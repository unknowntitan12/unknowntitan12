---
layout: page
title: "Lab3-w8: bigfile & symlinks"
short_title: "W8: bigfile & symlinks"
lab: lab3
week: 8
order: 1
date: 2026-03-05
---

<div class="info-box">
📦 <strong>Lab zip:</strong> xv6labs-w8 &nbsp;|&nbsp; 📊 <strong>Score:</strong> 100/100 &nbsp;|&nbsp; 📅 <strong>Due:</strong> 13 Mar 2026 11:59pm
</div>

## Task 1 — `bigfile` (Doubly-Indirect Blocks)

### The problem

xv6's original inode has 12 direct slots + 1 singly-indirect slot = 268 blocks max (~268 KB). The goal is to support 65,803 blocks (~64 MB) by adding a doubly-indirect slot.

### The "After" layout

| Slot | Type | Reaches |
|------|------|---------|
| 0–10 (11 slots) | Direct | 11 data blocks |
| 11 | Singly-indirect | 1 map block × 256 = 256 data blocks |
| 12 | Doubly-indirect | 256 map blocks × 256 = 65,536 data blocks |

Total: 11 + 256 + 65,536 = **65,803 blocks** ✓

### Files to modify

| File | What changes |
|------|-------------|
| `kernel/param.h` | `FSSIZE` 2000 → 200000 |
| `kernel/fs.h` | `NDIRECT` 12→11, add `NDINDIRECT`, update `MAXFILE`, `dinode.addrs[NDIRECT+2]` |
| `kernel/file.h` | `inode.addrs[NDIRECT+2]` |
| `kernel/fs.c` | `bmap()` — add doubly-indirect case; `itrunc()` — free doubly-indirect tree |

### `kernel/param.h`

```c
#define FSSIZE  200000   // was 2000
```

### `kernel/fs.h`

```c
#define NDIRECT     11                         // was 12 — sacrificed one for doubly-indirect
#define NINDIRECT   (BSIZE / sizeof(uint))     // 256
#define NDINDIRECT  (NINDIRECT * NINDIRECT)    // 256*256 = 65536
#define MAXFILE     (NDIRECT + NINDIRECT + NDINDIRECT)   // 65803

struct dinode {
    short type;
    short major;
    short minor;
    short nlink;
    uint  size;
    uint  addrs[NDIRECT + 2];   // +2: one for singly-indirect, one for doubly-indirect
};
```

### `kernel/file.h`

```c
struct inode {
    // ... existing fields ...
    uint addrs[NDIRECT + 2];    // match dinode
};
```

### `kernel/fs.c` — `bmap()`

`bmap(ip, bn)` translates logical block number `bn` to physical block address.

```c
static uint
bmap(struct inode *ip, uint bn)
{
    uint addr, *a;
    struct buf *bp;

    // ── Direct blocks ──
    if (bn < NDIRECT) {
        if ((addr = ip->addrs[bn]) == 0) {
            addr = balloc(ip->dev);
            if (addr == 0) return 0;
            ip->addrs[bn] = addr;
        }
        return addr;
    }
    bn -= NDIRECT;

    // ── Singly-indirect ──
    if (bn < NINDIRECT) {
        if ((addr = ip->addrs[NDIRECT]) == 0) {
            addr = balloc(ip->dev);
            if (addr == 0) return 0;
            ip->addrs[NDIRECT] = addr;
        }
        bp = bread(ip->dev, addr);
        a = (uint *)bp->data;
        if ((addr = a[bn]) == 0) {
            addr = balloc(ip->dev);
            if (addr) {
                a[bn] = addr;
                log_write(bp);
            }
        }
        brelse(bp);
        return addr;
    }
    bn -= NINDIRECT;

    // ── Doubly-indirect ──
    if (bn < NDINDIRECT) {
        // Level 1: master map block (slot NDIRECT+1 in inode)
        if ((addr = ip->addrs[NDIRECT + 1]) == 0) {
            addr = balloc(ip->dev);
            if (addr == 0) return 0;
            ip->addrs[NDIRECT + 1] = addr;
        }
        bp = bread(ip->dev, addr);
        a = (uint *)bp->data;

        uint idx1 = bn / NINDIRECT;   // which secondary map?
        uint idx2 = bn % NINDIRECT;   // which data block in that map?

        // Level 2: secondary map block
        if ((addr = a[idx1]) == 0) {
            addr = balloc(ip->dev);
            if (addr) {
                a[idx1] = addr;
                log_write(bp);
            }
        }
        brelse(bp);            // ALWAYS release before reading next level
        if (addr == 0) return 0;

        bp = bread(ip->dev, addr);
        a = (uint *)bp->data;

        // Level 3: data block
        if ((addr = a[idx2]) == 0) {
            addr = balloc(ip->dev);
            if (addr) {
                a[idx2] = addr;
                log_write(bp);
            }
        }
        brelse(bp);
        return addr;
    }

    panic("bmap: out of range");
}
```

### `kernel/fs.c` — `itrunc()`

Free all blocks when a file is deleted. Add the doubly-indirect section **after** the existing singly-indirect section:

```c
// After the singly-indirect itrunc block:

// ── Free doubly-indirect ──
if (ip->addrs[NDIRECT + 1]) {
    struct buf *bp1 = bread(ip->dev, ip->addrs[NDIRECT + 1]);
    uint *a1 = (uint *)bp1->data;

    for (int i = 0; i < NINDIRECT; i++) {
        if (a1[i]) {
            struct buf *bp2 = bread(ip->dev, a1[i]);
            uint *a2 = (uint *)bp2->data;

            for (int j = 0; j < NINDIRECT; j++) {
                if (a2[j])
                    bfree(ip->dev, a2[j]);
            }
            brelse(bp2);
            bfree(ip->dev, a1[i]);
        }
    }
    brelse(bp1);
    bfree(ip->dev, ip->addrs[NDIRECT + 1]);
    ip->addrs[NDIRECT + 1] = 0;
}
```

### Common mistakes

- **Forgetting `brelse`**: Every `bread()` must have a matching `brelse()`. Missing one causes the buffer cache to fill up and the kernel to panic.
- **Wrong index**: The doubly-indirect slot is `ip->addrs[NDIRECT + 1]` (slot 12, index 12), not `ip->addrs[NDIRECT]` (that's the singly-indirect slot 11).
- **Not running `make clean`**: Changing `NDIRECT` changes the on-disk format. Always delete `fs.img` and rebuild.

---

## Task 2 — `symlink` (Symbolic Links)

### Overview

A symlink is a special file that stores a path string. When `open()` encounters a symlink, it follows it to the target (up to 10 levels deep).

### Files to modify

| File | Change |
|------|--------|
| `kernel/stat.h` | Add `#define T_SYMLINK 4` |
| `kernel/fcntl.h` | Add `#define O_NOFOLLOW 0x800` |
| `kernel/syscall.h` | Add `#define SYS_symlink <next_number>` |
| `kernel/syscall.c` | extern + table entry |
| `kernel/sysfile.c` | `sys_symlink()` + modify `sys_open()` |
| `user/user.h` | `int symlink(const char*, const char*);` |
| `user/usys.pl` | `entry("symlink");` |
| `Makefile` | `$U/_symlinktest\` |

### `kernel/stat.h`

```c
#define T_DIR     1   // Directory
#define T_FILE    2   // File
#define T_DEVICE  3   // Device
#define T_SYMLINK 4   // Symbolic link  ← ADD
```

### `kernel/fcntl.h`

```c
#define O_RDONLY   0x000
#define O_WRONLY   0x001
#define O_RDWR     0x002
#define O_CREATE   0x200
#define O_TRUNC    0x400
#define O_NOFOLLOW 0x800   // ← ADD (must not overlap existing flags)
```

### `kernel/sysfile.c` — `sys_symlink()`

```c
uint64
sys_symlink(void)
{
    char target[MAXPATH], path[MAXPATH];
    struct inode *ip;

    // Get the two string arguments
    if (argstr(0, target, MAXPATH) < 0 || argstr(1, path, MAXPATH) < 0)
        return -1;

    begin_op();

    // Create a new inode of type T_SYMLINK
    ip = create(path, T_SYMLINK, 0, 0);
    if (ip == 0) {
        end_op();
        return -1;
    }

    // Write the target path into the inode's data blocks
    if (writei(ip, 0, (uint64)target, 0, strlen(target)) != strlen(target)) {
        iunlockput(ip);
        end_op();
        return -1;
    }

    iunlockput(ip);
    end_op();
    return 0;
}
```

### `kernel/sysfile.c` — modify `sys_open()`

Find the section in `sys_open()` that does `ilock(ip)`. Add the symlink-following loop **after** `ilock(ip)` but **before** the directory check:

```c
// After: ilock(ip);
// Before: if(ip->type == T_DIR && omode != O_RDONLY){ ...

// ── Follow symlinks ──
int depth = 0;
while (ip->type == T_SYMLINK && !(omode & O_NOFOLLOW)) {
    if (depth >= 10) {
        // Too many levels of indirection — circular link
        iunlockput(ip);
        end_op();
        return -1;
    }
    depth++;

    // Read the target path from the symlink's data
    char target[MAXPATH];
    int n = readi(ip, 0, (uint64)target, 0, MAXPATH - 1);
    if (n <= 0) {
        iunlockput(ip);
        end_op();
        return -1;
    }
    target[n] = '\0';

    // Unlock and release the current (link) inode
    iunlockput(ip);

    // Look up the target
    if ((ip = namei(target)) == 0) {
        end_op();
        return -1;   // dangling symlink
    }
    ilock(ip);
}
```

### Register the syscall

`kernel/syscall.h`:
```c
#define SYS_symlink  <next_num>   // e.g. 24 if monitor was 23
```

`kernel/syscall.c`:
```c
extern uint64 sys_symlink(void);
// in table:
[SYS_symlink] sys_symlink,
```

`user/user.h`:
```c
int symlink(const char*, const char*);
```

`user/usys.pl`:
```perl
entry("symlink");
```

### Common mistakes

- **Forgetting `O_NOFOLLOW` doesn't overlap**: Check existing flags. `O_CREATE` is `0x200`, `O_TRUNC` is `0x400`, so `0x800` is safe.
- **Deadlock on `ilock`**: Before calling `namei(target)`, you must call `iunlockput(ip)`. If you hold the lock of the current inode while trying to lock the new one, you can deadlock (both are in the same inode table).
- **Not null-terminating the target**: `readi` doesn't add a null byte — always do `target[n] = '\0'`.

---

## Variation Questions

<div class="var-question">
<div class="var-qnum">Q1 <span class="diff diff-easy">Easy</span></div>

Why must you call `make clean` and rebuild `fs.img` after changing `NDIRECT` from 12 to 11?

<div class="answer-content">

<code>NDIRECT</code> determines the <strong>on-disk layout</strong> of every inode. The <code>dinode.addrs[]</code> array is stored directly on disk. Changing <code>NDIRECT</code> changes the size of this array:

<ul>
  <li>Old: <code>addrs[13]</code> = 52 bytes</li>
  <li>New: <code>addrs[13]</code> = 52 bytes (same size — we kept total slots = 13, but changed meaning)</li>
</ul>
Actually the concern is more subtle: the mkfs tool uses <code>NDIRECT</code> to build the initial file system image. If the compiled kernel expects <code>NDIRECT=11</code> but <code>fs.img</code> was built with <code>NDIRECT=12</code>, the kernel will misinterpret the inode layout and potentially corrupt the file system.

<code>make clean</code> deletes <code>fs.img</code>, forcing mkfs to rebuild it from scratch with the new constants.

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q2 <span class="diff diff-med">Medium</span></div>

What would happen if `itrunc()` freed the doubly-indirect data blocks but forgot to call `bfree()` on the secondary map blocks themselves?

<div class="answer-content">

<p>The secondary map blocks (which hold 256 pointers each) would be leaked — they'd remain marked as "used" in the bitmap even though the file no longer references them. Over time:</p>

<ol>
  <li>The free block count would decrease even as files are deleted</li>
  <li>Eventually the file system would report "no free blocks" even with logical space available</li>
  <li>Running <code>fsck</code> (file system check) would report inconsistencies: blocks allocated in the bitmap but not referenced by any inode</li>
</ol>
This is a <strong>block leak</strong>, analogous to a memory leak but for disk blocks. In xv6, <code>mkfs</code> resets the entire image on the next <code>make clean</code>, hiding the bug during development. In a real OS, this would be a serious data-integrity issue.

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q3 <span class="diff diff-med">Medium</span></div>

A symlink `A` points to symlink `B`, which points back to `A`. Without the depth counter, what would happen when you `open("A")`? Trace through the kernel code.

<div class="answer-content">

<p>Without the depth counter, the while loop in <code>sys_open()</code> would run forever:</p>

<ol>
  <li><code>ip = namei("A")</code> → A's inode (T_SYMLINK)</li>
  <li><code>readi</code> → target = "B"</li>
  <li><code>ip = namei("B")</code> → B's inode (T_SYMLINK)</li>
  <li><code>readi</code> → target = "A"</li>
  <li><code>ip = namei("A")</code> → A's inode (T_SYMLINK)</li>
  <li>→ infinite loop</li>
</ol>
In kernel space, this infinite loop would exhaust the <strong>kernel stack</strong> (since each iteration calls <code>ilock</code>, <code>readi</code>, <code>iunlockput</code>, <code>namei</code> — all of which use stack space), causing a kernel stack overflow and a <code>panic</code>. The process can never be interrupted out of a tight kernel loop.

<p>With <code>depth >= 10</code> check: after 10 iterations, <code>sys_open</code> returns <code>-1</code> (error). The user-space <code>open()</code> call returns -1 with <code>errno = ELOOP</code> in a real OS.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q4 <span class="diff diff-hard">Hard</span></div>

Implement `readlink(const char *path, char *buf, int bufsize)` as a new xv6 system call. It should read the target of a symlink into `buf` without following it. What's the key difference from how `sys_open` handles symlinks?

<div class="answer-content">

<code>readlink</code> opens the symlink inode <strong>without following</strong> it (like <code>O_NOFOLLOW</code>), then reads its data:

<pre><code class="language-c">uint64
sys_readlink(void)
{
    char path[MAXPATH];
    uint64 ubuf;
    int bufsize;

<p>if (argstr(0, path, MAXPATH) &lt; 0) return -1;     argaddr(1, &amp;ubuf);     argint(2, &amp;bufsize);</p>

<p>begin_op();     struct inode *ip = namei(path);     if (ip == 0) { end_op(); return -1; }     ilock(ip);</p>

<p>if (ip-&gt;type != T_SYMLINK) {         iunlockput(ip);         end_op();         return -1;   // EINVAL — not a symlink     }</p>

<p>// Read target path from inode data     char target[MAXPATH];     int n = readi(ip, 0, (uint64)target, 0, MAXPATH - 1);     iunlockput(ip);     end_op();</p>

<p>if (n &lt;= 0) return -1;     target[n] = '\0';</p>

<p>// Copy to user space — truncate to bufsize     int copylen = n &lt; bufsize ? n : bufsize;     if (copyout(myproc()-&gt;pagetable, ubuf, target, copylen) &lt; 0)         return -1;     return copylen; } </code></pre></p>

<strong>Key difference from <code>sys_open</code>:</strong> <code>sys_open</code> follows the symlink to reach the target file. <code>sys_readlink</code> stops at the symlink itself and reads the path string stored inside it. It explicitly checks <code>ip->type == T_SYMLINK</code> and never calls <code>namei</code> on the target.

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q5 <span class="diff diff-hard">Hard</span></div>

The doubly-indirect implementation adds 65,536 blocks. If you wanted to support a file up to 1 TB (assuming `BSIZE = 4096`), how many levels of indirection would you need? How would you design a **triply-indirect** block scheme?

<div class="answer-content">

<strong>Calculation:</strong>

<ul>
  <li>Blocks needed: 1 TB / 4096 = 268,435,456 blocks (2^28)</li>
  <li>NINDIRECT with 4KB blocks: 4096 / 4 = 1024 (using 32-bit block numbers)</li>
  <li>Triply-indirect capacity: 1024 × 1024 × 1024 = 1,073,741,824 ≈ 1 billion blocks → 4 TB ✓</li>
</ul>
<strong>Triply-indirect design (3 levels):</strong>

<pre><code>inode.addrs[13]:  triply-indirect
  → L1 block (1024 pointers to L2 blocks)
    → L2 block (1024 pointers to L3 blocks)
      → L3 block (1024 pointers to data blocks)
</code></pre>

<p>In <code>bmap()</code>, add another level:</p>

<pre><code class="language-c">// After doubly-indirect:
bn -= NDINDIRECT;
if (bn &lt; NINDIRECT * NINDIRECT * NINDIRECT) {
    // Level 1
    // addr = ip-&gt;addrs[NDIRECT + 2], bread, check a[idx1]
    // Level 2
    // brelse L1, bread L2, check a[idx2]
    // Level 3
    // brelse L2, bread L3, check/alloc a[idx3]
    // brelse L3, return addr
}
</code></pre>

<p>Linux ext4 uses extents (range-based) rather than deep indirection trees for better performance. The Linux VFS also supports up to 48-bit block addresses, enabling files up to 16 TB on 4KB block systems.</p>

</div>
</div>
