---
title: Introduction to EBPF
date: 2025-06-20 12:00:00 +600
categories: [technical]
tags: [Kernel, EBPF]
---

What is EBPF?

eBPF is kernel tech that allows developer to write custom code to modify kernal behaviour safely.
eBPF programs can be added to or removed from the kernel at any time. Once a program is connected to an event, it will run every time that event happens, no matter what caused it.

BPF Maps
A map is a data structure that can accessed from an eBPF program and from user space

We are going to write a simple hello world ebpf program in Rust

we need to install following dependenciy

```bash
    cargo install bpf-linker
```