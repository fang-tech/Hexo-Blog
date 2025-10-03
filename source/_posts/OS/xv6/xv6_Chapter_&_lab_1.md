---
title: xv6第一章-系统概述 & lab1
tags:
  - Operating-System
  - xv6
  - xv6-lab
---
# Chapter-1-Overview
## 操作系统的目的
> 这种说法就像宇宙的目的这样感觉浪漫 @fung

> The job of an operating system is to share a computer among multiple programs and to provide a more useful set of services than the hardware alone supports. An operating system manages and abstracts the low-level hardware, so that, for example, a word processor need not concern itself with which type of disk hardware is being used. An operating system shares the hardware among multiple programs so that they run (or appear to run) at the same time. Finally, operating systems provide controlled ways for programs to interact, so that they can share data or work together @xv6 book

操作系统的目的就是
- 能让多个程序共享一个计算机 (而不是像单片机程序一样, 只能跑一个程序)
- 提供硬件层面的抽象使用接口和权限的管理

现代操作系统应该具有的功能有
- 管理内存和其他的系统资源 (Managing memory and other system resources.)
- 有安全和访问权限控制机制 (Imposing security and access policies.)
- 调度和多路复用进程与线程 (Scheduling and multiplexing processes and threads.)
- 能动态启动和关闭用户态程序 (Launching and closing user programs dynamically.)
- 提供用户访问硬件资源的接口 (Providing a basic user interface and application programmer interface.)

## 运行的环境
xv6使用的是ricsv指令集(不同于x86), 所以xv6是无法直接跑在常见的内核上的, 需要做转译, 这也就是另一层qemu虚拟机. xv6是运行在qemu虚拟机上使用ricsv指令集的操作系统.

```bash
# 编译并运行
make qemu
```
## 进程和内存模型
## I/O与文件描述符
## 管道机制
## 文件系统

# lab & homework 1
