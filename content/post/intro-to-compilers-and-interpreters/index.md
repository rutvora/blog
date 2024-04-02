---
title: Introduction to Compilers and Interpreters
description: Why is C so performant and Python so slow? Find Out!
date: 2021-10-13 05:22:00+0530
image: header.jpeg
categories:
    - Hello World
    - Overview
tags:
    - hello-world
    - overview

math: true
toc: true
---

For humans, it is easier to understand languages written from a script we are familiar with, like the Latin A-Z alphabets or the Devanagari script of northern India or others.

But for a computer, it is much easier to understand binary. That is 1s and 0s.

This is so because the computer essentially works with electronic circuits, and all you can do with them (easily, at least) is switch them on and off, hence the idea of 1 and 0. You might say that you can also vary voltage or current, but that is not something we can do accurately, and hence it is difficult to introduce them into computing. You don't want the result of 1+1 to be 2.0000001, do you?

Anyway, there are 2 types of programming languages in the world:

1. Compiled (e.g. C, C++, Java, Go, Rust)
2. Interpreted (e.g. Python, JS)

The code written in any compiled language has to compile, producing an executable that you can run. While in interpreted languages, an interpreter like the Python Interpreter (the prompt that comes up when you run Python or Python3 command on the terminal) compiles and runs each line of code every time the line has to be executed.

This interpreter hence introduces inefficiency. Here's an example for easier understanding. Imagine the pseudo-code below written in one compiled and one interpreted language of your choice.

```python
for i in range(1, 100):
    # Do Something Complex*
```
\* Complex instructions like matrix multiplication or matrix inverse etc.



What happens when you compile this code?

Short Answer: This code gets converted to binary and gets stored in a file. 

This means that `<some-complex-instructions>` are already converted to binary and ready to be executed. Hence, every time this loop runs, the computer runs the already compiled binary instruction/s.

On the contrary, in interpreted languages, `<some-complex-instructions>` have to be compiled every time that particular line executes. This is a significant overhead.

You might say that it can be optimised by caching (remembering) a compilation once it's compiled. But that is not always possible. And even if it is possible, there is a significant overhead of compiling it for the first time while running.

The most obvious next question is, "Why to use interpreted languages at all?"

The answer is that compilation means your executable is platform dependent. While the file that you execute in an interpreted language is not itself platform dependent and only requires the interpreter to be available for that particular platform (more on this later). Apart from that, they provide features like dynamic typing and ease of debugging.

You can 'pause' an interpreted code at any line you wish and check all the variables. While this is possible with the advent of gdb (for C, C++) and similar tools, it is still not a native language benefit and is usually messier for anyone to handle.

Many coders enjoy the fact that they don't have to bother about data types in Interpreted languages. Java kind of tries to take the best of both worlds. Java has a virtual machine called JVM (Java Virtual Machine). Think of it as a new Instruction Set Architecture or ISA (more on this later).

For this context, just think of ISA as the basic instructions a computer can understand. All of your functions have to be converted as repetition and sequence of the already available instructions. For example, for a computer with just an ADD function in the ISA, you have to multiply by adding n times because the computer only understands ADD.

JVM provides a new ISA that has an (almost) one to one mapping with ISAs available on computers (like amd64, x86, ARM, RISC-V, etc.). This means that each instruction in Java's ISA can be efficiently converted to your native ISA. 

Think of this as having only one word for something in 2 different languages. You do not have to worry about choosing the right word when translating. As an example, the Hindi statement "Mujhe maaf kariye" can be translated to either "Please forgive me" or "Pardon me". But if the word pardon did not exist, the translation would have been one-to-one, and you as a translator would not have to worry about which translation to use.

Now, all your programs written in Java can be compiled ahead of execution time to the JVM ISA. This ensures that all platforms which have a JVM implementation can run your program. This significantly increases efficiency.

## Phases of a Compiler

The resources [here](https://www.geeksforgeeks.org/phases-of-a-compiler/) and [here](https://www.tutorialspoint.com/compiler_design/compiler_design_phases_of_compiler.htm) are very helpful in understanding the details of a compiler.

