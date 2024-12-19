# Lab 6: Copy-on-write fork
---
实验6涉及的是实现 **懒复制** (Copy-On-Write, COW) 机制，在 `fork()` 系统调用中实现延迟复制内存的功能。该机制的基本思想是在创建子进程时，不立即复制父进程的内存，而是让父子进程共享相同的物理页面。只有在进程尝试修改这些页面时，才会触发页面错误，并进行真正的内存复制。

### 1. **Fork 时懒复制的实现**

在 `fork()` 中，我们不立即复制父进程的内存页面，而是将父进程的物理页面映射到子进程的地址空间，并且在页表中标记这些页面为只读且具有 COW 标志。这个时候父子进程仍然共享相同的物理页面。

### 2. **懒复制的实现步骤**

#### 修改 `uvmcopy()`：在 `fork()` 中实现懒复制

```c
int uvmcopy(pagetable_t old, pagetable_t new, uint64 sz) {
    pte_t *pte;
    uint64 pa, i;
    uint flags;

    for(i = 0; i < sz; i += PGSIZE) {
        if((pte = walk(old, i, 0)) == 0)
            panic("uvmcopy: pte should exist");
        if((*pte & PTE_V) == 0)
            panic("uvmcopy: page not present");
        pa = PTE2PA(*pte);
        if(*pte & PTE_W) {
            // 清除父进程的 PTE_W 标志位，设置 PTE_COW 标志位
            *pte = (*pte & ~PTE_W) | PTE_COW;
        }
        flags = PTE_FLAGS(*pte);
        // 将父进程的物理页直接映射到子进程
        if(mappages(new, i, PGSIZE, (uint64)pa, flags) != 0) {
            goto err;
        }
        // 增加该页面的引用计数
        krefpage((void*)pa);
    }
    return 0;

 err:
    uvmunmap(new, 0, i / PGSIZE, 1);
    return -1;
}
```

### 3. **检测懒复制页并执行实复制**

当进程尝试修改一个被标记为 COW 的页面时，会触发一个页面故障。此时，操作系统会检测该页面是否为 COW 页，并执行实际的页面复制操作，分配一个新的物理页面，并将数据复制过去。最终，更新页表以使其变为可写。

#### `usertrap()` 中捕获页面错误并执行复制操作

```c
void usertrap(void) {
    // 省略其他代码
    if((r_scause() == 13 || r_scause() == 15) && uvmcheckcowpage(r_stval())) { // 触发 COW 页面错误
        if(uvmcowcopy(r_stval()) == -1) { // 如果内存不足则杀死进程
            p->killed = 1;
        }
    } else {
        printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
        p->killed = 1;
    }
    // 省略其他代码
}
```

#### `uvmcheckcowpage()` 和 `uvmcowcopy()` 函数

检测并执行懒复制的核心函数：

```c
int uvmcheckcowpage(uint64 va) {
    pte_t *pte;
    struct proc *p = myproc();

    return va < p->sz && ((pte = walk(p->pagetable, va, 0)) != 0)
        && (*pte & PTE_V) // 页表项存在
        && (*pte & PTE_COW); // 页是 COW 页
}

int uvmcowcopy(uint64 va) {
    pte_t *pte;
    struct proc *p = myproc();

    if((pte = walk(p->pagetable, va, 0)) == 0)
        panic("uvmcowcopy: walk");

    uint64 pa = PTE2PA(*pte);
    uint64 new = (uint64)kcopy_n_deref((void*)pa); // 创建一个新的物理页并复制数据
    if(new == 0)
        return -1;

    uint64 flags = (PTE_FLAGS(*pte) | PTE_W) & ~PTE_COW;
    uvmunmap(p->pagetable, PGROUNDDOWN(va), 1, 0);
    if(mappages(p->pagetable, va, 1, new, flags) == -1) {
        panic("uvmcowcopy: mappages");
    }
    return 0;
}
```

### 4. **物理页的引用计数管理**

在支持懒复制的内存管理中，多个进程可能会引用同一个物理页面。为了保证只有最后一个进程释放页面时才回收该物理页，需要使用引用计数。

#### 引用计数操作

在 `kalloc()`、`kfree()`、`krefpage()` 和 `kcopy_n_deref()` 函数中加入引用计数操作：

```c
#define PA2PGREF_ID(p) (((p)-KERNBASE)/PGSIZE)
#define PGREF_MAX_ENTRIES PA2PGREF_ID(PHYSTOP)

struct spinlock pgreflock;
int pageref[PGREF_MAX_ENTRIES];

void krefpage(void *pa) {
    acquire(&pgreflock);
    PA2PGREF(pa)++;
    release(&pgreflock);
}

void kfree(void *pa) {
    struct run *r;

    if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
        panic("kfree");

    acquire(&pgreflock);
    if(--PA2PGREF(pa) <= 0) {
        memset(pa, 1, PGSIZE);  // 填充垃圾数据，防止悬空指针
        r = (struct run*)pa;

        acquire(&kmem.lock);
        r->next = kmem.freelist;
        kmem.freelist = r;
        release(&kmem.lock);
    }
    release(&pgreflock);
}

void *kalloc(void) {
    struct run *r;

    acquire(&kmem.lock);
    r = kmem.freelist;
    if(r)
        kmem.freelist = r->next;
    release(&kmem.lock);

    if(r) {
        memset((char*)r, 5, PGSIZE); // 填充垃圾数据
        PA2PGREF(r) = 1;  // 新分配的物理页引用计数为 1
    }

    return (void*)r;
}

void *kcopy_n_deref(void *pa) {
    acquire(&pgreflock);

    if(PA2PGREF(pa) <= 1) {
        release(&pgreflock);
        return pa;
    }

    uint64 newpa = (uint64)kalloc();
    if(newpa == 0) {
        release(&pgreflock);
        return 0; // 内存不足
    }
    memmove((void*)newpa, (void*)pa, PGSIZE);

    PA2PGREF(pa)--;

    release(&pgreflock);
    return (void*)newpa;
}
```

### 5. **总结**

1. **懒复制机制**：`fork()` 不会立即复制内存，而是让父子进程共享相同的物理页，直到某个进程修改这些页面时才进行复制。
2. **COW 页的实现**：在 `fork()` 时将页标记为 COW 页，在页面写入时触发页故障，系统通过 `uvmcowcopy()` 来执行实际的内存复制。
3. **引用计数**：物理页通过引用计数来管理，确保当没有任何进程使用该页面时，才释放内存。

通过以上步骤，你可以实现一个简单的懒复制机制，使得 `fork()` 的性能得到提升，同时有效减少内存占用。
