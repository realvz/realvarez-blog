---
title: How is eBPF efficient for observability
description: A brief look into how eBPF works
date: 2022-07-09
layout: layouts/post.njk
---

> Everybody says eBPF is fast, but how?

The Berkeley Packet Filter (BPF or eBPF) is a virtual machine based on registers, initially designed for filtering network packets, best known for its use in `tcpdump`. In this post, we learn what is exactly that makes BPF so appealing for a number of use cases.

## Introduction

BPF programs are small programs that run in the Linux kernel when events occur. You can think of BPF programs as event driven functions (like AWS Lambda).  BPF has access to a subset of kernel functions and memory. It is designed to be safe, that is poor BPF code won’t crash your kernel.

In the containers world, BPF is increasingly becoming relevant. Its most popular use case is Cilium, which uses eBPF to provide a [“sidecarless” service mesh](https://cilium.io/blog/2021/12/01/cilium-service-mesh-beta) and [Pixie](https://github.com/pixie-io/pixie), which uses eBPF to collect telemetry data.

Other popular BPF uses are:

- **Debugging and Tracing -** trace any `syscall` or kernel function or any user space program.
   - [bpftrace](https://github.com/iovisor/bpftrace) allows users to trace from the Linux command line.
- **Networking -** Inspect, filter, manipulate traffic.
   - User space program can attach a filter to any socket and inspect the traffic flowing through it. They can also allow, disallow, redirect packets.
- **Security monitoring and sandboxing**
   - BPF programs can detect and report the `syscalls` occurring on a system.
   - BPF programs can prevent applications from performing certain `syscalls` on a system (e.g., prevent deleting a file).

Linux has allowed us to perform above functions for decades, but BPF helps us perform these tasks more efficiently. BPF programs use fewer CPU and memory resources than traditional solutions.

## How is BPF faster?

BPF programs are faster because BPF code runs in kernel space.

Consider the steps a program has to take to calculate the number of bytes sent out on a Linux system. First, the kernel generates raw data as network activity occurs. This raw data packs a ton of information, most of which is irrelevant for calculating *bytes out* metrics. So, whatever is generating aggregated metrics will have to  filter the relevant data points repeatedly and run mathematical calculations on them. This process is repeated hundreds of times (or more) every minute.

Since traditional monitoring programs run in user space, all that raw data that the kernel generates has to be copied from kernel space into user space. This data copy and filtering operation can be very taxing on the CPU. This is why ptrace is slow (whereas [`bpftrace`](https://github.com/iovisor/bpftrace) is not).

eBPF avoids this copying of data from kernel space to user space. You can run a program in kernel space to aggregate the data you need and send just the output to user space.

Before BPF, large amounts of raw data from the kernel had to be copied over to user space for analysis. BPF allows creating histograms and filtering data within the kernel, which is much faster than exchanging massive amounts of data between user and kernel space.

## BPF Maps

BPF uses **BPF maps** to allow bidirectional data exchange between user and kernel space. In Linux, maps are a generic storage type for sharing data between user and kernel space. They are key value stores that reside in the kernel.

For metrics generation, BPF programs run calculations in kernel space and write results to BPF maps that a userspace  application can read (and also write to).

---

Now you know why eBPF is efficient. It’s because BPF provides a way to run programs in kernel space and avoid copying irrelevant data between kernel and userspace.

## References

[How eBPF Streamlines the Service Mesh](https://thenewstack.io/how-ebpf-streamlines-the-service-mesh)

[eBPF, sidecars, and the future of the service mesh](https://buoyant.io/2022/06/07/ebpf-sidecars-and-the-future-of-the-service-mesh/)

[TIB AV-Portal](https://av.tib.eu/media/44349)

[Linux Observability with BPF: Advanced Programming for Performance Analysis and Networking](https://www.amazon.com/Linux-Observability-BPF-Programming-Performance/dp/1492050202)

---

## Appendix

Containers use Linux Namespaces, a feature of the Linux kernel that partitions kernel resources such that one set of processes sees one set of resources while another set of processes sees a different set of resources.

Under the hood, containers use `iptables` to network. Linux developers didn’t intend to run thousands of `iptables` rules on every node. To overcome the performance limitations in `iptables`, kernel devs are replacing [`iptables` with BPF](https://cilium.io/blog/2018/04/17/why-is-the-kernel-community-replacing-iptables).

## BPF Verifier

BPF allows anyone to run code in kernel space without being a kernel dev, compiling your own kernel, or writing custom kernel extensions that can hugely reduce the stability of the kernel.

BPF programs are safe. The **BPF Verifier** ensures code is safe to run in kernel.   ← **Explain this**

Linux kernel includes a JIT compiler for BPF instructions. Once verification completes, JIT transforms bytecode to machine code.

## BPF programs

Written in a subset of C. LLVM compiles BPF programs into bytecode.

**BPF tracepoints** are static marks in the kernel’s codebase that allow you to inject any code for tracing and debugging.

## eBPF and mTLS

eBPF cannot do mTLS because it operates in kernel space entirely.

Proxy mTLS is handled by the sidecar, which runs in the user space.

## tcpdump and Seccomp

Generate BPF bytecode.

tcpdump is frontend to `libpcap`. It reads packets from the NIC and writes to a file. PCAP filter on Linux are compiled to BPF programs. When you use `tcpdump` , you compile and load a BPF program to filter packets.

## kubectl trace

[**kubectl-trace**](craftdocs://open?blockId=245EFFA2-BA97-4186-932A-F96888BF9F32&spaceId=9d54cc03-adfe-f72f-3389-565eb7356d1d)
