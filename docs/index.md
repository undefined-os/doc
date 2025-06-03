# Undefined OS 项目文档

欢迎来到Undefined OS 项目文档。文档将介绍OS的总体架构设计，具体的实现内容，开发经历，支持的应用程序，以及与一些其他OS的比较。

## 项目介绍

项目主体是基于[ArceOS](https://github.com/oscomp/arceos)和[StarryNext](https://github.com/oscomp/starry-next)编写的，对StarryNext的架构进行了较大的修改，修正和完善了ArceOS中的部分实现。在此基础上，我们实现了一百余个POSIX系统调用，提供了Linux兼容性，能够运行busybox等为Linux编译的实用程序。

## 项目特色

- 安全：使用Rust语言编写，提供了内存安全性，支持多CPU核心下的线程安全。
- 跨平台：支持x86_64/riscv64/aarch64/loongarch64 等多种指令架构，可以运行在qemu虚拟机和其他硬件开发板上。
- 兼容性：我们提供了兼容Linux的系统调用实现，同时保留了未来兼容其他OS的系统调用的接口。

## 项目地址

项目分为两部分进行开发：

- 底层的ArceOS部分：会积极合并来自上游的问题修复，同时添加新的提交，以问题修复为主；
- UndefinedOS部分：初版代码基于StarryNext，但现在的代码结构和内容已与原仓库有较大不同，提交以新功能开发和实现为主，对于上游的提交，以手工方式向UndefinedOS移植。

仓库分别为：

- ArceOS：[eternalcomet/arceos: An experimental modular OS written in Rust.](https://github.com/eternalcomet/arceos)
- UndefinedOS：[eternalcomet/undefined-os-final](https://github.com/eternalcomet/undefined-os-final)

## 主要功能

- 目前支持112个Linux系统调用
- 进程管理：支持多线程和多进程并发控制，支持进程组/会话管理
- 进程间通信：支持进程/线程间信号机制，支持共享内存
- 文件系统：支持ext4, fat32, ramfs文件系统，且提供了可扩展的接口设计以支持新的文件系统
- 内存管理：支持内存懒分配
- bash环境：支持运行在Linux根文件系统上，提供了基础的bash交互功能

```
888     888               888           .d888 d8b                        888         .d88888b.   .d8888b.  
888     888               888          d88P"  Y8P                        888        d88P" "Y88b d88P  Y88b 
888     888               888          888                               888        888     888 Y88b.      
888     888 88888b.   .d88888  .d88b.  888888 888 88888b.   .d88b.   .d88888        888     888  "Y888b.   
888     888 888 "88b d88" 888 d8P  Y8b 888    888 888 "88b d8P  Y8b d88" 888        888     888     "Y88b. 
888     888 888  888 888  888 88888888 888    888 888  888 88888888 888  888 888888 888     888       "888 
Y88b. .d88P 888  888 Y88b 888 Y8b.     888    888 888  888 Y8b.     Y88b 888        Y88b. .d88P Y88b  d88P 
 "Y88888P"  888  888  "Y88888  "Y8888  888    888 888  888  "Y8888   "Y88888         "Y88888P"   "Y8888P"  
```