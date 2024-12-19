# Lab5: Lazy Page Allocation

#### **实验目标**

本次实验的目标是实现一个懒分配机制，用于优化xv6中用户空间堆内存的分配。当进程请求堆内存时，操作系统并不立即分配实际的物理内存，而是仅仅记录分配请求。当程序实际访问这些内存时，操作系统才会进行物理内存的分配。通过这种延迟分配，系统可以减少不必要的内存分配开销，尤其适用于那些请求大量内存但仅使用其中一部分的程序。

---

### **实验内容**

#### **1. 从`sbrk()`中移除内存分配（简易部分）**

在实验的第一部分，我们修改了 `sys_sbrk()` 系统调用。原本在 `sbrk()` 中，操作系统会立即根据请求分配物理内存并更新页表。但是为了实现懒分配，我们改为仅调整进程的虚拟内存大小（`sz`），而不进行实际的物理内存分配。实际的内存分配将在进程访问这些内存时触发。

修改后的 `sys_sbrk()` 代码如下：

```c
uint64 sys_sbrk(void)
{
    int addr;
    int n;
    struct proc *p = myproc();
    if (argint(0, &n) < 0)
        return -1;

    addr = p->sz;
    if (n < 0) {
        uvmdealloc(p->pagetable, p->sz, p->sz + n); // 缩小堆内存时立即释放
    }
    p->sz += n; // 只更新进程的虚拟内存大小，不做物理内存分配
    return addr;
}
```

通过这样的修改，`sbrk()` 只是简单地调整了进程的内存空间大小，并没有进行内存的实际分配。这为后续的懒分配提供了基础。

---

#### **2. 实现懒分配（中等部分）**

接下来的任务是处理懒分配机制，具体来说，当进程访问尚未分配的内存时，我们需要捕获缺页异常（page fault），并在发生缺页时分配物理内存并建立映射。为了实现这一点，我们需要检测发生缺页异常时是否是访问懒分配的内存页。如果是懒分配的内存，操作系统将为其分配物理内存并建立页表映射。

首先，我们修改了 `usertrap()` 以处理用户态的缺页异常，代码如下：

```c
void usertrap(void)
{
    uint64 va = r_stval();
    if ((r_scause() == 13 || r_scause() == 15) && uvmshouldtouch(va)) {
        uvmlazytouch(va); // 对懒分配的内存页分配物理内存并映射
    } else {
        printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
        p->killed = 1;
    }
}
```

当检测到缺页异常且该异常发生在懒分配区域时，我们调用 `uvmlazytouch()` 函数进行实际的内存分配和页表更新：

```c
void uvmlazytouch(uint64 va)
{
    struct proc *p = myproc();
    char *mem = kalloc(); // 分配物理内存
    if (mem == 0) {
        printf("lazy alloc: out of memory\n");
        p->killed = 1;
    } else {
        memset(mem, 0, PGSIZE); // 初始化物理页
        if (mappages(p->pagetable, PGROUNDDOWN(va), PGSIZE, (uint64)mem, PTE_W | PTE_X | PTE_R | PTE_U) != 0) {
            printf("lazy alloc: failed to map page\n");
            kfree(mem);
            p->killed = 1;
        }
    }
}
```

此函数在访问懒分配页时分配一个新的物理页，并将其与相应的虚拟地址建立映射。通过这种方式，内存仅在实际访问时才会被分配和映射。

另外，我们定义了 `uvmshouldtouch()` 来判断某个虚拟地址是否需要进行懒分配：

```c
int uvmshouldtouch(uint64 va)
{
    pte_t *pte;
    struct proc *p = myproc();

    return va < p->sz && PGROUNDDOWN(va) != r_sp() &&
           (((pte = walk(p->pagetable, va, 0)) == 0) || ((*pte & PTE_V) == 0));
}
```

这个函数会检查虚拟地址是否在进程的堆空间范围内，是否不是栈的哨兵页，以及页表项是否有效。如果这些条件满足，表示该页需要进行懒分配。

---

#### **3. 修改内存管理函数以支持懒分配（中等部分）**

为了支持懒分配，我们需要对一些内存管理函数做修改，主要包括 `uvmunmap()`、`uvmcopy()`、`copyin()` 和 `copyout()`。这些函数在原始代码中假定所有内存页都已经分配并映射，因此需要更新以处理懒分配的页。

- **`uvmunmap()`**：取消内存映射时，需要跳过那些尚未映射（即懒分配）的页：

```c
void uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
{
    uint64 a;
    pte_t *pte;

    for (a = va; a < va + npages * PGSIZE; a += PGSIZE) {
        if ((pte = walk(pagetable, a, 0)) == 0 || (*pte & PTE_V) == 0) {
            continue; // 如果页未映射（懒加载页），跳过
        }
        // 执行取消映射操作
    }
}
```

- **`uvmcopy()`**：在复制进程的内存时，忽略懒分配的页：

```c
int uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
    uint64 i;
    pte_t *pte;

    for (i = 0; i < sz; i += PGSIZE) {
        if ((pte = walk(old, i, 0)) == 0 || (*pte & PTE_V) == 0) {
            continue; // 跳过懒加载页
        }
        // 进行内存复制
    }
    return 0;
}
```

- **`copyin()` 和 `copyout()`**：在进行用户态和内核态之间的数据拷贝时，我们确保访问的地址已经分配并映射。如果是懒分配页，我们首先触发懒分配：

```c
int copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
{
    if (uvmshouldtouch(dstva)) {
        uvmlazytouch(dstva); // 确保目标页已分配
    }
    // 执行拷贝操作
}

int copyin(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len)
{
    if (uvmshouldtouch(srcva)) {
        uvmlazytouch(srcva); // 确保源页已分配
    }
    // 执行拷贝操作
}
```

这些修改确保了在进行数据传输时，访问的虚拟地址对应的内存页已经被实际分配并映射。


---

### **遇到的问题与挑战**

- **栈保护页问题**：栈的保护页需要特别注意，因为栈的保护页不能进行懒分配。我们确保了在检测懒分配时，不会错误地对栈保护页进行内存分配。
- **内存释放问题**：懒分配的内存页在释放时需要特别处理，确保页面未被分配时不会发生错误。
