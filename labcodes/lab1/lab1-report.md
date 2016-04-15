# OS-lab1 report  
##2013011326  周建宇
##【练习1】
【练习1.1】 操作系统镜像文件 ucore.img 是如何一步一步生成的?(需要比较详细地解释 Makefile 中每一条相关命令和命令参数的含义,以及说明命令导致的结果)

答：
>生成bootblock

生成bootasm.o和bootmain.o

	生成bootmian.o:
	将bootmain.c 通过gcc编译成，makefile中对应的代码为：
	bootfiles = $(call listf_cc,boot)
	$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))
	
	生成bootasm.o:
	需要bootasm.S
	实际命令为：
	gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs \-nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc \-c boot/bootasm.S -o obj/boot/bootasm.o

生成sign:
	
	在makefile中对应得宏命令为：
	$(call add_files_host,tools/sign.c,sign,sign)
	$(call create_target_host,sign,sign)
	
生成bootblock.o:

	ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 \
	obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o
	
将bootblock.o转为bootblock.out(重定向输出)

	objcopy -S -O binary obj/bootblock.o obj/bootblock.out
	
将bootblock.out转为bootblock（用之前生成的sign）
	
	bin/sign obj/bootblock.out bin/bootblock

>生成kernel

生成obj/kern下的所有.o文件

	$(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,\
	$(KCFLAGS))

通过.o文件和kernel.ld生成kernel

	kernel = $(call totarget,kernel)

	$(kernel): tools/kernel.ld

	$(kernel): $(KOBJS)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)
	$(call create_target,kernel)

将kernel写入ucore.img

首先生成一个大小为10000个块的文件，每块地大小为512Bytes
bootblock中的内容写入第一个块（512Bytes）
kernel从第二块开始写入
makefile中具体对应的代码为：

	$(UCOREIMG): $(kernel) $(bootblock)
	$(V)dd if=/dev/zero of=$@ count=10000
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc
	$(call create_target,ucore.img)
	$(call finish_all)

【练习1.2】
一个被系统认为是符合规范的硬盘主引导扇区的特征是什么?

主引导扇区的大小是512字节。
第510个（倒数第二个）字节是0x55，
第511个（倒数第一个）字节是0xAA。

sign.c中具体代码如下：

	char buf[512];
    memset(buf, 0, sizeof(buf));
    FILE *ifp = fopen(argv[1], "rb");
    int size = fread(buf, 1, st.st_size, ifp);
    if (size != st.st_size) {
        fprintf(stderr, "read '%s' error, size is %d.\n", argv[1], size);
        return -1;
    }
    fclose(ifp);
    buf[510] = 0x55;
    buf[511] = 0xAA;
    FILE *ofp = fopen(argv[2], "wb+");
    size = fwrite(buf, 1, 512, ofp);

##【练习2】
【练习2.1】

答：在makefile中增加如下代码：

debug2: $(UCOREIMG)
		$(V)$(TERMINAL) -e "$(QEMU) -S -s -d in_asm -D $(BINDIR)/q.log -monitor stdio -hda $< -serial null"
		$(V)sleep 2
		$(V)$(TERMINAL) -e "gdb -q -x tools/gdbinit"
```

然后修改了tools/gdbinit:
```
set architecture i8086
target remote :1234
```
然后执行
```
make debug2

然后便可以进行单步跟踪了。

反汇编结果可以在bin/q.log中查看。

[练习2.2]在初始化位置0x7c00设置实地址断点,测试断点正常。
修改gdbinit文件为：

	file bin/kernel
	target remote :1234
	set architecture i8086
	b *0x7c00 
	c  
	x /20i $pc
	set architecture i386 
	
然后执行make debug2，输出结果为:

	Breakpoint 1, 0x00007c00 in ?? ()
	=> 0x7c00:	cli    
	   0x7c01:	cld    
	   0x7c02:	xor    %ax,%ax
	   0x7c04:	mov    %ax,%ds
	   0x7c06:	mov    %ax,%es
	   0x7c08:	mov    %ax,%ss
	   0x7c0a:	in     $0x64,%al
	   0x7c0c:	test   $0x2,%al
	   0x7c0e:	jne    0x7c0a
	   0x7c10:	mov    $0xd1,%al
	   0x7c12:	out    %al,$0x64
	   0x7c14:	in     $0x64,%al
	   0x7c16:	test   $0x2,%al
	   0x7c18:	jne    0x7c14
	   0x7c1a:	mov    $0xdf,%al
	   0x7c1c:	out    %al,$0x60
	   0x7c1e:	lgdtw  0x7c6c
	   0x7c23:	mov    %cr0,%eax

和boot/bootasm.S中代码一致，所以断点正常。

##【练习3】
答：
初始CS寄存器为0，IP寄存器为0x7c00.跳转到0x7c00地址后，首先将flag,DS,ES,SS置零：

	.code16
	    cli
	    cld
	    xorw %ax, %ax
	    movw %ax, %ds
	    movw %ax, %es
	    movw %ax, %ss
将A20拉高：

		seta20.1:           # 等待8042键盘控制器不忙
	    inb $0x64, %al      # 
	    testb $0x2, %al     #
	    jnz seta20.1        #
	
	    movb $0xd1, %al     # 发送写8042输出端口的指令
	    outb %al, $0x64     #
	
	seta20.1:               # 等待8042键盘控制器不忙
	    inb $0x64, %al      # 
	    testb $0x2, %al     #
	    jnz seta20.1        #
	
	    movb $0xdf, %al     # 打开A20
	    outb %al, $0x60     # 
加载全局中断表GDT：`lgdt gdtdesc`

将cr0寄存器PE位置1：

		movl %cr0, %eax
	    orl $CR0_PE_ON, %eax
	    movl %eax, %cr0

跳到32位的代码段地址：`ljmp $PROT_MODE_CSEG, $protcseg`

设置段寄存器，建立堆栈：

		movw $PROT_MODE_DSEG, %ax
	    movw %ax, %ds
	    movw %ax, %es
	    movw %ax, %fs
	    movw %ax, %gs
	    movw %ax, %ss
	    movl $0x0, %ebp
	    movl $start, %esp
完成模式切换，进入boot：`call bootmain`


##【练习4】
分析bootloader加载ELF格式的OS的过程。
从磁盘加载OS的主要代码集中在readsect,readseg和bootmain三个函数部分
readset负责读取单个扇区;readseg简单包装了readsect,可以从设备读取人因长度的内容;bootmain负责读取ELF。具体代码如下：

###readsect:

		static void
			readsect(void *dst, uint32_t secno) {
		    waitdisk();
		
		    outb(0x1F2, 1);                         // 设置读取扇区的数目为1
		    outb(0x1F3, secno & 0xFF);
		    outb(0x1F4, (secno >> 8) & 0xFF);
		    outb(0x1F5, (secno >> 16) & 0xFF);
		    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
		        // 上面四条指令联合制定了扇区号
		        // 在这4个字节线联合构成的32位参数中
		        //   29-31位强制设为1
		        //   28位(=0)表示访问"Disk 0"
		        //   0-27位是28位的偏移量
		    outb(0x1F7, 0x20);                      // 0x20命令，读取扇区
		
		    waitdisk();
	
		    insl(0x1F0, dst, SECTSIZE / 4);         // 读取到dst位置，
		                                            // 幻数4因为这里以DW为单位
		}
###readseg:

	readseg简单包装了readsect，可以从设备读取任意长度的内容。
	static void
	readseg(uintptr_t va, uint32_t count, uint32_t offset) {
		uintptr_t end_va = va + count;
		
		va -= offset % SECTSIZE;
		
		uint32_t secno = (offset / SECTSIZE) + 1; 
		// 加1因为0扇区被引导占用
		// ELF文件从1扇区开始
	
		for (; va < end_va; va += SECTSIZE, secno ++) {
	    	readsect((void *)va, secno);
		}
	}

###bootmain:

	void
	bootmain(void) {
	    // 首先读取ELF的头部
	    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);
	
	    // 通过储存在头部的幻数判断是否是合法的ELF文件
	    if (ELFHDR->e_magic != ELF_MAGIC) {
	        goto bad;
	    }
	
	    struct proghdr *ph, *eph;
	
	    // ELF头部有描述ELF文件应加载到内存什么位置的描述表，
	    // 先将描述表的头地址存在ph
	    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
	    eph = ph + ELFHDR->e_phnum;
	
	    // 按照描述表将ELF文件中数据载入内存
	    for (; ph < eph; ph ++) {
	        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
	    }
	    // ELF文件0x1000位置后面的0xd1ec比特被载入内存0x00100000
	    // ELF文件0xf000位置后面的0x1d20比特被载入内存0x0010e000

	    // 根据ELF头部储存的入口信息，找到内核的入口
	    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
	
	bad:
	    outw(0x8A00, 0x8A00);
	    outw(0x8A00, 0x8E00);
	    while (1);
	}
		
##【练习5】

在kdebug.c中实现print_stackframe如下：

	void
	print_stackframe(void) {
	     /* LAB1 YOUR CODE : STEP 1 */
	     /* (1) call read_ebp() to get the value of ebp. the type is (uint32_t);
	      * (2) call read_eip() to get the value of eip. the type is (uint32_t);
	      * (3) from 0 .. STACKFRAME_DEPTH
	      *    (3.1) printf value of ebp, eip
	      *    (3.2) (uint32_t)calling arguments [0..4] = the contents in address (unit32_t)ebp +2 [0..4]
	      *    (3.3) cprintf("\n");
	      *    (3.4) call print_debuginfo(eip-1) to print the C calling function name and line number, etc.
	      *    (3.5) popup a calling stackframe
	      *           NOTICE: the calling funciton's return addr eip  = ss:[ebp+4]
	      *                   the calling funciton's ebp = ss:[ebp]
	      */
	      uint32_t ebp = read_ebp();
	      uint32_t eip = read_eip();
	      int i,j;
	      for (i = 0; ebp!= 0 && i < STACKFRAME_DEPTH; i++){
	         cprintf("ebp:0x%08x eip:0x%08x ", ebp, eip);
	         cprintf("args: ");
	         for (j = 0; j < 4; j++)
	            cprintf("0x%08x ", ((uint32_t *)ebp + 2)[j]);
	         cprintf("\n");
	         print_debuginfo(eip - 1);
	         eip = *((uint32_t* )ebp+1);
	         ebp = *((uint32_t* )ebp);
	    }
	}


输出结果为：

Kernel executable memory footprint: 64KB
ebp:0x00007b08 eip:0x001009a6 args: 0x00010094 0x00000000 0x00007b38 0x00100092 
    kern/debug/kdebug.c:306: print_stackframe+21
ebp:0x00007b18 eip:0x00100c9b args: 0x00000000 0x00000000 0x00000000 0x00007b88 
    kern/debug/kmonitor.c:125: mon_backtrace+10
ebp:0x00007b38 eip:0x00100092 args: 0x00000000 0x00007b60 0xffff0000 0x00007b64 
    kern/init/init.c:48: grade_backtrace2+33
ebp:0x00007b58 eip:0x001000bb args: 0x00000000 0xffff0000 0x00007b84 0x00000029 
    kern/init/init.c:53: grade_backtrace1+38
ebp:0x00007b78 eip:0x001000d9 args: 0x00000000 0x00100000 0xffff0000 0x0000001d 
    kern/init/init.c:58: grade_backtrace0+23
ebp:0x00007b98 eip:0x001000fe args: 0x001032fc 0x001032e0 0x0000130a 0x00000000 
    kern/init/init.c:63: grade_backtrace+34
ebp:0x00007bc8 eip:0x00100055 args: 0x00000000 0x00000000 0x00000000 0x00010094 
    kern/init/init.c:28: kern_init+84
ebp:0x00007bf8 eip:0x00007d68 args: 0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8 
    <unknow>: -- 0x00007d67 --

最后一行各个数值的含义：
其对应着第一个使用堆栈的函数，即bootmain.c中的bootmain。意思是bootmain的ebp为0x7bf8。
##【练习6】
完善中断初始化和处理

[练习6.1] 中断描述符表（也可简称为保护模式下的中断向量表）中一个表项占多少字节？其中哪几位代表中断处理代码的入口？

在kern/mm/mmu.h中看到：

	/* Gate descriptors for interrupts and traps */
	struct gatedesc {
	    unsigned gd_off_15_0 : 16;        // low 16 bits of offset in segment
	    unsigned gd_ss : 16;            // segment selector
	    unsigned gd_args : 5;            // # args, 0 for interrupt/trap gates
	    unsigned gd_rsv1 : 3;            // reserved(should be zero I guess)
	    unsigned gd_type : 4;            // type(STS_{TG,IG32,TG32})
	    unsigned gd_s : 1;                // must be 0 (system)
	    unsigned gd_dpl : 2;            // descriptor(meaning new) privilege level
	    unsigned gd_p : 1;                // Present
	    unsigned gd_off_31_16 : 16;        // high bits of offset in segment
	};
16+16+5+3+4+1+2+1+16 = 64，所以一个表项占8个字节。	第2和第3个字节代表段选择子，即中断处理代码入口
。

[练习6.2] 请编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt_init。

代码如下：

	/* idt_init - initialize IDT to each of the entry points in kern/trap/vectors.S */
	void
	idt_init(void) {
	     /* LAB1 YOUR CODE : STEP 2 */
	     /* (1) Where are the entry addrs of each Interrupt Service Routine (ISR)?
	      *     All ISR's entry addrs are stored in __vectors. where is uintptr_t __vectors[] ?
	      *     __vectors[] is in kern/trap/vector.S which is produced by tools/vector.c
	      *     (try "make" command in lab1, then you will find vector.S in kern/trap DIR)
	      *     You can use  "extern uintptr_t __vectors[];" to define this extern variable which will be used later.
	      * (2) Now you should setup the entries of ISR in Interrupt Description Table (IDT).
	      *     Can you see idt[256] in this file? Yes, it's IDT! you can use SETGATE macro to setup each item of IDT
	      * (3) After setup the contents of IDT, you will let CPU know where is the IDT by using 'lidt' instruction.
	      *     You don't know the meaning of this instruction? just google it! and check the libs/x86.h to know more.
	      *     Notice: the argument of lidt is idt_pd. try to find it!
	      */
	      extern uintptr_t __vectors[];
	      int i;
	      for (i = 0; i < sizeof(idt) / sizeof(struct gatedesc); i++) {
	            if (i == T_SYSCALL) {
	                SETGATE(idt[i], 1, GD_KTEXT, __vectors[i], DPL_USER);
	             } 
	             else{
	               if (i == T_SWITCH_TOK) {
	                SETGATE(idt[i], 0, GD_KTEXT, __vectors[i], DPL_USER);
	                }    
	                else {
	                    SETGATE(idt[i], 0, GD_KTEXT, __vectors[i], DPL_KERNEL);
	                } 
	             } 
	       }
	       lidt(&idt_pd);
	}

将中断信息添至idt数组中。中断号为T_SYSCALL和T_SWITCH_TOK的中断进行了特殊处理。最后设置IDTR寄存器，值为idt表的首地址。


[练习6.3] 请编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写trap函数
在trap_dispatch函数中添加如下代码：

	ticks ++;
	if (ticks % TICK_NUM == 0) {
	    print_ticks();
	}
	break;

实现过程：
首先调用了kern/driver/clock.c中的全局变量ticks，ticks在kern/driver/clock.c已被初始化为0。然后每次增加100后调用一次print_ticks()。



##【练习7】
