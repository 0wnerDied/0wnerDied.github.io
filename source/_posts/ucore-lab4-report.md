---
title: uCore lab4 实验报告
categories:
  - uCore
tags:
  - OS
  - riscv
abbrlink: 57361
date: 2025-08-11 03:34:18
---

做这节实验的时候是真的头疼，光是新增的代码就读了好久，以前也确实没接触过文件系统相关的东西，感觉很折磨。后面写系统调用也是东一榔头西一棒槌的，感觉未来还是得再复盘。

## 本章任务

- `ch6b_usertest`
- merge ch5 的改动，然后再次测试 `ch6_usertest`。
- 完成本章问答作业。
- 完成本章编程作业。
- 最终，完成实验报告并 push 你的 ch6 分支到远程仓库。

## 编程作业

### 硬链接

你的电脑桌面是什么样的？放满了图标吗？反正我的 windows 是这样的。显然很少人会真的把可执行文件放到桌面上，桌面图标其实都是一些快捷方式。或者用 unix 的术语来说：软链接。为了减少工作量，我们今天来实现软链接的兄弟：[硬链接](https://en.wikipedia.org/wiki/Hard_link)。

硬链接要求两个不同的目录项指向同一个文件，在我们的文件系统中也就是两个不同名称目录项指向同一个磁盘块。本节要求实现三个系统调用 `sys_linkat`、`sys_unlinkat`、`sys_stat`。

**linkat**：

- syscall ID: 37
- 功能：创建一个文件的一个硬链接（[linkat标准接口](https://linux.die.net/man/2/linkat)）。
- 接口：`int linkat(int olddirfd, char* oldpath, int newdirfd, char* newpath, unsigned int flags)`
- 参数：
    - `olddirfd`，`newdirfd`: 仅为了兼容性考虑，本次实验中始终为 `AT_FDCWD` (-100)，可以忽略。
    - `flags`: 仅为了兼容性考虑，本次实验中始终为 0，可以忽略。
    - `oldpath`：原有文件路径。
    - `newpath`: 新的链接文件路径。
- 说明：
    - 为了方便，不考虑新文件路径已经存在的情况（属于未定义行为）。除非出现新旧名字一致的情况，此时需要返回 -1。
    - 返回值：如果出现了错误则返回 -1，否则返回 0。
- 可能的错误
    - 链接同名文件。

**unlinkat**:

- syscall ID: 35
- 功能：取消一个文件路径到文件的链接（[unlinkat标准接口](https://linux.die.net/man/2/unlinkat)）。
- 接口：`int unlinkat(int dirfd, char* path, unsigned int flags)`
- 参数：
    - `dirfd`: 仅为了兼容性考虑，本次实验中始终为 `AT_FDCWD` (-100)，可以忽略。
    - `flags`: 仅为了兼容性考虑，本次实验中始终为 0，可以忽略。
    - `path`：文件路径。
- 说明：
    - 需要注意 unlink 掉所有硬链接后彻底删除文件的情况。
- 返回值：如果出现了错误则返回 -1，否则返回 0。
- 可能的错误
    - 文件不存在。

**fstat**:

- syscall ID: 80
- 功能：获取文件状态。
- 接口：`int fstat(int fd, struct Stat* st)`
- 参数：
    - `fd`: 文件描述符
    - `st`: 文件状态结构体

        ```c
        struct Stat {
          uint64 dev,     // 文件所在磁盘驱动号，该实现写死为 0 即可。
          uint64 ino,     // inode 文件所在 inode 编号
          uint32 mode,    // 文件类型
          uint32 nlink,   // 硬链接数量，初始为1
          uint64 pad[7],  // 无需考虑，为了兼容性设计
        }

        // 文件类型只需要考虑:
        #define DIR 0x040000              // directory
        #define FILE 0x100000             // ordinary regular file
        ```

- 返回值：如果出现了错误则返回 -1，否则返回 0。
- 可能的错误
    - fd 无效。
    - st 地址非法。

### Tips

- 需要给 inode 和 dinode 都增加 link 的计数，但强烈建议不要改变整个数据结构的大小，事实上，推荐你修改一个 pad。
- os 和 nfs 的修改需要同步，只不过 nfs 比较简单，只需要初始化 link 计数为 1 就行（可以通过修改 `ialloc` 来实现）。
- unlink 有删除文件的语义，如果 link 计数为 0，需要删除 inode 和对应的数据块，为此你需要正确调用 `ivalid`、`iupdate`、`iput`（如果测试遇到bug了不妨再看看这句话），并取消 `iput` 中判断条件的注释。你可能需要修改 `iput` 注释中的变量名（如果你的计数变量不叫 nlink）。

## 编程作业答案

首先，简单把 ch5 的提交摘过来，或者直接合并分支也行，处理冲突很简单。测试一下 `make test BASE=2`，发现编译不通过，是 `spawn` 的系统调用迁移到 ch6 后，与新的 loader.c 不兼容导致的，看一下 ch6 的历史提交更改了什么就能很快修复了：

```patch
From 58af914164e469e653ee063a4050251fe8a5d86f Mon Sep 17 00:00:00 2001
From: 0wnerDied <z1281552865@gmail.com>
Date: Sat, 9 Aug 2025 15:06:32 +0800
Subject: [PATCH 08/11] syscall: adapt sys_spawn to ch6

os/syscall.c: In function 'sys_spawn':
os/syscall.c:188:7: error: implicit declaration of function 'get_id_by_name' [-Werror=implicit-function-declaration]
  188 |  id = get_id_by_name(name);
      |       ^~~~~~~~~~~~~~
os/syscall.c:197:6: error: implicit declaration of function 'loader'; did you mean 'bin_loader'? [-Werror=implicit-function-declaration]
  197 |  if (loader(id, np) < 0) {
      |      ^~~~~~
      |      bin_loader

This commit refactors the sys_spawn system call to make it compatible with the new
filesystem-based program loading mechanism introduced in ch6.

The previous implementation of spawn was designed for the ch5 architecture, where applications
were loaded from a pre-compiled, in-memory archive using an internal ID. With the transition to
a proper on-disk filesystem, this loading method became obsolete, causing the spawn system call to fail.

It now accepts a file path string from the user, uses namei() to resolve this path to
its corresponding inode on the disk. If the file is found, it allocates a new process
and uses the bin_loader() function to load the executable content from the inode into
the new process's address space. Otherwise, use init_stdio() to initialize standard I/O
for the new process to avoid errors like "[ERROR <pid>]invalid fd 1" because of
p->files[fd], that which files[1] is null. This change fully integrates spawn with the
filesystem, allowing it to correctly load and execute programs from disk.

Signed-off-by: 0wnerDied <z1281552865@gmail.com>
---
 os/syscall.c | 18 +++++++++++++-----
 1 file changed, 13 insertions(+), 5 deletions(-)

diff --git a/os/syscall.c b/os/syscall.c
index 2cb67f8..d8be06e 100644
--- a/os/syscall.c
+++ b/os/syscall.c
@@ -177,7 +177,7 @@ uint64 sys_spawn(uint64 va)
 	struct proc *p = curr_proc();
 	char name[MAX_STR_LEN];
 	struct proc *np;
-	int id;
+	struct inode *ip;
 
 	/*
 	 * Copy the filename from user space to kernel space.
@@ -185,19 +185,27 @@ uint64 sys_spawn(uint64 va)
 	if (copyinstr(p->pagetable, name, va, MAX_STR_LEN) < 0)
 		return -1;
 
-	id = get_id_by_name(name);
-	if (id < 0)
+	if ((ip = namei(name)) == 0)
 		return -1;
 
-	if ((np = allocproc()) == 0)
+	if ((np = allocproc()) == 0) {
+		iput(ip);
 		return -1;
+	}
+
+	/*
+	 * Initialize standard I/O for the new process.
+	 */
+	init_stdio(np);
 
 	np->parent = p;
 
-	if (loader(id, np) < 0) {
+	if (bin_loader(ip, np) < 0) {
 		freeproc(np);
+		iput(ip);
 		return -1;
 	}
+	iput(ip);
 
 	// add_task(np);
 
-- 
2.25.1

```

接下来测试一下 ch6_usertest，可以看见除了需要实现的三个系统调用以外，别的测例都是通过的，那么接下来实现实验要求的三个系统调用就可以了，这里我为了防止系统调用的函数体冗余，还新写了几个函数。简单一点的实现肯定也是没事的，因为前面实验指导书说过我们的 os 的文件系统只有一层，不过实现得更加细致肯定没有问题。修改结构体时别忘了对齐，不然镜像跑不起来。初始化也要相应做好，不然测例通不过。

```patch
From 4296a91d6325835336b6f848882d125d617b6255 Mon Sep 17 00:00:00 2001
From: 0wnerDied <z1281552865@gmail.com>
Date: Sat, 9 Aug 2025 22:44:04 +0800
Subject: [PATCH 09/11] Implement fstat syscall

Signed-off-by: 0wnerDied <z1281552865@gmail.com>
---
 nfs/fs.c     |  3 ++-
 nfs/fs.h     |  3 ++-
 os/file.h    |  3 ++-
 os/fs.c      |  9 ++++++---
 os/fs.h      | 14 +++++++++++++-
 os/syscall.c | 37 +++++++++++++++++++++++++++++++++++--
 6 files changed, 60 insertions(+), 9 deletions(-)

diff --git a/nfs/fs.c b/nfs/fs.c
index acff784..6a5d4c3 100644
--- a/nfs/fs.c
+++ b/nfs/fs.c
@@ -203,8 +203,9 @@ uint ialloc(ushort type)
 
 	bzero(&din, sizeof(din));
 	din.type = xshort(type);
-	din.size = xint(0);
 	// LAB4: You may want to init link count here
+	din.nlink = xshort(1);
+	din.size = xint(0);
 	winode(inum, &din);
 	return inum;
 }
diff --git a/nfs/fs.h b/nfs/fs.h
index 2e9670a..263d9b0 100644
--- a/nfs/fs.h
+++ b/nfs/fs.h
@@ -45,7 +45,8 @@ struct superblock {
 // On-disk inode structure
 struct dinode {
 	short type; // File type
-	short pad[3];
+	short pad[2];
+	short nlink;
 	uint size; // Size of file (bytes)
 	uint addrs[NDIRECT + 1]; // Data block addresses
 };
diff --git a/os/file.h b/os/file.h
index 433f0a1..aae675a 100644
--- a/os/file.h
+++ b/os/file.h
@@ -15,9 +15,10 @@ struct inode {
 	int ref; // Reference count
 	int valid; // inode has been read from disk?
 	short type; // copy of disk inode
+	// LAB4: You may need to add link count here
+	short nlink;
 	uint size;
 	uint addrs[NDIRECT + 1];
-	// LAB4: You may need to add link count here
 };
 
 // Defines a file in memory that provides information about the current use of the file and the corresponding inode location
diff --git a/os/fs.c b/os/fs.c
index beb8224..91a3d4e 100644
--- a/os/fs.c
+++ b/os/fs.c
@@ -114,6 +114,7 @@ struct inode *ialloc(uint dev, short type)
 		if (dip->type == 0) { // a free inode
 			memset(dip, 0, sizeof(*dip));
 			dip->type = type;
+			dip->nlink = 1;
 			bwrite(bp);
 			brelse(bp);
 			return iget(dev, inum);
@@ -135,8 +136,9 @@ void iupdate(struct inode *ip)
 	bp = bread(ip->dev, IBLOCK(ip->inum, sb));
 	dip = (struct dinode *)bp->data + ip->inum % IPB;
 	dip->type = ip->type;
-	dip->size = ip->size;
 	// LAB4: you may need to update link count here
+	dip->nlink = ip->nlink;
+	dip->size = ip->size;
 	memmove(dip->addrs, ip->addrs, sizeof(ip->addrs));
 	bwrite(bp);
 	brelse(bp);
@@ -188,8 +190,9 @@ void ivalid(struct inode *ip)
 		bp = bread(ip->dev, IBLOCK(ip->inum, sb));
 		dip = (struct dinode *)bp->data + ip->inum % IPB;
 		ip->type = dip->type;
-		ip->size = dip->size;
 		// LAB4: You may need to get lint count here
+		ip->nlink = dip->nlink;
+		ip->size = dip->size;
 		memmove(ip->addrs, dip->addrs, sizeof(ip->addrs));
 		brelse(bp);
 		ip->valid = 1;
@@ -208,7 +211,7 @@ void ivalid(struct inode *ip)
 void iput(struct inode *ip)
 {
 	// LAB4: Unmark the condition and change link count variable name (nlink) if needed
-	if (ip->ref == 1 && ip->valid && 0 /*&& ip->nlink == 0*/) {
+	if (ip->ref == 1 && ip->valid && ip->nlink == 0) {
 		// inode has no links and no other references: truncate and free.
 		itrunc(ip);
 		ip->type = 0;
diff --git a/os/fs.h b/os/fs.h
index efb4674..ddd640c 100644
--- a/os/fs.h
+++ b/os/fs.h
@@ -44,10 +44,11 @@ struct superblock {
 // On-disk inode structure
 struct dinode {
 	short type; // File type
-	short pad[3];
+	short pad[2];
 	// LAB4: you can reduce size of pad array and add link count below,
 	//       or you can just regard a pad as link count.
 	//       But keep in mind that you'd better keep sizeof(dinode) unchanged
+	short nlink;
 	uint size; // Size of file (bytes)
 	uint addrs[NDIRECT + 1]; // Data block addresses
 };
@@ -72,6 +73,17 @@ struct dirent {
 	char name[DIRSIZ];
 };
 
+struct stat {
+	uint64 dev;
+	uint64 ino;
+	uint32 mode;
+	uint32 nlink;
+	uint64 pad[7];
+};
+
+#define DIR 0x040000
+#define FILE 0x100000
+
 // file.h
 struct inode;
 
diff --git a/os/syscall.c b/os/syscall.c
index d8be06e..b880ec6 100644
--- a/os/syscall.c
+++ b/os/syscall.c
@@ -248,10 +248,43 @@ uint64 sys_close(int fd)
 	return 0;
 }
 
+/*
+ * sys_fstat - get file status
+ * @fd: The file descriptor to query.
+ * @stat: User-space address for the result.
+ *
+ * Retrieves information about an open file.
+ * It writes the data to a stat
+ * structure in user space. The structure
+ * includes device, inode number, type,
+ * and the number of hard links.
+ *
+ * Returns 0 on success, -1 on error.
+ */
 int sys_fstat(int fd, uint64 stat)
 {
-	//TODO: your job is to complete the syscall
-	return -1;
+	struct proc *p = curr_proc();
+	struct file *f = p->files[fd];
+
+	if (fd < 0 || fd > FD_BUFFER_SIZE || f == NULL)
+		return -1;
+
+	struct stat st;
+
+	st.dev = f->ip->dev;
+	st.ino = f->ip->inum;
+	if (f->ip->type == T_DIR)
+		st.mode = DIR;
+	else if (f->ip->type == T_FILE)
+		st.mode = FILE;
+	else
+		return -1;
+	st.nlink = f->ip->nlink;
+
+	if (copyout(p->pagetable, stat, (char *)&st, sizeof(st)) < 0)
+		return -1;
+
+	return 0;
 }
 
 int sys_linkat(int olddirfd, uint64 oldpath, int newdirfd, uint64 newpath,
-- 
2.25.1

```

```patch
From 5bafc45cca257777c87a0a863db321b65bd3c647 Mon Sep 17 00:00:00 2001
From: 0wnerDied <z1281552865@gmail.com>
Date: Sat, 9 Aug 2025 23:06:05 +0800
Subject: [PATCH 10/11] Implement linkat syscall

Signed-off-by: 0wnerDied <z1281552865@gmail.com>
---
 os/fs.c      | 33 +++++++++++++++++++++++++++++
 os/fs.h      |  1 +
 os/syscall.c | 59 +++++++++++++++++++++++++++++++++++++++++++++++++++-
 3 files changed, 92 insertions(+), 1 deletion(-)

diff --git a/os/fs.c b/os/fs.c
index 91a3d4e..8c38f4d 100644
--- a/os/fs.c
+++ b/os/fs.c
@@ -455,3 +455,36 @@ struct inode *namei(char *path)
 		panic("fs dumped.\n");
 	return dirlookup(dp, path + skip, 0);
 }
+
+/*
+ * @path: The full path to resolve.
+ *
+ * Finds the inode of a parent directory.
+ * It parses the given path to find
+ * the last component. Then it returns
+ * the inode of the containing directory.
+ * This is a helper for link and unlink.
+ *
+ * Returns inode of parent, or 0 on error.
+ */
+struct inode *nameiparent(char *path)
+{
+	char *s;
+	struct inode *ip;
+
+	// Find the last slash in the path string.
+	for (s = path + strlen(path) - 1; s >= path && *s != '/'; s--);
+
+	// If no slash, parent is the root directory.
+	if (s < path)
+		return root_dir();
+
+	// Temporarily truncate string at the slash
+    // to get the parent path.
+	*s = 0;
+	ip = namei(path);
+	// Restore the slash.
+	*s = '/';
+
+	return ip;
+}
diff --git a/os/fs.h b/os/fs.h
index ddd640c..01d6e79 100644
--- a/os/fs.h
+++ b/os/fs.h
@@ -99,6 +99,7 @@ void iunlock(struct inode *);
 void iunlockput(struct inode *);
 void iupdate(struct inode *);
 struct inode *namei(char *);
+struct inode *nameiparent(char *path);
 struct inode *root_dir();
 int readi(struct inode *, int, uint64, uint, uint);
 int writei(struct inode *, int, uint64, uint, uint);
diff --git a/os/syscall.c b/os/syscall.c
index b880ec6..1f7b761 100644
--- a/os/syscall.c
+++ b/os/syscall.c
@@ -287,10 +287,67 @@ int sys_fstat(int fd, uint64 stat)
 	return 0;
 }
 
+/*
+ * @oldpath: The existing file to link.
+ * @newpath: The new link name to create.
+ *
+ * Creates a new hard link to an
+ * existing file. It finds the inode
+ * of the old path. Then it increments
+ * the link count. A new directory entry
+ * is created for the new path. This
+ * entry points to the same inode.
+ * Directories cannot be hard-linked.
+ *
+ * Returns 0 on success, -1 on error.
+ */
 int sys_linkat(int olddirfd, uint64 oldpath, int newdirfd, uint64 newpath,
 	       uint64 flags)
 {
-	//TODO: your job is to complete the syscall
+	char old[MAXPATH], new[MAXPATH], *filename;
+	struct proc *p = curr_proc();
+	struct inode *dp, *ip;
+
+	if (copyinstr(p->pagetable, old, oldpath, MAXPATH) < 0 ||
+		copyinstr(p->pagetable, new, newpath, MAXPATH) < 0)
+		return -1;
+
+	// Can't be linked to itself.
+	if (strncmp(old, new, MAXPATH) == 0)
+		return -1;
+
+	if ((ip = namei(old)) == 0)
+		return -1;
+
+	ivalid(ip);
+
+	if (ip->type == T_DIR) {
+		iput(ip);
+		return -1;
+	}
+
+	ip->nlink++;
+	iupdate(ip);
+
+	if ((dp = nameiparent(new)) == 0)
+		goto fail;
+
+	for (filename = new + strlen(new) - 1;
+		 filename >= new && *filename != '/'; filename--);
+	filename++;
+
+	if (dirlink(dp, new, ip->inum) < 0) {
+		iput(dp);
+		goto fail;
+	}
+	iput(dp);
+	iput(ip);
+
+	return 0;
+fail:
+	ip->nlink--;
+	iupdate(ip);
+	iput(ip);
 	return -1;
 }
 
-- 
2.25.1

```

```patch
From 9064998e654b199eb4994876bda479c096f87b97 Mon Sep 17 00:00:00 2001
From: 0wnerDied <z1281552865@gmail.com>
Date: Sat, 9 Aug 2025 23:17:53 +0800
Subject: [PATCH 11/11] Implement unlinkat syscall

Signed-off-by: 0wnerDied <z1281552865@gmail.com>
---
 os/fs.c      | 54 +++++++++++++++++++++++++++++++++++++++++++++++++
 os/fs.h      |  2 ++
 os/syscall.c | 57 ++++++++++++++++++++++++++++++++++++++++++++++++++--
 3 files changed, 111 insertions(+), 2 deletions(-)

diff --git a/os/fs.c b/os/fs.c
index 8c38f4d..41f7180 100644
--- a/os/fs.c
+++ b/os/fs.c
@@ -488,3 +488,57 @@ struct inode *nameiparent(char *path)
 
 	return ip;
 }
+
+/*
+ * @dp:   The directory inode to modify.
+ * @off:  Offset of the entry to remove.
+ *
+ * Removes a directory entry by zeroing it.
+ * It first reads the entry to ensure
+ * it is a valid, non-empty one.
+ * Then it overwrites the entry with
+ * zeros, effectively unlinking it.
+ *
+ * Returns 0 on success, panics on error.
+ */
+int dirunlink(struct inode *dp, uint off)
+{
+	struct dirent de;
+
+	if (readi(dp, 0, (uint64)&de, off, sizeof(de)) != sizeof(de))
+		panic("dirunlink: read");
+	if (de.inum == 0)
+		panic("dirunlink: trying to unlink empty entry");
+
+	memset(&de, 0, sizeof(de));
+	if (writei(dp, 0, (uint64)&de, off, sizeof(de)) != sizeof(de))
+		panic("dirunlink: writei");
+
+	return 0;
+}
+
+/**
+ * @dp: The directory inode to check.
+ *
+ * Iterates through directory entries.
+ * It checks for any valid entries
+ * other than "." and "..". The loop
+ * starts after the first two entries.
+ *
+ * Returns 1 if empty, 0 otherwise.
+ */
+int isdirempty(struct inode *dp)
+{
+	struct dirent de;
+
+	// Iterate through directory entries. 
+	// Skip the first two entries, "." and "..".
+	for (uint off = 2 * sizeof(de); off < dp->size; off += sizeof(de)) {
+		if (readi(dp, 0, (uint64)&de, off, sizeof(de)) != sizeof(de))
+			panic("isdirempty: readi");
+		if (de.inum != 0)
+			return 0;
+	}
+
+	return 1;
+}
diff --git a/os/fs.h b/os/fs.h
index 01d6e79..d47bd76 100644
--- a/os/fs.h
+++ b/os/fs.h
@@ -105,4 +105,6 @@ int readi(struct inode *, int, uint64, uint, uint);
 int writei(struct inode *, int, uint64, uint, uint);
 void itrunc(struct inode *);
 int dirls(struct inode *);
+int dirunlink(struct inode *dp, uint off);
+int isdirempty(struct inode *dp);
 #endif //!__FS_H__
diff --git a/os/syscall.c b/os/syscall.c
index 1f7b761..278276c 100644
--- a/os/syscall.c
+++ b/os/syscall.c
@@ -351,10 +351,63 @@ fail:
 	return -1;
 }
 
+/*
+ * @name:   The path of the link to remove.
+ *
+ * Removes a link from a directory.
+ * It finds the parent directory and
+ * the target inode. If the target is
+ * a non-empty directory, it fails.
+ * The directory entry is then cleared.
+ * Finally, it decrements the inode's
+ * link count. If the link count
+ * drops to zero, the inode and its
+ * data blocks are freed by iput.
+ *
+ * Returns 0 on success, -1 on error.
+ */
 int sys_unlinkat(int dirfd, uint64 name, uint64 flags)
 {
-	//TODO: your job is to complete the syscall
-	return -1;
+	char path[MAXPATH], *filename;
+	struct proc *p = curr_proc();
+	struct inode *dp, *ip;
+	uint off;
+
+	if (copyinstr(p->pagetable, path, name, MAXPATH) < 0)
+		return -1;
+
+	for (filename = path + strlen(path) - 1;
+		 filename >= path && *filename != '/'; filename--);
+	filename++;
+
+	if ((dp = nameiparent(path)) == 0)
+		return -1;
+
+	if ((ip = dirlookup(dp, filename, &off)) == 0) {
+		iput(dp);
+		return -1;
+	}
+
+	ivalid(ip);
+
+	if (ip->type == T_DIR && !isdirempty(ip)) {
+		iput(dp);
+		iput(ip);
+		return -1;
+	}
+
+	if (dirunlink(dp, off) != 0) {
+		iput(dp);
+		iput(ip);
+		return -1;
+	}
+
+	ip->nlink--;
+	iupdate(ip);
+	iput(dp);
+	iput(ip);
+
+	return 0;
 }
 
 uint64 sys_sbrk(int n)
-- 
2.25.1

```

上面就是我对新增系统调用的完整实现了，我是按 `fstat`、`linkat`、`unlinkat` 的顺序实现的。懒得再写详细的 commit message，只做了一点简单的注释。再次 `make test BASE=2` 然后运行 `ch6_usertest`，最后看到 `ch6 Usertests passed!` 就行了。

## 问答作业

### ch6

1. 在我们的文件系统中，root inode起着什么作用？如果root inode中的内容损坏了，会发生什么？

### ch7

1. 举出使用 pipe 的一个实际应用的例子。

- tips:
    - 想想你平时咋使用 linux terminal 的？
    - 如何使用 cat 和 wc 完成一个文件的行数统计？

2. 如果需要在多个进程间互相通信，则需要为每一对进程建立一个管道，非常繁琐，请设计一个更易用的多进程通信机制。

## 问答作业答案

### ch6

1. 在我们的文件系统中，root inode 起着什么作用？如果 root inode 中的内容损坏了，会发生什么？

    root inode 代表根目录所对应的 inode，也是整个文件系统的起点，对其他的所有文件的操作都是在根目录的前提下完成的，如果损坏，那么整个文件系统就无法正常操作文件。

### ch7

1. 举出使用 pipe 的一个实际应用的例子。

- tips:
    - 想想你平时咋使用 linux terminal 的？
    - 如何使用 cat 和 wc 完成一个文件的行数统计？

    对一个文本文件排序并去重：`cat data.txt | sort | uniq`。

    用 cat 和 wc 完成一个文件的行数统计：`cat <filename> | wc -l`。

2. 如果需要在多个进程间互相通信，则需要为每一对进程建立一个管道，非常繁琐，请设计一个更易用的多进程通信机制。

    传统管道方案中，$n$ 个进程间通信需要建立 $n(n-1)/2$ 个管道，连接复杂度为 $O(n^2)$，管理困难繁琐。我认为，可以在内核中维护一个全局的消息缓冲区，所有进程的消息都写入这个统一的缓冲区。为每个进程维护独立的读取偏移量，这样每个进程可以按自己的进度读取消息，避免消息被其他进程读取后丢失的问题。为了避免安全问题，全局消息缓冲区只能在内核态访问，用户进程无法直接读写，并且所有消息操作必须通过系统调用进行，内核负责权限检查。
