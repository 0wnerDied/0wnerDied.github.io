---
title: uCore lab1 实验报告
categories:
  - uCore
tags:
  - OS
  - riscv
abbrlink: 28113
date: 2025-07-24 00:51:32
---

## 本章任务

### 获取任务信息

ch3 中，我们的系统已经能够支持多个任务分时轮流运行，我们希望引入一个新的系统调用 `sys_trace` （syscall ID: 410）用来追踪当前任务系统调用的历史信息、并进行任务内存的读写。定义如下：

```c
int sys_trace(int trace_request, unsigned long id, uint8_t data);
```

- syscall ID: 410

- 调用规范：

  - 这个系统调用有三种功能，根据 trace_request 的值不同，执行不同的操作：
  - 如果 trace_request 为 0，则 id 应被视作 uint8_t * ，表示读取当前任务 id 地址处一个字节的无符号整数值。此时应忽略 data 参数。返回值为 id 地址处的值。
  - 如果 trace_request 为 1，则 id 应被视作 uint8_t * ，表示写入 data （作为 uint8_t，即只考虑最低位的一个字节）到该用户程序 id 地址处。返回值应为0。
  - 如果 trace_request 为 2，表示查询当前任务调用编号为 id 的系统调用的次数，返回值为这个调用次数。本次调用也计入统计 。
  - 否则，忽略其他参数，返回值为 -1。

- 说明：

  - uint8_t 是C语言中的标准类型，表示一个无符号的8位整数。在代码中你可能需要使用 uint8 替代。
  - 你可能会注意到，这个调用的读写并不安全，使用不当可能导致崩溃。这是因为在下一章节实现地址空间之前，系统中缺乏隔离机制。所以我们 不要求你实现安全检查机制，只需通过测试用例即可。
  - 你还可能注意到，这个系统调用读写本任务内存的功能并不是很有用。这是因为作业的灵感来源 syscall 主要依靠 trace 功能追踪其他任务的信息，但在本章节我们还没有进程、线程等概念，所以简化了操作，只要求追踪自身的信息。

### 完成任务

下面是我对代码的修改，导出提交：

```patch
From b1836f667109436153481ae6f338a6a50a19e232 Mon Sep 17 00:00:00 2001
From: 0wnerDied <z1281552865@gmail.com>
Date: Wed, 23 Jul 2025 22:57:41 +0800
Subject: [PATCH] chapter3 practice

Signed-off-by: 0wnerDied <z1281552865@gmail.com>
---
 os/loader.c      |  1 +
 os/proc.c        |  1 +
 os/proc.h        |  1 +
 os/syscall.c     | 22 ++++++++++++++++++++++
 os/syscall_ids.h |  1 +
 5 files changed, 26 insertions(+)

diff --git a/os/loader.c b/os/loader.c
index be9f10a..72b49ed 100644
--- a/os/loader.c
+++ b/os/loader.c
@@ -51,6 +51,7 @@ int run_all_app()
 		/*
 		* LAB1: you may need to initialize your new fields of proc here
 		*/
+		memset(p->syscall_cnt, 0, sizeof(p->syscall_cnt));
 	}
 	return 0;
 }
\ No newline at end of file
diff --git a/os/proc.c b/os/proc.c
index 84240a8..c98b49e 100644
--- a/os/proc.c
+++ b/os/proc.c
@@ -34,6 +34,7 @@ void proc_init(void)
 		/*
 		* LAB1: you may need to initialize your new fields of proc here
 		*/
+		memset(p->syscall_cnt, 0, sizeof(p->syscall_cnt));
 	}
 	idle.kstack = (uint64)boot_stack_top;
 	idle.pid = 0;
diff --git a/os/proc.h b/os/proc.h
index f27369a..74cfe1c 100644
--- a/os/proc.h
+++ b/os/proc.h
@@ -38,6 +38,7 @@ struct proc {
 	/*
 	* LAB1: you may need to add some new fields here
 	*/
+	int syscall_cnt[512];
 };
 
 struct proc *curr_proc();
diff --git a/os/syscall.c b/os/syscall.c
index 3225031..50b0035 100644
--- a/os/syscall.c
+++ b/os/syscall.c
@@ -39,6 +39,22 @@ uint64 sys_gettimeofday(TimeVal *val, int _tz)
 /*
 * LAB1: you may need to define sys_trace here
 */
+int sys_trace(int trace_request, unsigned long id, uint8 data)
+{
+	struct proc *p = curr_proc();
+
+	switch (trace_request) {
+	case 0:
+		return *(uint8 *)id;
+	case 1:
+		*(uint8 *)id = data;
+		return 0;
+	case 2:
+		return (id < 512) ? p->syscall_cnt[id] : -1;
+	default:
+		return -1;
+	}
+}
 
 extern char trap_page[];
 
@@ -53,6 +69,9 @@ void syscall()
 	/*
 	* LAB1: you may need to update syscall counter here
 	*/
+	struct proc *p = curr_proc();
+	(id < 512) ? p->syscall_cnt[id]++ : 0;
+
 	switch (id) {
 	case SYS_write:
 		ret = sys_write(args[0], (char *)args[1], args[2]);
@@ -69,6 +88,9 @@ void syscall()
 	/*
 	* LAB1: you may need to add SYS_trace case here
 	*/
+	case SYS_trace:
+		ret = sys_trace(args[0], args[1], args[2]);
+		break;
 	default:
 		ret = -1;
 		errorf("unknown syscall %d", id);
diff --git a/os/syscall_ids.h b/os/syscall_ids.h
index e6fac51..06d8c00 100644
--- a/os/syscall_ids.h
+++ b/os/syscall_ids.h
@@ -280,6 +280,7 @@
 /*
 * LAB1: you may need to define SYS_trace here
 */
+#define SYS_trace 410
 #define SYS_pidfd_send_signal 424
 #define SYS_io_uring_setup 425
 #define SYS_io_uring_enter 426
-- 
2.25.1

```

然后 `make test` 检查即可

```
[rustsbi] RustSBI version 0.3.0-alpha.2, adapting to RISC-V SBI v1.0.0
.______       __    __      _______.___________.  _______..______   __
|   _  \     |  |  |  |    /       |           | /       ||   _  \ |  |
|  |_)  |    |  |  |  |   |   (----`---|  |----`|   (----`|  |_)  ||  |
|      /     |  |  |  |    \   \       |  |      \   \    |   _  < |  |
|  |\  \----.|  `--'  |.----)   |      |  |  .----)   |   |  |_)  ||  |
| _| `._____| \______/ |_______/       |__|  |_______/    |______/ |__|
[rustsbi] Implementation     : RustSBI-QEMU Version 0.2.0-alpha.2
[rustsbi] Platform Name      : riscv-virtio,qemu
[rustsbi] Platform SMP       : 1
[rustsbi] Platform Memory    : 0x80000000..0x88000000
[rustsbi] Boot HART          : 0
[rustsbi] Device Tree Region : 0x87000000..0x87000ef2
[rustsbi] Firmware Address   : 0x80000000
[rustsbi] Supervisor Address : 0x80200000
[rustsbi] pmp01: 0x00000000..0x80000000 (-wr)
[rustsbi] pmp02: 0x80000000..0x80200000 (---)
[rustsbi] pmp03: 0x80200000..0x88000000 (xwr)
[rustsbi] pmp04: 0x88000000..0x00000000 (-wr)
Hello world from user mode program!
Test hello_world OK!
3^10000=5079
3^20000=8202
3^30000=8824
3^40000=5750
3^50000=3824
3^60000=8516
3^70000=2510
3^80000=9379
3^90000=2621
3^100000=2749
Test power OK!
get_time OK! 11
current time_msec = 11
AAAAAAAAAA [1/5]
CCCCCCCCCC [1/5]
BBBBBBBBBB [1/5]
AAAAAAAAAA [2/5]
CCCCCCCCCC [2/5]
BBBBBBBBBB [2/5]
AAAAAAAAAA [3/5]
CCCCCCCCCC [3/5]
BBBBBBBBBB [3/5]
AAAAAAAAAA [4/5]
CCCCCCCCCC [4/5]
BBBBBBBBBB [4/5]
AAAAAAAAAA [5/5]
CCCCCCCCCC [5/5]
BBBBBBBBBB [5/5]
Test write A OK!
Test write C OK!
Test write B OK!
time_msec = 112 after sleeping 100 ticks, delta = 101ms!
Test sleep1 passed!
string from task trace test
Test trace OK!
Test sleep OK!
[PANIC 5] os/loader.c:14: all apps over
```

可以看到 `Test trace OK!`，修改成功通过了测试。

## 问答作业

1. 正确进入 U 态后，程序的特征还应有：使用 S 态特权指令，访问 S 态寄存器后会报错。请同学们可以自行测试这些内容（参考 前三个测例，描述程序出错行为，同时注意注明你使用的 sbi 及其版本。

2. 请结合用例理解 trampoline.S 中两个函数 `userret` 和 `uservec` 的作用，并回答如下几个问题:

    1. L79: 刚进入 `userret` 时，`a0`、`a1` 分别代表了什么值。

    2. L87-L88: `sfence` 指令有何作用？为什么要执行该指令，当前章节中，删掉该指令会导致错误吗？

        ```assembly
        csrw satp, a1
        sfence.vma zero, zero
        ```

    3. L96-L125: 为何注释中说要除去 a0？哪一个地址代表 a0？现在 a0 的值存在何处？

        ```assembly
        # restore all but a0 from TRAPFRAME
        ld ra, 40(a0)
        ld sp, 48(a0)
        ld t5, 272(a0)
        ld t6, 280(a0)
        ```

    4. `userret`：中发生状态切换在哪一条指令？为何执行之后会进入用户态？

    5. L29： 执行之后，`a0` 和 `sscratch` 中各是什么值，为什么？

        ```assembly
        csrrw a0, sscratch, a0
        ```

    6. L32-L61: 从 trapframe 第几项开始保存？为什么？是否从该项开始保存了所有的值，如果不是，为什么？

        ```assembly
        sd ra, 40(a0)
        sd sp, 48(a0)
        ...
        sd t5, 272(a0)
        sd t6, 280(a0)
        ```

    7. 进入 S 态是哪一条指令发生的？

    8. L75-L76: `ld t0, 16(a0)` 执行之后，`t0`中的值是什么，解释该值的由来？

        ```assembly
        ld t0, 16(a0)
        jr t0
        ```

## 问答作业解答

1. 正确进入 U 态后，程序的特征还应有：使用 S 态特权指令，访问 S 态寄存器后会报错。请同学们可以自行测试这些内容（参考前三个测例 ，描述程序出错行为，同时注意注明你使用的 sbi 及其版本。

    执行 `make test CHAPTER=2_bad` 后如下：

    ```
    [rustsbi] RustSBI version 0.3.0-alpha.2, adapting to RISC-V SBI v1.0.0
    .______       __    __      _______.___________.  _______..______   __
    |   _  \     |  |  |  |    /       |           | /       ||   _  \ |  |
    |  |_)  |    |  |  |  |   |   (----`---|  |----`|   (----`|  |_)  ||  |
    |      /     |  |  |  |    \   \       |  |      \   \    |   _  < |  |
    |  |\  \----.|  `--'  |.----)   |      |  |  .----)   |   |  |_)  ||  |
    | _| `._____| \______/ |_______/       |__|  |_______/    |______/ |__|
    [rustsbi] Implementation     : RustSBI-QEMU Version 0.2.0-alpha.2
    [rustsbi] Platform Name      : riscv-virtio,qemu
    [rustsbi] Platform SMP       : 1
    [rustsbi] Platform Memory    : 0x80000000..0x88000000
    [rustsbi] Boot HART          : 0
    [rustsbi] Device Tree Region : 0x87000000..0x87000ef2
    [rustsbi] Firmware Address   : 0x80000000
    [rustsbi] Supervisor Address : 0x80200000
    [rustsbi] pmp01: 0x00000000..0x80000000 (-wr)
    [rustsbi] pmp02: 0x80000000..0x80200000 (---)
    [rustsbi] pmp03: 0x80200000..0x88000000 (xwr)
    [rustsbi] pmp04: 0x88000000..0x00000000 (-wr)
    hello wrold!
    [ERROR 0]unknown trap: 0x0000000000000007, stval = 0x0000000000000000 sepc = 0x0000000080400004
    [ERROR 0]IllegalInstruction in application, epc = 0x0000000080400004, core dumped.
    [ERROR 0]IllegalInstruction in application, epc = 0x0000000080400004, core dumped.
    ALL DONE
    ```

    可以看到项目使用 RustSBI，版本为 0.3.0-alpha.2。

    别忘记 `trap` 类异常：

    ```c
    enum Exception {
      InstructionMisaligned = 0,
      InstructionAccessFault = 1,
      IllegalInstruction = 2,
      Breakpoint = 3,
      LoadMisaligned = 4,
      LoadAccessFault = 5,
      StoreMisaligned = 6,
      StoreAccessFault = 7,
      UserEnvCall = 8,
      SupervisorEnvCall = 9,
      MachineEnvCall = 11,
      InstructionPageFault = 12,
      LoadPageFault = 13,
      StorePageFault = 15,
    };
    ```

    三个测例如下：

    ```c
    // __ch2_bad_address.c
    #include <stdio.h>
    #include <unistd.h>

    int main()
    {
      int *p = (int *)0;
      *p = 0;
      return 0;
    }
    ```

    这是一个空指针解引用的测例，U 态程序尝试访问 S 态，显然运行会触发 `Store/AMO access fault` 异常，所以我们可以看到 `unknown trap: 0x0000000000000007`。

    ```c
    // __ch2_bad_instruction.c
    #include <stdio.h>
    #include <unistd.h>

    int main()
    {
      asm volatile("sret");
      return 0;
    }
    ```

    这是一个使用 S 态特权指令的测例，因为 U 态不能执行 S 态特权指令，所以会触发 `Illegal instruction` 异常。

    ```c
    // __ch2_bad_register.c
    #include <stdio.h>
    #include <unistd.h>

    int main()
    {
      uint64 x;
      asm volatile("csrr %0, sstatus" : "=r"(x));
      return 0;
    }
    ```

    这是一个访问 S 态寄存器的测例，同样地，U 态不能访问 S 态 CSR 寄存器，所以会触发 `Illegal instruction` 异常。

    通过上面的测试可以看到，成功输出了 `hello wrold!`，即用户态程序可以正常运行，且 RustSBI 正常工作，但存在对 U 态的特权级别限制，所以这几个测例会报错。

2. 请结合用例理解 trampoline.S 中两个函数 `userret` 和 `uservec` 的作用，并回答如下几个问题:

    1. L79: 刚进入 `userret` 时，`a0`、`a1` 分别代表了什么值。

        从注释中可以看到
            ```assembly
            userret:
                # userret(TRAPFRAME, pagetable)
                # switch from kernel to user.
                # usertrapret() calls here.
                # a0: TRAPFRAME, in user page table.
                # a1: user page table, for satp.
            ```
        所以 `a0` 指向 TRAPFRAME 的地址；`a1` 是用户页表的物理地址，将被设置为寄存器 `satp` 的值。

    2. L87-L88: `sfence` 指令有何作用？为什么要执行该指令，当前章节中，删掉该指令会导致错误吗？

        ```assembly
        csrw satp, a1
        sfence.vma zero, zero
        ```

        刷新 TLB。当修改 `satp` 寄存器切换页表时，TLB 中可能还缓存着旧页表的地址转换，必须刷新 TLB 确保后续的地址转换使用新页表。当前章节中删去不会出错，这里还不涉及用户页表切换。

    3. L96-L125: 为何注释中说要除去 `a0`？哪一个地址代表 `a0`？现在 `a0` 的值存在何处？

        ```assembly
        # restore all but a0 from TRAPFRAME
        ld ra, 40(a0)
        ld sp, 48(a0)
        ...
        ld t5, 272(a0)
        ld t6, 280(a0)
        ```

        当前 `a0` 存储的是 TRAPFRAME 的地址，需要用它访问保存的寄存器值，如果现在恢复 `a0`，就无法继续访问 TRAPFRAME 了。`a0` 的原始值保存在 `112(a0)` 位置，也就是TRAPFRAME 中的 `a0` 字段。在恢复过程中会先被加载到 `sscratch` 寄存器，最后通过 L128：`csrrw a0, sscratch, a0` 指令完成 `a0` 的恢复。

    4. `userret`：中发生状态切换在哪一条指令？为何执行之后会进入用户态？

        `sret`。`sret` 指令会根据 `sstatus` 寄存器的 `SPP` 位决定返回的特权级，在 `usertrapret()` 中，`x &= ~SSTATUS_SPP;` 已经设置了 `sstatus.SPP = 0`，表示返回用户态。

    5. L29： 执行之后，`a0` 和 `sscratch` 中各是什么值，为什么？

        ```assembly
        csrrw a0, sscratch, a0
        ```

        `csrrw` 是原子交换指令，同时读取和写入 `CSR` 寄存器，所以 `a0` 是 TRAPFRAME 的地址，`sscratch` 中是用户程序的原始 `a0` 值。

    6. L32-L61: 从 trapframe 第几项开始保存？为什么？是否从该项开始保存了所有的值，如果不是，为什么？

        ```assembly
        sd ra, 40(a0)
        sd sp, 48(a0)
        ...
        sd t5, 272(a0)
        sd t6, 280(a0)
        ```

        从 `ra` 开始保存，`ra` 的偏移量为 40，$40÷8=5$，所以从 `trapframe` 第六项开始保存。并没有保存所有的值，`a0` 没有保存，因为此时 `a0` 指向 `trapframe`，其原始值通过 `csrrw a0, sscratch, a0` 保存在 `sscratch` 中，后续在 `trapframe` 偏移量 112 处，也就是 `trapframe` 中的 `a0` 字段保存。

    7. 进入 S 态是哪一条指令发生的？

        用户程序中的 `ecall`。

    8. L75-L76: `ld t0, 16(a0)` 执行之后，`t0`中的值是什么，解释该值的由来？

        ```assembly
        ld t0, 16(a0)
        jr t0
        ```

        ```c
        // os/trap.c

        void usertrapret(struct trapframe *trapframe, uint64 kstack)
        {
          trapframe->kernel_satp = r_satp(); // kernel page table
          trapframe->kernel_sp = kstack + PGSIZE; // process's kernel stack
          trapframe->kernel_trap = (uint64)usertrap;
          trapframe->kernel_hartid = r_tp(); // hartid for cpuid()

          w_sepc(trapframe->epc);
          // set up the registers that trampoline.S's sret will use
          // to get to user space.

          // set S Previous Privilege mode to User.
          uint64 x = r_sstatus();
          x &= ~SSTATUS_SPP; // clear SPP to 0 for user mode
          x |= SSTATUS_SPIE; // enable interrupts in user mode
          w_sstatus(x);

          // tell trampoline.S the user page table to switch to.
          // uint64 satp = MAKE_SATP(p->pagetable);
          userret((uint64)trapframe);
        }
        ```

        可以看到 `trapframe->kernel_trap = (uint64)usertrap;`，所以 `t0` 中的值是 `usertrap` 函数的地址。当系统调用完成准备返回用户态时，`usertrapret()` 函数会被调用，`trapframe->kernel_trap = (uint64)usertrap;` 将 `usertrap` 函数地址存储在 `trapframe` 的偏移量 16 处，于是下次用户态发生 `trap` 时，`uservec` 就能正确跳转到`usertrap` 函数。
