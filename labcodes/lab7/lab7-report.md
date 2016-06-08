# Lab7 实验报告
##2013011326 周建宇

## [练习1]理解内核级信号量的实现和基于内核级信号量的哲学家就餐问题(不需要编码)

[练习1.1]请在实验报告中给出内核级信号量的设计描述,并说其大致执行流流程。

本实验的内核信号量建立在开关中断机制和wait queue的基础上进行了具体实现。信号量的数据结构定义如下：
	
	typedef struct {
	    int value;
	    wait_queue_t wait_queue;
	} semaphore_t;
	
semaphore_t是最基本的记录型信号量结构,包含了用于计数的整数值value,和一个进程等待队列
wait_queue,一个等待的进程会挂在此等待队列上。它的函数主要有初始化函数sem_Init(),PV操作的up()函数和down()函数，以及一个判断是否可以进行P操作的bool函数try_down()。
其中最主要的是是P操作函数`down(semaphore_t *sem)`和V操作函数`up(semaphore_t *sem)`。但这两个函数
的具体实现是`__down(semaphore_t *sem, uint32_t wait_state)`函数和`__up(semaphore_t	*sem, uint32_t wait_state)`函数。

	static __noinline uint32_t __down(semaphore_t *sem, uint32_t wait_state) {
	    bool intr_flag;
	    local_intr_save(intr_flag);
	    if (sem->value > 0) {
	        sem->value --;
	        local_intr_restore(intr_flag);
	        return 0;
	    }
	    wait_t __wait, *wait = &__wait;
	    wait_current_set(&(sem->wait_queue), wait, wait_state);
	    local_intr_restore(intr_flag);
	    schedule();
	    local_intr_save(intr_flag);
	    wait_current_del(&(sem->wait_queue), wait);
	    local_intr_restore(intr_flag);
	    if (wait->wakeup_flags != wait_state) {
	        return wait->wakeup_flags;
	    }
	    return 0;
	}

首先关掉中断,然后判断当前信号量的value是否大于0。如果是>0,则表明可以获得信号量,故让value减一,并打开中断返回即可;如果不是>0,则表明无法获得信号量,故需要将当前的进程加入到等待队列中,并打开中断,然后运行调度器选择另外一个进程执行。如果被V操作唤醒,则先关闭中断，把自身关联的wait从等待队列中删除，再打开中断。

###[练习1.2]请在实验报告中给出给用户态进程/线程提供信号量机制的设计方案,并比较说明给内核级提供信号量机制的异同。

在用户态进程/线程里实现信号量机制时，可以考虑使用软件的方法，如采用双线程的Peterson算法，多线程的Dekkers算法，使其在访问共享变量时不被打断，时刻保证临界区只有一个用户态进程/线程访问。
用户态进程/线程和内核态线程的主要区别在于，内核态线程可以基于开关中断机制，而用户态进程/线程没有开关中断机制，但是其有共享的变量，因此软件方法比较容易实现。但是，尽管实现方法有所不同，但二者的信号量数据结构以及在P操作和V操作功能上都是一致的。

## [练习2]完成内核级条件变量和基于内核级条件变量的哲学家就餐问题(需要编码)

[练习2.1]请在实验报告中给出内核级条件变量的设计描述,并说其大致执行流流程。

 在本实验中，管程中的条件变量的数据结构condvar_t定义如下:

	typedef struct condvar{
	    semaphore_t sem;        // the sem semaphore  is used to down the waiting proc, and the signaling proc should up the waiting proc
	    int count;              // the number of waiters on condvar
	    monitor_t * owner;      // the owner(monitor) of this condvar
	} condvar_t;

条件变量的定义中主要包含三个成员变量,信号量sem用于让发出`wait_cv`操作的等待某个条件C为真的进程睡眠,而让发出`signal_cv`操作的进程通过这个`sem`来唤醒睡眠的进程。`count`表示等在这个条件变量上的睡眠进程的个数。owner表示此条件变量的宿主是哪个管程。
这里,条件变量两个主要操作`wait_cv`和`signal_cv`分别实现为`cond_wait`和`cond_signal`函数，此外还有`monitor_init`初始化函数。简单分析一下`cond_wait`函数的实现，如果进程A执行了`cond_wait`函数,表示此进程等待某个条件C不为真,需要睡眠。因此表示等待此条件的睡眠进程个数`cv.count`要加一。接下来会出现两种情况。如果`monitor.next_count`如果大于0,表示有大于等于1个进程执行`cond_signal`函数且睡着了,就睡在了`monitor.next`信号量上。假定这些进程形成S进程链表。因此需要唤醒S进程链表中的一个进程B。然后进程A睡在`cv.sem`上,如果睡醒了,则让`cv.count`减一,表示等待此条件的睡眠进程个数少了一个,可继续执行了!这里隐含这一个现象,即某进程A在时间顺序上先执行了`signal_cv`,而另一个进程B后执行了`wait_cv`,这会导致进程A没有起到唤醒进程B的作用。这里还隐藏这一个问题,在`cond_wait`有`sem_signal(mutex)`,但没有看到哪里有`sem_wait(mutex)`,这好像没有成对出现,是否是错误的?其实在管程中的每一个函数的入口处会有`wait(mutex)`,这样二者就配好对了。第二种情况下，如果`monitor.next_count`如果小于等于0,表示目前没有进程执行`cond_signal`函数且睡着了,那需要唤醒的是由于互斥条件限制而无法进入管程的进程,所以要唤醒睡在`monitor.mutex`上的进程。然后进程A睡在`cv.sem`上,如果睡醒了,则让`cv.count`减一,表示等待此条件的睡眠进程个数少了一个,可继续执行了!对照着再来看`cond_signal`的实现。首先进程B判断`cv.count`,如果不大于0,则表示当前没有执行`cond_wait`而睡眠的进程,因此就没有被唤醒的对象了,直接函数返回即可;如果大于0,这表示当前有执行`cond_wait`而睡眠的进程A,因此需要唤醒等待在`cv.sem`上睡眠的进程A。由于只允许一个进程在管程中执行,所以一旦进程B唤醒了别人(进程A),那么自己就需要睡眠。故让`monitor.next_count`加一,且让自己(进程B)睡在信号量`monitor.next`上。如果睡醒了,这让`monitor.next_count`减一。
为了让整个管程正常运行,还需在管程中的每个函数的入口和出口增加相关操作，一方面，保证只有一个进程在执行管程中的函数，另一方面，避免由于执行了`cond_signal`函数而睡眠的进程无法被唤醒。

具体代码见`lab7/kern/check_sync.c`。

###[练习2.2]请在实验报告中给出给用户态进程/线程提供条件变量机制的设计方案,并比较说明给内核级提供条件变量机制的异同。

给用户态进程/线程提供条件变量机制，由于条件变量的实现基于信号量的实现，因此首先要提供在用户态的信号量机制，然后基于此实现条件变量。所以主要还是信号量的设计，假定信号量机制已经在用户态提供了，那么条件变量的实现和在内核态实现没有区别，这就转换为了练习1.2的问题。用户态的条件变量机制和内核态的条件变量机制不同之处在于一个基于用户态的信号量，一个基于内核态的信号量，其他的上层操作应该都是一致的。


