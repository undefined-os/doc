# oskernel-2024-pantheon

此部分是Undefined-OS与Pantheon-OS的分析与比较。

## 相关链接
- [Pantheon-OS源码仓库](https://gitlab.eduxiji.net/T202410336992584/oskernel-2024-pantheon)
- [Pantheon-OS项目文档](https://gitlab.eduxiji.net/T202410336992584/oskernel-2024-pantheon/-/blob/main/%E5%86%B3%E8%B5%9B%E7%AC%AC%E4%B8%80%E9%98%B6%E6%AE%B5%E6%96%87%E6%A1%A3.pdf)

## 简介

- Pantheon-OS是由杭州电子科技大学Pantheon队李梁锋,张锦轩,周家正同学使用rust语言编写的宏内核操作系统，运行于RISC-V64架构。

- Pantheon-OS实现了中断与 异常处理、进程管理、内存管理以及文件系统等操作系统基本模块，总共113个`syscall`。

- Pantheon-OS实现了我们尚未实现的无栈异步协程调度、异步网络模块。

## 分析和比较

比较了我们的是实现情况后，Pantheon-OS优点在于：

- 内存管理方面，实现了内存页的COW机制和懒分配
- 进程管理方面，实现了无栈异步协程调度
- 网络模块方面，实现了异步网络模块
- 开发策略设计方面，各子模块分部开发且易移植

Pantheon-OS的一些改进之处在于：

- 仅仅支持`FAT32`和`EXT4`文件系统，当前的`Starry`还支持`RAMFS`
- 部分`syscall`仅支持了测例用到的部分参数，存在一定`unimplemented!()`的情况，功能完整性还可以进一步完善

## 改进方向

- 内存管理：引入COW机制，在内存页分配时，延迟实际的内存复制，仅在写入时才复制页面内容在；懒分配，进程请求内存时，仅分配虚拟内存地址，实际物理内存分配推迟到页面首次访问时。减少不必要的内存复制，尤其在进程fork时，显著降低内存开销。懒分配避免预先分配未使用的物理内存，优化资源利用。进一步提升性能，减少内存复制和分配的开销，加快进程创建和内存访问速度，特别是在高负载场景下，有利于我们的操作系统支持更大规模的应用程序。

- 进程管理：实现无栈异步协程调度，采用轻量级协程模型，减少上下文切换开销，避免传统线程的栈分配。设计高效的协程调度机制，支持高并发任务，结合事件驱动模型实现非阻塞任务处理。无栈协程显著降低内存占用和上下文切换成本，适合I/O密集型任务（如网络服务器），提升系统吞吐量。相比传统线程模型，协程占用更少资源，支持更高并发度，适合嵌入式或资源受限环境。

- 网络模块：实现异步网络模块，基于事件驱动的I/O多路复用（如select、epoll或io_uring），实现非阻塞网络通信。异步网络模块减少线程阻塞，提高网络I/O吞吐量，适合高并发网络应用（如Web服务器、实时通信）。非阻塞I/O和零拷贝技术减少数据拷贝和上下文切换，缩短响应时间，提升用户体验。

