---
layout: post
title: "Intel MIC初探（一）：MIC架构及编程模型概览"
date: 2014-08-23 19:58:46 +0800
comments: true
categories: Intel MIC
---
#楔子
Intel MIC（Many Integrated Core）架构是将多个核心整合在一起的处理器，面向HPC（High Performance Computing）领域，旨在引领行业进入百亿亿次计算时代，在其计算机体系中，并非欲取代CPU，而是作为协处理器存在的。MIC芯片通常有数十个精简的x86核心，提供高度并行的计算能力。虽然Intel官方声称原生的CPU程序无需进行大的改动即可在MIC芯片上运行，但是笔者在具体的应用移植过程中发现，真实的应用在MIC架构下通常都会在规模、内存以及第三方库移植等各方面受到一定的制约。

本系列文章将结合实际使用场景对Intel MIC架构及相关技术做简要介绍，并针对两个具有代表性的应用，提出一些启发式的应用移植方法。另外，本文所有的引用文献以及相关参考资料均在文后列出，以资参考。

#硬件架构
片上对称多处理器（Symmetric Multiprocessor on-a-chip，SMP）是对Intel MIC架构的准确描述方式，Intel Xeon Phi协处理器是基于Intel MIC架构的首款产品。协处理器卡的核心是Intel Xeon Phi协处理器芯片，由61个IA（Intel Architecuture）核组成，这些核执行IA指令集，每个核有4个完全相同的硬件线程。

{% img center /images/blog/IntelMICArchiCore.png 250 %}

关于MIC的缓存组织，每个核的二级缓存组织包括一级数据缓存和指令缓存，大小均为32KB，另外每个核还设计有一个私有的（本地）512KB二级缓存。耳机缓存保持完全的缓存一致性并为片上每个核提供数据。私有的512KB二级缓存总共构成了容量可达30.5MB的片上缓存。所有的二级缓存由全局分布的 (global-distributed) 标签目录保持完全一致.该内存控制器和 PCIe 客户端逻辑分别向协处理器上的 GDDR5 内存和 PCIe 总线提供一种直接接口。所有这些组件都由环形互连连接在一起。被多个核共享的数据将会被复制到需要使用该数据的核所对应的本地二级缓存中，所以，如果每个核都已完美的同步方式共享相同的代码和数据，则有效的二级缓存容量只有512KB。所以，二级缓存的实际可用容量大小与代码和数据在核与线程间的共享情况密切相关。

{% img center /images/blog/Microarchitecture.jpg %}

协处理器同时支持4KB（标准）、64KB（非标准）和2MB（超大，标准）页面大小的虚拟内存管理。较小页面的访问会带来总体内存映射空间变小以及最快的访问速度。4KB的页面大小长期以来是Linux的标准设置。64KB的页面大小在现有的微处理器并不常见，同时需要Linux核心的特殊支持。2MB大型存储页支持一般是可用的，但是应用程序或运行环境需要经过一些特殊的修改。在访问大数据集和数组时，使用超大页面能够通过提高TLB命中率提升应用性能，这点在应用优化时需要额外注意。

协处理器芯片的每一个核都包含有一个512位宽的SIMD向量处理单元（VPU），并设计了通信向量化指令集。VPU每个时钟周期可同时处理16个单精度（32bit）或8个双精度（64bit）浮点运算。另外，还包含一个扩展数学单元（EMU，Extended Math Unit）用来实现单精度的超越函数指令集，即通过硬件实现指数、对数、倒数和倒数平方根等常用数学运算。

Intel Xeon Phi协处理器主要针对高度并行化的负载优化，同时支持多种常用的编程语言、编程模式和编程工具。除了大量的处理核心，协处理器还包括提供了并行功能的向量处理器单元。此外，协处理器的PCIe接口、DMA、电源管理、传感器以及散热监控等均有各自的设计特点。

#软件架构

Intel Xeon Phi协处理器最显著的一个特点是可以启动和运行一个包含网络功能的Linux操作系统，这也是它被成为“协处理器”而非“加速器”（需要依靠主机系统来管理软件应用程序）的主要原因。在协处理器上创建并运行应用程序及系统服务的软件架构包括两个主要部分：

- 开发工具集和运行时环境: Intel C/C++ Fortran编译器，各种函数库。
- Intel MPSS(Intel Manycore Platform Software Stack)：中间件接口、通信和控制相关的设备驱动，协处理器管理工具以及协处理器操作系统。

{% img center /images/blog/IntelMICSoftwareArchi.png %}

MIC的µOS建立的基本执行环境，是其他软件栈的基础。MIC的µOS是基于标准的Linux内核源码。MIC基于Linux的µOS是最小化的，是嵌入式Linux环境通过Linux标准基础（LSB,Linux Standard Base）核心库一直到MIC架构的产物，这也是个未签名的操作系统。µOS提供一些典型的能力，如：进程/任务创建、时序安排、内存管理等。µOS也提供设置、电源和服务器的管理能力。

Intel MPSS提供驱动程序，将PCIe总线映射成一个网络栈中的以太网设备，虚拟化TCP/IP堆栈。系统可以配置桥接TCP/IP网络于所连接的其他网络通信。使用户可以将其作为网络节点直接使用ssh连接到协处理器上。

{% img center /images/blog/IntelMICSoftwareStack.png %}

SCIF（Symmetric Communications Interface）实现了协处理器与主处理器之间，协处理器与协处理器之间的通信。它提供了一个统一的对称的API让主处理器和协处理器通过系统中的PCIe总线进行通信，SCIF可以使协处理器通过DMA方式对大数据块进行高速传输，同时可以将主处理器或协处理器的内存空间映射到任意运行在主处理器和协处理器上的进程地址空间中。

MicAccessAPI是的一组C/C++ API，用于监测和配置Intel Xeon Phi协处理器上的各项参数，如电源管理、CPU使用情况、PCI链路情况等。这些API需要依赖scif库（libscif.so）来实现内核软件栈中的通信。

{% img center /images/blog/IntelMICAccessAPIArchi.png %}

#编程模型
MIC拥有较为灵活的编程方式，MIC卡可以作为一个协处理器存在，也可以被看做是一个独立的节点。host端与MIC端的关系可以组合成一下5种关系。

{% img center /images/blog/IntelMICAppMode.png %}

而实际中的的应用模式通常不会这样复杂，常用的模式分为Native模式和Offload模式。

Native模式所有负载均在MIC端，通常使用于高并行计算程序，程序直接在MIC执行，这种方式对于应用移植来说难度较小，客观限制较多，如内存，第三方库函数等，提升的性能效果也比较有限，在后续的文章中会对此有所分析。

Offload模式是程序主函数由host发起，对于高度并行的计算部分分载到MIC端，由协处理器完成计算后返回结果。这种方式的优点是可以有更为明显的性能提升效果，缺点是移植较为复杂，特别是对于一些数据结构、算法逻辑较为复杂的代码，甚至需要完全重写。

#Reference
[1] Jim Jeffers, James Reinders等. *Intel Xeon Phi协处理器高性能编程指南*[M]. 人民邮电出版社，2014.04

[2] 王恩东, 张清等. *MIC高性能计算编程指南*[M]. 中国水利水电出版社, 2012.11

[3] George Chrysos. *英特尔® 至强融核™ 协处理器（代号 Knights Corner）*[EB/OL]. https://software.intel.com/zh-cn/articles/intel-xeon-phi-coprocessor-codename-knights-corner, 2013.01.15

[4] Intel Developer Zone. *Intel Xeon Phi systems software developers guide*[EB/OL]. https://software.intel.com/en-us/articles/intel-xeon-phi-coprocessor-system-software-developers-guide, 2012.11.12

[5] Intel Developer Zone. *Intel Xeon Phi coprocessor quick start developers guide*[EB/OL]. https://software.intel.com/en-us/articles/intel-xeon-phi-coprocessor-developers-quick-start-guide, 2012.11.12

[6] Intel Developer Zone. *An overview of programming for Intel Xeon processors and Intel Xeon Phi coprocessors*[EB/OL]. https://software.intel.com/en-us/articles/an-overview-of-programming-for-intel-xeon-processors-and-intel-xeon-phi-coprocessors, 2012.11.12