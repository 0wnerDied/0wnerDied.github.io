---
title: uCore lab3 实验报告
categories:
  - uCore
tags:
  - OS
  - riscv
abbrlink: 10007
date: 2025-08-02 15:21:47
---

## 本章任务

- `make test BASE=1` 执行 usershell，然后输入 `ch5b_usertest` 运行。
- merge ch4 的修改（如有需要，可以 `make test BASE=2` 然后输入 `ch5_mergetest` 运行，以此检查 merge 是否正确，但这不是实验必需要求）。
- 结合文档和代码理解 fork, exec, wait 的逻辑。结合课堂内容回答本章问答问题（注意第二问为选做）。
- 理解框架的调度机制，尤其要搞明白时钟中断的处理机制以及 yield 之后下一个进程的选择。在次基础上，完成本节的编程作业 (2)stride 调度算法。
- 完成本章编程作业。
- 最终，完成实验报告并 push 你的 ch5 分支到远程仓库。

## 编程作业

### 关于之前的 syscall

你仍需要迁移上一章的 `sys_gettimeofday` `sys_mmap` `sys_munmap` 以适应新的进程结构。不过，__从本章节开始，不再要求维护 `sys_trace` 这一系统调用__。

### 进程创建

大家一定好奇过为啥进程创建要用 fork + execve 这么一个奇怪的系统调用，就不能直接搞一个新进程吗？思而不学则殆，我们就来试一试！这章的编程练习请大家实现一个完全 DIY 的系统调用 spawn，用以创建一个新进程。

spawn 系统调用定义：

```c
int sys_spawn(char *filename)
```

- syscall ID: 400
- 功能：相当于 fork + exec，新建子进程并执行目标程序。
- 说明：成功返回子进程id，否则返回 -1。
- 可能的错误：
    - 无效的文件名。
    - 进程池满/内存不足等资源错误。

实现完成之后，你应该能通过 ch5_spawn* 对应的所有测例，在 shell 中执行 ch5_usertest 来执行所有测试，应当发现除了 setprio 相关的测例均正确。

#### tips:

- 注意 fork 的执行流，新进程 context 的 ra 和 sp 与父进程不同。所以你不能在内核中通过 fork 和 exec 的简单组合实现 spawn。
- 在 spawn 中不应该有任何形式的内存拷贝。

### stride 调度算法

lab3 中我们引入了任务调度的概念，可以在不同任务之间切换，目前我们实现的调度算法十分简单，存在一些问题且不存在优先级。现在我们要为我们的 os 实现一种带优先级的调度算法：stride 调度算法。

算法描述如下:

1. 为每个进程设置一个当前 stride，表示该进程当前已经运行的“长度”。另外设置其对应的 pass 值（只与进程的优先权有关系），表示对应进程在调度后，stride 需要进行的累加值。
2. 每次需要调度时，从当前 runnable 态的进程中选择 stride 最小的进程调度。对于获得调度的进程 P，将对应的 stride 加上其对应的步长 pass。
3. 一个时间片后，回到上一步骤，重新调度当前 stride 最小的进程。

可以证明，如果令 P.pass = BigStride / P.priority 其中 P.pass 为进程的 pass 值，P.priority 表示进程的优先权（大于 1），而 BigStride 表示一个预先定义的大常数，则该调度方案为每个进程分配的时间将与其优先级成正比。证明过程我们在这里略去，有兴趣的同学可以在网上查找相关资料。

其他实验细节：

- stride 调度要求进程优先级 ≥ 2，所以设定进程优先级 ≤ 1 会导致错误。
- 进程初始 stride 设置为 0 即可。
- 进程初始优先级设置为 16。

为了实现该调度算法，内核还要增加 `sys_set_priority` 系统调用:

```c
int sys_set_priority(long long prio)
```

- 功能描述
    - syscall ID: 140
    - 功能：设定进程优先级。
    - 说明：设定自身进程优先级，只要 prio 在 [2, isize_max] 就成功，返回 prio，否则返回 -1。

- 针对测例
    - ch5_setprio

完成之后你需要调整框架的代码调度机制，是的可以设置不同进程优先级之后可以按照 stride 算法进行调度。实现正确后，代码应该能够通过用户测例 ch5t_stride*。最终输出的 priority 和 exitcode 应该大致成正比，由于我们的时间片比较粗糙，qemu 的模拟也不是十分准确，我们最终的 CI 测试会允许最大 30% 的误差。

#### 实现 tips:

- 你应该给 proc 结构体加入新的字段来支持优先级。
- 我们的测例运行时间不很长，不要求处理 stride 的溢出（详见问答作业，当然处理了更好）。
- 为了减少整数除的误差，BIG_STRIDE 一般需要很大，但测例中的优先级都是 2 的整数次幂，结合第二点，BIG_STRIDE不需要太大，65536 是一个不错的数字。
- 用户态的 printf 支持了行缓冲，所以如果你想要增加用户程序的输出，记得换行。
- stride 算法要找到　stride 最小的进程，使用优先级队列是效率不错的办法，但是我们的实验测例很简单，所以效率完全不是问题。事实上，我很推荐使用暴力扫一遍的办法找最小值。
- 注意设置进程的初始优先级。

## 编程作业答案

### 迁移之前的 syscall

简单地 cherry-pick 过来，解决一下冲突就行了，记得不要留下任何 `sys_trace` 的相关代码。

```patch
From 65f6316dbf37e297d56bdeb84775164a5e6f7aef Mon Sep 17 00:00:00 2001
From: 0wnerDied <z1281552865@gmail.com>
Date: Fri, 25 Jul 2025 01:27:26 +0800
Subject: [PATCH 1/7] chapter4 practice

Signed-off-by: 0wnerDied <z1281552865@gmail.com>
---
 os/syscall.c | 118 ++++++++++++++++++++++++++++++++++++++++++++++++++-
 os/vm.h      |   1 +
 2 files changed, 117 insertions(+), 2 deletions(-)

diff --git a/os/syscall.c b/os/syscall.c
index fa22920..9cbcc81 100644
--- a/os/syscall.c
+++ b/os/syscall.c
@@ -48,6 +48,7 @@ uint64 sys_sched_yield()
 	return 0;
 }
 
+#if 0
 uint64 sys_gettimeofday(uint64 val, int _tz)
 {
 	struct proc *p = curr_proc();
@@ -58,6 +59,23 @@ uint64 sys_gettimeofday(uint64 val, int _tz)
 	copyout(p->pagetable, val, (char *)&t, sizeof(TimeVal));
 	return 0;
 }
+#else
+uint64 sys_gettimeofday(uint64 val_va, int _tz)
+{
+	struct proc *p = curr_proc();
+	TimeVal val;
+	uint64 cycle = get_cycle();
+
+	val.sec = cycle / CPU_FREQ;
+	val.usec = (cycle % CPU_FREQ) * 1000000 / CPU_FREQ;
+
+	if (copyout(p->pagetable, val_va, (char *)&val,
+			sizeof(TimeVal)) < 0)
+		return -1;
+
+	return 0;
+}
+#endif
 
 uint64 sys_getpid()
 {
@@ -114,6 +132,96 @@ uint64 sys_sbrk(int n)
         return addr;
 }
 
+int sys_mmap(uint64 start, uint64 len, int prot, int flags)
+{
+	struct proc *p = curr_proc();
+
+	if (start % PGSIZE != 0)
+		return -1;
+
+	if ((prot & ~0x7) != 0)
+		return -1;
+	if ((prot & 0x7) == 0)
+		return -1;
+
+	if (len == 0)
+		return 0;
+
+	uint64 end = PGROUNDUP(start + len);
+	for (uint64 va = start; va < end; va += PGSIZE) {
+		pte_t *pte = walk(p->pagetable, va, 0);
+		if (pte != 0 && (*pte & PTE_V))
+			return -1;
+	}
+#if 0
+	int xperm = 0;
+
+	if (prot & 0x2)
+		xperm |= PTE_W;
+	if (prot & 0x4)
+		xperm |= PTE_X;
+
+	if (uvmalloc(p->pagetable, start, end, xperm) == 0)
+		return -1;
+#else // #if 0
+	/*
+	 * Manual page allocation and mapping loop.
+	 * Unlike uvmalloc(), this gives us precise control
+	 * over permission bits to match the exact prot
+	 * flags specified by the user. We allocate physical
+	 * pages one by one, zero them for security,
+	 * set only the requested permission bits (crucial
+	 * for RISC-V compliance where PTE_W=1,PTE_R=0 is
+	 * illegal), and install the mapping via mappages().
+	 */
+	for (uint64 va = start; va < end; va += PGSIZE) {
+		char *pa = kalloc();
+		int pte_flags = PTE_V | PTE_U;
+
+		if (!pa)
+			return -1;
+
+		memset(pa, 0, PGSIZE);
+
+		if (prot & 0x1)
+			pte_flags |= PTE_R;
+		if (prot & 0x2)
+			pte_flags |= PTE_W;
+		if (prot & 0x4)
+			pte_flags |= PTE_X;
+
+		if (mappages(p->pagetable, va, PGSIZE, (uint64)pa, pte_flags) != 0) {
+			kfree(pa);
+			return -1;
+		}
+	}
+#endif // #if 0
+
+	return 0;
+}
+
+int sys_munmap(uint64 start, uint64 len)
+{
+	struct proc *p = curr_proc();
+
+	if (start % PGSIZE != 0)
+		return -1;
+
+	if (len == 0)
+		return 0;
+
+	uint64 end = PGROUNDUP(start + len);
+	for (uint64 va = start; va < end; va += PGSIZE) {
+		pte_t *pte = walk(p->pagetable, va, 0);
+		if (pte == 0 || (*pte & PTE_V) == 0)
+			return -1;
+	}
+
+	uvmunmap(p->pagetable, start, (end - start) / PGSIZE, 1);
+
+	return 0;
+}
+
 extern char trap_page[];
 
 void syscall()
@@ -159,8 +267,14 @@ void syscall()
 		ret = sys_spawn(args[0]);
 		break;
 	case SYS_sbrk:
-                ret = sys_sbrk(args[0]);
-                break;
+		ret = sys_sbrk(args[0]);
+		break;
+	case SYS_mmap:
+		ret = sys_mmap(args[0], args[1], args[2], args[3]);
+		break;
+	case SYS_munmap:
+		ret = sys_munmap(args[0], args[1]);
+		break;
 	default:
 		ret = -1;
 		errorf("unknown syscall %d", id);
diff --git a/os/vm.h b/os/vm.h
index 8618f5c..9975f6d 100644
--- a/os/vm.h
+++ b/os/vm.h
@@ -11,6 +11,7 @@ pagetable_t uvmcreate(uint64);
 int uvmcopy(pagetable_t, pagetable_t, uint64);
 void uvmfree(pagetable_t, uint64);
 void uvmunmap(pagetable_t, uint64, uint64, int);
+pte_t *walk(pagetable_t, uint64, int);
 uint64 walkaddr(pagetable_t, uint64);
 uint64 useraddr(pagetable_t, uint64);
 int copyout(pagetable_t, uint64, char *, uint64);
-- 
2.50.1

```

简单 `make test BASE=2` 然后输入 `ch5_mergetest` 运行一下，发现运行不了，运行到 mmap 的测例就会显示 `[PANIC 122] os/vm.c:198: freewalk: leaf`，看了一下应该是 max_page 的原因，于是进行以下修改：

```patch
From 838f072a0424c4dae3753136c05a91e57f28fc9e Mon Sep 17 00:00:00 2001
From: 0wnerDied <z1281552865@gmail.com>
Date: Fri, 1 Aug 2025 03:14:17 +0800
Subject: [PATCH 2/7] adapt mmap/munmap to ch5

Signed-off-by: 0wnerDied <z1281552865@gmail.com>
---
 os/syscall.c | 14 ++++++++++++++
 1 file changed, 14 insertions(+)

diff --git a/os/syscall.c b/os/syscall.c
index 9cbcc81..d0cf211 100644
--- a/os/syscall.c
+++ b/os/syscall.c
@@ -197,6 +197,10 @@ int sys_mmap(uint64 start, uint64 len, int prot, int flags)
 	}
 #endif // #if 0
 
+	uint64 end_page = end / PGSIZE;
+	if (end_page > p->max_page)
+		p->max_page = end_page;
+
 	return 0;
 }
 
@@ -219,6 +223,16 @@ int sys_munmap(uint64 start, uint64 len)
 
 	uvmunmap(p->pagetable, start, (end - start) / PGSIZE, 1);
 
+	for (uint64 pg = p->max_page; pg > 0; pg--) {
+		uint64 va  = (pg - 1) * PGSIZE;
+		pte_t *pte = walk(p->pagetable, va, 0);
+		if (pte && (*pte & PTE_V)) {
+			p->max_page = pg;
+			return 0;
+		}
+	}
+	p->max_page = 0;
+
 	return 0;
 }
 
-- 
2.50.1

```

改完之后再次运行 `make test BASE=2` 然后输入 `ch5_mergetest`，可以看到最后 `ch5 Mergetests passed!`，测试通过。

### 进程创建

spawn 的实现比较简单，看看 fork 和 execve 的逻辑就行了：

```patch
From 63314392ebafe3f203477c684709d8b611baef47 Mon Sep 17 00:00:00 2001
From: 0wnerDied <z1281552865@gmail.com>
Date: Fri, 1 Aug 2025 18:46:29 +0800
Subject: [PATCH 3/7] implement spawn syscall

This change brings a new `spawn` system call to provide an
efficient way to create a new process and execute a program
in a single operation.

This new system call is designed as a more performant alternative to
the traditional fork-then-exec model. It avoids the overhead of copying
the parent process's entire address space, which is immediately discarded
by exec. Instead, `spawn` creates a new, empty process and directly loads
the specified application into it.

Key implementation details:
- A new system call, `sys_spawn`, is added. It takes a user-space
  pointer to a filename as an argument.
- The filename is safely copied into the kernel using `copyinstr` to
  prevent security vulnerabilities.
- A new process is created by calling `allocproc`, which sets up a
  clean process state, including a PCB and a kernel stack.
- The application's binary is loaded into the new process's address
  space using the existing `loader` function.
- The parent-child relationship is established to maintain the process
  tree, which is essential for the `wait` system call.
- Robust error handling is implemented. If any step fails (e.g., invalid
  filename, process allocation failure, or loader error), allocated
  resources are cleaned up, and an error code (-1) is returned to the
  calling process.
- Expose the function `freeproc` from proc.h.

Signed-off-by: 0wnerDied <z1281552865@gmail.com>
---
 os/proc.h    |  1 +
 os/syscall.c | 42 ++++++++++++++++++++++++++++++++++++++++--
 2 files changed, 41 insertions(+), 2 deletions(-)

diff --git a/os/proc.h b/os/proc.h
index aabc2e8..85a527c 100644
--- a/os/proc.h
+++ b/os/proc.h
@@ -62,6 +62,7 @@ int wait(int, int *);
 void add_task(struct proc *);
 struct proc *pop_task();
 struct proc *allocproc();
+void freeproc(struct proc *p);
 int fdalloc(struct file *);
 // swtch.S
 void swtch(struct context *, struct context *);
diff --git a/os/syscall.c b/os/syscall.c
index d0cf211..b3a354a 100644
--- a/os/syscall.c
+++ b/os/syscall.c
@@ -110,10 +110,48 @@ uint64 sys_wait(int pid, uint64 va)
 	return wait(pid, code);
 }
 
+/*
+ * sys_spawn - Create a new process and execute a program.
+ * @va: User-space virtual address of the filename to execute.
+ *
+ * Creates a new process in a single step, avoiding the overhead of
+ * fork-then-exec. This is more efficient as it does not copy the
+ * parent's address space.
+ *
+ * Returns:
+ * The new process's PID on success.
+ * -1 on error (e.g., invalid filename, out of memory, or process limit reached).
+ */
 uint64 sys_spawn(uint64 va)
 {
-	// TODO: your job is to complete the sys call
-	return -1;
+	struct proc *p = curr_proc();
+	char name[MAX_STR_LEN];
+	struct proc *np;
+	int id;
+
+	/*
+	 * Copy the filename from user space to kernel space.
+	 */
+	if (copyinstr(p->pagetable, name, va, MAX_STR_LEN) < 0)
+		return -1;
+
+	id = get_id_by_name(name);
+	if (id < 0)
+		return -1;
+
+	if ((np = allocproc()) == 0)
+		return -1;
+
+	np->parent = p;
+
+	if (loader(id, np) < 0) {
+		freeproc(np);
+		return -1;
+	}
+
+	add_task(np);
+
+	return np->pid;
 }
 
 uint64 sys_set_priority(long long prio){
-- 
2.50.1

```

`make test BASE=2` 然后执行 `ch5_usertest` 进行测试，中间会有几个 `[ERROR 156]unknown syscall 140`，这不要紧，是因为我们还没实现优先级设置的系统调用导致的，下面实现完 `sys_set_priority` 后就不会再有这些错误了，现在只要看到最后的

```
ch5t usertest passed!
ch5 Usertests passed!
```

就证明我们的 spawn 系统调用完成了。

### stride 调度算法

这算是这个实验最复杂的一部分了吧，首先最简单的引入 stride 调度算法：

```patch
From 707d366e7a892d0b75e4dfc9e276ea13734496e6 Mon Sep 17 00:00:00 2001
From: 0wnerDied <z1281552865@gmail.com>
Date: Fri, 1 Aug 2025 22:17:09 +0800
Subject: [PATCH 4/7] Implement stride scheduling algorithm

This commit replaces the previous simple round-robin scheduler with the stride scheduling algo
to enable priority-based process scheduling. The new scheduler ensures that processes receive CPU time
proportional to their assigned priority, leading to a more fair and controllable execution environment.

Key changes include modifications to the process structure in proc.h, where stride and priority
fields have been added to struct proc. The main scheduler logic in proc.c is updated to select the process
with the minimum stride for execution and then increments its stride based on its priority.

New processes are initialized with a default priority of 16 and a stride of 0 in allocproc().
Child processes created via fork() now inherit the priority and stride from their parent.

Additionally, a new system call, sys_set_priority, is introduced in syscall.c. This allows a
process to set its own priority, provided it is 2 or greater.

Signed-off-by: 0wnerDied <z1281552865@gmail.com>
---
 os/const.h   |  4 ++++
 os/proc.c    | 43 ++++++++++++++++++++++++++++++++++++++++++-
 os/proc.h    |  3 +++
 os/syscall.c | 14 ++++++++++++--
 4 files changed, 61 insertions(+), 3 deletions(-)

diff --git a/os/const.h b/os/const.h
index a178e6c..2cce207 100644
--- a/os/const.h
+++ b/os/const.h
@@ -35,4 +35,8 @@ enum {
 #define MAX_STR_LEN (200)
 #define IDLE_PID (0)
 
+// scheduler algo
+#define BIG_STRIDE 65536
+#define DEFAULT_PRIO 16
+
 #endif // CONST_H
\ No newline at end of file
diff --git a/os/proc.c b/os/proc.c
index ce45bff..c59ef01 100644
--- a/os/proc.c
+++ b/os/proc.c
@@ -85,7 +85,10 @@ found:
 	p->exit_code = 0;
 	p->pagetable = uvmcreate((uint64)p->trapframe);
 	p->program_brk = 0;
-        p->heap_bottom = 0;
+	p->heap_bottom = 0;
+	p->stride = 0;
+	p->priority = DEFAULT_PRIO;
+	p->pass = BIG_STRIDE / p->priority;
 	memset(&p->context, 0, sizeof(p->context));
 	memset((void *)p->kstack, 0, KSTACK_SIZE);
 	memset((void *)p->trapframe, 0, TRAP_PAGE_SIZE);
@@ -99,6 +102,7 @@ found:
 //  - swtch to start running that process.
 //  - eventually that process transfers control
 //    via swtch back to the scheduler.
+#if 0
 void scheduler()
 {
 	struct proc *p;
@@ -126,6 +130,40 @@ void scheduler()
 		swtch(&idle.context, &p->context);
 	}
 }
+#else
+void scheduler()
+{
+	struct proc *p;
+	struct proc *nxt_proc;
+
+	for (;;) {
+		/*
+		 * Find the process with the smallest stride
+		 */
+		nxt_proc = 0;
+		for (p = pool; p < &pool[NPROC]; p++) {
+			if (p->state == RUNNABLE) {
+				if (nxt_proc == 0 || p->stride < nxt_proc->stride)
+					nxt_proc = p;
+			}
+		}
+
+		if (nxt_proc) {
+			p = nxt_proc;
+			p->state = RUNNING;
+			current_proc = p;
+
+			if (p->priority > 0)
+				p->stride += p->pass;
+
+			swtch(&idle.context, &p->context);
+			current_proc = &idle;
+		} else
+			// No runnable processes, wait for interrupt
+			asm volatile("wfi");
+	}
+}
+#endif
 
 // Switch to scheduler.  Must hold only p->lock
 // and have changed proc->state. Saves and restores
@@ -185,6 +223,9 @@ int fork()
 	// Cause fork to return 0 in the child.
 	np->trapframe->a0 = 0;
 	np->parent = p;
+	np->stride = p->stride;
+	np->priority = p->priority;
+	np->pass = p->pass;
 	np->state = RUNNABLE;
 	add_task(np);
 	return np->pid;
diff --git a/os/proc.h b/os/proc.h
index 85a527c..d7bb108 100644
--- a/os/proc.h
+++ b/os/proc.h
@@ -47,6 +47,9 @@ struct proc {
 	struct file *files[FD_BUFFER_SIZE];
 	uint64 program_brk;
 	uint64 heap_bottom;
+	uint64 stride;
+	int priority;
+	uint64 pass;
 };
 
 int cpuid();
diff --git a/os/syscall.c b/os/syscall.c
index b3a354a..2611e9c 100644
--- a/os/syscall.c
+++ b/os/syscall.c
@@ -155,8 +155,15 @@ uint64 sys_spawn(uint64 va)
 }
 
 uint64 sys_set_priority(long long prio){
-    // TODO: your job is to complete the sys call
-    return -1;
+	if (prio < 2)
+		return -1;
+
+	struct proc *p = curr_proc();
+
+	p->priority = prio;
+	p->pass = BIG_STRIDE / p->priority;
+
+	return prio;
 }
 
 
@@ -318,6 +325,9 @@ void syscall()
 	case SYS_spawn:
 		ret = sys_spawn(args[0]);
 		break;
+	case SYS_setpriority:
+		ret = sys_set_priority(args[0]);
+		break;
 	case SYS_sbrk:
 		ret = sys_sbrk(args[0]);
 		break;
-- 
2.50.1

```

开始我以为写得没问题了，结果跑了一下 `ch5_usertest` 显示 `[PANIC 5] os/queue.c:13: queue shouldn't be overflow`。看了一下代码，是因为和原来简单的 RR 调度冲突了，引入新的调度算法后，原来依靠队列 FIFO 的调度就不应该再被使用了，于是就有了以下更改：

```patch
From 556473656e043aa5dc748f632ed75a9c4a3d5aaf Mon Sep 17 00:00:00 2001
From: 0wnerDied <z1281552865@gmail.com>
Date: Fri, 1 Aug 2025 22:37:05 +0800
Subject: [PATCH 5/7] refactor scheduler from obsolete task queue

This change resolves a panic that occurred after implementing the stride scheduler.

The root cause of the issue was a design mismatch between the new scheduling logic
and legacy code. The Stride scheduler selects the next process by directly iterating
through the global process pool, making the task_queue obsolete. However, functions
like yield(), fork(), and etc were still calling add_task(), which continued to
enqueue processes into a queue that was never dequeued from, inevitably leading to an overflow.

Signed-off-by: 0wnerDied <z1281552865@gmail.com>
---
 os/loader.c  |  2 +-
 os/proc.c    | 23 +++++++++++++++--------
 os/proc.h    |  1 -
 os/queue.c   |  2 ++
 os/syscall.c |  2 +-
 5 files changed, 19 insertions(+), 11 deletions(-)

diff --git a/os/loader.c b/os/loader.c
index 77be56b..9e3fad4 100644
--- a/os/loader.c
+++ b/os/loader.c
@@ -99,6 +99,6 @@ int load_init_app()
 	}
 	debugf("load init proc %s", INIT_PROC);
 	loader(id, p);
-	add_task(p);
+	// add_task(p);
 	return 0;
 }
diff --git a/os/proc.c b/os/proc.c
index c59ef01..a12ffb8 100644
--- a/os/proc.c
+++ b/os/proc.c
@@ -3,7 +3,7 @@
 #include "loader.h"
 #include "trap.h"
 #include "vm.h"
-#include "queue.h"
+//#include "queue.h"
 
 struct proc pool[NPROC];
 __attribute__((aligned(16))) char kstack[NPROC][PAGE_SIZE];
@@ -12,7 +12,7 @@ __attribute__((aligned(4096))) char trapframe[NPROC][TRAP_PAGE_SIZE];
 extern char boot_stack_top[];
 struct proc *current_proc;
 struct proc idle;
-struct queue task_queue;
+//struct queue task_queue;
 
 int threadid()
 {
@@ -36,7 +36,7 @@ void proc_init()
 	idle.kstack = (uint64)boot_stack_top;
 	idle.pid = IDLE_PID;
 	current_proc = &idle;
-	init_queue(&task_queue);
+	// init_queue(&task_queue);
 }
 
 int allocpid()
@@ -45,6 +45,12 @@ int allocpid()
 	return PID++;
 }
 
+/*
+ * The fetch_task and add_task function is no longer
+ * needed with the stride scheduler, as the scheduler
+ * now iterates through the process pool directly.
+ */
+#if 0
 struct proc *fetch_task()
 {
 	int index = pop_queue(&task_queue);
@@ -58,9 +64,10 @@ struct proc *fetch_task()
 
 void add_task(struct proc *p)
 {
-	push_queue(&task_queue, p - pool);
-	debugf("add task %d(pid=%d) to task queue\n", p - pool, p->pid);
+	// push_queue(&task_queue, p - pool);
+	// debugf("add task %d(pid=%d) to task queue\n", p - pool, p->pid);
 }
+#endif
 
 // Look in the process table for an UNUSED proc.
 // If found, initialize state required to run in the kernel.
@@ -184,7 +191,7 @@ void sched()
 void yield()
 {
 	current_proc->state = RUNNABLE;
-	add_task(current_proc);
+	// add_task(current_proc);
 	sched();
 }
 
@@ -227,7 +234,7 @@ int fork()
 	np->priority = p->priority;
 	np->pass = p->pass;
 	np->state = RUNNABLE;
-	add_task(np);
+	// add_task(np);
 	return np->pid;
 }
 
@@ -269,7 +276,7 @@ int wait(int pid, int *code)
 			return -1;
 		}
 		p->state = RUNNABLE;
-		add_task(p);
+		// add_task(p);
 		sched();
 	}
 }
diff --git a/os/proc.h b/os/proc.h
index d7bb108..e32d656 100644
--- a/os/proc.h
+++ b/os/proc.h
@@ -63,7 +63,6 @@ int fork();
 int exec(char *);
 int wait(int, int *);
 void add_task(struct proc *);
-struct proc *pop_task();
 struct proc *allocproc();
 void freeproc(struct proc *p);
 int fdalloc(struct file *);
diff --git a/os/queue.c b/os/queue.c
index 5f04063..60d2860 100644
--- a/os/queue.c
+++ b/os/queue.c
@@ -1,3 +1,4 @@
+#if 0
 #include "queue.h"
 #include "defs.h"
 
@@ -27,3 +28,4 @@ int pop_queue(struct queue *q)
 		q->empty = 1;
 	return value;
 }
+#endif
diff --git a/os/syscall.c b/os/syscall.c
index 2611e9c..6dc9829 100644
--- a/os/syscall.c
+++ b/os/syscall.c
@@ -149,7 +149,7 @@ uint64 sys_spawn(uint64 va)
 		return -1;
 	}
 
-	add_task(np);
+	// add_task(np);
 
 	return np->pid;
 }
-- 
2.50.1

```

这样修完就没问题了。不过我们还没处理可能的 stride 的溢出，虽然实验也不要求但是我觉得还是处理一下更好：

```patch
From 529b7de53ab3dd0ebdc1de3e388797f97540f933 Mon Sep 17 00:00:00 2001
From: 0wnerDied <z1281552865@gmail.com>
Date: Fri, 1 Aug 2025 23:17:25 +0800
Subject: [PATCH 6/7] Prevent stride overflow in scheduler

This commit addresses a potential long-term stability issue in the stride scheduling algo
where the stride value could overflow its uint64 type. Such an overflow would wrap around
and disrupt the scheduler's fairness.

To resolve this, the scheduler logic in proc.c has been updated to normalize stride values
during each scheduling cycle. After identifying the runnable process with the minimum
stride, this minimum value is subtracted from the strides of all other runnable processes.

This operation preserves the relative differences between strides, ensuring the scheduling
order remains correct, while pulling all stride values down to a smaller range and preventing
them from growing indefinitely. This change makes the scheduler robust against overflow without
altering its proportional-share behavior, ensuring system stability over long periods of operation.

Signed-off-by: 0wnerDied <z1281552865@gmail.com>
---
 os/proc.c | 15 +++++++++++++++
 1 file changed, 15 insertions(+)

diff --git a/os/proc.c b/os/proc.c
index a12ffb8..78429cb 100644
--- a/os/proc.c
+++ b/os/proc.c
@@ -156,6 +156,21 @@ void scheduler()
 		}
 
 		if (nxt_proc) {
+			/*
+			 * To prevent stride overflow, subtract the minimum
+			 * stride from all runnable processes. This keeps the
+			 * relative order and prevents the stride values from
+			 * growing indefinitely.
+			 */
+			uint64 min_stride = nxt_proc->stride;
+			if (min_stride > 0) {
+				for (p = pool; p < &pool[NPROC]; p++) {
+					if (p->state == RUNNABLE) {
+						p->stride -= min_stride;
+					}
+				}
+			}
+
 			p = nxt_proc;
 			p->state = RUNNING;
 			current_proc = p;
-- 
2.50.1

```

做了上面的更改之后，我觉得真的是非常完美，一点问题都没有了。结果怎么着，运行完测例后输入 `quit` 想退出 usershell，结果没反应了？？左思右想想不通问题出在哪儿，又回去翻了一下原本的 `scheduler()` 函数实现，发现了问题。我在引入 stride 调度算法的提交中，如果没有 `nxt_proc`，会进入 wfi 状态，而在原本的实现中，如果没有新任务，会执行 `panic("all app are over!\n");`，这就是我输入 `quit` 后无法退出没有反应的问题所在，一直等待中断可不就没反应了吗？于是就有了下面最后这个修复提交：

```patch
From f6e510a5fe5e6fe3a8f40e9a883b12d38c5055d7 Mon Sep 17 00:00:00 2001
From: 0wnerDied <z1281552865@gmail.com>
Date: Sat, 2 Aug 2025 00:44:16 +0800
Subject: [PATCH 7/7] use panic() instead of inline assembly wfi

lol, if there's no existing process, we shouldn't wait for interrupt but
to panic when input is quit.

Signed-off-by: 0wnerDied <z1281552865@gmail.com>
---
 os/proc.c | 3 +--
 1 file changed, 1 insertion(+), 2 deletions(-)

diff --git a/os/proc.c b/os/proc.c
index 78429cb..2e18321 100644
--- a/os/proc.c
+++ b/os/proc.c
@@ -181,8 +181,7 @@ void scheduler()
 			swtch(&idle.context, &p->context);
 			current_proc = &idle;
 		} else
-			// No runnable processes, wait for interrupt
-			asm volatile("wfi");
+			panic("all app are over!\n");
 	}
 }
 #endif
-- 
2.50.1

```

~~我再也不用内联汇编了😭😭~~

## 问答作业

### stride 算法深入

*stride 算法原理非常简单，但是有一个比较大的问题。例如两个 pass = 10 的进程，使用 8bit 无符号整形储存 stride，p1.stride = 255，p2.stride = 250，在 p2 执行一个时间片后，理论上下一次应该 p1 执行。*

- *实际情况是轮到 p1 执行吗？为什么？*

*我们之前要求进程优先级 >= 2 其实就是为了解决这个问题。可以证明，__在不考虑溢出的情况下__, 在进程优先级全部 >= 2 的情况下，如果严格按照算法执行，那么 STRIDE_MAX – STRIDE_MIN <= BigStride / 2。*

- *为什么？尝试简单说明（传达思想即可，不要求严格证明）。*

*已知以上结论，__在考虑溢出的情况下__，假设我们通过逐个比较得到 Stride 最小的进程，请设计一个合适的比较函数，用来正确比较两个 Stride 的真正大小：*

```c
typedef unsigned long long Stride_t;
const Stride_t BIG_STRIDE = 0xffffffffffffffffULL;
int cmp(Stride_t a, Stride_t b) {
    // YOUR CODE HERE
    // return 1 if a > b
    // return -1 if a < b
    // return 0 if a == b
}
```

*例子：假设使用 8 bits 储存 stride, BigStride = 255。那么：*

- *cmp(125, 255) == 1*
- *cmp(129, 255) == -1*

## 问答作业答案

### stride 算法深入

*stride 算法原理非常简单，但是有一个比较大的问题。例如两个 pass = 10 的进程，使用 8bit 无符号整形储存 stride，p1.stride = 255，p2.stride = 250，在 p2 执行一个时间片后，理论上下一次应该 p1 执行。*

- *实际情况是轮到 p1 执行吗？为什么？*

    不是，p1.stride = 255，p2.stride = 250。p2 执行后，p2.stride = (250 + 10) % 256 = 4。于是 p1.stride = 255, p2.stride = 4，4 < 255，所以会选择 p2 继续执行。

*我们之前要求进程优先级 >= 2 其实就是为了解决这个问题。可以证明，__在不考虑溢出的情况下__, 在进程优先级全部 >= 2 的情况下，如果严格按照算法执行，那么 STRIDE_MAX – STRIDE_MIN <= BigStride / 2。*

- *为什么？尝试简单说明（传达思想即可，不要求严格证明）。*

    当所有进程优先级 ≥ 2 时，最大的 stride 增量是 BigStride / 2，于是在最坏情况下，一个进程一直不执行，其他进程轮流执行。假设进程 A 的 stride 最小，进程 B 的 stride 最大，当 B 再次被选中执行时，A 和 B 的 stride 差值最多是 BigStride / 2，因为每次增量最多是 BigStride/2，所以差值不会超过这个限制。

*已知以上结论，__在考虑溢出的情况下__，假设我们通过逐个比较得到 Stride 最小的进程，请设计一个合适的比较函数，用来正确比较两个 Stride 的真正大小：*

```c
typedef unsigned long long Stride_t;
const Stride_t BIG_STRIDE = 0xffffffffffffffffULL;

int cmp(Stride_t a, Stride_t b) {
    if (a == b)
        return 0;
    
    Stride_t diff = a - b;

    if (diff <= BIG_STRIDE / 2)
        return 1;
    else
        return -1;
}
```

*例子：假设使用 8 bits 储存 stride, BigStride = 255。那么：*

- *cmp(125, 255) == 1*
- *cmp(129, 255) == -1*

## 选做题目

### 选作题目列表

- （6分）相同页面共享（Same page sharing）fork时的Copy on Write
- （4分）实现多种(>3种)调度算法：可动态提升/降低优先级的多级反馈队列、实时调度等
- （7分）多核支持与多核调度（支持进程迁移和多核模式执行应用程序，但在内核中没有抢占和多核支持）

这次一道选做题目都没有写，时间不够了，如果之后有时间再补吧。~~（虽然大概率是没有的~~
