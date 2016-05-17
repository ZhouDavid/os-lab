#Lab4 实验报告
##2013011326   周建宇

##练习1：分配并初始化一个进程控制块

#####1.1 过程实现：
`alloc_proc`仅对线程做了最基本的初始化，即把`proc_struct`中的各个成员变量清零。少数变量设置了特殊值,具体如下：

	proc->state = PROC_UNINIT; 设置进程为“初始”态
	proc->pid = -1; 设置进程pid的未初始化值
	proc->cr3 = boot_cr3; 使用内核页目录表的基址

#####1.2 请说明`proc_struct中struct context context`和`struct trapframe *tf`成员变量含义和在本实验中的作用是啥？

context是用来保存进程切换的上下文，在进程切换的时候保存寄存器。`tf`用来辅助创建内核线程和fork，在创建内核线程或fork期间用来伪造中断栈，在最后跳转执行中断恢复，从而完成上下文切换。

context 和 trapframe 都是用来进行线程上下文切换的。context保存当前线程运行到什么位置，具体保存eip,esp,ebx,edx,esi,edi,ebp等寄存器的值。trapframe包括errocode,eip,cs,eflags等几部分。和中断一样，这些信息在线程切换时会被保存到内核堆栈里。如果涉及到线程特权级的切换，则被押入内核栈的还有esp,ss等信息。另外，trapframe中还有许多段寄存器信息和通用寄存器信息，这些用软件来保存。总的来说，trapframe就是保存了前一个被打断的进程或线程的运行信息。

##练习2：为新创建的内核线程分配资源

#####2.1 实现过程：


1. 调用alloc_proc函数去分配一个proc_struct的内存
2. 调用setup_kstack函数去为子进程分配一个内核栈
3. 调用copy_mm函数根据clone_flag选择复制或共享mm成员
4. 调用copy_thread函数去设置proc_struct中的tf和context成员
5. 将proc_struct插入hash_list与proc_list，给子进程分配pid
6. 调用wakeup_proc标记子进程为RUNNABLE状态
7. 设置返回值位子进程的pid

####请说明ucore是否做到给每个新fork的线程一个唯一的id？请说明你的分析和理由。

通过分析`get_pid`函数，发现只要已有线程数小于`(MAX_PID - 1)`，则一定可以做到给每个新fork的线程一个唯一的id，否则该函数会陷入死循环。但是因为ucore中的`MAX_PROCESS`为4096, `MAX_PID`为`MAX_PROCESS`的两倍, 因此, id的数量是足够用于分配给进程的。

分配ID的函数通过维护`next_safe`来标识可取的线程id最大不能到多少，通过不断地遍历`proc_list`来更新`next_safe`以及可能可用的`pid(last_pid`)直到找到一个未被占用的的pid并返回。

##练习3:阅读代码，理解 proc_run 函数和它调用的函数如何完成进程切换的

`proc_run`的参数`proc`就是待启动的进程，如果当前进程不是`proc`，如果不是则开始切换。切换的时候先屏蔽中断，然后通过`load_esp0`切换内核栈，通过lcr3切换页目录基址，再通过switch_to保存原进程的寄存器并恢复下一个进程的寄存器，最后打开中断。

####在本实验的执行过程中，创建且运行了几个内核线程？

两个，分别是`idle`和`init`。但真正运行的只用`init`

####语句`local_intr_save(intr_flag);....local_intr_restore(intr_flag);`在这里有何作用?请说明理由

屏蔽中断、打开中断。为了防止在对寄存器进行保存、回复的时候被打断。