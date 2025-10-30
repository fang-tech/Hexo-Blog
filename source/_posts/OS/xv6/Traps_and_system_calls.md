---
title: xv6 traps and system callls
tags:
  - Operating-System
  - xv6
  - syscall
date: 2025-10-29
---

# trap
trap有三种类型
- system call
- exception
- interrupt

每次触发trap的时候我们都会从用户态陷入到内核态, 并且这个过程对于用户程序来说应该是没有感知的, 在执行完trap以后, 从内核态回到用户态.

所有的trap都只在内核中执行, 这样能保证对物理设备访问的隔离性, 也能在处理异常的时候可以做出像kill用户进程这样的内核才有权限执行的响应方式

完成一个trap需要四步
1. 硬件CPU上的动作
2. 准备好的汇编代码, 用于进入到内核中对印的trap处理c函数上
3. 处理这个trap的c函数
4. 内核运行代码的内核进程

## RISC-V寄存器
### 指令
- 用于系统控制的特殊的寄存器, 不能随便地读写, 需要特殊的指令才能读写
	- `csrr`: CSR寄存器 read
	- `csrw`: CSR寄存器 write
### 寄存
- 系统控制寄存器
	- **sstatus: Supervisor Status Register**监督者状态
		- (第9个bit, 1L<< 8)记录异常发生前的CPU的特权级别, 1 = 来自Supervisor, 0 = 来自User
	- **stvec: Supervisor Trap Vector** 监督者陷阱向量
		- 指向发生trap的时候, CPU接下来要跳转到函数的地址
	- **sepc: Supervisor Exception Program Counter** 监督者异常pc
		- 保存发生trap的时候的指令地址, 以便异常处理完成以后能够返回
	- **scause: Supervisor Cause Register** 监督者原因
		- 记录陷入trap的原因
			- 8: 系统调用(ecall from U-mode)
			- 13: Load page fault (读页面错误)
			- 15: Store page fault (写页面错误)
	- **stval: Supervisor Trap Value** 监督者陷阱值
		- 用于提供额外的信息, 对于页面错误会导致错误的虚拟地址, 对于非法指令, 会提供指令本身
	- **satp: Supervisor Address Translation and Protection** 监督者地址转换和保护
		- 控制页表的地址, 用于虚拟地址到物理地址的转换, 准备用户进程的页表地址, 用于返回用户态
		- 访问函数: `#define MAKE_SATP(pagetable) (SATP_SV39 | (((uint64)pagetable) >> 12))`
		- 使用:  `uint64 satp = MAKE_SATP(p->pagetable)`
# xv6中进入到一个trap的过程讲解 
## 用户态陷入trap的处理函数
> kernel/trap.c: usertra

- 校验此刻CPU是不是U-Mode
- 将kernelvec函数的地址写入到stvec, 接下来执行kernelvec代码(在下一个小部分)
- 保存用户进程的pc(从sepc读取)
- 处理不同的异常情况 (读取scause的值)
	- 8 -> 来自系统调用: 
		- 将sepc的值 += 4, 指向下一条指令, 从而在返回的时候是返回到出现trap的指令的下一条指令
		- Supervisor Interrupt Enable, 将sstatus的bit 1设置为1, 开启中断, 允许系统在处理系统调用时响应其他的中断(比如timer系统终中断)
		- 执行syscall()函数, 调用对应的syscall函数
	- 13 / 15 ->
- 如果这是个timer中断, 让出cpu
- prepare_return()
- 切换到用户页表

> kernel/kernelvec.S

- 声明入口点

```S
.globl kerneltrap  # 将kerneltrap声明为全局的符号
.globl kernelvec  # 同样申请成全局符号以供外部调用
.align 4  # 将代码对齐到16字节边界, 提高性能
kernelvec:  # kernelvec入口标签, 在usertrap的第二步就会进入到这个入口中执行接下来的汇编代码
```

- 向下增长栈, 为保存通用寄存器留出空间

```S
# make room to save registers.
addi sp, sp, -256  # 栈指针向下移动256字节, 用于保存寄存器(RISC-V有32个通用寄存器=32*8 = 256字节)
```

- 保存用户态的寄存器

> RISC-V调用约定中, 寄存器分成两类
> 1. caller-saved registers: 调用函数前, 调用者必须保存这些寄存器
> 2. callee-saved registers: 被调用函数必须保存和恢复这些寄存器
> 这里我们只保存了caller-save register的原因是C函数会遵循RISC-V的约定, 保存callee-save register, 所以我们只用保存另一部分

```S
# save caller-saved registers.
sd ra, 0(sp)  # return address
# sd sp, 8(sp)
sd gp, 16(sp)  # global pointer
sd tp, 24(sp)  # thread pointer
sd t0, 32(sp)
sd t1, 40(sp)
sd t2, 48(sp)
sd a0, 72(sp)
sd a1, 80(sp)
sd a2, 88(sp)
sd a3, 96(sp)
sd a4, 104(sp)
sd a5, 112(sp)
sd a6, 120(sp)
sd a7, 128(sp)
sd t3, 216(sp)
sd t4, 224(sp)
sd t5, 232(sp)
sd t6, 240(sp)
```

- 调用kerneltrap C函数

```c
# call the C trap handler in trap.c
call kerneltrap
```

- 恢复caller-saved寄存器

```S
# restore registers.
ld ra, 0(sp)
# ld sp, 8(sp)
ld gp, 16(sp)
# not tp (contains hartid), in case we moved CPUs
ld t0, 32(sp)
ld t1, 40(sp)
ld t2, 48(sp)
ld a0, 72(sp)
ld a1, 80(sp)
ld a2, 88(sp)
ld a3, 96(sp)
ld a4, 104(sp)
ld a5, 112(sp)
ld a6, 120(sp)
ld a7, 128(sp)
ld t3, 216(sp)
ld t4, 224(sp)
ld t5, 232(sp)
ld t6, 240(sp)
```

```S
addi sp, sp, 256 # 将栈指针移回256字节前

# return to whatever we were doing in the kernel.
sret
```

> sret指令: 行为
> 1. 将PC设置成sepc的值
> 2. 将特权级别恢复成sstatus.SPP中保存的值
> 3. 将中断使能恢复成sstatus.SPIE中保存的值
> 4. 继续执行被中断的代码
## 用户态trap处理函数的返回准备函数
> kernel/trap.c: prepare_return

- intr_off(): 关闭中断
- 