#Lab4 实验报告
##2013011326   周建宇

##练习1：分配并初始化一个进程控制块

#####1.1 过程实现：
`alloc_proc`仅对线程做了最基本的初始化，即把`proc_struct`中的各个成员变量清零。少数变量设置了特殊值,具体如下：

	proc->state = PROC_UNINIT; 设置进程为“初始”态
	proc->pid = -1; 设置进程pid的未初始化值
	proc->cr3 = boot_cr3; 使用内核页目录表的基址

#####1.2 请说明`proc_struct中struct context context`和`struct trapframe *tf`成员变量含义和在本实验中的作用是啥？

context 和 trapframe 都是用来进行线程上下文切换的。context保存当前线程运行到什么位置，具体保存eip,esp,ebx,edx,esi,edi,ebp等寄存器的值。trapframe包括errocode,eip,cs,eflags等几部分。和中断一样，这些信息在线程切换时会被保存到内核堆栈里。如果涉及到线程特权级的切换，则被押入内核栈的还有esp,ss等信息。另外，trapframe中还有许多段寄存器信息和通用寄存器信息，这些用软件来保存。总的来说，trapframe就是保存了前一个被打断的进程或线程的运行信息。

##练习2：为新创建的内核线程分配资源

#####2.1 实现过程：

1. 调用`alloc_proc`函数来分配一个proc_struct：

		if((proc = alloc_proc()) == NULL) {
	      goto fork_out;
	    }

2. 调用`setup_kstack`函数去为子进程分配一个内核栈：

		if(setup_kstack(proc) != 0) {
	      goto bad_fork_cleanup_proc;
	    }

3. 调用`copy_mm`根据`clone_flag`选择复制或共享mm成员：

		if(copy_mm(clone_flags, proc) != 0) {
	      goto bad_fork_cleanup_kstack;
	    }
4. 调用`copy_thread`设置`proc_struct`中的tf和context成员：
5. 
		copy_thread(proc, stack, tf);

5. 将`proc_struct`插入`hash_list`与`proc_list`，给子进程分配pid

		local_intr_save(intr_flag);
	    {
	        proc->pid = get_pid();
	        hash_proc(proc);
	        list_add(&proc_list, &(proc->list_link));
	        nr_process++;
	    }
	    local_intr_restore(intr_flag);

6. 调用`wakeup_proc`标记子进程为RUNNABLE状态（wakeup_proc:  set proc->state = PROC_RUNNABLE）：

		wakeup_proc(proc);
		
7. 设置返回值位子进程的pid：

		ret = proc->pid;
		fork_out:
    	return ret;

#####2.2 请说明ucore是否做到给每个新fork的线程一个唯一的id？请说明你的分析和理由。

在`get_pid`中能够发现，如果当前线程数不大于`MAX_PID`,则直接将`last_pid+1`返回给新进程。否则
分配ID的函数通过维护`next_safe`来标识可取的线程id最大不能到多少，通过不断地遍历`proc_list`来更新`next_safe`以及可能可用的`pid(last_pid`)直到找到一个未被占用的的pid并返回。否则该函数将陷入死循环无法推退。但是ucore中`MAX_PROCESS`为4096，`MAX_PID`为8192，是`MAX_PROCESS`的两倍，因此实际上不会出现id数量不够分配的情况，也就是说每个线程都可以得到一个唯一的pid。

##练习3:阅读代码，理解 proc_run 函数和它调用的函数如何完成进程切换的

首先判断待启动的进程是不是当前进程，如果不是则先关闭中断，再用`load_esp0`完成内核堆栈切换，lcr3完成页目录基址切换，再用`switch_to`完成原进程和下一个进程的上下文保存和切换，最后开中断。

#####3.1 在本实验的执行过程中，创建且运行了几个内核线程？

创建了两个，分别是`idle`和`init`。仅运行了`init`。

#####3.2 语句`local_intr_save(intr_flag);....local_intr_restore(intr_flag);`在这里有何作用?请说明理由

屏蔽中断、打开中断。作用是为了防止在对寄存器进行保存、恢复的时候被打断。