# process模块（独立模块）

我们实现了一个独立模块process，它是操作系统无关的，可以作为一个基础组件提供给其他操作系统使用，没有依赖ArceOS或者UndefinedOS中的组件。

在Linux中，进程的组织结构包括session, process group, process这几个层次，POSIX thread属于特殊的process。此外，进程之间还存在父子关系。

在我们的设计中，我们认为调度的基本单元是线程，我们据此设计了操作系统无关的线程和进程模型，旨在提供一个通用的包，来提供进程和线程管理操作，基本覆盖了POSIX的所有进程/线程操作，如：

- 会话/进程组/进程/线程创建
- 线程/进程退出(包括exit_group)和收养
- 更改进程所在的进程组
- pid/tid查询和管理
- 进程fork和创建新线程

## 细节说明

**会话/进程组/进程/线程创建**

同一层次间，我们使用 BTreeMap 关联 id 与 指针来管理，支持会话/进程组/进程/线程创建，每一层实现相仿，出于作为一个通用模块的设计考量，创建新的会话/进程组/进程/线程的接口在模块内部公开，不封装在结构体内部，只需提供指定的 id 、parent 、所属上级结构即可调用。

这里以创建一个新的进程为例进行解释。

```rust
static PROCESS_TABLE: Mutex<BTreeMap<Pid, Arc<Process>>> =  Mutex::new(BTreeMap::<Pid, Arc<Process>>::new());

/// Create a new process if the process does not exist
fn create_process(pid: Pid, parent: Weak<Process>, group: Weak<ProcessGroup>) -> Arc<Process> {
    let mut process_table = PROCESS_TABLE.lock();
    if process_table.contains_key(&pid) {
        panic!("[process] process with id {} already exists", pid);
    }
    // create process
    let process = Process::new(pid, parent.clone(), group.clone());
    // add to process group
    let group = group.upgrade().unwrap();
    group.add_process(process.clone());
    // add to parent
    if let Some(parent) = parent.upgrade() {
        parent.children.lock().insert(pid, process.clone());
    }
    // create main thread
    create_thread(pid, Arc::downgrade(&process));
    // add to process table
    process_table.insert(pid, process.clone());
    process
}
```

调用 create_process 所需参数为 分配给新进程的 pid ，指向父进程的指针、指向新进程的 process group 的指针。

调用 create_process 后首先实例化一个新的 Process 结构体，将其与指定pid关联，加入全局进程表与指定的 process group 中，随后调用 create_thread ，为这个新的进程创建一个线程 。

**进程Fork和线程创建**

进程 fork 通过调用 create_process 实现，发起 fork 的进程调用 create_process 时需指定 parent 为自己，子进程与父进程属于同一个进程组，新创建出的子进程只有一个主线程。

**线程/进程退出(包括exit_group)和收养**

我们维护进程之间的父子关系，进程 fork 时成为新进程的父进程，并且实现了收养机制。

具体来说：

* 线程退出调用 `Process::remove_thread`更新线程表，最后一个线程退出触发进程退出
* 进程退出时通过 `is_zombie`原子标志位标记为僵尸状态，将自己从所属进程组移除
* 空进程组自动从会话中移除
* 空会话自动从全局会话表移除
* 进程退出时其子进程由最近的收养者或是init进程收养
* 线程调用 exit_group 先自己 exit ，然后向其他线程发送信号，使所有线程退出

**pid/tid查询和管理**

```rust
static NEXT_PID: AtomicU32 = AtomicU32::new(1);

fn generate_next_pid() -> Pid {
    NEXT_PID.fetch_add(1, Ordering::Acquire)
}
```

* 对于session, process group, process这几个层次结构，我们实现了和linux一致的 main thread 、process group leader 、session leader 等
* process id 实际上就是主线程的 thread id ， process group id 等于其 leader 的 process id , session id 等于其 leader 的 process group id
* 由于 process id 实际上就是主线程的 thread id，所以我们通过同一个原子计数器保证全局唯一PID/TID分配
* 进程/线程生命周期与ID绑定，对象销毁时释放ID

**更改进程所在的进程组**

实现上非常简单，先在旧的进程组中移除某进程，然后将其添加到新的进程组中，但需要遵循**约束条件**：

* session leader 无法迁移进程组
* 目标进程组必须存在于当前会话中，不能跨会话迁移进程组
* 创建新的进程组时一定有  pgid == pid  ，即进程组id等于创建它的进程的id
