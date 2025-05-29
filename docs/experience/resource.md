# resource limit 相关开发

## 相关 syscall

```rust
pub fn sys_prlimit64(
    pid: c_int,
    resource: c_int,
    new_limit: UserInPtr<ResourceLimit>,
    old_limit: UserOutPtr<ResourceLimit>,
) -> LinuxResult<isize>
```

```rust
pub fn sys_setrlimit(
    resource: c_int,
    resource_limit: UserInPtr<ResourceLimit>,
) -> LinuxResult<isize> 
```

```rust
pub fn sys_setrlimit(
    resource: c_int,
    resource_limit: UserInPtr<ResourceLimit>,
) -> LinuxResult<isize> 
```

## 相关设计

```rust
pub struct ProcessData {
    /// ...
    /// resource limits
    pub resource_limits: Arc<Mutex<ResourceLimits>>,
    /// ...
}
```

```rust
pub struct ResourceLimit {
    pub soft: u64, // 当前限制，不能超过 hard 
    pub hard: u64, // 硬限制，修改后的 hard 不能比原来大
}
```

共16种不同的resource limit ,如下所示

```rust
pub enum ResourceLimitType {
    CPU = RLIMIT_CPU,
    FSIZE = RLIMIT_FSIZE,
    DATA = RLIMIT_DATA,
    STACK = RLIMIT_STACK,
    CORE = RLIMIT_CORE,
    RSS = RLIMIT_RSS,
    NPROC = RLIMIT_NPROC,
    NOFILE = RLIMIT_NOFILE,
    MEMLOCK = RLIMIT_MEMLOCK,
    AS = RLIMIT_AS,
    LOCKS = RLIMIT_LOCKS,
    SIGPENDING = RLIMIT_SIGPENDING,
    MSGQUEUE = RLIMIT_MSGQUEUE,
    NICE = RLIMIT_NICE,
    RTPRIO = RLIMIT_RTPRIO,
    RTTIME = RLIMIT_RTTIME,
}
```

```rust
pub struct ResourceLimits([ResourceLimit; RLIM_NLIMITS as usize]);

impl ResourceLimits {
    pub fn new() -> Self {
        let mut limits = [ResourceLimit::new_infinite(); RLIM_NLIMITS as usize];
        limits[ResourceLimitType::STACK as usize] =
            ResourceLimit::new(axconfig::plat::USER_STACK_SIZE as u64, RLIMIT_INFINITY);
        limits[ResourceLimitType::CORE as usize] = ResourceLimit::new(0, RLIMIT_INFINITY);
        limits[ResourceLimitType::NPROC as usize] = ResourceLimit::new(10000, 10000);
        limits[ResourceLimitType::NOFILE as usize] = ResourceLimit::new(1024, 1024 * 1024); // 1024 files, 1M files max
        Self(limits)
    }

    pub fn get_soft(&self, resource: &ResourceLimitType) -> u64 {
        self.0[*resource as usize].soft
    }

    pub fn get(&self, resource: &ResourceLimitType) -> ResourceLimit {
        self.0[*resource as usize].clone()
    }

    pub fn set(&mut self, resource: &ResourceLimitType, limit: ResourceLimit) -> bool {
        if limit.soft > limit.hard {
            return false;
        }
        self.0[*resource as usize] = limit;
        true
    }
}

```

此外，为了让 resource limit 真正生效，还需要修改部分 syscall 的逻辑，判定当前操作是否会导致超出limit 限制。