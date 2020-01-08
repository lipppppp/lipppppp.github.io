---
layout: post
title: Hadoop Tuning Guide (2)
date: '2020-01-06 10:49'
subtitle: Translate Hadoop Performance Tuning Guide
author: Eric Yin
catalog: 'true'
tags:
  - Hadoop
---

> 翻译《Hadoop Performance Tuning Guide》-AMD

# **JVM配置调优**

一旦解决了堆栈Hadoop端的所有可能的瓶颈，或者至少在很大程度上减轻了瓶颈，就值得进一步优化JVM。

# JVM FLAGS

JVM供应商努力提高JVM的开箱即用性能，这样用户就不必为他们的应用程序执行Flag挖掘实验。但是，仍然有可能通过JVM Flag优化来提高性能，而无需在本练习中花费大量精力。以下是一些值得探索的Flag：

- AggressiveOpts----这是一个整体标志，用于控制JVM是否启用了某些其他优化。这些标志启用的优化可能因JVM版本而异，因此使用此标志进行测试非常重要，特别是当系统升级到较新的JVM时。打开我们的集群通过启用AggressiveOpts功能，我们看到了大约3.4%的改进。请注意，在Oracle JDK7更新5上默认禁用AggressiveOpts。
- UseCompressedOops----压缩的普通对象指针或简单的压缩指针有助于减少64位jvm的内存占用，同时减少可寻址Java堆的总空间。此功能在较新的Oracle JVM版本中默认启用。如果使用的是旧版本的JVM，并且禁用了压缩指针，请尝试启用它们。通过禁用此功能，我们看到TeraSort性能下降不到1%。注意，UseCompressedOops在oraclejdk7update5上默认启用。
- Oracle HotSpot JDK中的UseBiasedLocking-Piciated锁定功能提高了在通常不争用锁的情况下的性能。通过禁用此功能，我们看到TeraSort性能下降不到1%。注意，UseBiasedLocking在oraclejdk7update5上默认启用。

# JVM GC

对Map和Reduce JVM的垃圾收集分析和调优不如尝试使用不同的JVM标志那么简单。但是，如果您对GC分析和调优相当熟悉，那么通过调优GC行为获得的收益可以抵消此练习中涉及的时间和精力。在一个不同于本调整指南中用于实验的测试平台上，我们观察到通过调整Map和Reduce jvm的GC行为在性能上有了适度的改进。鼓励对GC优化的其他细节感兴趣的读者阅读本文。

![](/img/tuning5.png)

# **OS配置调优**

在本节中，我们将讨论对Hadoop工作负载的性能有积极影响的OS配置。

## TRANSPARENT HUGE TUNING

RHEL6.2及以上版本的透明大页面（THP）功能旨在简化大页面的管理。此功能已显示出可提高广泛应用程序的开箱即用性能。但是，在启用THP特性的情况下运行Hadoop工作负载时，由于THP特性的压缩过程，显示出较高的内核空间CPU利用率。这会导致大的倒退。在启用THP的情况下运行TeraSort工作负载时，我们注意到多达66%的回归。如果在运行Hadoop工作负载时观察到类似的症状，则需要禁用THP功能。有关更多详细信息，请参见CDH4.0.1发行说明[19]的"CDH4中的已知问题和解决方法"一节。这里的一个折衷方案是，如果同一集群上的其他生产应用程序从THP中受益，那么它们的性能将受到影响。

## 文件系统选择和属性

Linux支持的默认文件系统（FS）可能因发行版和发行版的版本而异。考虑到对FS的内部进行了持续的分析和改进，FS的选择会对Hadoop工作负载的性能产生重大影响，特别是对于IO密集型工作负载。RHEL6.3支持EXT4作为默认的FS，根据我们过去的经验，我们发现与ext3fs相比，EXT4的性能明显更好。

在没有noatime属性的情况下，每个文件读取操作都会触发一个磁盘写入操作，以保持文件的最后访问时间。通过向数据磁盘的FS mount选项添加noatime（这也意味着nodiratime）属性，可以禁用此元数据日志记录。我们注意到，通过使用noatime FS mount属性，TeraSort的性能提高了29%。

## IO调度程序选择

大多数Linux发行版支持4种不同类型的IO调度程序：CFQ、deadline、noop和expectory。根据IO操作的模式、IO子系统的硬件配置和应用程序的要求，特定IO调度器可能比其他IO调度器执行得更好。默认的IO调度程序可能因Linux发行版而异。例如，Ubuntu11.04将deadline IO调度程序作为默认调度程序，而RHEL6.3将CFQ作为默认调度程序。正如我们在 <http://dl.acm.org/citation.cfm?id=2188323> 上的前一组实验所观察到的那样我们注意到使用CFQ调度程序而不是deadline调度程序提高了15%。

![](/img/tuning6.png)
