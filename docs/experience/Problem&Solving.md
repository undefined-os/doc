# 问题与解决

这一部分主要介绍我们开发过程中遇到的各类问题和解决方法。


## 内存问题


- 原有框架下，`iozone`测试会发现写操作的内容读回来结果错误，发现是默认线性映射的问题，全都将缓冲区加载到内核态，问题解决。

```rust
//线性映射地址转换
pub const fn virt_to_phys(vaddr: VirtAddr) -> PhysAddr {
    pa!(vaddr.as_usize() - PHYS_VIRT_OFFSET)
}

pub const fn phys_to_virt(paddr: PhysAddr) -> VirtAddr {
    va!(paddr.as_usize() + PHYS_VIRT_OFFSET)
}

pub fn write(&mut self, buf: &[u8]) -> AxResult<usize> {
    let offset = if self.is_append {
        self.get_attr()?.size()
    } else {
        self.offset
    };
    //拷贝我们的OS所在的内核态，默认线性映射
    let buf = buf.to_vec();
    let node = self.access_node(Cap::WRITE)?;
    let write_len = node.write_at(offset, &&*buf)?;
    self.offset = offset + write_len as u64;
    Ok(write_len)
}
```

- `aarch64`架构下测评会发现找不到已经映射的页面，更新了依赖`page_table_multiarch`的正确兼容`aarch64`的版本后问题解决。
## 测例问题

- 测例架构支持：最早`basic`测例仅仅支持`x86_64`架构，初始的`starry-next`也只支持`x86_64`架构，所以最开始可以通过`basic`测例，但我们支持了四个架构的`stat`系列`syscall`之后`basic`测例无法通过，后更换了测例镜像使得`basic`测例支持四个架构后问题解决。

- 无效测例：在调试测例`fpclassify_invalid_ld80`的过程中，发现本地`wsl`运行此测例行为和输出结果与在实验框架下相同，但流水线测评为`fail`。经过沟通和确认发现此测例代码在`x86_64`下编译工具链会出现问题，后联系了编写测评流水线的同学进行修改，`ban`掉了这个测例。

- 错误测例：反馈了`busybox`中不合理的`kill 10`测例，测例要求成功`kill`掉`pid = 10`的进程，但并未保证`pid = 10`的进程一定存在，后测例修改为`fork`一个子进程后`kill`父进程，测例得到修正。

## 流水线问题
- 测评流水线逻辑问题：在评测脚本中通过统计正确输出次数是否为4(对应4个架构)来判定测例是否通过，但如果两个测例有相同的正确输出，会导致正确输出统计量翻倍，导致错误，联系了维护测评流水线的同学进行修改，问题解决。

## 架构设计

- 调用嵌套严重：初步接手`starry-next`的时候有相当大一部分`syscall`的实现调用嵌套非常严重，有些甚至从最顶层的`syscall`调用到底层的真实现需要来回调用五次甚至六次，以`sys_mmap`这个系统调用为例。


```rust   
    //syscall层第一次调用
    pub fn sys_mmap(
        addr: UserInPtr<usize>,
        length: usize,
        prot: i32,
        flags: i32,
        fd: i32,
        offset: isize,
    ) -> LinuxResult<isize> {
        //...
        aspace.map_alloc(
            start_addr,
            aligned_length,
            permission_flags.into(),
            populate,
        )?;
        //...
    }
    //第二层调用
    pub fn map_alloc(
        &mut self,
        start: VirtAddr,
        size: usize,
        flags: MappingFlags,
        populate: bool,
    ) -> AxResult {
        //...
        self.areas
            .map(area, &mut self.pt, false)
            .map_err(mapping_err_to_ax_err)?;
        //...
    }
    //第三层调用
    pub fn map(
        &mut self,
        area: MemoryArea<B>,
        page_table: &mut B::PageTable,
        unmap_overlap: bool,
    ) -> MappingResult {
        //实际分配与映射
        area.map_area(page_table)?;
        //...
    }
    //第四层调用
    pub(crate) fn map_area(&self, page_table: &mut B::PageTable) -> MappingResult {
        self.backend
            .map(self.start(), self.size(), self.flags, page_table)
            .then_some(())
            .ok_or(MappingError::BadState)
    }
    //第五层调用
    fn map(&self, start: VirtAddr, size: usize, flags: MappingFlags, pt: &mut PageTable) -> bool {
        match *self {
            Self::Linear { pa_va_offset } => Self::map_linear(start, size, flags, pt, pa_va_offset),
            Self::Alloc { populate } => Self::map_alloc(start, size, flags, pt, populate),
        }
    }
    pub(crate) fn map_alloc(
        start: VirtAddr,
        size: usize,
        flags: MappingFlags,
        pt: &mut PageTable,
        populate: bool,
    ) -> bool {
        //...
        if let Some(frame) = alloc_frame(true) {
            if let Ok(tlb) = pt.map(addr, frame, PageSize::Size4K, flags) {
                tlb.ignore(); // TLB flush on map is unnecessary, as there are no outdated mappings.
            } else {
                return false;
            }
        }
        //...
    }
    //第六层调用
    fn alloc_frame(zeroed: bool) -> Option<PhysAddr> {
        let vaddr = VirtAddr::from(global_allocator().alloc_pages(1, PAGE_SIZE_4K).ok()?);
        if zeroed {
            unsafe { core::ptr::write_bytes(vaddr.as_mut_ptr(), 0, PAGE_SIZE_4K) };
        }
        let paddr = virt_to_phys(vaddr);
        Some(paddr)
    }
```

这种现象不利于后续`syscall`的实现，也降低了代码的可读性和可移植性，我们执行了以下的措施：

- 实现新的`syscall`时，尽量严格执行`接口层-实现层-Arceos底层函数`的架构设计，嵌套层数限制在2~3次内。
- 对于已经实现的嵌套现象严重的`syscall`进行重构，这部分工作还在进行中，最终希望可以完全去除`arceos_posix_api`库。