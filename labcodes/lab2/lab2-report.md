# Lab2 实验报告
##2013011326   周建宇

##练习1：实现 first-fit 连续物理内存分配算法（需要编程）

需要修改 default_pmm.c中 default_alloc_pages()和default_free_pages()两个函数。

###default_alloc_pages
first-fit 需要从链表中找到首个block size>=n 的表项。之后对page的property做判断，如果正好等于n则直接删除该表项即可；如果大于n则在更新property -= n，并且将新的page_link插入list，覆盖之前的表项。

	static struct Page *
	default_alloc_pages(size_t n) {
    assert(n > 0);
    if (n > nr_free) {
        return NULL;
    }
    struct Page *page = NULL;
    list_entry_t *le = &free_list;
    list_entry_t *pre_le = &free_list;
    for (le = list_next(le);le!=&free_list;le = list_next(le)) {
        struct Page *p = le2page(le, page_link);
        if (p->property >= n) {
            page = p;
            break;
        }
        pre_le = le;
    	}
    if (page != NULL) {
        if(page->property > n) {
            list_del(&(page->page_link));
            struct Page *p = page + n;
            p -> property = page -> property - n;
            list_add_after(pre_le, &(p->page_link));
            page ->property = n;
        }
        else if(page->property == n) {
            list_del(&(page->page_link));
        }
        nr_free -= n;
        ClearPageProperty(page);
    }
    return page;
	}

###default_free_pages
首先判断传入页面参数是否合法，合法的话我们从`free_list`的头开始遍历，找到第一个地址不小于base+n的位置，然后把要释放的n个页面插入到`free_list`即可。然后将base的引用位置零，property置n。在进行合并时，由于所有空闲页面都在`free_list`中，因此在遍历的时候如果发现有在base前的临近页面则合并，有在base+n的临近页面也合并。并更新base和property

	static void
	default_free_pages(struct Page *base, size_t n) {
    assert(n > 0);
    struct Page *p = base;
    for (; p != base + n; p ++) {
        assert(!PageReserved(p) && !PageProperty(p));
        p->flags = 0;
        set_page_ref(p, 0);
    }
    base->property = n;
    SetPageProperty(base);
    list_entry_t *le = list_next(&free_list);
    list_entry_t *after_le = &free_list;
    for (;le != &free_list;le = list_next(le)) {
        p = le2page(le, page_link);
	if (base + base->property < p) {
            after_le = &(p->page_link);
            break;
        }
        else if (base + base->property == p) {
            base->property += p->property;
            ClearPageProperty(p);
            list_del(&(p->page_link));
        }
        else if (p + p->property == base) {
            p->property += base->property;
            ClearPageProperty(base);
            base = p;
            list_del(&(p->page_link));
        }
    }
    nr_free += n;
    list_add_before(after_le, &(base->page_link));
	}


###可以改进的地方
有可以改进的地方。因为free_list中有很多property为0的页面，可以在free的时候忽略掉这些页面。另外遍历的复杂度是O(n),如果链表所有项按照地址排序，可以用二分查找在log(n)的时间内找到应该插入的位置，再看看两边是否可以合并即可。

##练习2：实现寻找虚拟地址对应的页表项

###2.1 实现方法：
根据传入的页目录表的首地址和线性地址，首先判断线性地址所对应得页目录表项是否存在，如果存在，则根据表项的值找到对应的二级页表首地址，并根据线性地址找到对应二级页表的某个页表项，返回该页表项的地址即可。如果不存在，那么需要创建一个包含该页目录项。get_pte函数中的具体代码如下：

	pde_t *pdep = &pgdir[PDX(la)];
    if(!(*pdep & PTE_P)) {
        struct Page * page;
        if(!create || (page = alloc_page()) == NULL) {
            return NULL;
        }
        set_page_ref(page, 1);
        uintptr_t pa = page2pa(page);
        memset(KADDR(pa), 0, PGSIZE);
        *pdep = pa | PTE_U | PTE_W | PTE_P;
    }
    return &((pte_t *)KADDR(PDE_ADDR(*pdep)))[PTX(la)];

###2.2 请描述页目录项（Pag Director Entry）和页表（Page Table Entry）中每个组成部分的含义和以及对ucore而言的潜在用处。

	 PTE_P           0x001                   // 物理页面是否存在
	 PTE_W           0x002                   // 是否可写
	 PTE_U           0x004                   // 用户程序是否可访问
	 PTE_PWT         0x008                   // 是否写直达
	 PTE_PCD         0x010                   // 是否启用缓存
	 PTE_A           0x020                   // 是否被访问过
	 PTE_D           0x040                   // PTE中表示是否被写过，PDE中无意义
	 PTE_PS          0x080                   // PTE为0，PDE中表示页大小
	 PTE_MBZ         0x180                   // 全部为零
	 PTE_AVAIL       0xE00                   // 表示能否被应用用户程序使用

###2.3 如果ucore执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？
首先CPU在内核栈保存现场，向栈内一次压入EFLAGS、CS、EIP、errorCode。操作系统通过IDT发现这是一个缺页异常，从而跳转到内核的异常处理代码，然后执行页面的替换。最后恢复现场，再次执行该条指令。

##练习3：释放某虚地址所在的页并取消对应二级页表项的映射（需要编程）
###3.1 实现方法
首先判断该页面是否存在，如果页面存在，则将该页面的记数-1，如果减到0，则释放该页面，同时删除对应二级页表项并更新TLB。代码如下：
	
    if(*ptep & PTE_P) { //  判断页表项是否存在且对应页面已经被分配出去
	    struct Page * page = pte2page(*ptep);
	    if(page_ref_dec(page) == 0) {  //引用-1并判断是否减到0
	        free_page(page);
	    }
	    *ptep = 0;
    	tlb_invalidate(pgdir, la);  //更新TLB
    }

###3.2 数据结构Page的全局变量（其实是一个数组）的每一项与页表中的页目录项和页表项有无对应关系？如果有，其对应关系是啥？

二者有对应关系。一个页表项对应Page数组中的一个page。具体来讲，每个页所对应的虚拟地址的高10位为页目录表中的索引（序号），通过PDTR中保存的页目录基址二者相加可得到页目录表项地址。再通过该页目录表项中存的页表基址和虚拟地址的中间10(index)能够找到对应的页表项，其中保存了物理地址的高20位，再与虚拟地址的低12位拼接即可得到物理地址。

###3.3 如果希望虚拟地址与物理地址相等，则需要如何修改lab2，完成此事？鼓励通过编程来具体完成这个问题。
如果想修改LAB2完成这个事情，需要初始化PTE的时候把PTE中的的PFN变成PTX:PDX，即将ucore起始地址与内核虚地址设置为相同的地址。