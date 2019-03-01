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

