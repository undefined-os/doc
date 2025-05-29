# 共享内存(Shared Memory)开发

## `syscall`实现
```rust
pub fn sys_shmget(key: c_int, size: c_ulong, shm_flag: c_int);
pub fn sys_shmat(shm_id: c_int, shm_addr: c_ulong, shm_flag: c_int);
pub fn sys_shmctl(shm_id: c_int, op: c_int, buf: c_ulong);
pub fn sys_shmdt(shm_addr: c_ulong);
```
## 模块设计
```rust
  pub struct SharedMemory {
      /// The key of the shared memory segment
      pub key: u32,
      /// Virtual kernel address of the shared memory segment
      pub addr: usize,
      /// Page count of the shared memory segment
      pub page_count: usize,
  }
  
  impl Drop for SharedMemory {
      fn drop(&mut self) {
          let allocator = global_allocator();
          allocator.dealloc_pages(self.addr, self.page_count);
      }
  }
  
  pub struct SharedMemoryManager {
      mem_map: Mutex<BTreeMap<u32, Arc<SharedMemory>>>,
      next_key: AtomicU32,
  }
  
  impl SharedMemoryManager {
  	//...
      pub fn delete(&self, key: u32) -> bool {
           // sys_shmctl在IPC_RMID参数下的行为就是将对应的SharedMemory从全局的SHARED_MEMORY_MANAGER中remove
          self.mem_map.lock().remove(&key).is_some()
      }
  }
  
  pub static SHARED_MEMORY_MANAGER: SharedMemoryManager = SharedMemoryManager::new();
  
  pub struct ProcessData {
  	//方便sys_shmdt找到对应的SharedMemory，其参数是一个虚拟地址
      /// Shared memory
      pub shared_memory: Mutex<BTreeMap<VirtAddr, Arc<SharedMemory>>>,
  }
```


## 开发经历

- 在`sys_shmctl`将一个共享内存块设置为待删除时，我们的做法是直接将其从全局表单中移除，这样就可以在所有`attach`一段共享内存的进程`detach`或者结束后，利用`Rust`的`Drop`机制，优雅地实现自动内存释放，也不再需要维护`SharedMemory`的`deleted`成员变量。
- 同时这种设计的另一个好处是：规定被`sys_shmctl`标记为待删除的共享内存段不应还能够被`sys_shmat`映射，我们直接删除的做法可以让全局表单中查不到这个共享内存段，从而杜绝了这种行为，否则还需要再`sys_shmat`中进行判定校验。
- `ProcessData`的`shared_memory`类型定义为`Mutex<BTreeMap<VirtAddr, Arc<SharedMemory>>>`，原因是`sys_shmdt`的参数类型是`shm_addr`，这样设置方便直接找到虚拟地址所对应的共享内存段。
- 我们发现由于`Loongarch64`的内存布局和其他架构不同，需要给其分配更大的内存才足够通过测试，我们对可能的原因进行了调研，结果认为有可能是因为`Loongarch64`架构的虚拟地址空间存在“空洞”，即部分地址不可用，这可能导致需要分配更大的内存以弥补空洞造成的内存损失。

## 设计思路

关于共享内存自动释放的问题，有时OS会面临一种特殊的情况————一段通过共享内存段，如果所有通过`sys_shmat`绑定这段内存的进程都通过`sys_shmdt`解绑了，但是没有任何进程调用`sys_shmctl`把它标记为待删除，操作系统是否应该释放这段内存？此时一般有两种做法。

- 全局表单中`SharedMemoryManager`中使用强引用，也就是我们现在的实现方式，只有在有进程调用了`sys_shmctl`把某段共享内存设置为待删除同时所有绑定的进程解绑的时候才会进行释放，充分尊重了用户程序的行为，但是在用户程序不规范的情况下会造成内存泄露。
- 全局表单中`SharedMemoryManager`中使用弱引用，即使没有进程调用了`sys_shmctl`把某段共享内存设置为待删除，只要所有绑定的进程解绑就进行释放，一定程度上解决了内存释放的问题，但是OS无法假设所有进程解绑后共享内存就不再需要，因为其他进程可能在未来重新通过`sys_shmat`附加到该内存段。

面临这种`trade-off`，我们去查阅了`linux`操作系统的行为，具体如下：

- 通过`sys_shmget`创建的共享内存段是持久性系统级资源，声明周期独立于创建它的进程。它的生命周期不与任何单个进程绑定。即使所有进程都已调用`sys_shmdt`解绑或进程终止，共享内存段仍然存在于内核中，占用内存资源，直到显式调用`sys_shmctl`或系统重启，总体来说采取前者的实现。

- 那么如何解决共享内存导致的内存泄漏问题呢？

前面提到的系统重启的时候会清空所有 System V IPC 资源，包括共享内存段。同时`Linux`提供了工具，允许用户或管理员手动检查和删除不再需要的共享内存段，以解决内存泄漏问题。

  - `ipcs`命令：列出系统中当前存在的共享内存段（以及其他 IPC 资源，如信号量和消息队列），帮助用户识别可能未被正确清理的共享内存段。如果某共享内存段`nattch`表示附加的进程数。如果为`0`，说明没有进程使用该内存段，可能是泄漏的候选对象。
  - `ipcrm`命令：手动删除指定的共享内存段，允许管理员清理不再需要的共享内存段，解决内存泄漏。使用strace命令检查后发现`ipcrm`会调用`sys_shmctl`进行共享内存段的删除。


```bash
ipcs -m
```
```bash
------ Shared Memory Segments --------
key        shmid      owner      perms      bytes      nattch     status      
0x00000000 12345      user       666        4096       0          
```
```bash
strace ipcrm -m 12345
```
```bash
//...
shmctl(12345, IPC_RMID, NULL)           = 0
//...
+++ exited with 1 +++
```










  
