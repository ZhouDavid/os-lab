# Lab3 实验报告
##2013011326   周建宇

##练习1:给未被映射的地址映射上物理页

###1.1 实现过程：
在`do_pgfault`函数中，首先根据给出的页目录（pgdir）和地址（addr）尝试找到对应的PTE(利用`get_pte`函数)，如果*pte==0,则表示物理地址不存在，这时应分配一个页并建立好物理地址和逻辑地址的映射，具体代码如下：

	 if ((ptep = get_pte(mm->pgdir, addr, 1)) == NULL) {
        cprintf("get_pte in do_pgfault failed\n");
        goto failed;
    }

    if (*ptep == 0) {
        if(pgdir_alloc_page(mm->pgdir, addr, perm) == NULL) {
            cprintf("pgdir_alloc_page in do_pgfault failed\n");
            goto failed;
        }
    }
    else {
        if(swap_init_ok) {
            struct Page * page = NULL;
            if ((ret = swap_in(mm, addr, &page)) != 0) {
                cprintf("swap_in in do_pgfault failed\n");
                goto failed;
            }
            page_insert(mm->pgdir, page, addr, perm);
            swap_map_swappable(mm, addr, page, 1);
            page->pra_vaddr = addr;
        }
        else {
            cprintf("no swap_init_ok but ptep is %x, failed\n",*ptep);
            goto failed;
        }
    }
   	ret = 0;


###1.2 请描述页目录项（Pag Director Entry）和页表（Page Table Entry）中组成部分对ucore实现页替换算法的潜在用处。

在`mmu.h`中能够看到PTE中各个位的具体含义，`PTE_D`表明该页是否已经被写过但还没写回相应地址，用于页面置换算法中的页换出时候是否需要写回。`PTE_W`标识该页是否可写，如果该页只读则写该页是虚拟内存管理机制需要阻止访问。`PTE_A`表示该页表项对应的页当前是否被访问过，可用于各种页面置换算法的判断。

###1.3 如果ucore的缺页服务例程在执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？

既然是异常也要做一些常规的事情，即保存当前被打断的程序现场，依次将EFLAGS，CS,EIP，errorCode压入内核栈中。再将程序控制权交给相应的中断服务例程。

##练习2：补充完成基于FIFO的页面替换算法

###2.1 实现过称：

`_fifo_map_swappable` 找到循环链表后，将新的页面插入链表头部。`_fifo_swap_out_victim` 找到链表头部的prev(即链表尾部)删除即可。代码如下：
	
	static int
	_fifo_map_swappable(struct mm_struct *mm, uintptr_t addr, struct Page *page, int swap_in)
	{
	    list_entry_t *head=(list_entry_t*) mm->sm_priv;
	    list_entry_t *entry=&(page->pra_page_link);
	 
	    assert(entry != NULL && head != NULL);
	    //record the page access situlation
	    /*LAB3 EXERCISE 2: 2013011326*/ 
	    //(1)link the most recent arrival page at the back of the pra_list_head qeueue.
	    list_add(head, entry);
	    return 0;
	}

	static int
	_fifo_swap_out_victim(struct mm_struct *mm, struct Page ** ptr_page, int in_tick)
	{
	     list_entry_t *head=(list_entry_t*) mm->sm_priv;
	         assert(head != NULL);
	     assert(in_tick==0);
	     /* Select the victim */
	     /*LAB3 EXERCISE 2: 2013011326*/ 
	     //(1)  unlink the  earliest arrival page in front of pra_list_head qeueue
	     //(2)  set the addr of addr of this page to ptr_page
	     list_entry_t *le = head->prev;
	     assert(head != le);
	     struct Page *p = le2page(le, pra_page_link);
	     list_del(le);
	     assert(p != NULL);
	     *ptr_page = p;
	     return 0;
	}


###2.2 如果要在ucore上实现"extended clock页替换算法"请给你的设计方案，现有的swap_manager框架是否足以支持在ucore中实现此算法？如果是，请给你的设计方案。如果不是，请给出你的新的扩展和基此扩展的设计方案。
现有框架可以实现clock页面替换算法。

#####2.2.1 设计方案：

1. 在`mm_struct`当中加入一个指向某一页的指针，该指针由`mm_struct`维护。
2. 在`swap_out_victim`函数中加入获取当前指针（即`mm_struct`中维护的指针）指向页面所对应的PTE部分。
3. 对得到的PTE进行引用位和修改位的检查，如果引用位为0则淘汰该页面，否则向前移动指针。

#####2.2.2 需要被换出的页的特征是什么？

	PTE_A和PTE_D都被置为0.

#####2.2.3 在ucore中如何判断具有这样特征的页？

	读取当前页的PTE，得到引用位和修改位判断即可。

#####2.2.4 何时进行换入和换出操作？

	页面访问合法且试图得到空闲页失败（没有页空闲时），这时会利用各种页面替换算法替换