# NPUcore-IMPACT

此部分是 Undefined-OS 与 NPUcore-IMPACT的分析与比较。

## 参考链接

[Fediory/NPUcore-IMPACT: “全国大学生计算机系统能力大赛 - 操作系统设计赛(全国)- OS内核实现赛道龙芯LA2K1000分赛道” 一等奖参赛作品](https://github.com/Fediory/NPUcore-IMPACT/tree/NPUcore-FF)

## 简介

NPUcore-IMPACT 是西北工业大学 NPUcore-IMPACT!!! 队伍 的冯宜湑、张瀚宸、张逸飞三位同学使用Rust编写的基于 LoongArch 架构的类Unix操作系统。

NPUcore-IMPACT 基于原先的 NPUcore(RISCV) 开发迭代并扩展，实现POSIX标准系统调用106个，支持信号机制及线程。

( NPUcore 是西北工业大学的操作系统内核构建实践型教学操作系统，曾获得2022年OSKernel大赛内核实现赛道一等奖，不支持LoongArch 架构)

## 分析和比较

参考 NPUcore-IMPACT 的仓库与文档，他们总共实现了106个POSIX标准系统调用，数量上来看和我们相仿。与 Undefined-OS 相比，主要特点是

- 从零开始适配 LoongArch 架构
- 进程/线程管理较简单（没有优先级调度，使用FIFO策略）
- 没有任务优先级设置，因此采用FIFO策略进行任务调度
- CoW
- 懒分配
- 适配了网络模块（但贡献主要属于同届的 NPUcore-重生之我是菜狗 队伍）

### 写时复制（CoW）

在进行fork系统调用构造出新的进程时，不需要将父进程地址空间的全部内存拷贝一份，而是让**子进程与父进程共享物理内存页**，这样做的开销就只有修改页表。同时如果某个内存段是懒分配的内存段，便不需要共享物理页，直接新增一个虚拟地址内存段即可。

懒分配：在**用户空间内存和文件映射区域**的物理页分配时先在 **虚拟地址空间**建立映射关系，但物理页帧只在**真正访问（发生缺页异常）时才分配** 。

总体而言NPUcore 实现的比较简单，其文档中自述，由于是第一年开启LoongArch赛道，花了大量时间精力来适配 LoongArch 与 开发板，所以除了“独自实现了2022版本的NPUcore到2k1000平台（龙芯架构）的适配，并封装为一个arch包，方便后人持续开发”与“基本完成了NPUcore在ext4文件系统的适配，但仍有少部分bug”外，其他方面贡献与创新有限。

