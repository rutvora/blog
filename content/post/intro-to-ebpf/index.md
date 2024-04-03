---
title: Dynamically Extending a Linux Kernel - eBPF
description: eBPF allows someone to dynamically extend certain components of the Linux kernel, without recompiling the entire kernel. It also ensures the safety of the code, unlike kernel modules, which have unfettered access to kernel code!
date: 2022-02-02 22:14:00+0530
image: header.png
categories:
    - NextGen Tech
    - eBPF
tags:
    - nextgen
    - ebpf

math: true
toc: true
---

A simple Google search of "eBPF" will tell you that it stands for "extended Berkeley Packet Filter". The words "packet filter" in the name makes me think this has something to do with Computer Networks. But, while it started as a packet filter system, it has extended much beyond that functionality since then. Facebook also had Whatsapp, Instagram, Oculus etc., under its belt, and hence the board decided to rename the company to Meta to symbolise this. In the same way, maybe eBPF should be renamed to represent what it is!

So, what exactly is eBPF? To understand eBPF, let us first understand why it is required.

A short video explaining eBPF:
{{< youtube zxhpmVgTQu0 >}}

## Pre-requisites

In this journey, the first thing to learn is the difference between userspace and kernel space. Each application is given a virtual memory address where it resides. And a hardware chip stores the mapping of the actual physical memory address (in the RAM/cache) to that virtual location and serves it as required. On the other hand, the kernel has full access to the entire Physical Memory without any virtual addressing. In this post, we will not get into the details of physical vs virtual addressing.

What if you wanted to apply an OS-wide filter for network packets (blocking or accepting requests from a particular source) or log the [system calls](https://en.wikipedia.org/wiki/System_call) that an application makes. One way to do this is to develop a regular application using your favourite programming language. But that program would run in userspace, which has its own caveats. 

For example, each network packet you want to filter will still be de-packetised until L4 (Transport layer) in the kernel and then served to userspace. Only then can you filter it. Also, there is no direct way to capture the syscalls of an application without explicitly running the program in association with other tracing programs like 'strace'. Needless to say, all of these consume comparatively more processing power and memory than programs just running in kernel space. And it would be entirely impossible to 'see' a program if it's run and destroyed in a tiny amount of time, as userspace applications would poll the system with programs like 'ps' to see which programs are running at any given time.

Running programs in the kernel is much more helpful for the aforementioned use-cases. For example, you can attach a program to run every time a particular system call is made and then log which process called it. You can also block or re-route a network packet directly from the kernel by attaching a program to run whenever a network packet is received. The possibilities are endless!

## Introduction

This brings us to what eBPF is. It's a kernel extension that allows users to run programs inside the kernel by attaching their execution to a particular kernel event. eBPF works by having a extremely tiny Virtual Machine inside the Linux Kernel. Don't worry; this Virtual Machine is not at all RAM hoggy. This VM can be related more to the Java Virtual Machine than the VirtualBox/VMWare/KVM systems. In Java, your code is compiled to an architecture-agnostic bytecode, and this (mostly) has a one-one correspondence to most architecture-specific bytecodes. Similarly, the BPF VM is a target machine for compiling programs to. [LLVM](https://llvm.org/) has BPF VM as a target.

But there are obvious caveats to running a program in the kernel. The first one that comes to mind is to run an infinite loop every time a system call is made. The Linux kernel's scheduler determines which program to run next on a particular CPU core for any userspace program. And this is designed in such a way that, after a specific quantum of time/some event, the scheduler program runs, pre-empting (removing) any existing program from the core and executing itself. But, this is possible as the other programs are in ring 3, while the scheduler executes in ring 0 and has access to additional CPU features. Hence, if I run an infinite loop in a program inside the kernel, the kernel will never yield the processor, and your computer would be stuck. This is devastating.

Hence, eBPF is programmed with a subset of C to avoid these caveats. C is a [Turing Complete](https://en.wikipedia.org/wiki/Turing_completeness) language. A Turning Complete language, in the simplest terms, is a language that can do what any other Turing Complete language can do. i.e. you can do with C, whatever you can with Java, Python, Go, Rust, JS etc. The subset of C that eBPF permits is not Turing Complete. It does not allow features like loops. It also has a hard limit on the number of branches the code can have. Every time you load an eBPF program in the kernel, a verifier first verifies whether the program will always terminate.

The eBPF functionality was first introduced in the Linux kernel v3.18. Since then, eBPF has evolved quite a lot. But one of the most notable changes to eBPF was in Linux kernel v5.2. Before v5.2, eBPF programs had a maximum limit of 4096 instructions per program, which was confusing. In addition, the total number of instructions across all branches were limited to 232, beyond which the verifier would give up and not allow your code to execute. v5.2 got rid of the 4096 instruction limit and extended the verifier limit to 1M instructions, allowing developers to write more involved programs.

## Hello World!

You can use the interactive example below, or continue reading on how to set this up on your own machine!

<iframe src="https://killercoda.com/rutvora/course/eBPF/hello_world_python" width="100%" height="790px" frameBorder="0" style="border: 0;"></iframe><br>Brought to you by <a href="" target="_blank">KillerCoda Development Environment</a>

Let us walk through a Hello World program with eBPF. As eBPF is a Linux functionality, the first thing you need is a Linux Machine. Don't worry if you are on Windows/Mac, though. You can use Multipass (WSL does not support this!). Windows Home users need VirtualBox alongside Multipass, but Mac users do not require it. After the installation in either OS, use the following code in "cmd" or "Terminal".

```bash
multipass launch --name eBPF
multipass exec eBPF -- bash
```

The first line launches an instance of the latest Ubuntu LTS with 1 core and 1GB RAM. Note that this will take some time as your device will download the Ubuntu image and set up the VM. The second line logs you into the running instance. After logging into the instance, run the following commands:

```bash
# Update repositories and packages
sudo apt update && sudo apt upgrade -y

# Install BPF Compiler Collection and other required stuff
sudo apt install bpfcc-tools libbpfcc-dev linux-headers-$(uname -r) python3-bpfcc

# Fetch the hello world code from pastebin (also given below)
wget -O hello_ebpf.py https://pastebin.com/raw/Jj7xB7ZU

# Run the program
sudo python3 hello_ebpf.py
```

The program is set to run when any application calls the 'getdents64' system call, which is the call to access the list of files/folders in a particular folder. It is the underlying system call for 'ls', a command similar to 'dir' in Windows. To test this, in another terminal/cmd, run:

```bash
multipass exec eBPF -- bash
```

You will notice that the first terminal/cmd now has a list of programs that tried accessing different files from different folders, just to log you in! You can now type 'ls' in the 2nd terminal to check whether a new entry shows up in the first terminal. Spoiler: It will!

## Code Walkthrough

```python
from bcc import BPF

BPF_PROGRAM = r"""
int hello(void *ctx) {
  bpf_trace_printk("getdents64 detected: \n");
  return 0;
}
"""

bpf = BPF(text=BPF_PROGRAM)
bpf.attach_kprobe(event=bpf.get_syscall_fnname("getdents64"), fn_name="hello")

while True:
    try:
        (task, pid, cpu, flags, ts, msg) = bpf.trace_fields()
        print(msg.decode('utf8'), task.decode('utf8'), pid, cpu, flags.decode('utf8'), ts)
    except ValueError:
        print('ValueError')
        continue
    except KeyboardInterrupt:
        break
```

1. The first line imports BPF into python. This is a wrapper around the `bpf()` system calls.
2. BPF_PROGRAM in Line#3 is a multiline string, which is the actual eBPF code. 
3. Line #5 has the function [`bpf_trace_printk`](https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md#:~:text=Output-,1.%20bpf_trace_printk(),-Syntax%3A%20int%20bpf_trace_printk), which prints to a specific well-known pipe*, but has limitations. The message is padded with extra informaion like the process name that triggered it and the process's id.
4. Line #10 generates and gives the context of the eBPF program to python.
5. Line #11 instructs the kernel to attach that eBPF program to a kernel event (a call to 'getdents64' in this case)
6. Line #13 is a loop in the userspace Python program and not the eBPF program, and hence is allowed.
7. Line #15 assigns the result of [`trace_fields()`](https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md#:~:text=2.-,trace_fields(),-Syntax%3A%20BPF.trace_fields) to a tuple. 

## Further Reading
1. [BCC Reference Guide](https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md)
2. [eBPF Maps](https://prototype-kernel.readthedocs.io/en/latest/bpf/ebpf_maps.html)
