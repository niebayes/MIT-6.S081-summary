# MIT 6.S081 & OSTEP

trap

1. trap 由硬件（CPU）和软件（kernel）协同完成，分四个步骤：
(1) RISC-V 硬件做一些操作，从 user mode 提升为 kernel mode.
(2) 一些汇编代码做一些不需要栈的关于寄存器的操作。这些代码称为 vector 代码（vector = jump），意为跳板代码，因为此时处于 user space 与 kernel space 的交界（注意区分 mode 与 space）。
(3) 一些 C 代码来 handle 这个 trap。此时在 kernel space.
(4) 之后从 kernel space 返回到 user space.
在 RISC-V 中，硬件只做最小的工作，其他交由软件完成。这样可以给系统设计者更多的灵活性。
2. 相关的寄存器和指令：
(1) stvec 寄存器：由 kernel 设置，下次 trap 发生时，从哪里开始 handle 这个 trap.
(2) sepc 寄存器：trap 发生时，user pc 被保存在这里，之后被 retrieve back 以 resume user control flow.
(3) scause 寄存器：描述发生此次 trap 的原因。
(4) sscratch 寄存器：scratch = temporary，用于临时保存某些东西。
(5) sstatus 寄存器：有两个 bits 比较重要：SPP bit，表示发生 trap 时处于哪个 mode，之后返回时就返回到这个 mode；SIE bit，表示在 kernel mode 时是否使能 intr。这里的 intr 包括硬件中断与软件中断，后者即包括由 ecall 引发的 trap（存疑？）。
(6) ecall 指令： ecall = environment call. 引发一个 environment call exception. E.g. syscall.
(7) sret 指令： ret = return from trap。该指令会将 sepc 寄存器内的值复制到 pc 寄存器中，即在执行该指令之后，执行的下一条指令是什么。
3. 发生 trap 时，硬件做的操作：
(1) device intr && SIE bit cleared ，则不 trap. 直接结束。
(2) clear SIE bit.
(3) pc -> epc
(4) mode -> sstatus
(5) cause num -> scause, e.g. environment call (e.g syscall) = 8, store page fault = 15, load page fault = 13, instruction page fault = 12.
(6) set to kernel mode.
(7) stvec -> pc.
4. 发生 trap 时，软件做的操作：
(1) 硬件完成以上任务后，CPU 接着执行 pc，故此时来到预设好的位于 `trapoline.S` 文件中的 `uservec` 函数的起始处。
(2) `uservec` 函数作为跳板，其工作即为之后运行在 kernel mode 下做准备。一是保存当前寄存器中的值到 trapframe 中，以便之后 resume。此处需要 sscratch 寄存器作为一个辅助临时寄存器。二是切换到 内核页表。注意切换页表后，CPU 还能正确执行指令，是因为 kernel page table 与 user page table 都在同一段 va 地址段映射了 trapoline page.
(3) 现在跳转到 `trap.c` 文件中的 `usertrap` 函数。由于在 kernel space 中也可能发生 trap，故需要更改 stvec 寄存器的值，使得这些 trap 被 kernelvec 所 handle。同时需要将 epc 寄存器的值（存的是 user pc）存到 user trapframe 中，以免被下一个 trap 所覆盖。此时是不会被打断的，因为硬件已经 disable 了 intr.
(4) 根据 sscause 的值，handle syscall, device intr 或 exception。在 handle syscall 之前，需要打开 intr，因为可能是一个 write to disk syscall，则为了 overlap I/O 与 CPU，故需要打开 intr.
(5) 如果不是 exception，则跳转到 `usertrapret` 函数。此时先关闭 intr，因为要保证修改寄存器值时不被中途打断，导致 inconsistent. 该函数的主要工作就是为下一次 trap 进 kernel 做准备。
(6) 之后来到 `trampoline.S` 中的 `userret` 函数，该函数即取回之前保存在 trapframe 中的 user 所使用的寄存器的值，再切换到 user page table。最后执行 sret 执行，以切换到 user mode，并将 spec 复制到 pc。至此，resume 了 user control flow.

syscall

1. user 调用的 syscall 实际上是 C wrapper functions. 这个 function 对应的汇编码会将 syscall num 存到 a7 寄存器中，之后执行 ecall 指令以 trap 进内核。
2. 根据 C calling convention，这些 wrap func 参数的值会存到寄存器中。这些寄存器的值在 trap 时，又会存到 trapframe 中。而 trapframe 所在的物理段被 direct map 到 kernel virtual addr space。
故 kernel 可以通过 `argraw` 函数从 trapframe 中取回这些寄存器的值.
3. kernel 的 `syscall` 函数根据 syscall num 的值，在 syscall table 中找到对应的 syscall func 的函数指针，并调用它。
4. 当 syscall 调用结束后，返回值被存到 a0 寄存器中，以便 user 取回。

start

1. `make qemu` 会根据 Makefile 编译生成 `kernel` 目标文件。该文件存在 disk 中。
2. 根据 Makefile 第 213 行，`make qemu` 会在编译完成后，立即运行 `kernel`，即根据 `kernel.ld` 将其 load 进内存，以供执行。
3. `kernel.ld` 配置了该 load 的输出，即按 ELF 格式指定了 `kernel` 文件中 .text 等 sections 加载到内存后的地址。特别地，设置 `0x80000000` 为起始地址。
4. 查看 `kernel.asm` 可知，起始地址对应了函数 `_entry`，该函数的汇编码可在 `entry.S` 中查看。因此，`_entry` 为启动后第一个运行的函数。
5. 由于之后的启动过程有比较复杂的逻辑，使用汇编较难实现，故 xv6 使用了 C 代码。又由于该段 C 代码中存在函数调用，因此必须事先准备 stack。故 `_entry` 函数的唯一作用就是开辟一段内存，以作为之后运行 C 代码所需的 stack。
特别地，由于有多个 CPU，因此为每个 CPU 开辟互不重合的一个 page 作为 stack。之后执行 call 指令，调用 `start` 函数。
6. `start` 函数在文件 `start.c` 中。到目前为止，CPU 处于启动后默认的 machine mode，该 mode 拥有所有硬件的控制权。因此 `start` 函数的主要工作是配置硬件 timer，以定期发送 timer intr。
之后的初始化任务由 `main.c` 文件中的 `main` 函数完成。为了在 `start` 函数之后以 supervisor mode 执行 `main` 函数，`start` 函数做了如下操作：
(1) 设置 `mstatus` 寄存器。该寄存器中存在一个 `MPP` bit，即 machine previous priviledge bit，表示之前机器处于哪个 mode。将其设置为 `MPP_S`，即表示 supervisor mode。
(2) 设置 `mepc` 寄存器。该寄存器存的是切换到 supervisor mode 后，下一个运行的指令。此处设为 `main` 函数的起始地址，即跳转到 supervisor mode 后，立即执行 `main` 函数。
(3) 使能 supervisor mode 下的所有 intrs 和 exceptions。
做完这些工作后，调用 `mret` 以“跳回”到 supervisor mode。刚刚设定的 `main` 函数此时得以运行。
7. 由于每个 CPU 控制独立的寄存器组，因此以上流程所有 CPU 都需要独立完成。而之后的初始化任务涉及到内存、磁盘等非 CPU 部件，故并不需要每个 CPU 都走一遍该初始化流程。因此，`main`函数委托 cpuid 为 `0` 的 CPU 完成大部分与 CPU 无关的初始化流程。在其完成之后，再初始化与 CPU 相关的东西。
特别地，`main` 函数会调用 `userinit` 函数，以开启第一个用户进程。该用户进程是必要的，因为目前为止，系统内只存在 NCPU 个内核进程，如果没有用户进程，那么用户无法与内核交互，即无法使用该电脑。
8. `userinit` 函数位于文件 `proc.c` 中。其工作可分为四个部分：
(1) 调用 `allocproc` 在 process table 中找到一个空闲的 `proc` struct，并将其保存到全局变量 `initproc` 中。`allocproc` 为该 proc 分配一个物理页，并构建 page table 映射关系。
之后，将 initproc->context 中的 `ra` 设为 `forkret` 函数的起始地址；并设置 `sp`，以指定 initproc 在内核中运行时所使用的内核栈（除了 scheduler 进程以外，每个进程都有一个独立的用户栈，以及一个独立的内核栈，分别支持该进程在用户空间与内核空间运行 C 代码）。
(2) 调用 `uvminit` 函数，将 `initcode` 这一段字节码映射到 initproc 的 vm 空间的最开头部分，即 va = 0 开始的一个 page。
(3) 设置 initproc->trapframe 的 `epc`，以在之后跳转到 user space 后，立即执行刚刚映射到该地址的 `initcode` 所对应的用户程序，即 `initcode.S` 文件所描述的程序（实际上，该文件只是 MIT 提供的辅助阅读源码的文件，该文件对应的汇编程序即对应 initcode 二进制编码）。
(4) 设置 initproc->state 为 RUNNABLE，以供 `scheduler` 函数调度。
9. 在初始化完成之后，`main` 函数调用 `scheduler` 函数，进入一个 indefinite loop。该 loop 会 linear scan 出一个 state 为 RUNNABLE 的 proc，并 context switch 到该 proc。
具体而言，scheduler 是当前正在运行的内核进程，由 cpu->p field 指明。所谓的 context switch，即将 cpu->p 替换为所找到的某个 proc，即在内核中 resume 该 proc，同时 suspend 原来的某个 proc，此处即 scheduler 进程。
此时，唯一可切换的 proc 即为 initproc。
10. 关于 context switch 以及 user/kernel space 的切换，留待之后详细讲解，此处作简要说明。
(1) context switch 由 `swtch.S` 文件中的 `swtch` 函数完成。该函数的重点在于：将 initproc->context 的 `ra` 以及 `sp` 替换 scheduler 进程的 `ra` 以及 `sp`。`ra` 寄存器，即 return address 寄存器，表示当该函数执行完成之后，所返回的指令的地址。
因此，当 `swtch` 函数执行完毕，CPU 立即接着执行 `ra` 所指向的地址。之前，`ra` 被设为 `forkret` 函数的起始地址，故 `forkret` 函数此时被执行。`forkret` 函数执行时所使用的内核栈，即为 `sp` 所指定的内核栈。
(2) `forkret` 函数完成切换到 user space 的任务。简而言之，其调用 `trap.c` 文件中的 `usertrapret` 函数，该函数将 initproc 使用的内核栈的有关信息存到 user vm 中的 trampframe 中，以供 re-enter 进内核时使用。
并设置 `satp` 寄存器，以切换到 user page table。还设置 `mstatus` 寄存器与 `spec` 寄存器，以使得执行 `sret` 指令后，能进入 user mode，并执行刚刚保存到 `sepc` 寄存器中的值，即 `initcode` 字节码的首地址。
最后调用 `trampoline.S` 中的 `userret` 函数，在完成一定操作后，执行 `sret` 指令，切换到 user space。
11. `initcode` 字节码所做的唯一工作即使用 exec syscall 加载 `init.c` 文件进内存，并执行它。
12. `init.c` 首先创建一个 console 设备，再 fork 一个子进程以 exec shell （init 所在的父进程会一直 wait shell 子进程退出，以 reap 它）。之后又经过一轮 user/kernel 切换后 （因为涉及到 syscall），shell 程序终于在 initproc 中以 user mode 开始运行。xv6 至此启动完毕。

scheduling

1. thread (线程) vs. process (进程)
(1) thread，链、串，顾名思义，一种定义为串行执行代码的单元。
线程状态（即什么定义了一个线程）： 程序运行所需的寄存器
(a) 程序计数器（pc寄存器）,
(b) callee saved register 的值，
(c) caller saved register 的值（保存在 stack 中，由 stack pointer (sp) 寄存器表征）.
通过 restore 线程的状态，i.e. 将当前 CPU 寄存器的值替换为之前保存的线程运行时寄存器的值，即可恢复线程的运行。
因此，一般说 线程是 OS 调度的基本单元。（实际上，在 xv6 中，进程是 OS 调度的基本单元。故需要看具体实现）
多线程是实现 multi-tasking 的一种方式，其他还有 event-driven, state machine etc.
(2) process，一种定义为 进程 = 线程 + 线程间共享的内存
进程状态：线程状态 + 页表 + etc.
因为进程有很多相关资源 (内存、文件等），故进程是 OS 资源管理的基本单元。
2. xv6 的用户线程与内核线程
当程序在 user space 运行时，使用的是用户线程；当该程序 trap 到内核时，使用的是内核线程。
即一个进程有用户、内核两个线程（虽然属于不同的页表）。
由于 xv6 一个进程同时只能运行一个线程，故所有的用户线程不共享内存（除非刻意，例如 COW fork）。
而根据 xv6 设计，kernel page table 映射了所有进程的 kernel stack，故所有进程的内核线程共享内存。
3. xv6 的调度器线程
该线程承继 boot 时所用的线程，没有对应的进程。
scheduler 线程的 stack 在 _entry 函数中开辟。
4. xv6 的 schedule (high level view)
(1) 利用 timer intr 强制将 CPU 控制权从 user space 转移到 kernel space，即从某个进程的用户线程转移到这个进程的内核线程。
内核线程再调用 yield，资源释放 CPU 控制权。
之后 scheduler 线程拿到控制权，开始调度。
前者为 pre-emptive schedule, 后者为 voluntary schedule.
(2) 为支持调度，需要区分进程（线程）状态：
RUNNING: CPU 当前正在运行的线程
RUNNABLE: 可以被 resume 的线程。
SLEEPING: 调用 sleep，主动 yield 的线程。例如在等待 I/O，当前不希望被调度。
(3) 一个 CPU 同时只能运行一个线程。且一个线程不会同时被多个 CPU 运行。
(4) 因为 timer intr, 从一个用户线程切换到另一个用户线程的过程：
(a) 进程 A 因为 timer intr，引发 trap，从用户线程切换到内核线程。
(b) 内核线程 yield CPU，切换到调度器线程。
(c) 调度器线程 schedule 到进程 B 的内核线程, resume 它。
(d) 进程 B 的内核线程继续执行，直至 trap out. 则进程 B 的用户线程继续执行。
(5) 几种 context switch
(a) 由一个用户线程切换到另一个用户线程，包含 (2), (3).
(b) 由同一个进程的用户线程切换到内核线程
(c) 由某个进程的内核线程切换到调度器线程，或相反。
(6) context 与 trapframe 的区别：
context: 保存某个内核线程的状态.
trapframe: 保存某个用户线程的状态，以及与 trap 相关的信息。
(7) swtch 过程中被挂起的那个线程不能持有锁。因为 swtch 后，它就不运行了，则不会释放锁。而 resume 的线程可能会 acquire 这个锁，则导致死锁。
5. 因为 timer intr, 从一个用户线程切换到另一个用户线程的完整过程
(1) trap 进内核。到达 usertrap 函数。即从用户线程切换到内核线程。
(2) devintr 函数返回 device num = 2, 即 timer intr. 则调用 yield.
(3) 因为可能有多个 CPU，为防止其他 CPU 看到 inconsistent 的进程状态，故先该进程的锁，再更改 state 为 RUNNABLE，即挂起该内核线程。
(4) yield 调用 sched，进行一些检查，之后调用 swtch 函数，将该内核线程的 context 存到 cpu->proc 中，再取回原来存在 cpu->proc 中的 context。即调度器线程的 context.
注意 swtch 函数只 swap 了 callee saved registers 以及 ra register 的值。这是因为其他 registers 的值被保存在 kernel stack 中。
(5) swtch 函数重点在于设置 ra 及 sp，再加上隐式取回的 caller saved registers 的值，即 swap 了线程状态，即 context.
注意不需要存取 pc，因为 ret 指令会将 pc 设为 ra.
swap 寄存器后，swtch 函数已经隐式地从 sched 中的 swtch 跳到了 scheduler 中的 swtch，即调度器线程被 resume.
(6) 调度器线程 linear scan 出一个 RUNNABLE 的线程，调用 swtch，将调度器线程的 context 暂存到 cpu->proc 中，并激活所找到的这个内核线程的 context.
(7) 这个内核线程接着执行。例如也是由于 timer intr 而 trap 内核，故会回到 sched -> yield -> usertrap。之后调用 usertrapret trap out.
至此，这个内核线程对应的用户线程被 resume.

pipe

1. pipe 基本原理
alloc 一个 pipe 即在内核中创建一个 pipe struct，包含一个 data buf, lock, 并暴露出 read/write fds.
线程通过 read/write fds，进行 data 传输。传输时，data 暂存到 data buf 内。这个 data buf 通常为 circle buffer array.
buf 大小固定，在满时会 block，直到某个 consumer consume 了。在空时也会 block，直到某个 producer produces 了。
2. xv6 实现
(1) pipewrite
先获取 pi->lock。
如果 buf 满了，wakeup consumers. 这里的 chan 设置为 pi->nread，即只 wakeup 那些正在等待第 nread 个 data 的 consumer。
这可以使 coordination 更 efficient，因为可以减少 spurious wakeup 和 thundering herd.
wakeup 后，sleep。这里的 chan 设置为 pi->nwrite，道理同上。
write 完成后，wakeup pi->nread
释放 pi->lock.

(2) piperead
先获取 pi->lock.
如果 buf 空，直接 sleep on pi->nread，即设置 chan = pi->nread。
read 完成后，wakeup pi->nwrite。
释放 pi->lock.

paging

1. 一个 RISC-V 处理器包含多个 cores。每个 core 包含 CPU, MMU, TLB 等。CPU发出的指令会经过 MMU 转译，即 va->pa.
2. MMU 首先查看 TLB 中是否存了该 va 的 cache，如果有，则直接取。
3. 如果没有，MMU 利用 CPU 提供的 satp 寄存器中的值，到物理内存某个位置读 page table。利用该 page table，找到 va 对应的 pa。并更新 TLB。
4. page table 怎么翻译 va->pa 的？
(1) 首先要明确，内存被组织为 pages，即内存的存取以 page 为单位进行（e.g. swap in/out）。xv6 选择 page size = 4096 bytes.
(2) 另一方面，地址的读写以 byte 为单位进行。由于 page size = 4096 bytes = 2^12 bytes，则在页内部进行 index，需要 12 bits.
(3) xv6 又设计了 2^56 bytes 大小的物理地址空间，56 - 12 = 44，则 page table 中的每个 PTE 除了存 flag bits，至少还需要存 44 bits. 向上取整，则每个 PTE 占用 64 bit，即 8 bytes.
(4) 则知一个 page 可以存 4096 / 8 = 512 个 PTEs。index 这些 PTEs，需要 log512 = log 2^9 = 9 bits.
(5) 综合考虑效率与空间占用，xv6 选择了三级页表。则总共需要 3 * 9 = 27 bits。加上用于页内 index 的 12 bits，故总共需要 39 bits.
5. 每次 switch page table，都需要 flush TLB，以 invalidate old mappings.

一些问题：
(1) 为何页表保存在内存中？
因为 MMU 容量不足。
(2) 任意时刻，页表所有内容都在内存中吗？
否。可能有些被 swap 到了 disk 中。
(3) 倒排页表？
以物理页为 key，维护和访问一个表，表中 entry 存映射到该物理页的所有虚拟页，以及一些 flags，
例如 ASID，即标识该虚拟页是哪个进程的（当然可能是“内核”）。
显然，这样可以最大限度地节省内存，因为是纯粹的按需分配。
然而，一个显然的问题是需要 scan 这个倒排页表，才能完成 va-pa 映射，则提高了复杂性。对于是否降低效率，还不好说，
因为即便是 scan，也可以用哈希表支持，则可能比多级页表更快，毕竟不需要额外的内存访问来 walk 多级页表。
(4) 基于 paging, TLB, swap space 的一次完整的地址翻译流程
用户程序发起一个内存访问，CPU 拿到 va.
CPU 将该 va 转发给 MMU.
MMU 转发给 TLB.
从 va 中拿到 VPN，作为 key 访问 TLB，拿到 TLB entry，回到 MMU 中。
如 TLB hit，根据 flags 判断是否可以访问，例如 protection bit，U/S bit 等。
根据结果做相应操作，如合法 va，则从 TLB entry 中拿到 PPN，再组合得到 pa.
如非法，则 raise exception（PROTECTION_FAULT），让 OS 去处理。
如 TLB miss，则根据 PTBR 和 VPN ，拿到 PTE（如果是多级页表，则根据 PDBR 拿到下一层的页表起始位置，再拿到 PDE。
如此 walk，直到最后一层。如果中间某个页表没有分配，可以选择分配）。
根据 PTE 中的 flags，做相应操作。
（1）如合法，则从 PTE 中拿到 PPN，与 VPN 组成 VPN-PPN key-value pair，
存到 TLB 中，当然 TLB entry 中也要存与 PTE entry 中存的一些 flag bits.
之后，则重新执行该内存访问指令，此时，则为 TLB hit.
（2）如非法，则 raise exception，根据 flags 情况，判断。
如 invalid，则为 SEGMENTATION_FAULT；
如valid 但是 权限不对，即 protection bit 不对，则为 PROTECTION_FAULT；
如 valid ，但不 present，则为 PAGE_FAULT。都让 OS 去处理。

OS 的 page fault handler 的一般逻辑为：
从空闲的物理页中，选一个，作为承载换入的磁盘页的空间。
如果没有空闲的，则要么自己执行页置换 routine，将一个物理页换出到 swap space，则 block 等它执行完毕。
要么通知 swap daemon ，让它以 clustering 的方式，批量 evict 一些页，将 low watermark 调整到 high watermark。
swap daemon 执行完毕之后，再通知 page fault handler。
腾出内存空间后，执行 disk I/O 从磁盘中拷贝目标页。
拷贝完毕后（通常为 block，然后等 disk 通知），设置引发 page fault 的虚拟页的 PTE，
设置 PPN 为承载该换入页的地址，以及设置 present bit 为 1 等。
重新执行该内存访问指令。

关于谁去 insert 该页到 TLB 中，根据 hardware-managed 或 OS-managed，来决定。

page fault

核心思想：惰性（laziness，即 on demand）。
将一些工作推迟到之后真正需要时才执行，可以提高程序在当前的反应速度（responsiveness）。
更重要的是，很多工作事实上并不真正需要，而是一种冗余或保险。

1. lazy allocation (惰性分配, 又称 on-demand allocation)
(1) motivation: 用户程序在动态分配内存时，通常有很大的冗余量，例如一个很大的 buffer ，实际上只有很小一部分会被用到。
由于物理内存的分配比较慢（例如隐式链表结构），因此如果只记录地址空间的分配情况，而不立即进行物理内存分配，
只在有需要时（on demand）时才进行分配，这样可以大幅提高动态内存分配的性能。
(2) implementation: 在 sys_sbrk 中，仅修改 p->sz，而不调用 growproc 以真正分配物理内存并 install 页表。
在根据 p->sz 进行页表访问时，如果遇到 present bit 为 0 的虚页，则会 trap 进内核（present bit: 表明该页是 present 在内存中，还是 present 在磁盘中。为0则不 present 在内存中）。
根据 scause 的值，内核中的 usertrap 函数会 判断是否是 page fault。由于只有 heap 段才会使用 lazy allocation，因此只需判断是否是 load/store page fault。如果是，内核会引导进 lazy allocation page fault 的 handler。
这个 handler 首先判断这个内存访问是否是合法的，即是否在 [heap_start, p->sz] 之间。如果是，再调用 kalloc 进行物理内存的分配。如果此次因为物理内存用完了而分配失败，则会 kill 这个 faulting process.
为了支持该机制，很多由于 walk, walkaddr 而引发的 panic，均因被 comment out，因为这是正常的行为。
(3) Linux 相关： 在 linux 系统中，top 命令会显式系统的实时信息，其中有两栏 VIRT 和 RES，分别对应地址空间的大小与真正分配的物理内存的大小。
通常来说，RES 的值会远小于 VIRT 的值。
2. copy-on-write fork (写时复制)
(1) motivation: 由于 fork 会复制父进程完整的地址空间，包括页表与物理内存，因此此处有两个点非常影响性能：
(a) 找到空闲的物理内存。可能从空闲链表中取，或者从预分配内存池中取，也可能 evict 一些物理页，以腾出空间，这甚至涉及到磁盘 I/O。
(b) 将父进程的物理内存中的内容完整拷贝都子进程物理内存中。
因此 fork 的性能较差。
进一步，这些复制的物理内存，在 fork + exec pattern 中实际上并没有被真正使用。
COW fork 即为：在 fork 时，只复制页表，而并不复制物理内存，可以提高性能。
(2) implementation: 首先要明确，COW 只会发生在可写的虚页上，相应地，也只会产生 store page fault。
其次，由于一些 物理页 被共享，因此在 free 时，必须判断所有 spy on 这个物理页的进程（通过页表 spy on）都放弃了对该页的共享。由于 xv6 用于普通程序的物理内存只有 [kernel_data_start, PHYSTOP] 这一段（注意到由于这一段与内核地址空间是 direct mapping，我们才能使用内核的段来界定），总共大小为 0x0640'0000，而页大小为 4096，因此总共也就 32768 个页，故可以使用固定大小的数组作为 counter，对共享页的引用进行计数。

在 fork 时，uvmcopy 进行父、子进程虚拟地址空间的拷贝。对于可写的虚页，父、子进程的 PTE 都清除 可写标记，并标记为 COW。COW bit 取自 RSW bit，即 reserved bit（实际上并不是必须要额外设计一个 COW bit，但这样比较简单，其他怎么做不明）。
在 usertrap 中，通过 scause 识别 store page fault，即 COW page 访问。涉及相应的 handler，处理它，主要逻辑为：（1）判断 faulting va 是否合法，不能是 guard page 或 只读段，例如 trampoline 等。（2）判断 faulting va 是否对应一个 COW 页，即必须有对应的物理页，必须不可写（因为之前创建时设置为不可写），COW bit 必须被 set，用户空间必须有权限（即页表的复制仅发生在用户页表，由于内核页表只有一个，不可能复制）。创建一个新物理页，并修改 faulting process 的页表中的 pte 的映射关系，即修改它的 PPN（当然还有 flags），把 COW 页的物理内存内容完整拷贝过来。最后尝试 free COW 物理页，因为 faulting process 不再 share 这个页了。

注意到，在 handle 之前，先检查引用计数是否已经为 1（不可能为 0），如果是，则说明已经不需要共享了，则直接重设 flags，即标记为 可写 和 非 COW，直接 trap out 内核即可。这样可以减少一次复制，考虑一个进程 a ，fork 了一个进程 b。进程 a 马上放弃了对这些物理页的共享（例如通过写这些页，或销毁进程），但是进程 b 不知道。故在它写这些页时，还是会发生一次由于 COW 引发的复制。
但这个方法还是会有一次 store page fault，而它需要 trap in/out 内核，故消耗还挺大。为了减少这次消耗，可以通过两个方式做到（个人思考，没有验证及实现）：（1）由于 xv6 只有固定的 64 个进程，因此可在 boot 时对它们创建并查集，如果两个进程有共享的物理页，则将它们 cluster 到一起。（2）由于 fork 创建的子进程自动与父进程分到一组，因此可以通过信号的广播进制，在一个进程放弃共享时，通知其他进程。

注意到，由于这是 page fault，因此当 trap out 到用户空间，会重新执行 faulting epc，则 all goes well。

为了维护共享物理页的引用计数，（1）在 kalloc 中，置计数为 1。（2）在 fork 时，增 1。（3）在 kfree 中减 1，如果减到了 0，则才真正 free。

1. demand zero (aka. zero fill on demand)
(1) motivation: 当调用 exec 时，会将可执行文件装载进内存中，对 ELF 文件不同的段，可以有不同的内存分配方式。
对于 bss 段，即未显式初始化或初始化为 0 的全局变量或静态变量所在的段，可以将它们对应的虚拟内存都映射到一个 填充 0 （zero fill）的物理页。
这样在 exec 时，效率就更高。同时，与 lazy allocation 类似，这样显然可以节省物理内存。
(2) implementation（未验证）: 首先要明确，只有 data store 才会引发对 demand zero 页的改变。
将 demand zero 的页对应的虚拟页设为只读，并（可能）需要 RSW bit ，给 OS 做标记，表明这是一个 demand zero 页。
然后当写这个页时，新分配一个页（注意置 0，保险操作），再重新映射 va-pa，注意重设 pte，使得该虚拟页不再是 demand zero 页。
最后 trap out ，重新执行 spec，all goes well.
2. demand paging
(1) motivation: 很多可执行文件并不是只有少数几个函数，而是十数个甚至更多个函数，即 text 段很大。对于 data 段来说也是如此。
然而每次执行并不是都会用到其中所有的这些 text 和 data。因此可以在 exec 时，不采取 eager 的方式装载，而是仅给它们分配页表 pte，并将它们设为 invalid。
之后，当程序真正需要访问它们时，会引发 page fault，从而将它们真正装载进内存。
如此，可以节省物理内存和提高 exec 的执行效率。
(2) implementation: 大部分和 demand zero 类似。需要提的是，内核需要在某个地方（例如下面所述的 VMA 结构体）存储这个 page 对应的文件信息，如此才能在某个虚拟页 fault 时，从文件中拉取内容。
3. memory-mapped files (内存映射文件)
(1) motivation: 将完整或者部分文件加载到内存中，这样就可以通过内存地址相关的load或者store指令来操纵文件。
(2) implementation: mmap syscall:
这个系统调用会接收一个虚拟内存地址（VA），长度（len），protection，一些标志位，一个打开文件的文件描述符，和偏移量（offset）。
这里的语义就是，从文件描述符对应的文件的偏移量的位置开始，映射长度为len的内容到虚拟内存地址VA，同时我们需要加上一些保护，比如只读或者读写。
通常以 lazy 的方式实现，即：不会立即将文件内容拷贝到内存中，而是先记录一下这个PTE属于这个文件描述符。
相应的信息通常在VMA结构体中保存，VMA全称是Virtual Memory Area。在VMA中我们会记录文件描述符，偏移量等等，
这些信息用来表示对应的内存虚拟地址的实际内容在哪，这样当我们得到一个位于VMA地址范围的page fault时，
内核可以从磁盘中读数据，并加载到内存中。

interrupt

1. 中断的分类与区别
@source [https://blog.csdn.net/imilano/article/details/83690133](https://blog.csdn.net/imilano/article/details/83690133)
2. 硬件外部中断 与 syscall 的区别
(1) syscall 是 trap，即有意的中断。而硬件中断不可控。
(2) 硬件中断是 async 的。syscall 是 sync 的。即 syscall trap 进内核后，进程没变（页表变了），即还是同一个进程。
而硬件中断会单起一个进程，从而与 CPU 当前正在执行的进程并行。
3. PLIC (Platform Level Interrupt Controller)
一个 RISC-V 处理器包含多个 CPU cores 及一个 PLIC。所有的外部设备与 PLIC 相连。
当一个中断发生时，PLIC 做如下操作：
(1) 通知所有 cores，当前有一个新的 intr 待 handle.
(2) 在一个 core claim 这个 intr 之前，PLIC 会 hold 这个 intr，即将 intr info 保存在 PLIC 内部某个地方。
(3) 当一个 core claim 之后，PLIC 标记该 intr 为正在 handling.
(4) core handle 完后，通知 PLIC，PLIC 再删除存储的 intr info.
4. OS 如何与外部设备交互?
在 kernel 启动时，会将设备的 I/O 接口映射到内核的特定 va 地址段，即构建 memory-mapped files.
再通过普通的 load/store 指令，即可与设备进行交互。OS 操作的对象是设备暴露出来的寄存器，具体来说是 control registers。
即通过更改这些寄存器的 bits，可以控制设备的行为。
5. 需要哪些操作 xv6 才能接收中断？
(1) 与中断有关的寄存器：
(a) sie 寄存器：SIE = supervisor intr enable. 其中 E bit 控制接收外部中断，S bit 控制接收软件中断，T bit 控制接收 timer intr.
(b) sstatus 寄存器：里面有一个 SIE bit，控制接收所有中断，即 sie 的几个 bits 控制的。
(c) sip 寄存器：sip = supervisor intr pending. 中断发生时，中断的类型 num 被存在这个寄存器中。
(d) sscause 寄存器.
(e) stvec 寄存器.
(2) 设置发送中断的外部设备: 通过给定的 device manual，利用 mmap 将设备暴露的寄存器映射到 va 特定的地址段。通过写这些地址，来配置设备。
(3) 设置 PLIC，以路由设备。方法同 (2).
(4) 设置 CPU 相关寄存器，即 sie 寄存器与 sstatus 寄存器，使能 CPU 接收中断。
6. xv6 启动时，命令行的第一个 '$' 如何打印出来的？
@source [https://mit-public-courses-cn-translatio.gitbook.io/mit6-s081/lec09-interrupts/9.5-uart-driver-top](https://mit-public-courses-cn-translatio.gitbook.io/mit6-s081/lec09-interrupts/9.5-uart-driver-top)
7. 用户程序写入的字符，是如何显示在 console 中的？
@source [https://mit-public-courses-cn-translatio.gitbook.io/mit6-s081/lec09-interrupts/9.6-uart-driver-bottom](https://mit-public-courses-cn-translatio.gitbook.io/mit6-s081/lec09-interrupts/9.6-uart-driver-bottom)
8. 键盘输入的字符，是如何显式在 console 中的？
@source [https://mit-public-courses-cn-translatio.gitbook.io/mit6-s081/lec09-interrupts/9.8-uart-read-keyboard](https://mit-public-courses-cn-translatio.gitbook.io/mit6-s081/lec09-interrupts/9.8-uart-read-keyboard)
9. intr 两种方式：polling 与 intr
短时间内中断数量大，例如高性能网卡，应该使用 polling，因为如果使用 intr，则 CPU 的 cycles 全部用来 handle intr 了，而没有做实际的工作。
10. programmed I/O vs. DMA (direct memory access)
programmed I/O: read/send data a byte at a time from/to control register of the device. 需要 CPU 参与。
DMA: read/send data from/to memory. 不需要 CPU 参与。

fs

1. xv6 创建一个文件的流程.
(1) 用户空间调用 open wrapper func, 传入要创建的文件的名称，并设置 omode 为 O_CREATE，表示创建一个新文件。
(2) 内核空间调用 sys_open func，从用户空间复制 open 函数参数，即 filepath 和 omode flag. 根据 omode 中 set 的 O_CREATE bit，
则会调用 create 函数执行文件创建。
注意，在此之间，会调用 begin_op，表征 txn 的开始。因为 create 涉及到 on-disk data structrue 的 modification.
(3) create 函数首先检查 filepath 的合法性，即必须在一个已经存在的 父path 下。再检查该文件是否已经被创建，如果是，直接返回。
再调用 ialloc 函数分配一个 inode 给这个新文件。
(4) ialloc 函数从 disk 中的 inode 段中依次读取 inode （以 block 为单位，利用 bread 读取）。如果某个 inode 的 type 为 0，则表示是一个
unallocated inode，则通过设置 type 字段为给定的 type（例如 file, device, etc.），表示该 inode 已分配。
每个 inode 对应一个 inode num，即该 inode 是 disk 上的第几个 inode slot。ialloc 以该 inode num 为参数，调用 iget，将 on-disk inode 的
in-memory inode 返回给 create 函数。
(5) xv6 在 kernel space 中专门开辟了一段固定大小的内存给 inodes，以 cache in-memory inodes. iget 首先检查这个 inode （由 inum 表征）是否已经 cache 了。
如果没有 cache，则找到一个 empty cache slot 给这个 inode。
注意此时尚未从 disk 中读取 inode（ip->valid = 0 表示还未从 disk 读取），只是将 inode cache slot 的 inum 设置了。
(6) 之后 create 函数调用 ilock，将该 inode cache slot 锁住，并复制 on-disk inode 到 in-memory inode 中（如果尚未读取的话）。此时 ip->valid = 1.
(7) 之后 create 函数调用 iunlockput，即首先调用 iunlock 解锁 inode cache slot，则该 slot 可被复用。再调用 iput。
iput 会检查 in-memory inode 的 ref 与 nlink fields。每次 iget 会 ++ref，而每次 iput 会 --ref，即 ref 代表了当前有多少个 C pointers 指向了这个 inode.
这些 C pointers 可能是 fd, cwd 或者由于 exec 临时打开文件所致。nlink 则表示该文件作为 dirent 出现了几次，即有多少个 link（初始时，nlink = 1）。
如果 iput 检测到 ref = 1 以及 nlink = 0，则该文件不再被目录引用，同时也无其他 C pointers 指向它，则 iput 会调用 itrunc 将该 inode 的字段抹去。
对于一个新创建的文件，其 nlink = 1，则其不会被 free.
注意，由于 inode 中存了 ip->addrs ，即 direct/indirect data blocks 的引用，则会首先 free 这些 data blocks，即通过调用 bfree 在 bitmap block 中标记这些 block 为 free。（实际 content 会作为 grabage 留下来）
再置0这个 in-memory inode。
注意，每次修改 in-memory inode，都要立即调用 iupdate，同步更新 on-disk inode. 这是因为每次 ilock 后，会调用 iunlock。则 inode cache slot 可能会被复用。(FIXME:??)
(8) 之后调用 filealloc，从全局的 file table 中，找到一个 ref = 0，即空闲的 file slot 给这个新文件，返回这个 slot 的 file pointer.
(9) 再调用 fdalloc 从当前 proc 的 open file 中，找到一个空闲的 file slot，给这个新文件，即将之前返回的 file pointer 存在这个 file slot 中。
这样的设计，使得同一个文件可以被多个 proc 打开。但是共享一个全局的 file，即共享 offset.
(10) 再将 f->ip 与之前的 ip 联系起来，f->readable/writable，即权限。至此新文件创建完成，返回 fd 给用户空间。
2. xv6 写入一个文件的流程.
(1) 用户空间调用 write wrapper func, 传入参数 fd, buf addr, write length。trap 进内核.
(2) 内核空间调用 sys_write func，从用户地址空间复制 write 函数参数，其中 fd 作为当前 proc 的 file table 的 index，以 fetch 对应的 file pointer.
(3) sys_write 调用 filewrite. filewrite 则根据 file 的 type field，调用具体的 write 函数。例如 pipe 对应的 pipewrite, 某个 device 的 devsw。或者文件对应的 writei.
(4) 对于文件的写入，由于 xv6 使用了 write ahead rule 的 logging，且在 disk 中分配了一个 fixed size 的 log space （由 superblock 中的 logstart 与 nlog fields 界定，单位 #blocks），即限制了一次文件写入所能写入的最大大小。
故对于大文件写入，xv6 会将其分为多次小文件写入，每次小文件写入 wrap 进一个 txn.
(5) 对于一个 txn，首先调用 begin_op，标志 txn 的起始。
(a) 如果当前已经有一个 txn 在 commit，当前这个 txn 会 sleep。这是因为如果不 sleep，那么由于 txns 共用同一个 log space，
那么正在 commit 的这个 txn 可能会 commit 当前这个 txn 新写入的 log，显然是不对的。
(b) 由于 log space 是 fixed size 的，xv6 conservatively 为每个 txn 设置了一个最大能使用的 blocks 的数量 MAXOPBLOCKS，同时用 log.outstanding field 记录了当前尚未 commit 的 outstanding 的 txns 的数量.
如果加上当前这个 txn 之后，可能会 overflow log size，则当前这个 txn 也要 sleep.
(c) 如果以上都通过，则将 log.outstanding +1，之后开始这个 txn.
注意，begin_op 开头 acquire 了 log 的 lock，当其他 txn commit 后，会 release lock，并 wakeup sleep 在 begin_op 中的 txn.
(6) 调用 ilock，锁住每个小文件对应的 inode slot，这样 xv6 不会将 slot 分配给其他文件。如果 inode->valid = 0，表明该 inode 还未 cache 在内存中，则会先利用 bread 与 buffer cache 层交互，将该 inode 对应的
on-disk inode fetch 到某个 buf 中，在将 buf->data cast 成 dinode，以复制 on-disk inode 至 in-memory inode.
(7) 之后调用 writei 将用户要写入的内容（存在内存某个区域，用户空间或者内核空间），写入到分配给该 inode 的 data blocks 中。
(a) writei 首先做一些合理性检查，再检测该次写入是否会超出最大文件大小，如果检查不通过，则 return -1 以 indicate error.
(b) 再根据当前的 off 与 BSIZE （即 block size），计算该次写入所应写入的 bn。每个 inode 内部有个 addrs array，其 index 即 bn （区别于全局的 blockno）.
该 array 大小为13，前12个 slot 保存 direct data blocks 的 blockno，第13个 slot 保存一个 indirect data block 的地址，该 block 中存的是其他分配给该 inode 的 data blocks 的 blockno。
如果根据该 bn 所 index 的 addrs array 对应的 slot 为空，则说明需要为该次写入新分配一个 data block. 如果该 bn <= 12，则分配一个 direct data block，如果超出了，
则需要在 indirect data block 中新增一个所分配的 data block 的 addr.
(c) writei 则将该调用 bread 将所需的 data block fetch 到 buffer cache 中. 再将用户要写入的内容复制到对应的 buf->data 中。
(8) 每次将数据成功复制到 buf->data 之后，表明用户要写入的数据已经 cache 在了 in-memory buffer 中，但还没有写入到 disk 中。此时调用 log_write，在 log 中 append 一个新的 block modification 记录，
即将本次 writei 写入的 block 的 blockno append 到 log header 的 log array 中，并 increment log header 的 n 字段，表明新增了一个 log record.
注意，在 append 之前，会先检查该 txn 在之前是否已写入了这个 block，如果是，则不新增，如此可减少 log record，方便 recovery 并 save log space。此即 log absorbtion.
另一个要注意的是，对于新增的 log record，会调用 bpin 以 increment 这个 buf 的 refcnt。这是因为之前是用 bread 将某个 disk block 读入到 buffer cache 中，而每个 bread 都会 follow 一个 brelse。
则在 bread 中 ++refcnt 会在 brelse 中 --refcnt，则由于 refcnt 可能为0，则这个 buf 可能被 evict。而根据 write ahead rule，所有数据在 commit 之前不能写入到 disk 中。故这样显然是不对的。
为了防止被 evict，则需要调用 bpin ++refcnt.
(9) 当 writei 执行成功后，调用 iunlock，将 inode slot 解锁，则该 slot 可被分配给其他文件。(FIXME: ??)
(10) 调用 end_op，其首先 decrement log.outstanding，表明这个 txn 的结束。之后判断是否因为此次 decrement，导致 log.outstanding = 0。如果是，则由这个 txn 完成 commit，即调用 commit 函数。如果不是，则将 commit 的任务留给最后这段时期最后的
那个 txn. 此即 group commit，即虽然每次 file syscall 都会有 begin_op, end_op，但一段时期内开展的多个 syscall 会被 wrap 到一个 txn
(11) commit 分为四个步骤:
(a) 首先调用 write_log，将 in-memory log cache 写入到 on-disk log space 中。即根据 log header block 中存的 log blockno，从 buffer cache 中 fetch 对应的 buf (一定在，因为 log_write 会 pin 住它们)，将它们的 data 写入对应的 log space 中。
(b) 再调用 write_header，将 in-memory log header block 写入到 on-disk log header block 中。当此次写入完成之后，这个 txn 即真正地 commit 了。
注意，disk 在硬件上保证一个 block(sector) 的写入是 atomic 的。
注意，一定是先写 log blocks 再写 header block.
注意，所有的写入都是通过 buffer cache 层，即不是直接写入到 disk，而是通过 buffer cache 层提供的 bwrite 函数。
(c) 再调用 install_trans，将 on-disk log blocks 写入到 on-disk data blocks 中（通过 buffer cache 层）。
(d) 再将 header block 的 n 清 0，表示没有待 install 的 log.
3. crash recovery
(0) 关心的东西：文件系统操作不要使得文件系统陷入一个 inconsistent 状态。
这里的 inconsistent，举例说明：
当创建一个新文件时，包含两个步骤（并不止）：
(a) 分配一个新的 inode，则要在 inode block 中找一个 idle 的 inode entry，将其 type 设置为非 0，即已分配。
(b) 将该新文件添加到其所在的目录，即更新某个 type 为 dir 文件的 data block.
如果 pc 在这两步之间 crash 了，则我们无法在任何一个目录中找到这个文件，然而根据 inode entry，这个 inode 已经被分配了。
其后果就是，无法 GC 这个 inode，则导致系统资源浪费。

注意，并不关心用户的意图是否实现。举例说明：用户想写入一个大文件到文件系统。然而这个文件太大，以至于 logging 系统无法将其 wrap 进一个 txn，
则某些文件系统通常会将该大文件拆分成多个小文件，再分别写入。文件系统只会保证每个小文件的写入不会引发 inconsistent 状态，并不会保证所有内容都被写入。
(1) xv6 在 boot 时，会启动第一个 user proc，即运行 shell。为了运行 shell，会遵循一般的 trap out 路径，其中会调用 forkret。
对于第一次 forkret 的调用（由一个全局量 first 表明），会调用 fsinit 函数，以初始化 in-memory 文件系统，其中即包括了 recovery.
在 shell 的内核线程调用 fsinit，而不再 boot 的 main 中调用，是因为 main 所在的 scheduler 线程本身不能被 sleep，而 fsinit 会调用 sleep.
(2) fsinit 首先将 superblock 从disk中读取到 buffer cache 的某个 buf 中。再调用 initlog 函数。
(3) initlog 函数从 superblock 中读取 on-disk log space 的 start 和 size (单位: block)，再调用 recover_from_log 函数。
(4) recover_from_log 首先调用 read_head，读取 log header block 到内存中。再调用 install_trans，将 on-disk log blocks 和应写入的 block 都读取到内存，再在内存中进行数据复制.
写入到 disk 则通过调用 buffer cache 层的 bwrite 函数完成。
写入完成后，清空内存中 log header block 的 n field，并同步更新到 disk，以表征所有的 log 已经 install 了。 至此，recovery 完成。

1. xv6 读一个文件的流程.
略。

fork 

(1) fork 调用 alloc_proc 分配一个进程。
(a) alloc_proc 首先在 proc table 中找到一个空闲的 slot，拿到对应的 proc struct.
(b) 为这个 proc 分配一个 pid（xv6 设计 pid 为全局递增）.
(c) 再分配一个 physical page 作为这个 proc 的 trapframe.
注意由于 PHYSTOP 之下都是 direct mapping，所以 user/kernel 都可以使用这个 trapframe 完成 trap in/out.
(e) 再将这个 trapframe 和 trapoline page 都映射到这个 proc 的用户地址空间。构建出一个基本的页表。
(f) 最后设置 p->context.ra = forkret，即一个新创建的进程在被 schedule 到之后，会从 forkret 开始运行，模仿一般的 trap out.
(2) fork 再 将父进程的页表复制给子进程。即根据父进程页表的 PTEs，填充子进程的 PTEs。
如果没有 COW，则对于父进程映射的每个物理页，都分配一个新的物理页，并复制内容。即父子进程拥有完全独立的物理空间，虽然它们所存储的内容是完全一致的。
(3) 将父进程的 context 复制给子进程，这样当子进程 trap out 之后，会和父进程从同一个指令开始运行。
(4) 设置 proc->parent 为父进程，构建父子关系，用于之后的 reap.
(5) 对于父进程打开的每个 file，即 ofiles 数组，调用 filedup，++ref，表示有一个新的进程打开了对应的文件。
(6) 将 proc->trapframe->a0 设为 0，这样子进程 trap out 之后，fork wrapper 会返回 0。而父进程会返回子进程的 pid.
(7) 最后将 proc->state 设为 RUNNABLE，这样子进程就可以被调度了。

exit wait kill

1. 需要明确的点
(1) 进程自己不能完全 kill 自己，因为 kill proc，意味着回收页表，而进程的执行需要页表。
故如一个进程要 kill 自己，需要先由自己申请，再让他人完成进程清理工作。
由于每个进程必定有父进程（init 进程除外），因此该 “他人” 即每个进程的父进程。
(2) 某个进程不能安全地强制销毁另外一个进程。因为另外一个进程可能在某个“原子”操作的中间位置，例如写入文件系统的某个 txn.
P.S. xv6 中没有权限概念。在 Linux 中，每个进程属于某个用户，只有属于相同用户的进程才能 kill 彼此。
2. kill 自己
(1) high level view
进程调用 exit，将自己设为 zombie。
父进程调用 wait，检测到该孩子为 zombie，销毁它的页表，回收该进程。
(2) xv6 exit 实现
首先关闭进程打开的文件，此处涉及到 inode 引用计数，比较复杂。
再 wakeup init 进程。（此步非必需，但 harmless）
获取父进程的锁，再获取该进程的锁.
因为调用 wait 的是父进程，而当它 scan proc table 时，才获取子进程的锁。故必需按照这个顺序获取锁以避免死锁。
此处还需要注意的是，因为两次获取锁之间有间隔，在这个窗口期内，该进程的父进程可能将该进程过继给 init。
故在获取父进程的锁之前，用一个变量保存当前进程的父进程，如此才能保证待会解除的一定是同一个父进程的锁。
将所有孩子过继给 init 进程。
wakeup 父进程，因为父进程可能 sleep 在 wait 中。
将 state 设为 zombie。
调用 sched，即释放 CPU。
(3) xv6 wait 实现
wait 首先获取当前进程的锁。因为子进程在 exit 时，会获取父进程的锁（即这里的该进程），这是为了应对下面这种情况：
子进程尚未执行 exit，父进程调用 wait，发现没有 zombie 的 kids，则 sleep on wait。condition lock 为父进程的 p->lock.
则不会 lost 子进程的 wakeup.
其他情况都可以正常进行。
scan proc table，根据 np->parent，找到自己的 kid。如其 state = zombie，将 exit status 复制到给定的地址。再销毁其页表（包括回收物理内存）。
释放该进程的锁，再释放子进程的锁。
3. kill 他人
根据需要明确的点（2），kill 唯一安全的操作就是将给定的进程的 state 设为 killed.
如给定的进程在 sleeping，wakeup 它。
则给定的进程会继续执行，知道因为 syscall 或 timer intr trap 进内核，然后在 usertrap 中检测到 killed = 1，再自己调用 exit。
其他某些 while loop 也会检测 killed（如果它们认为打断这个操作是安全的）。

exec 

exec(path, argv)
(1) 首先调用 namei 解析 path，descend into directory，将目标文件从 disk 加载到 内存。
(2) 首先从内存中复制 ELF header 到 exec 的局部内存中，根据 MAGIC num 确认该文件确实是 ELF 格式的可执行文件。
ELF格式的可执行文件由一个 ELF header 和多个 program section header 组成。前者描述所有 sections，后者则描述具体的单个 section.
xv6 只有一个 section，即没有区分 data, text, etc.
(3) 根据 ELF header，对于每个 section，首先从 disk 加载到 kernel 内存中， 再调用 loadseg 复制每个 section 到用户空间指定的位置。
位置由 section header 中的 vaddr 指定。
注意在复制之前，需要在用户创建一个新的页表，并分配 PTE 及对应的物理内存。
此时，用户页表已经映射了 trampoline, trapframe 以及 sections.
(4) 接下来，为用户页表创建 stack. 注意并不是直接在用户页表创建，而是现在 kernel 中模拟一个 stack，再直接复制到用户页表对应的位置。
(a) 维护一个 sz 变量，初始值为 0，表示用户页表起始地址为 0. 在刚才 load sections 时，sz 已经更新到了最后一个 section 的末端。
(b) 执行 sz += 2 * PAGESIZE，隐式地表示为用户空间分配两个物理页。其中上一个物理页作为 user stack，下一个物理页作为 guard page，即将对应的 PTE 的 PTE_U bit unset.
(c) 再维护一个 sp 变量，表示 user stack pointer，初始时为 sp = sz，即指向 user stack 的底部。注意 stack grows downward.
(d) 对于 exec 的参数 argv，其类型为 char**，即 char*[]，即它是一个数组，数组内每个元素为 char*，表示一个字符串，对应一个参数。
对于每个参数字符串，将其复制到 user stack 对应的位置。即压栈。
将每个参数字符串复制后的位置的起始地址（由动态更新的 sp 变量表示），存入exec中的局部变量 ustack 数组内。
在压完所有参数字符串后，将 ustack 的内容也压入栈中。
则用户根据 argv[i] 的方式可以拿到每个参数字符串在用户空间的地址，再 indirection 它，即可获取对应的参数字符串。
(5) exec 的目的是模拟 main(argc, argv) 的调用，故其：
(a) 将 argv 存入 p->trapframe->a1，再 return argc （则 syscall 函数会执行 p->trapframe->a0 = argc）。
由于 a0, a1 寄存器对应函数的第 1，2个参数，故模拟了 main 函数的调用。
(b) 将 epc 设置为 ELF header 中包含的 entry field，则在 trap out 后，user program 会从 entry 开始执行，即模拟 main 函数的执行。
(6) 最后销毁旧页表，将新页表已经其他信息赋给当前的 proc struct，则完成了一次替换式的 exec 调用。
之后，该 proc struct 即执行新的程序。

coordination

1. coordination (aka. conditional coordination)
一个线程主动 sleep, 释放 CPU，等待某个 event 发生。
另一个线程完成这个 event，再 wakeup 这个 sleeping 线程。
适用于 event 耗时较长 或不知道什么时候会完成。

其他的 sync 方式：
e.g. busy-waiting, 即 polling. 很耗 CPU。适用于 event 完成耗时很短。

1. xv6 使用 sleep, wakeup 实现 coordination
(1) 如何通知 sleeping proc?
(a) state
要通知一个 sleeping proc，proc->state 必须被设置为 sleeping，以与 RUNNABLE 区分。否则会被 scheduler 调度，根本不会等待 event，即没有 sync.
(b) chan
为了通知特定的 sleeping proc，需要给每个 sleeping proc 一个 unique 的 id，即 chan。（实际上，可以不 unique。但是会有 spurious wakeup 问题）
(2) sleep 实现：
对于这个 event，sleeper 和 waker 都需要 access 这个 event 完成的 condition，则是一个共享 obj，则需要锁，称为 condition lock.
故 sleeper 和 waker 在 access 这个 condition 之前，都需要 acquire 这个锁。
而由于要进行 context switch，sleeper 不能持有除了 p->lock 之外的其他锁，否则可能死锁。故需要在 sleep 之前，释放 condition lock.
为避免 lost wakeup，应先获取 p->lock，再释放 condition lock. 如果先释放 condition lock，再获取 p->lock，
则会有一个 window time，期间 sleeper 没有持有任何锁。则此时 waker 可以运行。如 waker 此时恰好运行，则会发现 一个 state 为 sleeping，但实际没有 sleep 的 proc.
则会产生 lost wakeup 问题。
(3) wakeup 实现：
必须在获取 condition lock 的前提下，才能调用，因为 waker 要设置 condition.
因为要设置 p->state = RUNNABLE，则需要获取 p->lock.

address space 

@ref kernel_page_table.png and user_page_table.png

之后应该就不和 xv6 强相关了

要学好 os，看 xv6 源码。例如对照着 ostep，看一个章节或者一个小点，然后看 xv6 是怎么实现的。这样就学好了

scheduling policies

@ref OSTEP

riscv 

@ref riscv_registers.png and riscv_isa.png

os basics

1. Why OS?
OS 的功能与人们的需求有关。
首先，当然需要一台电脑能同时运行多个程序，此即 multitasking，或称 multiprogramming。
对程序而言，要求对资源进行复用 multiplexing，即互不干扰的使用资源。
对资源而言，要求能被多个程序同时使用，即共享，sharing。

实现这个的核心机制，即抽象 abstraction，或更进一步，虚拟化 virtualization。
虚拟化提供了一个核心机制： level of indirection。这使得软件与硬件、软件与软件不会直接接触，而是通过中间的 OS 进程有限的可控的接触。
对于两个重要的资源：CPU、内存，则分别有 CPU 虚拟化 与 内存 虚拟化。

注意到，要多个程序同时互不干扰地运行，要全局资源被多个程序共享，则显然需要一些保护和同步机制。
保护机制由 内存 的虚拟化，即虚拟内存，提供。同时，也实现了隔离。
同步，则涉及到并发。
因此，并发是支持 OS 的核心机制之一。
注意，并发的核心在于并，即我们当然可以使用一个 queue 来规划所有任务，这样就永远不会干涉。但是，这样做性能太差。
故需要使用并发，让多个任务在同一个资源上执行，并且不干涉，此即并发问题的困难所在。

另一方面，程序在不运行时会放到磁盘，因此需要 OS 来管理磁盘，并提供接口给用户以操纵磁盘。此即持久化涉及的问题。

其他的，每台电脑上通常都会有 网络 （network）设备、显式设备（graphics），这些都要 OS 来管理。

还有，安全问题。OS 要介入很多地方，防止 malicious 行为或无意但损害很大的行为。

Summary:
核心思想：间接性。即实现软件与软件、软件与硬件、硬件与硬件有限的可控的间接交互。即由 OS 作为中间对象，由 OS 完成交流、控制、组织的工作。
核心机制：虚拟化（virtualization）。
multitasking：虚拟化 CPU，以复用 CPU，使得多个程序能同时运行，而互补干扰。
sharing：虚拟化 内存，以共享 内存，使得多个程序能同时使用唯一的全局的内存，而互补干扰。
当然，还有虚拟化硬件，则可以共享硬件，统一硬件被软件使用的接口。

辅助机制：并发（concurrency）。
再辅助机制：持久化（persistence）。

1. virtualization
虚拟化是实现复用（multiplexing）的一种手段。
举个例子：
有个全局的、唯一的桃子，但是有很多人要同时吃它。此即对桃子的复用。
复用即为了实现同时吃这个桃子，即管理对于这个桃子的同时的吃。
如何实现复用呢？
一种方法即虚拟化，可以理解为对这个实体桃子做投影，给每个要吃的人都投影一个桃子的虚像。
通过良好的设计，即可以让这些人不意识到他们面前的桃子是虚的，每个人的吃也不会干扰到其他人。
通过这个投影，即提供了一层间接性，操作系统即利用这个间接性，作为这个中间方，实现很多功能。

如何设计，才能让这些意识不到呢？
由于一个人吃了一口后，要咀嚼，则这段时间，其他人可以吃。故 OS 可以进行调度，让其他空闲的人来吃。调度做得越好，则这个桃子虚像就越真，因为每个人要吃的时候都有的吃。此即，时间复用，即对 CPU 的虚拟化。
由于同一个桃子被多个人吃，肯定要防止一个人吃别人吃的部分，此即保护及隔离。即需要分配好每个人吃的部分。保护、分配地越好，这个桃子虚像就越真，因为不会吃到别人吃的。此即，空间复用，即对内存的虚拟化。

但是，不管怎么设计，都有可能产生干扰。、
因此，需要同步机制，限制这些人干扰别人。此即并发涉及东西，也即 OS 需要并发功能。

一个合格的虚拟化机制应保证：
transparency
efficiency
protection

1. great ideas in OS design
(1) abstraction: 抽象是思想，virtualization 是手段，最关键的在于提供了 间接性 level of indirection。则具有了 isolation 与 protection.
(2) laziness (aka. on demand)
(3) clustering （稍同义 batch，以 amortize cost）
(4) background（daemon，可以实现 overlap，例如 CPU, disk overlap）
(5) time-space tradeoff
(6) hybrid: 综合两种策略的特点。
例如一个调度策略 max TAT ，另一个策略 max RPT，看是否能结合这两个策略，则形成一个新的 hybrid 的 同时 比较 max TAT 与 RPT。例如 MLFQ 调度策略。
例如，有些 OS hybrid 了 分段与分页。
(7) modularity: e.g. 机制（mechanism by h/w）与 策略（policy by s/w）的模块化分离。
(8) locality (局部性原理)
(9) leveling: e.g. 多级页表，多级 cache, 多级存储结构, MLFQ 调度策略中的多级队列 etc.
(10) 软硬件协同: 硬件做少点，软件灵活性就大点，但对应地，安全性和性能可能差点。

TODO locking, how to implement, types of lock

dynamic memory allocation

1. 
2. out-of-memory (OOM) 时如何处理？
由于物理内存有限，而有些应用要求使用超过这个限度的内存，则很可能出现需要一个存在磁盘中的页，但是内存已经满了（OOM），
则此时需要利用类似 cache 的方法，evict 一个存在内存中的页，将其撤回到磁盘中，再换入这个需要的页。
这些被撤出的页，会被存在交换空间（swap space）中，等待被再次换入（不一定发生）。
对于 evict 的选择，通常需要根据 dirty bit 和 access bit 来判断，并据此设计 page selection 策略，
以决定需要 evict 哪个页。如果一个页被写了，则 dirty bit 为 1。如果一个页被访问了（写或读），则 access bit 为 1。
通常会选择撤回 non-dirty 页，因为目前的先验信息告诉我们，这些页被使用的概率较低。
一个常用的策略是 LRU，即最近最少使用策略。为支持它，需要维护时序信息。
一种利用时序信息的方式是：OS 会每隔一段时间间隔，刷新 access bit。这样在一段较长的时间内，
可以通过每个时间间隔检测 access bit，来统计这段时间内每个页的访问频次，以支持 LRU 策略。
3. swap space water mark
通常，不会等到物理内存满了，才换出到交换空间中。
而是会执行一个 swap daemon，并设置两条水位线：low watermark（LW） 与 high watermark（HW）。
当某次物理内存分配后，OS 发现空闲的物理内存低于 LW，则会批量地 evict 部分物理页（根据所采用的页置换策略），直至空闲的物理页数量达到 HW。
使用 daemon，能够将待执行的任务堆到后台，待合适的时机批量执行。
特别在磁盘 I/O 中，批量操作有益于降低总的磁盘寻址时间。另一方面，可以提高 responsiveness，降低 latency。
有时候，还可以避免不必要的工作，例如并不是所有页最终都要存到内存，可能存完之后，马上又换出来，则有更大的弹性。
当然，要实现这个，必须保留较大的交换空间。

cache 

cache locality

storage level

thrashing

write through, write back

c calling convention 

1. caller saved vs. callee saved
2. param regs, return value reg
3. call stack
(1)
stack pointer
frame pointer
stack frame
(2) 一次完整的函数调用流程:
grow downwards

TODO: concurrency 

basics

issues

primitives

problems

常见 solution paradigms