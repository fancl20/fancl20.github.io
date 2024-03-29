---
title: Toward decoupled system architecture
authors: [fancl20]
date: 2021-06-27
---

## At the end of Moore's law
Moore's law was a prediction made in 1965 by Gordon Moore, which states that computer power doubles every two years at the same cost. The law has been predicted [[1]](#1) to be ended at some point - around 2020. So we are literally at the end of Moore's law as nobody is going to change their prediction.

![Faith no Moore](faith-no-moore.svg)

If we only consider single-core performance, modern CPUs have already failed to keep up with Moore's law since 2005, when multi-core processor start becoming the majority. It's too hard to improve fabrication - Intel is stuck in 14nm from 2014 until now (2021).

At the same time, I/O bandwidth is being doubled every 2 years from PCI-e 4.0 to PCI-e 6.0. Although it doesn't mean disk bandwidth can be doubled, network bandwidth increased dramatically because less constrained by the physical law compare to the disk [[2]](#2). CPU performance improvements failing to match the I/O bandwidth means I/O drains a much larger portion of general computational resources [[3]](#3).

In short:

- CPU performance is reaching its physical ceiling.
- I/O performance (both bandwidth and latency) is much better than we thought it should be.

We need an architecture to make it possible to process the data.

## Computation happened closer to the hardware
We are seeing a rising trend that computation is moving closer to the I/O devices, which means, bypassing every abstraction provided. Consider a network card, generally, we have three levels of abstractions:

- Application: This is the place where usually all computation happened. Userspace application talks to I/O devices using syscall.
- Kernel: Kernel provides a set of syscall to manipulate hardware. It's also responsible for handling events or interruptions from hardware and notify or schedule the userspace application in some way.
- Hardware: The physical I/O device.

Traditionally, all data computation should happen in the application, then sending or receiving the data using syscall. But syscall becomes more and more expensive due to the less capable CPU compare to I/O device. So we have different ways to bypass or mitigate the cost of abstraction:

- DPDK/SPDK are some of the commonly used technologies to provide a userspace driver for special hardware, which widely used for high-performance I/O devices [[4]](#4) [[5]](#5). It requires taking over the devices and reimplements the drivers in userspace.
- SR-IOV is a hardware virtualization solution that allows partitioning of physical PCI functions into independent virtual PCI functions [[6]](#6). It usually used with unikernel or libos to run a user program in kernel space, by which the user program becomes part of the kernel/libos [[7]](#7).
- eBPF or BPF is an in-kernel virtual machine that provides the ability to run a limited, verified program in kernel space. It is even described as "A New Type of Software [[8]](#8)". XDP, a Programmable and high performance networking data path, has been built on top of the eBPF [[9]](#9). eBPF is still quite limited and has some performance overhead (compare to DPDK/SPDK), but the adoption is generally much painless compare to DPDK/SPDK or unikernel.

![System Architechture](traditional-architechture.svg)

All of these methods move the computation closer to the hardware: DPDK/SPDK move the driver into userspace and bypass kernel; libos/unikernel move Application into the (maybe virtualized) kernel; eBPF split the application into multiple parts, userspace application is used as the control plane of eBPF programs. But can we go further? Why computation must be separated from the I/O device?

> For many years, the highest energy cost in processing has been data movement rather than computation, and energy is the limiting factor in processor design. As the data needed for a single application grows to exabytes, there is an opportunity to design a bandwidth-optimized architecture for big data computation by specialising hardware for data movement. [[10]](#10)

Moving computation to the place where I/O happened removes all of the overhead from every abstraction layer. We see a rising interest in Data Processing Unit or DPU, specialised hardware for high-speed data processing on I/O devices, letting CPU offload a large portion of computation.

## Toward decoupled system architecture

The system architecture is becoming decoupled as computation distributed to different parts of systems instead of a centralized application. Although the integrated design may improve reliability a bit by reducing communication errors, we have to handle CPU errors in the program when corrupt execution errors can't be simply considered almost impossible [[11]](#11). Based on the End-to-End Argument [[12]](#12), handling hardware errors in a program is inevitable when the system is complex enough. Splitting system into multiple parts and communicate explicitly can improve fault isolation, which can result in a more reliable system.

All these imply that we can't improve the system by adding complexities to the existing components. To adapt to the hardware trends, we are seeing more microkernel-like system design, moving features from kernel to userspace [[13]](#13) or splitting kernel into isolated parts as "eBPF is turning the Linux kernel into a microkernel." [[14]](#14). The motivation is we can't ship a kernel which extremely optimised for specialised needs while keeping generalised and reliable.

The kernel will be more like a control plane of all the components. We can have dedicated processes for managing hardware or specialised computation similar to microkernel. RDMA-like API or io\_uring can largely reduce the overhead of IPC, and more computation will be done on the hardware without CPU involved.

Having said that, there probably won’t be any paradigm shift in the operating system. All these changes will likely happen by gradually patching on the existent Linux kernel. But designing an “after Moore” system architecture from the ground up must be interesting work.

## References

<a id="1">[1]</a> 2016. After Moore’s Law. The Economist: Technology Quarterly. Retrieved from https://www.economist.com/technology-quarterly/2016-03-12/after-moores-law

<a id="2">[2]</a> Shelby Thomas, Geoffrey M. Voelker, and George Porter. 2018. Cachecloud: towards speed-of-light datacenter communication. In _Proceedings of the 10th USENIX Conference on Hot Topics in Cloud Computing_ (HotCloud’18), USENIX Association, Boston, MA, USA, 9.

<a id="3">[3]</a> Pekka Enberg, Ashwin Rao, and Sasu Tarkoma. 2019. I/O Is Faster Than the CPU: Let’s Partition Resources and Eliminate (Most) OS Abstractions. In _Proceedings of the Workshop on Hot Topics in Operating Systems_ (HotOS ’19), Association for Computing Machinery, Bertinoro, Italy, 81–87. DOI:https://doi.org/10.1145/3317550.3321426

<a id="4">[4]</a> Kalia, Michael Kaminsky, and David G. Andersen. 2019. Datacenter RPCs can be general and fast. In _Proceedings of the 16th USENIX Conference on Networked Systems Design and Implementation_ (NSDI’19), USENIX Association, Boston, MA, USA, 1–16.

<a id="5">[5]</a> Tian Zhang, Dong Xie, Feifei Li, and Ryan Stutsman. 2019. Narrowing the Gap Between Serverless and its State with Storage Functions. In _Proceedings of the ACM Symposium on Cloud Computing_ (SoCC ’19), Association for Computing Machinery, Santa Cruz, CA, USA, 1–12. DOI:https://doi.org/10.1145/3357223.3362723

<a id="6">[6]</a> Patrick Kutch. 2011. PCI-SIG SR-IOV Primer: An Introduction to SR-IOV Technology.

<a id="7">[7]</a> Simon Peter, Jialin Li, Irene Zhang, Dan R. K. Ports, Doug Woos, Arvind Krishnamurthy, Thomas Anderson, and Timothy Roscoe. 2015. Arrakis: The Operating System Is the Control Plane. _ACM Trans. Comput. Syst._ 33, 4 (November 2015), 11:1-11:30. DOI:https://doi.org/10.1145/2812806

<a id="8">[8]</a> Brendan Gregg. 2019. BPF: A New Type of Software. Retrieved from http://www.brendangregg.com/blog/2019-12-02/bpf-a-new-type-of-software.html

<a id="9">[9]</a> Toke Høiland-Jørgensen, Jesper Dangaard Brouer, Daniel Borkmann, John Fastabend, Tom Herbert, David Ahern, and David Miller. 2018. The eXpress data path: fast programmable packet processing in the operating system kernel. In _Proceedings of the 14th International Conference on emerging Networking EXperiments and Technologies_ (CoNEXT ’18), Association for Computing Machinery, Heraklion, Greece, 54–66. DOI:https://doi.org/10.1145/3281411.3281443

<a id="10">[10]</a> Sandeep R Agrawal, Sam Idicula, Arun Raghavan, Evangelos Vlachos, Venkatraman Govindaraju, Venkatanathan Varadarajan, Cagri Balkesen, Georgios Giannikis, Charlie Roth, Nipun Agarwal, and Eric Sedlar. 2017. A many-core architecture for in-memory data processing. In _Proceedings of the 50th Annual IEEE/ACM International Symposium on Microarchitecture_ (MICRO-50 ’17), Association for Computing Machinery, Cambridge, Massachusetts, 245–258. DOI:https://doi.org/10.1145/3123939.3123985

<a id="13">[11]</a> Peter H. Hochschild, Paul Turner, Jeffrey C. Mogul, Rama Govindaraju, Parthasarathy Ranganathan, David E. Culler, and Amin Vahdat. 2021. Cores that don’t count. In Proceedings of the Workshop on Hot Topics in Operating Systems, ACM, Ann Arbor Michigan, 9–16. DOI:https://doi.org/10.1145/3458336.3465297

<a id="14">[12]</a> J. H. Saltzer, D. P. Reed, and D. D. Clark. 1984. End-to-end arguments in system design. ACM Trans. Comput. Syst. 2, 4 (November 1984), 277–288. DOI:https://doi.org/10.1145/357401.357402

<a id="11">[13]</a> Michael Marty, Marc de Kruijf, Jacob Adriaens, Christopher Alfeld, Sean Bauer, Carlo Contavalli, Michael Dalton, Nandita Dukkipati, William C. Evans, Steve Gribble, Nicholas Kidd, Roman Kononov, Gautam Kumar, Carl Mauer, Emily Musick, Lena Olson, Erik Rubow, Michael Ryan, Kevin Springborn, Paul Turner, Valas Valancius, Xi Wang, and Amin Vahdat. 2019. Snap: a microkernel approach to host networking. In _Proceedings of the 27th ACM Symposium on Operating Systems Principles_, ACM, Huntsville Ontario Canada, 399–413. DOI:https://doi.org/10.1145/3341301.3359657

<a id="12">[14]</a> Thomas Graf. 2020. eBPF - Rethinking the Linux Kernel. Retrieved from https://www.infoq.com/presentations/facebook-google-bpf-linux-kernel/