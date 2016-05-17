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

####实现过称：

`_fifo_map_swappable`找到循环链表后，将新的页面插入链表头部。`_fifo_swap_out_victim`找到链表头部的prev(即链表尾部)删除即可。

####如果要在ucore上实现"extended clock页替换算法"请给你的设计方案，现有的swap_manager框架是否足以支持在ucore中实现此算法？如果是，请给你的设计方案。如果不是，请给出你的新的扩展和基此扩展的设计方案。

现有的`swap_manager`框架足以支持在ucore中实现此算法，与fifo相比不同的是需要在`swap out`的时候对当前指针指向的页所对应的PTE进行查询，得到（引用位,修改位）。同时拓展时钟算法需要一个“时针”，因此需要在struct mm中添加一项只想swap链表中某项的指针。

* 需要被换出的页的特征是什么？

	PTE_A和PTE_D都被置为了零.

* 在ucore中如何判断具有这样特征的页？

	读取当前页的PTE，得到（引用位，修改位），做出判断。

 * 何时进行换入和换出操作？

	当试图得到空闲页时，发现当前没有空闲的物理页可供分配，这时才开始查找“不常用”页面，并把一个或多个这样的页换出到硬盘上。