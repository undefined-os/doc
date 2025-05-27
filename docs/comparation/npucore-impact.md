# 共享内存(Shared Memory)开发


```rust
pub fn sys_shmget(key: c_int, size: c_ulong, shm_flag: c_int);
pub fn sys_shmat(shm_id: c_int, shm_addr: c_ulong, shm_flag: c_int);
pub fn sys_shmctl(shm_id: c_int, op: c_int, buf: c_ulong);
pub fn sys_shmdt(shm_addr: c_ulong);
```

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





  
