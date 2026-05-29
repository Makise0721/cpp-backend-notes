# 09 — 文件系统：VFS、inode、Page Cache、硬软链接

> 关联篇目：[03 fork（fd 继承）](./03-process-create-ipc.md) | [06 页面置换](./06-page-replacement-allocator.md)

---

## 1. 文件描述符——一切都是 fd

### 1.1 三层表结构

```
进程级              系统级                   文件系统级
┌──────────┐     ┌──────────────┐     ┌──────────────┐
│ fd 表     │     │ 打开文件表    │     │  inode 表    │
│ (per proc)│     │ (system-wide)│     │              │
├──────────┤     ├──────────────┤     ├──────────────┤
│ 0→stdin  │──→  │ refcnt=1     │──→  │ inode #1234  │
│ 1→stdout │──→  │ offset=0     │     │ (真正的文件)  │
│ 2→stderr │     │ flags=O_RDONLY│    │ size/perms   │
│ 3→"a.txt"│──→  │              │     │ block ptrs   │
└──────────┘     └──────────────┘     └──────────────┘
```

**fork 后父子进程的 fd 表**：子进程复制 fd 表，但指向**同一打开文件表项**（共享 offset！）。这就是管道能工作的原因。

### 1.2 关键结论

- 同一个文件可以被多个进程打开，各自有独立的 offset
- `dup`/`dup2` 复制的 fd 共享同一个打开文件表项（共享 offset）
- `fork` 后父子 fd 也共享打开文件表项
- 文件描述符是**进程内最小的未使用整数**

---

## 2. VFS（虚拟文件系统）——统一接口

Linux 可以挂载多种文件系统（ext4、xfs、btrfs、nfs、proc...），VFS 提供统一抽象：

```
用户程序
    │ open/read/write/close
    ▼
┌──────────┐
│   VFS    │  ← 统一接口
├──────────┤
│ ext4 │ xfs │ nfs │ proc │ ...  ← 具体文件系统实现
└──────┴─────┴─────┴──────┘
    │
    ▼
块设备驱动 → 磁盘
```

### 2.1 VFS 四大核心对象

| 对象 | 数据结构 | 含义 | 生命周期 |
|------|----------|------|------|
| **superblock** | `super_block` | 文件系统元信息（类型、大小、块大小） | 挂载时创建 |
| **inode** | `inode` | 文件的元数据（大小、权限、时间戳、数据块指针） | 文件存在就存在 |
| **dentry** | `dentry` | 目录项缓存（将路径名映射到 inode） | 缓存，内存不足时回收 |
| **file** | `file` | 打开文件的实例（offset、flags） | open→close |

---

## 3. inode——文件的身份证

```
inode 包含（stat 能看到的信息）:
- 文件类型（普通文件/目录/软链接/设备...）
- 权限（rwxrwxrwx）
- 所有者（uid/gid）
- 大小
- 时间戳（atime/mtime/ctime）
- 链接数（硬链接计数）
- 数据块指针（指向磁盘上实际数据的块号）

inode 不包含: 文件名！
文件名存在目录的 dentry 中
```

```bash
$ stat file.txt
  File: file.txt
  Size: 1024       Blocks: 8
  Inode: 1234567   Links: 2
  Access: (0644/-rw-r--r--)
```

---

## 4. 文件权限模型

```
权限位: rwx rwx rwx
        │   │   │
        │   │   └─ 其他用户
        │   └───── 组
        └───────── 所有者

特殊位:
SUID (4xxx): 以文件所有者的权限执行（如 /usr/bin/passwd）
SGID (2xxx): 以文件所属组的权限执行
Sticky (1xxx): 目录下只有文件所有者能删除（如 /tmp）
```

---

## 5. Page Cache——文件读写的核心缓存

### 5.1 读取流程

```
首次读 "file.txt":
1. read(fd, buf, size)
2. → 查 Page Cache → 未命中
3. → 从磁盘读取到 Page Cache
4. → 从 Page Cache 拷贝到用户 buf
5. 返回

再次读同一位置:
1. read(fd, buf, size)
2. → 查 Page Cache → 命中！
3. → 直接从 Page Cache 拷贝 → 零磁盘 I/O
```

**这就是为什么第二次跑同一个程序比第一次快**——可执行文件和库都在 Page Cache 中。

### 5.2 写入与回写

```
write(fd, buf, size):
1. 数据写入 Page Cache（标记为 dirty）
2. 立即返回（writeback 模式）——异步！
3. 内核后台线程（flusher）定期将 dirty 页写回磁盘

fsync(fd):  强制将该文件的 dirty 页写回磁盘 + 等待完成
fdatasync:  同上，但不刷新元数据（只刷数据，更快）
sync:       刷新所有 dirty 页
```

---

## 6. 标准 IO vs 系统调用 IO

| | 标准 IO (`fopen/fread`) | 系统调用 IO (`open/read`) |
|------|------|------|
| 缓冲 | 用户态缓冲区（`FILE*` 内部） | 无用户态缓冲 |
| 系统调用频率 | 低（缓冲合并多次读写） | 每次 `read` 都进内核 |
| 适用 | 普通文件读写 | 实时性要求、网络 IO、设备 |

```
标准 IO 的缓冲层:
用户程序 ←→ [用户态 stdio 缓冲] ←→ [内核 Page Cache] ←→ 磁盘
                  ↑                        ↑
            setvbuf 控制            OS 自动管理
```

---

## 7. 硬链接 vs 软链接

```
硬链接:                           软链接（符号链接）:
┌─────────┐                       ┌─────────┐
│ dentry  │                       │ dentry  │
│ "a.txt" │──→ inode #123        │ "b.txt" │──→ inode #456
└─────────┘    ┌──────────┐      └─────────┘    ┌──────────┐
               │ 数据块    │                     │ "/a.txt" │
┌─────────┐    │ "hello"  │                     │ (存路径)  │──→ ...
│ dentry  │    └──────────┘                     └──────────┘
│ "c.txt" │──→ inode #123
└─────────┘    (同一个 inode!)
```

| | 硬链接 | 软链接 |
|------|------|------|
| 本质 | 多个目录项指向同一 inode | 一个特殊文件，内容是目标路径 |
| `ls -l` 看到的 | 普通文件（`links` 数 > 1） | `lrwxrwxrwx → target` |
| 跨文件系统 | ❌（inode 是 per-fs 的） | ✅（存的是路径字符串） |
| 删原文件后 | 数据还在（links 减到 0 才删除） | ❌ **悬空链接** |
| 创建 | `ln a.txt c.txt` | `ln -s a.txt b.txt` |

---

## 小结

| 概念 | 一句话 |
|------|--------|
| fd 表 | 进程级→系统级→inode 三级结构 |
| VFS | 统一文件系统接口，抽象 superblock/inode/dentry/file |
| inode | 文件元数据（权限/大小/块指针），不含文件名 |
| Page Cache | 磁盘数据的缓存，读命中零 I/O，写异步回刷 |
| 硬链接 | 同 inode 多名字；软链接 = 存路径的特殊文件 |

下一篇 [10 系统调用 / proc / cgroups / eBPF](./10-system-basics.md)。
