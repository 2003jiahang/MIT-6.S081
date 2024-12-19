# Lab 9: File Systems

### 概述

本实验的目标是扩展 `xv6` 的文件系统，支持大文件和符号链接。这项工作涉及对现有的文件系统进行修改，以便支持更大的文件，并实现符号链接文件的功能。实验的难度适中，要求理解文件系统底层的实现原理，尤其是 inode 结构和磁盘块的管理方式。

### 大文件支持

#### 问题背景
在原始的 `xv6` 文件系统中，每个 inode 结构最多能直接存储 12 个数据块（每个块大小为 1024 字节），这意味着一个文件最大只能占用 12 * 1024 = 12KB 的数据。如果文件超出了这个大小，文件系统会通过单级间接块来存储额外的块地址。

为了支持更大的文件，我们需要扩展当前的索引机制，加入 **二级间接索引**。这样，一个文件可以通过一级和二级间接块来管理更多的磁盘块，支持更大的文件。

#### 修改 inode 结构
为了支持更大的文件，我们修改了 `inode` 和 `dinode` 结构，增加了一个二级间接块。具体来说，inode 中有 11 个直接块、1 个单级间接块和 1 个二级间接块。这样最大支持的文件大小得到了扩展。

```c
// kernel/fs.h
#define NDIRECT 11 // 由原来的12减少为11
#define NINDIRECT (BSIZE / sizeof(uint)) // 每个间接块可以包含的块数
#define MAXFILE (NDIRECT + NINDIRECT + NINDIRECT * NINDIRECT) // 最大文件大小

// 磁盘上的 inode 结构体
struct dinode {
  short type;           // 文件类型
  short major;          // 设备的主设备号 (只有 T_DEVICE 类型的文件才有)
  short minor;          // 设备的次设备号
  short nlink;          // inode 的硬链接数量
  uint size;            // 文件大小（字节）
  uint addrs[NDIRECT+2]; // 包含直接、单级间接和二级间接的地址
};
```

#### 修改 `bmap` 函数
`bmap` 函数用于获取文件中某个块的物理地址。我们需要修改它，增加对单级间接块和二级间接块的支持。当文件的块数超过直接块的数量时，`bmap` 会逐层查找间接块，直到找到正确的磁盘块。

```c
// 获取文件中第 bn 个块的磁盘地址。
// 如果没有对应的块，会分配一个新的块。
static uint
bmap(struct inode *ip, uint bn)
{
  uint addr, *a;
  struct buf *bp;

  if(bn < NDIRECT) { // 直接块
    if((addr = ip->addrs[bn]) == 0)
      ip->addrs[bn] = addr = balloc(ip->dev);  // 分配新的块
    return addr;
  }
  bn -= NDIRECT;

  if(bn < NINDIRECT) { // 单级间接块
    if((addr = ip->addrs[NDIRECT]) == 0)
      ip->addrs[NDIRECT] = addr = balloc(ip->dev);
    bp = bread(ip->dev, addr);  // 读取间接块
    a = (uint*)bp->data;
    if((addr = a[bn]) == 0){
      a[bn] = addr = balloc(ip->dev);  // 分配新的块
      log_write(bp);  // 写回间接块
    }
    brelse(bp);  // 释放缓冲区
    return addr;
  }

  bn -= NINDIRECT;

  if(bn < NINDIRECT * NINDIRECT) { // 二级间接块
    if((addr = ip->addrs[NDIRECT+1]) == 0)
      ip->addrs[NDIRECT+1] = addr = balloc(ip->dev);
    bp = bread(ip->dev, addr);  // 读取一级间接块
    a = (uint*)bp->data;
    if((addr = a[bn / NINDIRECT]) == 0){
      a[bn / NINDIRECT] = addr = balloc(ip->dev);  // 分配二级间接块
      log_write(bp);  // 写回一级间接块
    }
    brelse(bp);  // 释放缓冲区

    bn %= NINDIRECT;
    bp = bread(ip->dev, addr);  // 读取二级间接块
    a = (uint*)bp->data;
    if((addr = a[bn]) == 0){
      a[bn] = addr = balloc(ip->dev);  // 分配数据块
      log_write(bp);  // 写回二级间接块
    }
    brelse(bp);  // 释放缓冲区
    return addr;
  }

  panic("bmap: out of range");
}
```

#### 修改 `itrunc` 函数
`itrunc` 函数负责释放 inode 占用的所有磁盘块。我们在这里也需要处理二级间接块，确保释放掉所有不再需要的块。

```c
// 截断 inode（释放文件内容）
void
itrunc(struct inode *ip)
{
  int i, j;
  struct buf *bp;
  uint *a;

  for(i = 0; i < NDIRECT; i++){
    if(ip->addrs[i]){
      bfree(ip->dev, ip->addrs[i]);  // 释放直接块
      ip->addrs[i] = 0;
    }
  }

  if(ip->addrs[NDIRECT]) {
    bp = bread(ip->dev, ip->addrs[NDIRECT]);
    a = (uint*)bp->data;
    for(j = 0; j < NINDIRECT; j++){
      if(a[j])
        bfree(ip->dev, a[j]);  // 释放单级间接块
    }
    brelse(bp);
    bfree(ip->dev, ip->addrs[NDIRECT]);
    ip->addrs[NDIRECT] = 0;
  }

  if(ip->addrs[NDIRECT+1]) {
    bp = bread(ip->dev, ip->addrs[NDIRECT+1]);
    a = (uint*)bp->data;
    for(j = 0; j < NINDIRECT; j++) {
      if(a[j]) {
        struct buf *bp2 = bread(ip->dev, a[j]);
        uint *a2 = (uint*)bp2->data;
        for(int k = 0; k < NINDIRECT; k++) {
          if(a2[k])
            bfree(ip->dev, a2[k]);  // 释放二级间接块
        }
        brelse(bp2);
        bfree(ip->dev, a[j]);
      }
    }
    brelse(bp);
    bfree(ip->dev, ip->addrs[NDIRECT+1]);
    ip->addrs[NDIRECT+1] = 0;
  }

  ip->size = 0;
  iupdate(ip);  // 更新 inode
}
```

### 符号链接

#### 问题背景
符号链接（Symlink）是一个特殊的文件，它存储指向另一个文件的路径。符号链接的实现需要在文件系统中进行一些额外的处理。通常，符号链接是一个占用 inode 的文件，它的第一个直接块用于存储目标文件的路径。

#### 实现符号链接的系统调用
首先，我们需要实现 `symlink` 系统调用，用于创建符号链接。符号链接的 inode 与普通文件的 inode 类似，但它的第一个数据块存储的是目标文件的路径。

```c
// kernel/sysfile.c
uint64
sys_symlink(void)
{
  struct inode *ip;
  char target[MAXPATH], path[MAXPATH];
  if(argstr(0, target, MAXPATH) < 0 || argstr(1, path, MAXPATH) < 0)
    return -1;

  begin_op();

  ip = create(path, T_SYMLINK, 0, 0);  // 创建符号链接
  if(ip == 0){
    end_op();
    return -1;
  }

  // 使用第一个数据块存储目标路径
  if(writei(ip, 0, (uint64)target, 0, strlen(target)) < 0) {
    end_op();
    return -1;
  }

  iunlockput(ip);
  end_op();
  return 0;
}
```

#### 修改 `sys_open` 处理符号链接
当打开文件时，如果遇到符号链接，系统调用会根据需要递归跟随符号链接，直到遇到一个普通文件或目录，除非指定了 `O_NOFOLLOW` 标志。

```c
// kernel/sysfile.c
uint64
sys_open(void)
{
  char path[MAXPATH];
  int fd, omode;
  struct file *f;
  struct inode *ip;
  int n;

  if((n = argstr(

0, path, MAXPATH)) < 0 || argint(1, &omode) < 0)
    return -1;

  begin_op();

  if(omode & O_CREATE) {
    ip = create(path, T_FILE, 0, 0);
    if(ip == 0){
      end_op();
      return -1;
    }
  } else {
    int symlink_depth = 0;
    while(1) { // 递归跟随符号链接
      if((ip = namei(path)) == 0){
        end_op();
        return -1;
      }
      ilock(ip);
      if(ip->type == T_SYMLINK && (omode & O_NOFOLLOW) == 0) {
        if(++symlink_depth > 10) {  // 防止符号链接循环
          iunlockput(ip);
          end_op();
          return -1;
        }
        if(readi(ip, 0, (uint64)path, 0, MAXPATH) < 0) {
          iunlockput(ip);
          end_op();
          return -1;
        }
      } else {
        break;  // 如果不是符号链接，停止循环
      }
    }
  }

  // 进一步处理打开文件的操作
  ...
}
```

### 总结

本实验通过修改 `xv6` 文件系统来支持大文件和符号链接，主要完成了以下任务：
- 扩展了 inode 结构，支持通过直接块、单级间接块和二级间接块存储更多的文件数据。
- 修改了 `bmap` 和 `itrunc` 函数，支持大文件的磁盘块分配和回收。
- 实现了符号链接的创建和打开功能，符号链接存储目标路径，并在打开时会递归跟随链接。
