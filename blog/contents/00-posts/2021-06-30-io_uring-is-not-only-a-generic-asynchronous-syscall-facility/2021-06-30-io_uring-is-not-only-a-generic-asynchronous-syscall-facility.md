---
title: io_uring is not (only) a generic asynchronous syscall facility
authors: [fancl20]
date: 2021-06-30
---

TL;DR\: After reading [`io_uring` is not an event system](https://despairlabs.com/posts/2021-06-16-io-uring-is-not-an-event-system/), I think there is another way to consider why `io_uring` can adapt to every use case: `io_uring` is more than a generic asynchronous syscall facility. It's the state-of-the-art asynchronous interface for communication between subsystems implemented between the kernel and the userspace.

----

Starting from describing an abstract interface, a typical `io_uring` like interface contains these parts:

- Control Plane
    - Send control signal to the subsystem.
    - Usually synchronous (e.g. `io_uring_enter` is synchronous syscall because we wait for the control signal itself finished).
- Data Plane
    - Exchange data between subsystems.
    - Usually implemented by sharing cache or storage for reducing copying data.
    - Can be synchronous, although it must be asynchronous if it's an `io_uring` like interface.
- Interrupt
    - Send events in a reverse direction to the control flow.
    - Nice to have: We can poll the events if the interrupt is not available with some busy looping penalties.

These three components can describe not only the design of `io_uring` but also lots of other system designs, including Hardware DMA interface, RDMA interface, [netmap](https://dl.acm.org/doi/10.5555/2342821.2342830), [Snap](https://doi.org/10.1145/3341301.3359657). All these system architectures share the same view of the subsystem, that the standalone subsystem will run continuously regardless of the application's state. In contrast, the synchronous view will be that the "remote function call" is part of the application instruction flow.

The growing interest in `io_uring` means we are changing the view of syscall as a function call to that kernel is a standalone subsystem. That even makes more sense when [comes to using eBPF with `io_uring`](https://lwn.net/Articles/847951/). Hardware subsystems have their asynchronous nature and kernel is becoming one of them when more complex and customized computation happened in the kernel.

What's the future of `io_uring`? One possible future is that if we keep improving the performance of `io_uring`, adding fast user-level interrupt, it will become a userspace API mapping to hardware DMA. That means we can build all other syscalls in userspace on top of the DMA mapping.

## Updates

2021-10-03: Intel is adding [x86 User Interrupts support](https://lwn.net/ml/linux-kernel/20210913200132.3396598-1-sohil.mehta@intel.com/). Although the current implementation only support interrpts from userspace to userspace and doesn't provide a full control of scheduling.