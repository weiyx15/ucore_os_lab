# lab1实验报告

## 练习1：理解通过make生成执行文件的过程

- ucore.img是如何一步一步生成的？

  ```makefile
  178 # create ucore.img
  179 UCOREIMG    := $(call totarget,ucore.img)
  180           
  181 $(UCOREIMG): $(kernel) $(bootblock)
  182     $(V)dd if=/dev/zero of=$@ count=10000
  183     $(V)dd if=$(bootblock) of=$@ conv=notrunc
  184     $(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc
  185           
  186 $(call create_target,ucore.img)
  ```

  ucore.img依赖kernel和bootblock

  生成ucore.img后用dd移动/dev/zero, bootblock, kernel到指定shell参数指定的位置

  - kernel

    ```makefile
    140 # create kernel target             
    141 kernel = $(call totarget,kernel)   
    142                                    
    143 $(kernel): tools/kernel.ld         
    144                                    
    145 $(kernel): $(KOBJS)                
    146     @echo + ld $@                  
    147     $(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)                                                                                                                                              
    148     @$(OBJDUMP) -S $@ > $(call asmfile,kernel)
    149     @$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)
    150                                    
    151 $(call create_target,kernel)       
    ```

    kernel依赖libs中的头文件

    生成kernel后链接tools/kernel.ld

  - bootblock

    ```makefile
    155 # create bootblock                                                     
    156 bootfiles = $(call listf_cc,boot)                                      
    157 $(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))
    158                                                                        
    159 bootblock = $(call totarget,bootblock)                                 
    160                                                                        
    161 $(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)       
    162     @echo + ld $@                                                      
    163     $(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)                                                                                                                        
    164     @$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
    165     @$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
    166     @$(call totarget,sign) $(call outfile,bootblock) $(bootblock)      
    167                                                                        
    168 $(call create_target,bootblock)                             
    ```

    bootblock依赖bootfiles和sign

    链接bootfiles得到的obj文件并将其反编译为汇编

    - bootfiles

      用gcc编译boot/下的c文件

    - sign

      ```makefile
      172 # create 'sign' tools        
      173 $(call add_files_host,tools/sign.c,sign,sign
      174 $(call create_target_host,sign,sign)
      ```

      用gcc编译tools/sign.c

- 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？

  总共有512个字节，第510和511个字节分别是0x55, 0xAA.

## 练习2：使用qemu执行并调试lab1中的软件.

- 从CPU加电后执行的第一条指令开始，单步跟踪BIOS的执行

  BIOS在实模式下运行，故需要设置为i8086架构，即修改tools/gdbinit为

  ```
  file bin/kernel						// 内核文件
  set architecture i8086				// 设置i8086（实模式）
  target remote localhost :1234		// 连接qemu
  ```

  在命令行窗口运行make debug，可以看到位于0xfff0处的第一条指令

- 在初始化位置0x7c00设置实地址断点，测试断点正常

  新建gdb配置文件toos/lab1init

  ```
  file bin/kernel						// 内核文件
  target remote :1234					// 连接qemu
  set architecture i8086				// 设置i8086（实模式）
  break *0x7c00						// 在0x7c00处设置断点
  continue							// 继续运行
  x $pc								// 打印断点处的指令
  ```

  修改Makefile，加入lab1-mon

  ```makefile
  210 lab1-mon: $(UCOREIMG)                                                       
  211     $(V)$(TERMINAL) -e "$(QEMU) -S -s -d in_asm -D $(BINDIR)/q.log -monitor     stdio -hda $< -serial null"
  212     $(V)sleep 2
  213     $(V)$(TERMINAL) -e "gdb -q -x tools/lab1init"
  ```

  在命令行窗口运行make lab1-mon，可以在0x7c00处停下

- 从0x7c00开始跟踪代码运行，将单步跟踪反汇编得到的代码与bootasm.S进行比较

  在上一步的gdb调试窗口运行命令x /43i $pc，打印当前指令寄存器$pc之后20条指令的内容如下：

  ```assembly
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
     0x7c26:	or     $0x1,%eax
     0x7c2a:	mov    %eax,%cr0
     0x7c2d:	ljmp   $0x8,$0x7c32
     0x7c32:	mov    $0xd88e0010,%eax
     0x7c38:	mov    %ax,%es
     0x7c3a:	mov    %ax,%fs
     0x7c3c:	mov    %ax,%gs
     0x7c3e:	mov    %ax,%ss
     0x7c40:	mov    $0x0,%bp
     0x7c43:	add    %al,(%bx,%si)
     0x7c45:	mov    $0x7c00,%sp
     0x7c48:	add    %al,(%bx,%si)
     0x7c4a:	call   0x7d0b
     0x7c4d:	add    %al,(%bx,%si)
     0x7c4f:	jmp    0x7c4f
  ```

  与bootasm.S中的内容相同

- 自己找一个bootloader或内核中的代码位置，设置断点并进行测试

  修改tools/gdbinit为：

  ```
  file bin/kernel
  target remote :1234
  break kern_init                                                                     continue
  ```

  在命令行窗口运行make debug，在kern/init/init.c的函数kernel_init入口处中断。



## 练习3：分析bootloader进入保护模式的过程

- 为何开启A20，如果开启A20

  In file **boot/bootasm.S**:

  ```assembly
  29     seta20.1:     
  30     inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
  31     testb $0x2, %al
  32     jnz seta20.1
  33               
  34     movb $0xd1, %al                                 # 0xd1 -> port 0x64
  35     outb %al, $0x64                                 # 0xd1 means: write data to 8042's P2 port
  36               
  37 seta20.2:     
  38     inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
  39     testb $0x2, %al
  40     jnz seta20.2
  41               
  42     movb $0xdf, %al                                 # 0xdf -> port 0x60
  43     outb %al, $0x60                                 # 0xdf = 11011111, means set P2's A20 bit(the 1 bit) to 1
  
  ```

  **为何开启A20**：实模式下只有1MB内存空间可见，切换到保护模式时需要取消限制

  **如何开启A20**：

  ```assembly
  movb $0xdf, %al
  outb %al, $0x60
  ```

  将0xdf(11011111)赋给0x60端口，表示将A20置1，可以访问4GB内存空间

- 如何初始化GDT表

  ```assembly
  lgdt gdtdesc
  ```

- 如何使能和进入保护模式

  ```assembly
  orl $CR0_PE_ON, %eax 
  ```

  将cr0寄存器置1进入保护模式

## 练习4：分析bootloader加载ELF格式的OS的过程

- bootloader如何读取硬盘扇区？

  在boot/bootmain.c中，通过函数readsect读取512字节的数据

  ```c
   44 static void
   45 readsect(void *dst, uint32_t secno) {
   46     // wait for disk to be ready
   47     waitdisk();
   48       
   49     outb(0x1F2, 1);                         // count = 1
   50     outb(0x1F3, secno & 0xFF);
   51     outb(0x1F4, (secno >> 8) & 0xFF);
   52     outb(0x1F5, (secno >> 16) & 0xFF);
   53     outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
   54     outb(0x1F7, 0x20);                      // cmd 0x20 - read sectors
   55       
   56     // wait for disk to be ready
   57     waitdisk();
   58     
   59     // read a sector
   60     insl(0x1F0, dst, SECTSIZE / 4);
   61 }   
  ```

  然后用readseg函数封装了readsect函数，可以一次读取多个扇区

  ```c
  /* *                            
   64  * readseg - read @count bytes at @offset from kernel into virtual address @va,
   65  * might copy more than asked.  
   66  * */                           
   67 static void                     
   68 readseg(uintptr_t va, uint32_t count, uint32_t offset) {
   69     uintptr_t end_va = va + count;
   70                                 
   71     // round down to sector boundary
   72     va -= offset % SECTSIZE;    
   73                                 
   74     // translate from bytes to sectors; kernel starts at sector 1
   75     uint32_t secno = (offset / SECTSIZE) + 1;
   76                                 
   77     // If this is too slow, we could read lots of sectors at a time.
   78     // We'd write more to memory than asked, but it doesn't matter --
   79     // we load in increasing order.
   80     for (; va < end_va; va += SECTSIZE, secno ++) {
   81         readsect((void *)va, secno);                                                82     }                           
   83 }             
  ```

- bootloader如何加载ELF格式的OS？

  用e_magic属性判断扇区是否是ELF格式

  ```c
   /* bootmain - the entry of bootloader */
   86 void                           
   87 bootmain(void) {
   88     // 读取扇区头
   89     readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);
   90  
   91     // 判断是否是ELF格式
   92     if (ELFHDR->e_magic != ELF_MAGIC) {
   93         goto bad;
   94     }
   95  
   96     struct proghdr *ph, *eph;
   97  
   98     // 分扇区加载OS
   99     ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
  100     eph = ph + ELFHDR->e_phnum;
  101     for (; ph < eph; ph ++) {
  102         readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
  103     }
  104  
  105     // 跳转到OS的代码
  106     // note: does not return
  107     ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
  108  
  109 bad:
  110     outw(0x8A00, 0x8A00);
  111     outw(0x8A00, 0x8E00);
  112  
  113     /* do nothing */
  114     while (1);
  115	}
  ```

## 练习5： 实现函数调用堆栈跟踪函数

- 完成kern/debug/kdebug.c中print_stackframe函数的实现

  ```c
  void
  print_stackframe(void) {
       /* LAB1 YOUR CODE : STEP 1 */
       /* (1) call read_ebp() to get the value of ebp. the type is (uint32_t);
        * (2) call read_eip() to get the value of eip. the type is (uint32_t);
        * (3) from 0 .. STACKFRAME_DEPTH
        *    (3.1) printf value of ebp, eip
        *    (3.2) (uint32_t)calling arguments [0..4] = the contents in address (uint32_t)ebp +2 [0..4]
        *    (3.3) cprintf("\n");
        *    (3.4) call print_debuginfo(eip-1) to print the C calling function name and line number, etc.
        *    (3.5) popup a calling stackframe
        *           NOTICE: the calling funciton's return addr eip  = ss:[ebp+4]
        *                   the calling funciton's ebp = ss:[ebp]
        */
       uint32_t ebp = read_ebp();             // in file: ../../lib/x86.h
       uint32_t eip = read_eip();
  
       int i, j;
       for (i=0; i<STACKFRAME_DEPTH; i++)
       {
          cprintf("ebp:%08x eip:%08x args:", ebp, eip);
          for (j=0; j<4; j++)
          {
              cprintf("%08x ", ((uint32_t*)(ebp+2))[j]);
          }
          cprintf("\n");
          print_debuginfo(eip-1);
          eip = ((uint32_t*)(ebp+4))[0];
          ebp = ((uint32_t*)(ebp))[0];
       }
  }
  ```

  命令行输入make qemu输出

  ```shell
  Kernel executable memory footprint: 64KB
  ebp:00007b38 eip:00100a3c args:0d300000 00940010 00940001 7b680001 
      kern/debug/kdebug.c:306: print_stackframe+21
  ebp:00007b48 eip:00100d30 args:007f0000 00000010 00000000 00000000 
      kern/debug/kmonitor.c:125: mon_backtrace+10
  ebp:00007b68 eip:0010007f args:00a10000 00000010 7b900000 00000000 
      kern/init/init.c:48: grade_backtrace2+19
  ebp:00007b88 eip:001000a1 args:00be0000 00000010 00000000 7bb4ffff 
      kern/init/init.c:53: grade_backtrace1+27
  ebp:00007ba8 eip:001000be args:00df0000 00000010 00000000 00000010 
      kern/init/init.c:58: grade_backtrace0+19
  ebp:00007bc8 eip:001000df args:00500000 00000010 00000000 00000000 
      kern/init/init.c:63: grade_backtrace+26
  ebp:00007be8 eip:00100050 args:7d6e0000 00000000 00000000 00000000 
      kern/init/init.c:28: kern_init+79
  ebp:00007bf8 eip:00007d6e args:7c4f0000 fcfa0000 d88ec031 d08ec08e 
      <unknow>: -- 0x00007d6d --
  ```

## 练习6：完善中断初始化和处理

- 中断描述符表中一个表项占多少字节？其中哪几位代表中断处理代码的入口？

  在文件./kern/mm/mmu.h中结构体gatedesc描述了一个中断描述符表项。

  ```c
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
  ```

  一个中断描述符表项占64位(即8字节)。16bit(即2字节)的属性gd_ss表示段选择子，2个16bit(即2字节)的属性gd_off_31_16, gd_off_15_0拼起来(共32bit，4字节)表示段内偏移。

- 编程完善对中断向量表进行初始化的函数idt_init

  ```c
  void
  idt_init(void) {
      extern uintptr_t __vectors[];                               // 中断向量表
      int i = 0;
      for (i=0; i<256; i++)
      {
          SETGATE(idt[i], 0, GD_KTEXT, __vectors[i], DPL_KERNEL); // 除系统调用中断外的所有中断都是内核态权限
      }
      SETGATE(idt[T_SWITCH_TOK], 0, GD_KTEXT, __vectors[T_SWITCH_TOK], DPL_USER);       // 系统调用中断是用户态权限
      lidt(&idt_pd);                                              // 加载中断描述符表
  }
  ```

  - 首先从vector.S中引入__vectors表示中断向量表
  - 然后用SETGATE宏对中断描述符表项进行初始化
  - 与其他中断不同，系统调用中断的权限为用户态权限
  - 用lidt加载中断描述表

- 编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分，使操作系统遇到100次时钟中断后，向屏幕打印一行"100 ticks".

  ```c
  case IRQ_OFFSET + IRQ_TIMER:
  	ticks++;
      if (ticks % TICK_NUM == 0)
      {
      	print_ticks();
          ticks = 0;
  	}
      break;
  ```

  完成上述两处修改后，在命令行运行make qemu，可以看到操作系统约1秒会打印一次"100 ticks"，且会响应键盘输入。

  ![1551595123379](./pictures/ex1_6.png)

