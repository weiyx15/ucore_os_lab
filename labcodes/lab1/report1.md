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



