# syscall-trace模块（独立模块）

在StarryNext项目原本的代码中，许多系统调用的参数和返回值都是人工编写代码手动打印的，还有些代码没有引入打印功能。现有的部分打印系统调用名称和返回值的代码可能远远不够，因为我们还需要打印系统调用的参数和作为结构体返回给用户的信息。

手动实现的打印代码是低效的，且容易遗漏，在改变函数签名时，可能会遗漏打印代码的修改。同时，这些打印代码可能不能合适地打印（包装）指针指向的结构体的内容。

## 架构设计与实现

因此，我们希望实现一个宏，能够自动生成类似 `strace`输出的系统调用日志打印代码，便于调试和分析系统调用的执行过程。这个宏可以在编译时根据函数签名自动生成打印代码，避免手动实现的错误和遗漏。我目前选择的方案是使用rust的过程宏，如果为函数加上 `#[syscall_trace]`属性，编译器会自动生成打印代码，在进入函数时打印函数名和参数（含非 `UserOutPtr`指针指向的内容），然后在闭包中执行函数体，最后打印返回值。

未来，我还将添加以下功能：

- 实现flags的打印
- 忽略参数中的tf(trap frame)
- 使得框架更通用化

## 使用举例

使用方法可以简单地在系统调用函数上添加 `#[syscall_trace]`即可。

例如：

```rust
#[syscall_trace]
pub fn sys_fstatat(
    dir_fd: c_int,
    path: UserInPtr<c_char>,
    stat_buf: UserOutPtr<UserStat>,
    flags: c_int,
) -> LinuxResult<isize> {
    // ...
}
```

输出可能类似：

```rust
[syscall] <= sys_fstatat(dir_fd = -100, path = "busybox" @ VA:0x3ffffffcb, stat_buf = ... @ VA:0x3fffffab0, flags = 0)
[syscall] => sys_fstatat(dir_fd = -100, path = "busybox" @ VA:0x3ffffffcb, stat_buf = UserStat { st_dev: 0, st_ino: 1, st_mode: 33188, st_nlink: 1, st_uid: 1000, st_gid: 1000, st_rdev: 0, _pad0: 0, st_size: 1104040, st_blksize: 512, _pad1: 0, st_blocks: 2157, st_atime: TimeSpec { seconds: 0, nanoseconds: 0 }, st_mtime: TimeSpec { seconds: 0, nanoseconds: 0 }, st_ctime: TimeSpec { seconds: 0, nanoseconds: 0 }, _unused: [0, 0] } @ VA:0x3fffffab0, flags = 0) = 0
```
