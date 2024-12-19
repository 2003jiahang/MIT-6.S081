# Lab 10: mmap

在这个实验中，我们将实现一个简化版的 `mmap` 系统调用，该系统调用能够将文件映射到进程的虚拟地址空间，并支持懒加载及将修改内容写回文件。这一过程涉及以下几个关键的操作系统概念：虚拟内存（Virtual Memory），文件映射，懒加载（Lazy Allocation），以及进程间的内存管理。我们将逐步实现这些功能。

---

### 1. 定义 `vma`（Virtual Memory Area）结构体

我们首先定义一个结构体 `vma` 来管理每个进程的虚拟内存区域，它保存了映射文件的信息，包括映射的起始地址、大小、文件偏移量、权限等信息。每个进程最多有 16 个 `vma`，用于管理文件映射的区域。

```c
// kernel/proc.h
#define NVMA 16  // 最大支持的 vma 数量

struct vma {
    int valid;       // 是否有效
    uint64 vastart;  // 映射区域的起始地址
    uint64 sz;       // 映射的大小
    struct file *f;  // 映射的文件
    int prot;        // 映射的权限
    int flags;       // 映射的标志
    uint64 offset;   // 文件的偏移量
};
```

### 2. 更新 `proc` 结构体

在 `proc` 结构体中，增加一个 `vmas` 数组，来存储每个进程的虚拟内存区域。

```c
// kernel/proc.h
struct proc {
    struct spinlock lock;

    // 省略其他成员

    struct vma vmas[NVMA];  // 虚拟内存区域
};
```

### 3. 实现 `sys_mmap` 系统调用

`mmap` 系统调用的主要工作是将文件映射到进程的虚拟内存中。我们需要找到一个空闲的 `vma`，然后计算文件映射的虚拟地址范围，并将文件映射到该区域。需要注意检查文件的权限，确保映射过程中的权限符合要求。

```c
// kernel/sysfile.c
uint64 sys_mmap(void) {
    uint64 addr, sz, offset;
    int prot, flags, fd;
    struct file *f;

    if (argaddr(0, &addr) < 0 || argaddr(1, &sz) < 0 || argint(2, &prot) < 0
        || argint(3, &flags) < 0 || argfd(4, &fd, &f) < 0 || argaddr(5, &offset) < 0 || sz == 0)
        return -1;

    // 检查文件的可读写权限
    if ((!f->readable && (prot & PROT_READ)) || (!f->writable && (prot & PROT_WRITE) && !(flags & MAP_PRIVATE))) {
        return -1;
    }

    sz = PGROUNDUP(sz);  // 确保大小为页对齐

    struct proc *p = myproc();
    struct vma *v = 0;
    uint64 vaend = MMAPEND;  // 从高地址开始向下分配

    // 查找空闲的 vma 并计算映射地址
    for (int i = 0; i < NVMA; i++) {
        struct vma *vv = &p->vmas[i];
        if (vv->valid == 0) {
            if (v == 0) {
                v = &p->vmas[i];
                v->valid = 1;
            }
        } else if (vv->vastart < vaend) {
            vaend = PGROUNDDOWN(vv->vastart);  // 确保不重叠
        }
    }

    if (v == 0) {
        panic("mmap: no free vma");
    }

    v->vastart = vaend - sz;
    v->sz = sz;
    v->prot = prot;
    v->flags = flags;
    v->f = f;  // 假设文件类型为 FD_INODE
    v->offset = offset;

    filedup(v->f);  // 增加文件引用计数

    return v->vastart;  // 返回映射的起始地址
}
```

### 4. 懒加载机制

为了优化内存的使用，`mmap` 映射的文件使用懒加载的策略，只有在访问映射的内存时，才会将对应的文件内容从磁盘加载到内存中。这可以通过 `usertrap` 和 `vmatrylazytouch` 来实现。

```c
// kernel/trap.c
void usertrap(void) {
    int which_dev = 0;

    // ...其他处理

    // 检查是否发生页面错误，进行懒加载
    uint64 va = r_stval();
    if (r_scause() == 13 || r_scause() == 15) {  // 页面错误
        if (!vmatrylazytouch(va)) {
            goto unexpected_scause;
        }
    } else {
        unexpected_scause:
        printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
        p->killed = 1;
    }

    usertrapret();
}

// kernel/sysfile.c
int vmatrylazytouch(uint64 va) {
    struct proc *p = myproc();
    struct vma *v = findvma(p, va);
    if (v == 0) {
        return 0;  // 未找到映射
    }

    // 分配物理页并从文件加载内容
    void *pa = kalloc();
    if (pa == 0) {
        panic("vmatrylazytouch: kalloc");
    }
    memset(pa, 0, PGSIZE);

    // 从文件读取数据到物理页
    begin_op();
    ilock(v->f->ip);
    readi(v->f->ip, 0, (uint64)pa, v->offset + PGROUNDDOWN(va - v->vastart), PGSIZE);
    iunlock(v->f->ip);
    end_op();

    // 映射到进程的虚拟地址空间
    int perm = PTE_U;  // 用户权限
    if (v->prot & PROT_READ) perm |= PTE_R;
    if (v->prot & PROT_WRITE) perm |= PTE_W;
    if (v->prot & PROT_EXEC) perm |= PTE_X;

    if (mappages(p->pagetable, va, PGSIZE, (uint64)pa, perm) < 0) {
        panic("vmatrylazytouch: mappages");
    }

    return 1;
}
```

### 5. 实现 `munmap` 系统调用

`munmap` 系统调用用于释放映射的内存区域。它需要处理内存页的释放，并在必要时将数据写回磁盘。

```c
// kernel/sysfile.c
uint64 sys_munmap(void) {
    uint64 addr, sz;
    if (argaddr(0, &addr) < 0 || argaddr(1, &sz) < 0 || sz == 0)
        return -1;

    struct proc *p = myproc();
    struct vma *v = findvma(p, addr);
    if (v == 0) {
        return -1;  // 找不到映射
    }

    if (addr > v->vastart && addr + sz < v->vastart + v->sz) {
        return -1;  // 不允许在中间“挖洞”
    }

    uint64 addr_aligned = addr;
    if (addr > v->vastart) {
        addr_aligned = PGROUNDUP(addr);  // 页对齐
    }

    int nunmap = sz - (addr_aligned - addr);  // 计算实际要释放的内存字节数
    if (nunmap < 0) nunmap = 0;

    vmaunmap(p->pagetable, addr_aligned, nunmap, v);  // 自定义的释放映射内存函数

    if (addr <= v->vastart && addr + sz > v->vastart) {
        v->offset += addr + sz - v->vastart;
        v->vastart = addr + sz;
    }

    v->sz -= sz;

    if (v->sz <= 0) {
        fileclose(v->f);  // 关闭文件引用
        v->valid = 0;  // 标记 vma 无效
    }

    return 0;
}
```

### 6. 内存页的释放与写回磁盘

在 `vmaunmap` 中，我们处理了对内存页的释放，确保如果页面被修改过（即 `PTE_D` 被设置），会将修改的数据写回到文件。

```c
// kernel/vm.c
void vmaunmap(pagetable_t pagetable, uint64 va, uint

64 sz, struct vma *v) {
    uint64 end = va + sz;
    for (uint64 addr = va; addr < end; addr += PGSIZE) {
        pte_t *pte = walk(pagetable, addr, 0);
        if (*pte & PTE_V) {
            uint64 pa = PTE2PA(*pte);
            if (*pte & PTE_D) {
                // 如果页面被修改，写回文件
                begin_op();
                ilock(v->f->ip);
                writei(v->f->ip, 0, pa, v->offset + (addr - v->vastart), PGSIZE);
                iunlock(v->f->ip);
                end_op();
            }
            freepages(pa);  // 释放物理页
            *pte = 0;       // 清除页表项
        }
    }
}
```

通过这些步骤，我们实现了一个简化版的 `mmap` 系统调用，它支持文件映射、懒加载以及内存的回收和写回磁盘。
