---
layout: post
title: Hadoop Tuning Guide (1)
date: '2019-12-31 11:22'
subtitle: Translate Hadoop Performance Tuning Guide
author: Eric Yin
catalog: 'true'
tags:
  - Hadoop
---

> 翻译《Hadoop Performance Tuning Guide》-AMD

# **准备工作**

根据Hadoop工作负载的性质，某些OS参数和Hadoop配置参数的默认值可能会导致Map / Reduce任务或Hadoop作业失败，或者可能导致性能大幅下降。根据我们在TeraSort工作负载方面的经验，以下配置参数值得关注：

## OS参数

- 使用ulimit命令配置的默认最大打开File Descriptors（FD）数量有时会导致FD耗尽，具体取决于Hadoop工作负载的性质。 这会导致作业失败异常。 在此研究中，我们在测试群集上将此值设置为32768。
- 使用开箱即用的设置运行Hadoop作业时，获取失败可能会发生。 有时，这些失败可能是由于net.core.somaxconn Linux内核参数的值较低引起的。 此参数的默认值为128。如果需要，则需要将该值增加到足够大的值。 我们在测试集群上将其设置为1024。

## Hadoop参数

- 根据Hadoop工作负载的性质，Map / Reduce任务在一段时间内可能不会指示其进度超过mapred.task.timeout中的Hadoop配置属性设置 mapred-site.xml。 默认情况下，此属性设置为600秒。 如果需要，应根据工作负载要求增加此属性的值。但是请注意，如果由于硬件，集群设置或工作负载实施问题而确实存在任务挂起的情况，则将该值增加到异常大的值可能会将其从Hadoop框架中屏蔽掉。这又可能导致性能下降。
- 如果遇到java.net.SocketTimeoutException这样的异常，例如"timeout while waiting for channel to be ready for read/write"，则hdfs-site.xml中的dfs.socket.timeout和dfs.datanode.socket.write.timeout属性值需要增加到可接受的水平。同样，这里要权衡的是，与这些属性的值无关的真正的超时情况可能会在可接受的时间段内被忽略。


# **Hadoop配置调优**

一旦验证了Hadoop集群的硬件和软件组件能够以最佳的开箱即用的性能水平运行，就可以开始深入研究以精调配置旋钮。 Hadoop堆栈的所有级别（Hadoop框架，JVM和OS）上的配置参数都会极大地影响Hadoop工作负载的性能。在本节中，基于我们在调优TeraSort工作负载方面的经验，我们将提供一组准则，可以帮助调整和最大化Hadoop工作负载的性能。这些指南中的步骤可以按照介绍的顺序执行，以实现最大的性能优势。在适当情况下，我们还将提供经验数据，以显示这些准则中建议的性能影响。以下各节将介绍为Hadoop框架，JVM和OS调整配置参数的准则。请注意，本指南中的Hadoop调整建议适用于Hadoop（CDH4.0.1）。

在本节中，我们将讨论如何为Hadoop工作负载提出基准配置，然后如何继续调整该配置以实现最大程度的资源利用率和性能。

如前所述，我们将在此处使用TeraSort作为示例工作负载。 在没有压缩的情况下，TeraSort工作负载的Map阶段执行1TB的磁盘IO读取和1TB的磁盘IO写入操作，这不包括由于溢出和推测性执行而引起的IO操作。 Reduce阶段还执行了1TB的磁盘IO读取和1TB的磁盘IO写入操作，但不包括Reduce端溢出和推测性执行所引起的IO操作。 总体而言，TeraSort在执行时间上花费了大量时间来执行IO操作。 根据Hadoop工作负载的性质，以下各节中介绍的一些准则可能会或可能不会显示最佳结果。

## 基准配置

对于某些Hadoop工作负载，默认的Map / Reduce插槽数和Java堆大小选项可能不足。 因此，定义基准配置的第一步是将这些配置参数修改为可接受的水平。 此处的目标是最大程度地利用群集的CPU和其他硬件资源。 Hadoop工作负载的Java堆大小要求，工作负载产生的Map / Reduce任务数量，系统上可用的硬件核心/线程数量，可用IO带宽和可用内存量都将影响基线配置 。 从我们的经验中可以帮助我们的方法是：

1. 足够数量的硬盘驱动器可以满足基础硬盘的预期存储要求工作量
2. 配置Map和Reduce插槽的数量，以使Map阶段获得尽可能多的CPU资源，并且在Reduce阶段也要利用所有处理器核心
3. 配置Map和Reduce JVM进程的Java堆大小，以便在满足Map / Reduce任务的堆要求之后，为OS缓冲区和缓存留出足够的内存。

此处的相关Hadoop配置参数为mapred.map.tasks，mapred.tasktracker.map.tasks.maximum，mapred.reduce.tasks，mapred.tasktracker.reduce.tasks.maximum，mapred.map.child.java.opts和 mapred.reduce.child.java.opts，可以在mapred-site.xml文件中找到。 使用此方法，对于基线配置，我们决定：

1. 每个DataNode使用四个数据磁盘驱动
2. 每个硬件核心使用2个Map和1个Reduce
3. 为Map和Reduce JVM进程分配1GB的初始和最大Java堆大小

因此，在任何给定的时间，每个硬件核心产生3个JVM进程，每个硬件核心消耗多达3GB的堆。 系统上每个硬件核心都有4GB的RAM。 每个硬件内核剩余的1GB内存留给OS缓冲区和缓存。 使用此配置作为基准配置，我们开始关注调整机会。请注意，基线配置是在了解TeraSort工作负载的工作知识的基础上得出的。基线配置可能会根据基础Hadoop工作负载的特性而有所不同。

## 数据磁盘扩展

定义基准配置之后的下一步应该是了解工作负载随可用数据磁盘数量的扩展程度。 需要配置mapred-site.xml中的mapred.local.dir和hdfs-site.xml中的dfs.name.dir和dfs.data.dir来更改Hadoop框架使用的数据磁盘数。 鉴于TeraSort工作负载的IO繁重性质，我们看到随着磁盘数量的增加，性能可以很好地扩展。 图1包含一个图表，显示了不同数量的数据磁盘上TeraSort的归一化性能差异。

![](/img/hadoop_tuning_result1.png)

## 压缩

Hadoop支持3个不同级别的压缩-输入数据，中间Map输出数据和Reduce输出数据。它还支持可用于执行压缩和解压缩的多个编解码器。一些编解码器具有更好的压缩系数，但是压缩和解压缩都需要更长的时间。一些编解码器在压缩因子与压缩和解压缩活动的开销之间达到了很好的平衡。TeraSort工作负载不支持输入数据压缩，也不支持Reduce输出数据压缩。因此，我们可以尝试压缩中间Map输出。启用Map输出压缩可减少磁盘和网络IO开销，但以压缩和解压缩所需的CPU周期为代价。因此，应该评估在支持的不同级别Hadoop工作负载中，所有压缩/解压缩的权衡。这里应该感兴趣的属性是 – mapred.compress.map.output，mapred.map.output.compression.codec，mapred.output.compress，mapred.output.compression.type和mapred.output.compression.codec在mapred-site.xml中找到。仅在以下情况下进行权衡：CPU资源有限，并且底层工作负载非常占用CPU，在这种情况下，处理器不会因IO活动而停顿，在此期间，压缩/解压缩线程可以接收CPU上的周期。

![](/img/tuning2.png)

使用Snappy编解码器时，我们发现运行之间的差异约为28％。 图2中Snappy编解码器的比例是通过排除引起运行的方差得出的。由于我们观察到LZO编解码器的差异要小得多（1.5％），因此我们决定继续使用它作为性能最佳的编解码器。

## JVM重用策略

Hadoop支持一个名为mapred.job.reuse.jvm.num.tasks的配置参数，该参数控制是否将重复使用的Map/Reduce JVM进程重新用于运行多个任务。可以在mapred-site.xml中找到此属性，并且此参数的默认值为1，这意味着JVM不重新用于运行多个任务。将此值设置为-1表示可以在特定的JVM实例上调度无限数量的任务。启用JVM重用策略可减少JVM启动和拆卸的开销。由于JVM花费更少的时间来解释Java字节码，因此它还提高了性能，因为某些早期任务有望触发热方法的JIT编译。在运行大量非常短的任务的情况下，JVM重用效果非常明显。我们注意到通过启用JVM重用，性能提高了近2％。

## HDFS块大小

每个Map任务都在所谓的输入拆分上进行操作。 （mapred-site.xml的mapred.min.split.size），（hdrfs-site.xml的dfs.block.size）和（mapred-site.xml的mapred.max.split.size）配置参数决定输入split的大小。Hadoop工作负载的输入split大小和总输入数据大小决定了Hadoop框架产生的Map任务总数。 对于像TeraSort这样的工作负载，最自然的更改输入split大小的方法是使用dfs.block.size参数更改HDFS块大小值。 如果Hadoop作业正在产生，则使用更大的HDFS块大小尝试大量的Map任务。通过这种方式减少Map任务的数量可以减少启动和拆卸Map JVM的开销。它还可以减少在Reduce阶段合并Map输出段所涉及的成本。较大的块大小也有助于增加每个Map任务的执行时间。最好运行少量长时间运行的Map任务，而不是运行大量非常短的运行Map任务。请注意，如果Map输出大小与HDFS块大小成比例，那么如果未适当调整与溢出相关的属性，则更大的块大小可能会导致其他Map溢出。 图3显示了性能如何随不同的块大小而变化。 我们发现256M是我们配置的最佳大小。

![](/img/tuning3.png)

## Map端泄露

当Map任务运行时，生成的中间输出存储在缓冲区中。 该缓冲区是一部分保留内存，它是Map JVM堆空间的一部分。 该缓冲区的默认大小为100 MB。 这由io.sort.mb（mapred-site.xml的）配置参数的值控制。 该缓冲区的一部分保留用于存储溢出记录的元数据。 默认情况下，它设置为io.sort.mb的0.05（5％），因此等于5MB。 这由mapred-site.xml的io.sort.record.percent配置参数控制。 每个元数据记录的长度为16个字节。 因此，总共327680条记录的元数据可以存储在记录缓冲区中。 一旦缓冲区的输出数据部分或缓冲区的元数据部分达到一定的占用阈值，此缓冲区的内容就会溢出到磁盘上。 默认情况下，由mapred-site.xml的io.sort.spill.percent配置参数控制的阈值设置为0.8（80％）。

将Map输出多次溢出到磁盘上（在最终溢出之前）会导致读取和合并溢出记录的额外开销。 影响Map端溢出行为的属性默认值仅对于约304字节长的Map输入记录而言是最佳的。如果有足够的Java堆内存可用于Map JVM，则此处的目标应该是消除所有中间溢出，仅溢出最终输出。如果对可用堆有限制，则应尝试通过调整io.sort.record.percent参数值来最大程度地减少溢出次数。一种简单的检测Map阶段是否正在执行其他溢出的方法是在Map阶段完成后立即查看JobTracker Web界面的" Map Out records"和"Spilled Records"计数器。如果溢出的记录数大于Map输出记录，则将发生其他溢出。

可以完全利用Map输出缓冲区的一种方法是确定Map输出的总大小以及此输出中包含的记录总数，然后可以将其用于计算记录缓冲区所需的空间。然后，可以将io.sort.mb属性配置为容纳数据和记录缓冲区需求，并且可以将io.sort.spill.percent值设置为0.99，以使缓冲区几乎充满其容量，因为我们将在Map任务完成后，进行第一次和最后一次溢出。请注意，此缓冲区的大小将需要略微超过总缓冲区空间要求。当我们在Map输出数据和记录缓冲区要求上不允许额外的空间时，我们发现溢出。例如，对于256MB的HDFS块大小设置（每个记录为100字节长），我们将io.sort.mb设置为316MB，将io.sort.record.percent设置为0.162（316MB的16.2％）和io.sort.spill.percent到0.99（填充阈值的99％）以完全消除地图侧溢出。我们注意到，消除了Map-side溢出，执行时间缩短了2.64％

## COPY/SHUFFLE阶段调优

如果发现在执行所有Map任务后，Reduce阶段没有立即完成复制Map输出的操作，则表明集群配置不正确。复制阶段进展缓慢可能有多种原因：

- 默认情况下，由mapred-site.xml中的mapred.reduce.parallel.copies控制的并行Map输出copier线程的最大数量设置为5。这可能是复制操作吞吐量的限制因素。
- 用于将Map输出提供给Reducer的TaskTracker级别的工作线程最大数量由tasktracker.http.threads控制，默认情况下设置为40。这是TaskTracker级别的属性。可以尝试逐步增加此值以查看其对Copy阶段的影响。
- 配置参数，例如dfs.datanode.handler.count（位于hdfs-site.xml中），dfs.namenode.handler.count（位于hdfs-site.xml中）和mapred.job.tracker.handler.count（位于mapred-site.xml中），还可以探索以进行进一步的微调。
- Reduce瓶颈也可能减慢复制阶段的进度。旨在避免在Reduce端发生磁盘溢出的调整可能会加快复制阶段。在我们的案例中，结果确实是减少复制方面的瓶颈。
- 与网络硬件相关的瓶颈也可能是一个促成因素。使用Netperf之类的基准测试，可以确认所有辅助节点上的网络带宽都处于可接受的水平。

## Reduce泄露

Reduce阶段对于影响Hadoop工作负载的总执行时间至关重要。Reduce阶段是更密集的网络，并且可能是更密集的IO，这取决于Hadoop工作负载的性质。毕竟，由大量潜在的Map任务生成的所有输出都需要复制、聚合/合并、处理，而且可能所有这些数据都需要写回HDFS。因此，取决于分配给Hadoop作业的Reduce个数，Reduce jvm的Java堆需求可能比Map jvm的需求要高得多，特别是在考虑最佳性能的情况下。

一旦Map任务开始完成其工作，Map输出将按每个Reducer进行排序和分区，并写入TaskTracker节点的磁盘。然后将这些Map分区复制到相应的Reducer tasktracker。由mapred-site.xml的mapred.job.shuffle.input.buffer.percent配置参数控制的缓冲区，如果足够大，将保存此Map输出数据。否则，Map输出将溢出到磁盘。默认情况下，mapred.job.shuffle.input.buffer.percent设置为0.70。这意味着Reduce JVM堆空间的70%将用于存储复制Map的输出数据。当这个缓冲区 达到占用率的某个阈值（由mapredsite.xml的mapred.job.shuffle.merge.percent属性控制，默认值为0.66），累积的Map输出将合并并溢出到磁盘。一旦Reduce-side排序阶段结束了Reduce JVM堆的一部分，那么 mapred-site.xml的mapred.job.reduce.input.buffer.percent可用于在将Map输出输入到reduce阶段的最终reduce函数之前保留Map输出。mapred.job.reduce.input.buffer.percent参数默认设置为0.0，这意味着所有reduce JVM堆都被分配给最终reduce函数。

基本上，拥有大的mapred.job.shuffle.input.buffer.percent和mapred.job.reduce.input.buffer.percent缓冲区将有助于避免由reduce side溢出导致的不必要的IO操作。在标识Reducer（如TeraSort工作负载使用的那个）的情况下，reduce函数不需要很大的Java堆内存。在这种情况下，可以通过将mapred.job.reduce.input.buffer.percent值增加到尽可能接近1.0以获得性能改进。

通过将 mapred.job.shuffle.input.buffer.percent参数增加到0.9和 mapred.job.reduce.input.buffer.percent设置为0.8。很明显，TeraSort的Reduce阶段可以在额外的堆上茁壮成长。在这一点上，我们决定在每个硬件核心有2个Map与每个硬件核心只有1个Map和将所有释放的Map Java堆空间分配给Reduce jvm之间进行权衡。Reduce端的堆空间扩大显著提高了性能。

我们注意到Reduce阶段优化带来约10%的改善。图4以图表的形式显示了这些数据。

![](/img/tuning4.png)

## 潜在限制

最后，在所使用的Hadoop环境中可能存在某些已知的限制或错误，这些限制或错误可能会导致性能或稳定性问题。例如，CDH 4.0.1和早期版本受到MR框架中的错误影响，在MR框架中，Map/Reduce任务的一小部分可能会发生故障。这些失败会导致执行时间出现较大的回归和方差。例如，如果长时间运行的Reduce任务由于此错误而失败，并且没有其他可用的Reduce，则Hadoop框架不会重新调度失败的任务，直到有可用的Reduce。此外，mapred-site.xml的mapred.max.tracker.failures配置参数在TaskTracker被列入黑名单时进行控制，默认设置为4。这意味着，如果任何一个tasktracker有4个或更多的任务失败，那么这些tasktracker将被列入黑名单，并且不会对它们进行进一步的任务调度。要绕过这些回归，可以将mapred.max.tracker.failures属性的值设置为足够大的值，或者使用修复此问题的Hadoop发行版。需要注意确保 mapred.max.tracker.failures属性会影响集群的稳定性。在我们的设置中，在包含了这个问题的修复之后，我们从源代码构建了Hadoop核心jar文件。注意，最近发布的CDH 4.1确实包含了这个bug的修复。

因此，重要的是确定是否有任何已知问题可能会损害性能或集群的稳定性。
