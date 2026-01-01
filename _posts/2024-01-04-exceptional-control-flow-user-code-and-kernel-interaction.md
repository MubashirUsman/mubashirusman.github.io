---
layout: post
title:  "Exceptional Control Flow: User Code and Kernel Interaction"
date:   2024-02-04 5:30:47 +0100
categories: linux
---
### How user code and kernel coexist through exceptional control flow. Understanding the mechanisms that allow safe transitions between user and kernel space.
A process is a running instance of a program consisting of memory and register values. Memory contains both user code and kernel code. To change system state (writing to disk, sending network packets, responding to Ctrl+C), control must transfer to the kernel. This is because we don't want user programs performing sensitive operations directly.

#### Key Concepts
1. Kernel - Collection of exception handlers called when events occur
2. Exception Table - Maps exception numbers to handler addresses
3. Process Context - Current operating state (memory, registers) that must be stored for resumption

#### Context vs Process Switch:
1. Context switch - Happens during interrupt/syscall handling, requires less state storage
1. Process switch - Complete removal from execution, requires full state storage and memory mapping changes
*Important: Root user privileges are OS-defined, while kernel mode privileges are CPU-defined. These are completely different concepts!*