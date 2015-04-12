# Lab1 实验报告
@(cxlyc007 的笔记本)[读书，inbox]
### 练习1:  理解通过make生成执行文件流程
大致来说，Make首先寻找本机上是否有恰当的开发工具链，包括正确的i386-elf-gcc和qemu等，如果没有，报错。接着编译所有的项目文件，进行相应的链接，最后把生成的文件封装成kernel和ucore.img。最后设置若干的宏命令，如**lab1-mon**等。由于我是在Mac下开发，因此需要重写对Makefile文件进行编辑，做了如下的改动：
``` shell
CDPWD := cd $(shell pwd)

runcomand = $(shell osascript -e 'tell application "Terminal" to do script "$(CDPWD);$1"')

LAB1MON = $(QEMU) -S -s -d in_asm -D $(BINDIR)/q.log -monitor stdio -hda $< -serial null

LAB1INIT = gdb -q -x tools/lab1init

lab1-mon: $(UCOREIMG)
	$(call runcommand, $(LAB1MON))
	sleep 1
	$(call runcommand, $(LAB1INIT))

```

可以看出主要是封装了一个在新窗口启动qemu和gdb的程序，苹果只能通过`tell application "Terminal" to do script xxx`来实现


### 练习2：使用qemu执行调试lab1中的软件
首先加载相应的符号表，接着连接qemu，然后设置断点到入口0x7c00，最后运行程序, 新建一个labinit的文件，加入以下信息：
``` shell
file bin/kernel
target remote :1234
set architecture i8086

define hook-stop
x/i $pc
end

b *0x7c00
continue
x /2i $pc
```
这样的话，就可以正确地进行调试和开发了。
具体调试的时候，可以利用
- `b breakpoint`来设置断点
- `info b`来查看所有断点
- `delete`来删除断点
- `x/10i`来打印10条汇编指令

### 练习3: 分析bootloader进入保护模式的过程
开启A20：通过将键盘控制器上的A20线置于高电位，全部32条地址线可用，
可以访问4G的内存空间。
```
	seta20.1:               # 等待8042键盘控制器不忙
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
```

初始化GDT表：一个简单的GDT表和其描述符已经静态储存在引导区中，载入即可
```
	    lgdt gdtdesc
```

进入保护模式：通过将cr0寄存器PE位置1便开启了保护模式
```
	    movl %cr0, %eax
	    orl $CR0_PE_ON, %eax
	    movl %eax, %cr0
```

### 练习4： 分析Bootloader加载ELF的过程
主要过程是在bootmain.c中的，核心代码如下
``` shell
void
bootmain(void) {
    // read the 1st page off disk
    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);

    // is this a valid ELF?
    if (ELFHDR->e_magic != ELF_MAGIC) {
        goto bad;
    }

    struct proghdr *ph, *eph;

    // load each program segment (ignores ph flags)
    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
    eph = ph + ELFHDR->e_phnum;
    for (; ph < eph; ph ++) {
        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
    }

    // call the entry point from the ELF header
    // note: does not return
    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();

```

首先读入磁盘第一扇区的512字节数据，然后通过elf的头判断是否是合法的elf文件，接着通过elf里面的结构信息把相应的数据从磁盘上读到内存的相应位置。一旦所有的信息加载完毕，程序交给`ELFHDR->e_entry`接口，进入`kernerl_init`


### 5. 实现函数堆栈调用跟踪
由于在调用过程中，会依次执行以下过程：
- 当调用call函数的时候，首先把返回地址压栈
- 接着把当前的%ebp值压栈，接着把%esp的值赋给%ebp，接着开始执行被调用函数的主体
- 在返回的时候，首先将旧的%ebp数据赋值给%ebp，然后暴露出了返回地址，接着直接调用`ret`指令，函数返回

因此顺着以上的思路，可以编写如下代码：
``` shell
void
print_stackframe(void) {
    uint32_t ebp = read_ebp();
    uint32_t eip = read_eip();
    int i,j=0;
    for(i=0; i<STACKFRAME_DEPTH; i++){
        cprintf("ebp:0x%08x eip:0x%08x args:", ebp, eip);
        uint32_t *args = (uint32_t *)ebp + 2;
        for(j=0; j<4; j++){
            cprintf("0x%08x ", args[j]);
        }
        cprintf("\n");
        print_debuginfo(eip-1);
        eip = ((uint32_t *)ebp)[1];
        ebp = ((uint32_t *)ebp)[0];
    }
}
```

### 实验6: 完善中断初始化和处理
首先是初始化，主要的目的就是填写好IDT然后调用`LIDT`，代码如下：
``` shell
    extern uint32_t __vectors[];
    int i;
    for(i=0; i<sizeof(idt) / sizeof(struct gatedesc); i++){
        SETGATE(idt[i], 0, GD_KTEXT, __vectors[i], DPL_KERNEL);
    }
    lidt(&idt_pd);

```

接着就是当有时钟中断的时候，进行处理，计数器加1，如果到了最大值，输出信息并且归零，代码如下：
``` shell
    case IRQ_OFFSET + IRQ_TIMER:
        ticks++;
        if(ticks%TICK_NUM == 0){
            ticks = 0;
            print_ticks();
        }
        break;

```
