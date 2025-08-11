---
title: uCore lab5 实验报告
categories:
  - uCore
tags:
  - OS
  - riscv
abbrlink: 8465
date: 2025-08-12 02:11:04
---

这次实验应该是最难的了，虽然不用考虑内核/用户态的切换，也不用移植前面实验的内容，但是光读一遍新增的锁就很累了。还要对一些算法有一定认识，后面再细讲。

## 本章任务

- 本次任务对应 lab5，也是本学期最后一次实验，祝你好运。
- 老规矩，先 `make test BASE=1` 看下啥情况。
- **理解框架的多线程机制，了解几种锁的运行原理。在此基础上，实现本章编程作业死锁检测。**
    - 如果时间有限，多线程机制的一些细节大可跳过，但至少应知道多线程基本原理和本章在任务调度粒度上的调整
    - 与实验息息相关的是互斥锁(mutex)与信号量(semaphore)，条件变量(condvar)供阅读
    - **框架包含 `LAB5` 字样的注释中给出了一个供参考的实现位置和顺序，你可以按顺序完成（下面的标号与注释中的一种）：**
        - 1: 定义并初始化部分 PCB 的部分变量，包括控制死锁检测启动与死锁检测算法用到的变量，你可以先定义一部分，后面发现有需要时再做添加；
        - 2: 完成系统调用 `sys_enable_deadlock_detect`，只需要修改变量，不必考虑是否正确实现了死锁。完成这一步后你可以顺利跑完 `ch8_sem2_deadlock`，这个测例开启了死锁检测但并没有死锁；
        - 3: 尝试写一个函数实现下面提到的死锁检测算法，注释中给了供参考的函数签名。这是一个和 OS 独立的函数，你可以自行设计数据单独运行它以测试；
        - 4-1: 维护 mutex 相关的死锁检测变量，并调用死锁检测算法，完成后你可以顺利跑完测例 `ch8_mut1_deadlock`；
        - 4-2: 维护 semaphore 相关的死锁检测变量，并调用死锁检测算法，完成后你可以顺利跑完测例 `ch8_sem1_deadlock`；
- 最终，完成实验报告并 push 你的 ch8 分支到远程仓库。push 代码后会自动执行 CI，代码给分以 CI 给分为准。

## 编程作业

<details>
<summary>⚠警告</summary>

本次实验框架变动较大，且改动较为复杂，为降低同学们的工作量，本次实验不要求合并之前的实验内容，可以直接 checkout 到 ch8 框架开始实验，最终只需通过 ch8 系列的测例和前面章节的基础测例即可。

</details>

<details>
<summary>🖊注解</summary>

本次实验实现死锁检测算法本身只需要40行左右代码，但加上系统调用实现、变量声明与初始化、以及在锁的创建、锁、释放时维护死锁检测 Available、Allocation、Request 数组，总代码量预计在 100 行左右。助教的参考实现约为 90 行。

</details>

### 死锁检测

目前的 mutex 和 semaphore 相关的系统调用不会分析资源的依赖情况，用户程序可能出现死锁。 我们希望在系统中加入死锁检测机制，当发现可能发生死锁时拒绝对应的资源获取请求。一种检测死锁的算法如下：

定义如下三个数据结构：

- 可利用资源向量 Available：含有 m 个元素的一维数组，每个元素代表可利用的某一类资源的数目，其初值是该类资源的全部可用数目，其值随该类资源的分配和回收而动态地改变。Available[j] = k，表示第 j 类资源的可用数量为 k。
- 分配矩阵 Allocation：n * m 矩阵，表示每类资源已分配给每个线程的资源数。Allocation[i, j] = g，则表示线程 i 当前己分得第 j 类资源的数量为 g。
- 需求矩阵 Request：n * m 的矩阵，表示每个线程还需要的各类资源数量。Request[i, j] = d，则表示线程 i 还需要第 j 类资源的数量为 d。

算法运行过程如下：

1. 设置两个向量: 工作向量 Work，表示操作系统可提供给线程继续运行所需的各类资源数目，它含有 m 个元素。初始时，Work = Available；结束向量 Finish，表示系统是否有足够的资源分配给线程，使之运行完成。初始时 Finish[0 ~ n - 1] = false，表示所有线程都没结束；当有足够资源分配给线程时，设置 Finish[i] = true。
2. 从线程集合中找到一个能满足下述条件的线程 i

> 1) Finish[i] == false;
> 2) Request[i, 0 ~ n - 1] ≤ Work[0 ~ n - 1];

若找到，执行步骤 3，否则执行步骤 4。

3. 当线程 i 获得资源后，可顺利执行，直至完成，并释放出分配给它的资源，故应执行:

> 1) Work[0 ~ n - 1] = Work[0 ~ n - 1] + Allocation[i, 0 ~ n - 1];
> 2) Finish[i] = true;

跳转回步骤2

4. 如果 Finish[0 ~ n - 1] 都为 true，则表示系统处于安全状态；否则表示系统处于不安全状态，即出现死锁。

出于兼容性和灵活性考虑，我们允许进程按需开启或关闭死锁检测功能。为此我们将实现一个新的系统调用：`sys_enable_deadlock_detect`。

**enable_deadlock_detect：**

- syscall ID: 469
- 功能：为当前进程启用或禁用死锁检测功能。
- 接口：`int enable_deadlock_detect(int is_enable)`
- 参数：
    - `is_enable`: 为 1 表示启用死锁检测，0 表示禁用死锁检测。
- 说明：
    - 开启死锁检测功能后，`mutex_lock` 和 `semaphore_down` 如果检测到死锁，应拒绝相应操作并返回 `-0xDEAD` (十六进制值)。
    - 简便起见可对 mutex 和 semaphore 分别进行检测，无需考虑二者(以及 `waittid` 等)混合使用导致的死锁。
- 返回值：如果出现了错误则返回 -1，否则返回 0。
- 可能的错误
    - 参数不合法
    - 死锁检测开启失败

## 编程作业答案

就按实验要求的顺序一步一步实现吧，下面是我的更改：

```patch
From 4ff1db2543e1394afd4b5a862fbb552e7f08b7d3 Mon Sep 17 00:00:00 2001
From: 0wnerDied <z1281552865@gmail.com>
Date: Mon, 11 Aug 2025 23:12:01 +0800
Subject: [PATCH 1/6] proc: Initialize needed variables in PCB

Signed-off-by: 0wnerDied <z1281552865@gmail.com>
---
 os/proc.c | 7 +++++++
 os/proc.h | 7 +++++++
 2 files changed, 14 insertions(+)

diff --git a/os/proc.c b/os/proc.c
index 9baaeb3..c964b99 100644
--- a/os/proc.c
+++ b/os/proc.c
@@ -143,6 +143,13 @@ found:
 	p->next_semaphore_id = 0;
 	p->next_condvar_id = 0;
 	// LAB5: (1) you may initialize your new proc variables here
+	p->deadlock_detect_enabled = 0;
+	memset(p->mutex_avail, 0, sizeof(p->mutex_avail));
+	memset(p->mutex_alloc, 0, sizeof(p->mutex_alloc));
+	memset(p->mutex_req, 0, sizeof(p->mutex_req));
+	memset(p->sem_avail, 0, sizeof(p->sem_avail));
+	memset(p->sem_alloc, 0, sizeof(p->sem_alloc));
+	memset(p->sem_req, 0, sizeof(p->sem_req));
 	return p;
 }
 
diff --git a/os/proc.h b/os/proc.h
index 0398e36..ba43261 100644
--- a/os/proc.h
+++ b/os/proc.h
@@ -66,6 +66,13 @@ struct proc {
 	// LAB5: (1) Define your variables for deadlock detect here.
 	//			 You may need a flag to record if detection enabled,
 	//       and some arrays for detection algorithm.
+	int deadlock_detect_enabled;
+	int mutex_avail[LOCK_POOL_SIZE];
+	int mutex_alloc[NTHREAD][LOCK_POOL_SIZE];
+	int mutex_req[NTHREAD][LOCK_POOL_SIZE];
+	int sem_avail[LOCK_POOL_SIZE];
+	int sem_alloc[NTHREAD][LOCK_POOL_SIZE];
+	int sem_req[NTHREAD][LOCK_POOL_SIZE];
 };
 
 int cpuid();
-- 
2.25.1

```

```patch
From e5dfeb92213efd43fba0830e0f75175c55c33d83 Mon Sep 17 00:00:00 2001
From: 0wnerDied <z1281552865@gmail.com>
Date: Mon, 11 Aug 2025 23:17:04 +0800
Subject: [PATCH 2/6] Implement enable_deadlock_detect syscall

Signed-off-by: 0wnerDied <z1281552865@gmail.com>
---
 os/syscall.c | 14 ++++++++++++++
 1 file changed, 14 insertions(+)

diff --git a/os/syscall.c b/os/syscall.c
index 84eab14..05c919b 100644
--- a/os/syscall.c
+++ b/os/syscall.c
@@ -362,6 +362,17 @@ int sys_condvar_wait(int cond_id, int mutex_id)
 }
 
 // LAB5: (2) you may need to define function enable_deadlock_detect here
+int sys_enable_deadlock_detect(int is_enable)
+{
+	if (is_enable != 0 && is_enable != 1)
+		return -1;
+
+	struct proc *p = curr_proc();
+
+	p->deadlock_detect_enabled = is_enable;
+
+	return 0;
+}
 
 extern char trap_page[];
 
@@ -455,6 +466,9 @@ void syscall()
 		ret = sys_condvar_wait(args[0], args[1]);
 		break;
 	// LAB5: (2) you may need to add case SYS_enable_deadlock_detect here
+	case SYS_enable_deadlock_detect:
+		ret = sys_enable_deadlock_detect(args[0]);
+		break;
 	default:
 		ret = -1;
 		errorf("unknown syscall %d", id);
-- 
2.25.1

```

```patch
From 448844a705e085d827ef5ece51ef0c64487e6501 Mon Sep 17 00:00:00 2001
From: 0wnerDied <z1281552865@gmail.com>
Date: Mon, 11 Aug 2025 23:46:13 +0800
Subject: [PATCH 3/6] Implement deadlock detect algo

Signed-off-by: 0wnerDied <z1281552865@gmail.com>
---
 os/syscall.c | 53 ++++++++++++++++++++++++++++++++++++++++++++++++++++
 1 file changed, 53 insertions(+)

diff --git a/os/syscall.c b/os/syscall.c
index 05c919b..03610ba 100644
--- a/os/syscall.c
+++ b/os/syscall.c
@@ -248,7 +248,60 @@ int sys_waittid(int tid)
 *						const int request[NTHREAD][LOCK_POOL_SIZE])
 *				for both mutex and semaphore detect, you can also
 *				use this idea or just ignore it.
+*	@thread_cnt:	The total number of threads in the system.
+*	@resource_cnt:	The total number of resource types, this corresponds
+					to the number of currently created mutexes or
+					semaphores (e.g., p->next_mutex_id).
 */
+int deadlock_detect(int thread_cnt,
+		 int resource_cnt, int *avail,
+		 int alloc[NTHREAD][LOCK_POOL_SIZE],
+		 int req[NTHREAD][LOCK_POOL_SIZE])
+{
+	// Initialize Work and Finish arrays.
+	int work[LOCK_POOL_SIZE], finish[NTHREAD];
+
+	for (int j = 0; j < resource_cnt; j++)
+		work[j] = avail[j];
+	for (int i = 0; i < thread_cnt; i++)
+		finish[i] = 0;
+
+	// Find a thread that can finish and release its resources.
+	while (1) {
+		int found = 0;
+		for (int i = 0; i < thread_cnt; i++) {
+			// Find a thread [i] which is not finished and its request can be satisfied
+			if (finish[i] == 0) {
+				int unsatisfied = 0;
+				for (int j = 0; j < resource_cnt; j++) {
+					if (req[i][j] > work[j]) {
+						unsatisfied = 1;
+						break;
+					}
+				}
+				// If found, pretend to release its resources
+				if (!unsatisfied) {
+					for (int j = 0; j < resource_cnt; j++)
+						work[j] += alloc[i][j];
+					finish[i] = 1;
+					found = 1;
+				}
+			}
+		}
+		// If no such thread was found in a full pass, exit the loop
+		if (!found)
+			break;
+	}
+
+	// Check if all threads are finished.
+	for (int i = 0; i < thread_cnt; i++) {
+		if (finish[i] == 0)
+			// If any thread cannot finish, the system is in an unsafe state, detect deadlock.
+			return 1;
+	}
+
+	return 0;
+}
 
 int sys_mutex_create(int blocking)
 {
-- 
2.25.1

```

```patch
From fa4e86f787b90e4517d941cc74e3e3131ef7cf68 Mon Sep 17 00:00:00 2001
From: 0wnerDied <z1281552865@gmail.com>
Date: Tue, 12 Aug 2025 00:09:59 +0800
Subject: [PATCH 4/6] Maintain deadlock detector and variables for mutex

Signed-off-by: 0wnerDied <z1281552865@gmail.com>
---
 os/syscall.c | 55 +++++++++++++++++++++++++++++++++++++++++++++++++---
 1 file changed, 52 insertions(+), 3 deletions(-)

diff --git a/os/syscall.c b/os/syscall.c
index 03610ba..bb2791d 100644
--- a/os/syscall.c
+++ b/os/syscall.c
@@ -311,7 +311,14 @@ int sys_mutex_create(int blocking)
 		return -1;
 	}
 	// LAB5: (4-1) You may want to maintain some variables for detect here
-	int mutex_id = m - curr_proc()->mutex_pool;
+	struct proc *p = curr_proc();
+	int mutex_id = m - p->mutex_pool;
+
+	// If deadlock detection is enabled, initialize
+	// the state for this new resource.
+	if (p->deadlock_detect_enabled)
+		p->mutex_avail[mutex_id] = 1;
+
 	debugf("create mutex %d", mutex_id);
 	return mutex_id;
 }
@@ -324,7 +331,37 @@ int sys_mutex_lock(int mutex_id)
 	}
 	// LAB5: (4-1) You may want to maintain some variables for detect
 	//       or call your detect algorithm here
-	mutex_lock(&curr_proc()->mutex_pool[mutex_id]);
+	struct proc *p = curr_proc();
+	struct thread *t = curr_thread();
+	struct mutex *m = &p->mutex_pool[mutex_id];
+
+	if (p->deadlock_detect_enabled) {
+		p->mutex_req[t->tid][mutex_id] = 1;
+		// Check for potential deadlock ONLY if the resource is
+		// not immediately available. If the mutex is already
+		// locked, the thread will have to wait. This is a
+		// potential deadlock situation.
+		if (m->locked) {
+			if (deadlock_detect(NTHREAD, p->next_mutex_id,
+				 p->mutex_avail, p->mutex_alloc, p->mutex_req)) {
+				p->mutex_req[t->tid][mutex_id] = 0;
+				return -0xDEAD;
+			}
+		}
+	}
+
+	mutex_lock(m);
+
+	// Update state matrices after successfully acquiring the lock.
+	if (p->deadlock_detect_enabled) {
+		// The request has been fulfilled, clear it now.
+		p->mutex_req[t->tid][mutex_id] = 0;
+		// Now the resource is allocated to this thread.
+		p->mutex_alloc[t->tid][mutex_id] = 1;
+		// It's unavailable now.
+		p->mutex_avail[mutex_id] = 0;
+	}
+
 	return 0;
 }
 
@@ -335,7 +372,19 @@ int sys_mutex_unlock(int mutex_id)
 		return -1;
 	}
 	// LAB5: (4-1) You may want to maintain some variables for detect here
-	mutex_unlock(&curr_proc()->mutex_pool[mutex_id]);
+	struct proc *p = curr_proc();
+	struct thread *t = curr_thread();
+
+	// Update state matrices to reflect the resource release.
+	if (p->deadlock_detect_enabled) {
+		// No longer holds the resource. Clear it.
+		p->mutex_alloc[t->tid][mutex_id] = 0;
+		// Now it's available.
+		p->mutex_avail[mutex_id] = 1;
+	}
+
+	mutex_unlock(&p->mutex_pool[mutex_id]);
+
 	return 0;
 }
 
-- 
2.25.1

```

```patch
From ee4d9d1c584cb450a83256e53d17adeb3ebe2d05 Mon Sep 17 00:00:00 2001
From: 0wnerDied <z1281552865@gmail.com>
Date: Tue, 12 Aug 2025 00:31:44 +0800
Subject: [PATCH 5/6] Maintain deadlock detector and variables for sem

Signed-off-by: 0wnerDied <z1281552865@gmail.com>
---
 os/syscall.c | 53 +++++++++++++++++++++++++++++++++++++++++++++++++---
 1 file changed, 50 insertions(+), 3 deletions(-)

diff --git a/os/syscall.c b/os/syscall.c
index bb2791d..6149a98 100644
--- a/os/syscall.c
+++ b/os/syscall.c
@@ -396,7 +396,14 @@ int sys_semaphore_create(int res_count)
 		return -1;
 	}
 	// LAB5: (4-2) You may want to maintain some variables for detect here
-	int sem_id = s - curr_proc()->semaphore_pool;
+	struct proc *p = curr_proc();
+	int sem_id = s - p->semaphore_pool;
+
+	// If deadlock detection is enabled,
+	// initialize the state for this new resource.
+	if (p->deadlock_detect_enabled)
+		p->sem_avail[sem_id] = res_count;
+
 	debugf("create semaphore %d", sem_id);
 	return sem_id;
 }
@@ -409,7 +416,18 @@ int sys_semaphore_up(int semaphore_id)
 		return -1;
 	}
 	// LAB5: (4-2) You may want to maintain some variables for detect here
-	semaphore_up(&curr_proc()->semaphore_pool[semaphore_id]);
+	struct proc *p = curr_proc();
+	struct thread *t = curr_thread();
+
+	// Update state matrices to reflect V operation.
+	if (p->deadlock_detect_enabled) {
+		// To release a semaphore, decrease the thread's allocation of this resource.
+		p->sem_alloc[t->tid][semaphore_id]--;
+		// Consequently, the number of available instances of this resource increases.
+		p->sem_avail[semaphore_id]++;
+	}
+
+	semaphore_up(&p->semaphore_pool[semaphore_id]);
 	return 0;
 }
 
@@ -422,7 +440,36 @@ int sys_semaphore_down(int semaphore_id)
 	}
 	// LAB5: (4-2) You may want to maintain some variables for detect
 	//       or call your detect algorithm here
-	semaphore_down(&curr_proc()->semaphore_pool[semaphore_id]);
+	struct proc *p = curr_proc();
+	struct thread *t = curr_thread();
+	struct semaphore *s = &p->semaphore_pool[semaphore_id];
+
+	if (p->deadlock_detect_enabled) {
+		p->sem_req[t->tid][semaphore_id] = 1;
+		// Check for potential deadlock if the thread might wait.
+		// A thread will wait if the semaphore count is not positive.
+		if (s->count <= 0) {
+			if (deadlock_detect(NTHREAD, p->next_semaphore_id,
+					p->sem_avail, p->sem_alloc, p->sem_req)) {
+				// Cancel the request.
+				p->sem_req[t->tid][semaphore_id] = 0;
+				return -0xDEAD;
+			}
+		}
+	}
+
+	semaphore_down(s);
+
+	// Update state matrices after successfully acquiring the resource.
+	if (p->deadlock_detect_enabled) {
+		// The request has been fulfilled, clear it now.
+		p->sem_req[t->tid][semaphore_id] = 0;
+		// Increase the thread's allocation of this resource.
+		p->sem_alloc[t->tid][semaphore_id]++;
+		// Decrease the thread's available of this resource.
+		p->sem_avail[semaphore_id]--;
+	}
+
 	return 0;
 }
 
-- 
2.25.1

```

最后这个提交不是必需的，只是我看 `mutex_lock` 和 `semaphore_down` 里面的重复判断有点难受，所以简化了一下实现，~~感觉好看多了~~：

```patch
From 97ef7e592e8118b1a0f1ccf53d320f88d94d196d Mon Sep 17 00:00:00 2001
From: 0wnerDied <z1281552865@gmail.com>
Date: Tue, 12 Aug 2025 01:26:20 +0800
Subject: [PATCH 6/6] refactor mutex_lock and semaphore_down syscall

Signed-off-by: 0wnerDied <z1281552865@gmail.com>
---
 os/syscall.c | 90 ++++++++++++++++++++++++++++------------------------
 1 file changed, 49 insertions(+), 41 deletions(-)

diff --git a/os/syscall.c b/os/syscall.c
index 6149a98..c38d7c5 100644
--- a/os/syscall.c
+++ b/os/syscall.c
@@ -335,33 +335,37 @@ int sys_mutex_lock(int mutex_id)
 	struct thread *t = curr_thread();
 	struct mutex *m = &p->mutex_pool[mutex_id];
 
-	if (p->deadlock_detect_enabled) {
-		p->mutex_req[t->tid][mutex_id] = 1;
-		// Check for potential deadlock ONLY if the resource is
-		// not immediately available. If the mutex is already
-		// locked, the thread will have to wait. This is a
-		// potential deadlock situation.
-		if (m->locked) {
-			if (deadlock_detect(NTHREAD, p->next_mutex_id,
-				 p->mutex_avail, p->mutex_alloc, p->mutex_req)) {
-				p->mutex_req[t->tid][mutex_id] = 0;
-				return -0xDEAD;
-			}
+	if (!p->deadlock_detect_enabled)
+		goto acquire_lock;
+
+	p->mutex_req[t->tid][mutex_id] = 1;
+	// Check for potential deadlock ONLY if the resource is
+	// not immediately available. If the mutex is already
+	// locked, the thread will have to wait. This is a
+	// potential deadlock situation.
+	if (m->locked) {
+		if (deadlock_detect(NTHREAD, p->next_mutex_id,
+			 p->mutex_avail, p->mutex_alloc, p->mutex_req)) {
+			p->mutex_req[t->tid][mutex_id] = 0;
+			return -0xDEAD;
 		}
 	}
 
+acquire_lock:
 	mutex_lock(m);
 
-	// Update state matrices after successfully acquiring the lock.
-	if (p->deadlock_detect_enabled) {
-		// The request has been fulfilled, clear it now.
-		p->mutex_req[t->tid][mutex_id] = 0;
-		// Now the resource is allocated to this thread.
-		p->mutex_alloc[t->tid][mutex_id] = 1;
-		// It's unavailable now.
-		p->mutex_avail[mutex_id] = 0;
-	}
+	if (!p->deadlock_detect_enabled)
+		goto ret;
 
+	// Update state matrices after successfully acquiring the lock.
+	// The request has been fulfilled, clear it now.
+	p->mutex_req[t->tid][mutex_id] = 0;
+	// Now the resource is allocated to this thread.
+	p->mutex_alloc[t->tid][mutex_id] = 1;
+	// It's unavailable now.
+	p->mutex_avail[mutex_id] = 0;
+
+ret:
 	return 0;
 }
 
@@ -444,32 +448,36 @@ int sys_semaphore_down(int semaphore_id)
 	struct thread *t = curr_thread();
 	struct semaphore *s = &p->semaphore_pool[semaphore_id];
 
-	if (p->deadlock_detect_enabled) {
-		p->sem_req[t->tid][semaphore_id] = 1;
-		// Check for potential deadlock if the thread might wait.
-		// A thread will wait if the semaphore count is not positive.
-		if (s->count <= 0) {
-			if (deadlock_detect(NTHREAD, p->next_semaphore_id,
-					p->sem_avail, p->sem_alloc, p->sem_req)) {
-				// Cancel the request.
-				p->sem_req[t->tid][semaphore_id] = 0;
-				return -0xDEAD;
-			}
+	if (!p->deadlock_detect_enabled)
+		goto acquire_sem;
+
+	p->sem_req[t->tid][semaphore_id] = 1;
+	// Check for potential deadlock if the thread might wait.
+	// A thread will wait if the semaphore count is not positive.
+	if (s->count <= 0) {
+		if (deadlock_detect(NTHREAD, p->next_semaphore_id,
+			 p->sem_avail, p->sem_alloc, p->sem_req)) {
+			// Cancel the request.
+			p->sem_req[t->tid][semaphore_id] = 0;
+			return -0xDEAD;
 		}
 	}
 
+acquire_sem:
 	semaphore_down(s);
 
-	// Update state matrices after successfully acquiring the resource.
-	if (p->deadlock_detect_enabled) {
-		// The request has been fulfilled, clear it now.
-		p->sem_req[t->tid][semaphore_id] = 0;
-		// Increase the thread's allocation of this resource.
-		p->sem_alloc[t->tid][semaphore_id]++;
-		// Decrease the thread's available of this resource.
-		p->sem_avail[semaphore_id]--;
-	}
+	if (!p->deadlock_detect_enabled)
+		goto ret;
 
+	// Update state matrices after successfully acquiring the resource.
+	// The request has been fulfilled, clear it now.
+	p->sem_req[t->tid][semaphore_id] = 0;
+	// Increase the thread's allocation of this resource.
+	p->sem_alloc[t->tid][semaphore_id]++;
+	// Decrease the thread's available of this resource.
+	p->sem_avail[semaphore_id]--;
+
+ret:
 	return 0;
 }
 
-- 
2.25.1

```

上面的提交我就不详细解释了，注释都有。

## 问答作业

1. 在我们的多线程实现中，当主线程 (即 0 号线程) 退出时，视为整个进程退出， 此时需要结束该进程管理的所有线程并回收其资源。

- 需要回收的资源有哪些？
- 其他线程的 `struct thread` 可能在哪些位置被引用，分别是否需要回收，为什么？

2. 对比以下两种 `mutex_unlock` 中阻塞锁的实现，二者有什么区别？这些区别可能会导致什么问题？（假设无论哪种实现，对应的 `mutex_lock` 均正确处理了 `m->locked`）

    ```c
    void mutex_unlock_v1(struct mutex *m)
    {
      if (m->blocking) {
          m->locked = 0;
          struct thread *t = id_to_task(pop_queue(&m->wait_queue));
          if (t != NULL) {
            t->state = RUNNABLE;
            add_task(t);
          }
      } else ...
    }

    void mutex_unlock_v2(struct mutex *m)
    {
      if (m->blocking) {
          struct thread *t = id_to_task(pop_queue(&m->wait_queue));
          if (t == NULL) {
            m->locked = 0;
          } else {
            t->state = RUNNABLE;
            add_task(t);
          }
      } else ...
    }
    ```

## 问答作业答案

1. 在我们的多线程实现中，当主线程 (即 0 号线程) 退出时，视为整个进程退出， 此时需要结束该进程管理的所有线程并回收其资源。

- 需要回收的资源有哪些？

    当主线程退出导致整个进程终止时，需要回收所有相关资源以防泄漏。主要包括：内存资源，通过 `freepagetable` 函数释放整个用户地址空间（代码、数据、堆以及所有线程的用户栈），并回收每个线程的内核栈、`trapframe` 页、页表结构本身以及像管道（Pipe）一样由内核动态分配的内存；进程与线程的控制结构，将全局 pool 数组中的 `struct proc` 条目和其下所有的 `struct thread` 条目清理并标记为未使用状态，以供后续分配；文件与设备资源，关闭进程打开的所有文件描述符，并正确处理文件的引用计数；同步与调度资源，包括将进程中所有线程从调度器的就绪队列以及各类同步原语的等待队列中彻底移除。

- 其他线程的 `struct thread` 可能在哪些位置被引用，分别是否需要回收，为什么？

    在一个运行的系统中，除了进程自身的 `threads` 数组，其他线程的 `struct thread` 引用主要存在于两个关键位置，并且都必须被回收。首先是全局的就绪队列（`task_queue`），等待调度的线程 ID 会存放在这里。当进程终止时，必须将其所有 `RUNNABLE` 的线程从这个队列中移除，否则调度器未来可能会尝试运行一个已被销毁的线程，这样就会访问到无效的内存，从而导致系统崩溃。其次是各类同步原语（如互斥锁、信号量）的等待队列中，因阻塞而处于 SLEEPING 状态的线程 ID 会存放在这里，这些也必须被移除，否则这些线程将永远无法被唤醒，造成线程资源永久泄漏，并可能导致其他依赖这些资源的进程产生问题。

2. 对比以下两种 `mutex_unlock` 中阻塞锁的实现，二者有什么区别？这些区别可能会导致什么问题？（假设无论哪种实现，对应的 `mutex_lock` 均正确处理了 `m->locked`）

    ```c
    void mutex_unlock_v1(struct mutex *m)
    {
      if (m->blocking) {
          m->locked = 0;
          struct thread *t = id_to_task(pop_queue(&m->wait_queue));
          if (t != NULL) {
            t->state = RUNNABLE;
            add_task(t);
          }
      } else ...
    }

    void mutex_unlock_v2(struct mutex *m)
    {
      if (m->blocking) {
          struct thread *t = id_to_task(pop_queue(&m->wait_queue));
          if (t == NULL) {
            m->locked = 0;
          } else {
            t->state = RUNNABLE;
            add_task(t);
          }
      } else ...
    }
    ```

    `mutex_unlock_v1` 先解锁再唤醒，也就是先将 `m->locked` 置为 0，再唤醒等待队列中的线程。这样会导致在锁被释放到等待线程被唤醒之间的微小时间窗口内，如果有更高优先级的线程进入，这个线程会抢先获得锁，导致本应获得锁的等待线程被再次阻塞，引发线程饥饿问题，并因无效的唤醒和上下文切换而带来性能开销。`mutex_unlock_v2` 就没有这个问题，它先检查等待队列，如果队列不为空，会直接将等待线程置为就绪态，同时保持锁的状态为 `locked`，也就相当于把锁的所有权直接传递到下一个线程，只有当等待队列为空时，才将锁真正释放。这样就保证了 FIFO 的公平性。

## 用到的算法

最后提一下这个实验里很重要的银行家算法吧，让 Gemini 讲一下：

**银行家算法（Banker's Algorithm）**是一种在操作系统中用于**避免死锁（Deadlock Avoidance）**的著名算法，由艾兹格·迪科斯彻（Edsger Dijkstra）在 1965 年为 THE 操作系统设计。

### 核心思想

银行家算法的名字来源于它模拟了银行家向客户贷款的场景：

- **银行家（操作系统）**：拥有一定数量的资金（系统资源）。
- **客户（进程）**：有多个客户，每个客户在开始时会声明他完成项目所需的最大资金额（进程所需的最大资源量）。
- **贷款（资源分配）**：客户可以分批次申请贷款，但总额不能超过他声明的最大值。
- **还款（资源释放）**：客户完成项目后，会把所有借款一次性还给银行。

银行家的策略是：**只有当他确信，即使批准了当前的贷款请求，也存在一种安全的放贷顺序，能让所有客户都最终完成项目并还款，他才会批准这次贷款。** 否则，他会拒绝请求，让客户等待，直到有其他客户还款，资金变得充裕。

通过这种方式，银行（操作系统）永远不会进入一个无法收回所有贷款（无法满足所有进程的资源需求）的“坏账”状态，从而避免了死锁。

### 关键数据结构

为了实现这个算法，操作系统需要维护以下几个核心的数据结构（假设有 `n` 个进程和 `m` 种资源）：

1. **`Available` (可用资源向量)**
    - 一个长度为 `m` 的向量。
    - `Available[j] = k` 表示系统中当前有 `k` 个 `Rj` 类型的可用资源。

2. **`Max` (最大需求矩阵)**
    - 一个 `n x m` 的矩阵。
    - `Max[i][j] = k` 表示进程 `Pi` 最多需要 `k` 个 `Rj` 类型的资源。这个值是在进程创建时就声明的。

3. **`Allocation` (已分配矩阵)**
    - 一个 `n x m` 的矩阵。
    - `Allocation[i][j] = k` 表示进程 `Pi` 当前已经被分配了 `k` 个 `Rj` 类型的资源。

4. **`Need` (需求矩阵)**
    - 一个 `n x m` 的矩阵。
    - `Need[i][j] = k` 表示进程 `Pi` **还**需要 `k` 个 `Rj` 类型的资源才能完成任务。
    - 这个矩阵不是凭空来的，而是通过计算得出的：
        **`Need[i][j] = Max[i][j] - Allocation[i][j]`**

### 核心算法

银行家算法主要包含两个部分：**安全性算法**和**资源请求算法**。

#### 安全性算法 (Safety Algorithm)

这个算法用于检查当前系统状态是否是**安全**的。一个状态是安全的，指的是系统能找到一个**安全序列（Safe Sequence）**。

**安全序列**是指一个进程的排列 `<P1, P2, ..., Pn>`，按照这个顺序为每个进程分配其所需的全部资源，可以保证每个进程都能顺利执行完毕。

**算法步骤：**

1. **初始化**：
    - 创建一个工作向量 `Work`，`Work = Available`。
    - 创建一个布尔向量 `Finish`，长度为 `n`，初始时所有元素都为 `false`。`Finish[i] = false` 表示进程 `Pi` 尚未完成。

2. **寻找可完成的进程**：
    - 在所有进程中，寻找一个满足以下两个条件的进程 `Pi`：
        - `Finish[i] == false` (该进程还没完成)
        - `Need[i] <= Work` (该进程所需资源小于或等于当前系统可用资源)
    - 如果找不到这样的进程，则跳转到第 4 步。

3. **模拟执行与释放**：
    - 如果找到了这样的进程 `Pi`，则假定它执行完毕并释放了所有资源。
    - 更新 `Work`：`Work = Work + Allocation[i]` (把 `Pi` 占有的资源加回到可用资源中)
    - 标记 `Pi` 为完成：`Finish[i] = true`
    - 返回第 2 步，继续寻找下一个可以完成的进程。

4. **判断结果**：
    - 如果 `Finish` 向量中所有的元素都为 `true`，则说明系统处于**安全状态**，并且已经找到了一个安全序列。
    - 否则，系统处于**不安全状态**。

#### 资源请求算法 (Resource-Request Algorithm)

当进程 `Pi` 发出资源请求时，系统按以下步骤操作。假设请求向量为 `Request_i`，`Request_i[j] = k` 表示进程 `Pi` 请求 `k` 个 `Rj` 类型的资源。

1. **检查请求的合法性**：
    - 判断 `Request_i <= Need[i]` 是否成立。
    - 如果 `false`，说明进程请求的资源超过了它之前声明的最大需求，这是一个错误，应终止该进程。

2. **检查系统是否有足够资源**：
    - 判断 `Request_i <= Available` 是否成立。
    - 如果 `false`，说明系统当前没有足够的资源来满足请求，进程 `Pi` 必须**等待**。

3. **尝试分配（假装分配）**：
    - 如果前两步都通过，系统会**尝试**将资源分配给 `Pi`，并更新数据结构：
        - `Available = Available - Request_i`
        - `Allocation[i] = Allocation[i] + Request_i`
        - `Need[i] = Need[i] - Request_i`

4. **执行安全性检查**：
    - 调用**安全性算法**，检查经过第 3 步更新后的系统状态是否仍然是安全的。
    - **如果安全**：正式批准 `Pi` 的请求，分配完成。
    - **如果不安全**：系统必须**撤销**第 3 步的尝试分配操作（将 `Available`, `Allocation`, `Need` 恢复原状），并让进程 `Pi` **等待**。

### 优缺点

- **优点**：
    - 能够有效地避免死锁的发生。
    - 资源利用率相较于一些更保守的策略（如一次性分配所有资源）要高。
- **缺点**：
    - **需要预知最大资源需求**：这在实际的通用操作系统中很难做到，因为程序的行为是动态的。
    - **进程数量和资源种类必须固定**：算法要求参与者集合是稳定不变的。
    - **开销大**：每次资源请求都需要运行一次安全性算法，其时间复杂度为 O(m * n²)，这在大型系统中可能是无法接受的。
    - **资源释放模型不灵活**：算法要求进程在执行完毕后一次性释放所有资源，而实际情况中进程可能是边执行边释放资源的。
