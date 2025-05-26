# process模块（独立模块）

我们实现了一个独立模块process，它是操作系统无关的，可以作为一个基础组件提供给其他操作系统使用，没有依赖ArceOS或者UndefinedOS中的组件。

在Linux中，进程的组织结构包括session, process group, process这几个层次，POSIX thread属于特殊的process。此外，进程之间还存在父子关系。

在我们的设计中，我们认为调度的基本单元是线程，我们据此设计了操作系统无关的线程和进程模型，旨在提供一个通用的包，来提供进程和线程管理操作，基本覆盖了POSIX的所有进程/线程操作。
