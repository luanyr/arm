#### UBOOT启动流程分析

##### uboot的链接地址和运行地址的区别

首先有三个概念：存储地址，运行地址，链接地址

存储地址：代码下载到flash中的地址。

链接地址：链接脚本中指定的代码运行位置（程序员期望的运行地址）

运行地址：当前程序正在运行的地址（PC）

理论上来讲可以将代码加载到内存的任意位置然后跳转到相应位置运行，如果代码是位置有关码且和链接地址不一致可能会出现问题。

##### uboot链接脚本解析

ENTRY(_start)_

<!--ENTRY标号指定的是可执行函数的入口，_start函数是为main函数执行做准备工作。对于用户程序来说main是执行入口，但对整个可执行文件来说，_start才是执行入口。-->

`SECTIONS {` 

​	`. = 0x00000000;`

<!--链接起始地址，ld最终链接时会被-Ttext $(TEXT_BASE)更新，生成的uboot.bin的链接地址都是从TEXT_BASE开始的-->

```
	. = ALIGN(4);

​	.text : 

​	{ 	

​		*(.__image_copy_start) 	
```

<!--
__image_copy_start的定义如下：
*char __image_copy_start[0] __attribute__((section(".__image_copy_start")));在链接时不占用链接空间地址。定义了一个零长度的数组，并指定存放到section(".__image_copy_start")中，
因为零长数组不占用空间，所以就代表了uboot的起始地址，相当于创建了一个位置标签
-->

​		`*(.vectors) `

<!--
将vectors段链接到此处
-->

​		`CPUDIR/start.o (.text*)` 	

<!--
根据CPU类型编译start.S,编译出start.o中的.text段链接到此处
-->

​		`*(.text*)`

<!--
剩余所有代码到链接到此处
-->

```
	}

​		. = ALIGN(4);

​		.rodata : { *(SORT_BY_ALIGNMENT(SORT_BY_NAME(.rodata*))) }

​		. = ALIGN(4); 

​		.data : { 

​			*(.data*)

​		 }		

​		.u_boot_list : { 	

​				KEEP(*(SORT(.u_boot_list*))); 

​		}
```

<!--
  所有以u_boot_list名称开始的段都链接到这，主要是一些uboot命令相关的代码，KEEP是为了保证所有段都被加进来，SORT排序
-->

```
. = ALIGN(4);

.image_copy_end : { 

​	*(.__image_copy_end) 

}
```

<!--
拷贝结束时的位置标签
-->

```
 `.rel_dyn_start :
 {
  *(.__rel_dyn_start)
 }

 .rel.dyn : {
  *(.rel*)
 }
 .rel_dyn_end :
 {
  *(.__rel_dyn_end)}.end :{*(.__end)}_image_binary_end = .;. = ALIGN(4096);.mmutable : {*(.mmutable)}

​	......

}
```

<!--
位置标签，重定位时全局变量的LABLE的起始地址，下面end时结束地址
-->

##### 重定位理解

代码位于arch/arm/lib/relocate.S

```
ENTRY(relocate_code)

  ldr r1, =__image_copy_start /* r1 <- SRC &__image_copy_start */

  subs  r4, r0, r1   /* r4 <- relocation offset */

  beq relocate_done    /* skip relocation */

  ldr r2, =__image_copy_end  /* r2 <- SRC &__image_copy_end */

copy_loop:

  ldmia  r1!, {r10-r11}   /* copy from source address [r1]  */

  stmia  r0!, {r10-r11}   /* copy to  target address [r0]  */

  cmp r1, r2     /* until source end address [r2]  */

  blo copy_loop`
```

<!--__image_copy_start是在连接脚本中的uboot镜像起始位置标签，将起始位置标签中的地址赋给r1,r0为重定位后的地址，r0-r1的值就是搬运的偏移值存放于r4，判断r4是否为0，为0则跳转到重定位完成。image_copy_end是在连接脚本中的uboot镜像结束位置标签，将标签位置地址赋给r2。-->

<!--随后开始进入uboot镜像拷贝环节，ldmia:load memory  after increase,将r1指向的地址与r1指向的地址+4赋给r10-r11，随后stmid（store memory  after increase）r10-r11的值赋给r0指向的地址和r0指向的地址+4，r1!和r0!的表示每次地址+4字节。完成一次拷贝后比较一下r1和r2指向的地址是否相同，如果不同继续执行拷贝代码，如果地址相同则表示拷贝已经完成。-->

```
  ldr r2, =__rel_dyn_start  /* r2 <- SRC &__rel_dyn_start */

  ldr r3, =__rel_dyn_end /* r3 <- SRC &__rel_dyn_end */

fixloop:

  ldmia  r2!, {r0-r1}    /* (r0,r1) <- (SRC location,fixup) */

  and r1, r1, #0xff

  cmp r1, #R_ARM_RELATIVE

  bne fixnext

  /* relative fix: increase location by offset */

  add r0, r0, r4

  ldr r1, [r0]

  add r1, r1, r4

  str r1, [r0]

fixnext:

  cmp r2, r3

  blo fixloop

relocate_done:
```

<!--当为编译器指定编译选项-fpic火-fpie时会产生位置无关码。uboot中只指定了-pie给ld，但没有指定-fPIC给gcc。-->

<!--指定-pie后编译生成的uboot中就会有一个rel.dyn段，uboot通过rel.dyn段实现完美的重定位。rel_dyn_start和rel_dyn_end 是头部label和尾部label的地址。-->

<!--将头部label和尾部label的地址赋给r2,r3。然后将ldmia将r2!指向地址的值赋给r0,r1。r0储存变量地址，r1的固定值00000017，所以通过r1来验证rel.dyn段是否正确，R_ARM_RELATIVE的值就是17,如果r1不为17,比较r2,r3指向的地址是否一致判断是否完成拷贝label，如果r1为17，r0=r0+r4，r0存放label变量地址加上r4中存放的偏移地址量，就得到重定位后的label地址。然后将r0指向的地址存放的值（也就是变量的地址）赋给r1，r1=r1+r4，变量地址也同样加上偏移地址量得到重定位后的变量地址。最后将r1指向的地址存放到r0指向的地址的值中，继续重定位偏移知道覆盖整个rel.dyn段。-->

#### 启动代码流程分析

##### start.S(arch/arm/cpu/armv7/start.S)

```
  mrs r0, cpsr

  and r1, r0, #0x1f    @ mask mode bits

  teq r1, #0x1a    @ test for HYP mode

  bicne  r0, r0, #0x1f    @ clear all mode bits

  orrne  r0, r0, #0x13    @ set SVC mode

  orr r0, r0, #0xc0    @ disable FIQ and IRQ

  msr cpsr,r0
```

设置CPU为SVC模式，关闭中断和快中断使能；

```
  ldr r0, =_start

  mcr p15, 0, r0, c12, c0, 0 @Set VBAR
```

设置异常向量表地址；

```
bl cpu_init_cp15



ENTRY(cpu_init_cp15)

  /*

   \* Invalidate L1 I/D

   */

  mov r0, #0     @ set up for MCR

  mcr p15, 0, r0, c8, c7, 0  @ invalidate TLBs

  mcr p15, 0, r0, c7, c5, 0  @ invalidate icache

  mcr p15, 0, r0, c7, c5, 6  @ invalidate BP array

  mcr   p15, 0, r0, c7, c10, 4 @ DSB

  mcr   p15, 0, r0, c7, c5, 4  @ ISB



  /*

   \* disable MMU stuff and caches

   */

  mrc p15, 0, r0, c1, c0, 0

  bic r0, r0, #0x00002000 @ clear bits 13 (--V-)

  bic r0, r0, #0x00000007 @ clear bits 2:0 (-CAM)

  orr r0, r0, #0x00000002 @ set bit 1 (--A-) Align

  orr r0, r0, #0x00000800 @ set bit 11 (Z---) BTB

\#if CONFIG_IS_ENABLED(SYS_ICACHE_OFF)

  bic r0, r0, #0x00001000 @ clear bit 12 (I) I-cache

\#else

  orr r0, r0, #0x00001000 @ set bit 12 (I) I-cache

\#endif

  mcr p15, 0, r0, c1, c0, 0
```

关闭cache和mmu，关闭cache和mmu的原因：cache可以cpu与内存之间提高读写数据效率，cache是CPU内部2级缓存，将常用的指令和数据放在CPU内部通过cp15协处理器来管理，当cpu刚上电时，i-cache可关可不关，但是d-cache必须关闭，因为刚开始cpu去取数据可能从cache中取，但此时的内存中的数据还没有缓存到cache中，会导致数据预取异常；mmu在设备刚上电没有任何作用，包括初始化设备时都是访问的实际地址，mmu的打开起不到作用所以关闭。

```
lowlevel_init
```

设置栈，进入C阶段；关闭看门狗，初始化时钟，内存；

```
#if defined(CONFIG_SPL_BUILD) && defined(CONFIG_SPL_STACK)
	ldr	sp, =CONFIG_SPL_STACK
#else
	ldr	sp, =CONFIG_SYS_INIT_SP_ADDR
#endif
	bic	sp, sp, #7 /* 8-byte alignment for ABI compliance */
#ifdef CONFIG_SPL_DM
	mov	r9, #0
#else
	/*
	 * Set up global data for boards that still need it. This will be
	 * removed soon.
	 */
#ifdef CONFIG_SPL_BUILD
	ldr	r9, =gdata
#else
	sub	sp, sp, #GD_SIZE
	bic	sp, sp, #7
	mov	r9, sp
#endif
#endif
	/*
	 * Save the old lr(passed in ip) and the current lr to stack
	 */
	push	{ip, lr}

	/*
	 * Call the very early init function. This should do only the
	 * absolute bare minimum to get started. It should not:
	 *
	 * - set up DRAM
	 * - use global_data
	 * - clear BSS
	 * - try to start a console
	 *
	 * For boards with SPL this should be empty since SPL can do all of
	 * this init in the SPL board_init_f() function which is called
	 * immediately after this.
	 */
	bl	s_init
	pop	{ip, pc}
```

设置一个临时的堆栈；

```
ENTRY(_main)



/*

 \* Set up initial C runtime environment and call board_init_f(0).

 */



\#if defined(CONFIG_TPL_BUILD) && defined(CONFIG_TPL_NEEDS_SEPARATE_STACK)

  ldr r0, =(CONFIG_TPL_STACK)

\#elif defined(CONFIG_SPL_BUILD) && defined(CONFIG_SPL_STACK)

  ldr r0, =(CONFIG_SPL_STACK)

\#else

  ldr r0, =(CONFIG_SYS_INIT_SP_ADDR)

\#endif

  bic r0, r0, #7 /* 8-byte alignment for ABI compliance */

  mov sp, r0

  bl board_init_f_alloc_reserve

  mov sp, r0

  /* set up gd here, outside any C code */

  mov r9, r0

  bl board_init_f_init_reserve



\#if defined(CONFIG_SPL_BUILD) && defined(CONFIG_SPL_EARLY_BSS)

  CLEAR_BSS

\#endif



  mov r0, #0

  bl board_init_f



\#if ! defined(CONFIG_SPL_BUILD)



/*

 \* Set up intermediate environment (new sp and gd) and call

 \* relocate_code(addr_moni). Trick here is that we'll return

 \* 'here' but relocated.

 */



  ldr r0, [r9, #GD_START_ADDR_SP] /* sp = gd->start_addr_sp */

  bic r0, r0, #7 /* 8-byte alignment for ABI compliance */

  mov sp, r0

  ldr r9, [r9, #GD_NEW_GD]    /* r9 <- gd->new_gd */



  adr lr, here

  ldr r0, [r9, #GD_RELOC_OFF]   /* r0 = gd->reloc_off */

  add lr, lr, r0

\#if defined(CONFIG_CPU_V7M)

  orr lr, #1       /* As required by Thumb-only */

\#endif

  ldr r0, [r9, #GD_RELOCADDR]   /* r0 = gd->relocaddr */

  b  relocate_code

here:

/*

 \* now relocate vectors

 */



  bl relocate_vectors



/* Set up final (full) environment */



  bl c_runtime_cpu_setup /* we still call old routine here */

\#endif

\#if !defined(CONFIG_SPL_BUILD) || CONFIG_IS_ENABLED(FRAMEWORK)



\#if !defined(CONFIG_SPL_BUILD) || !defined(CONFIG_SPL_EARLY_BSS)

  CLEAR_BSS

\#endif



\# ifdef CONFIG_SPL_BUILD

  /* Use a DRAM stack for the rest of SPL, if requested */

  bl spl_relocate_stack_gd

  cmp r0, #0

  movne  sp, r0

  movne  r9, r0

\# endif



\#if ! defined(CONFIG_SPL_BUILD)

  bl coloured_LED_init

  bl red_led_on

\#endif

  /* call board_init_r(gd_t *id, ulong dest_addr) */

  mov   r0, r9         /* gd_t */

  ldr r1, [r9, #GD_RELOCADDR] /* dest_addr */

  /* call board_init_r */

\#if CONFIG_IS_ENABLED(SYS_THUMB_BUILD)

  ldr lr, =board_init_r  /* this is auto-relocated! */

  bx lr

\#else

  ldr pc, =board_init_r  /* this is auto-relocated! */

\#endif

  /* we should not return here. */

\#endif



ENDPROC(_main)
```

mov sp, r0 此时r0和sp都指向CONFIG_SYS_INIT_SP_ADDR;设置C语言运行环境；

```
ulong board_init_f_alloc_reserve(ulong top)

{

  /* Reserve early malloc arena */

\#if CONFIG_VAL(SYS_MALLOC_F_LEN)

  top -= CONFIG_VAL(SYS_MALLOC_F_LEN);

\#endif

  /* LAST : reserve GD (rounded up to a multiple of 16 bytes) */

  top = rounddown(top-sizeof(struct global_data), 16);

  return top;

}
```

参数ulong top是R0，即当前sp指向地址，top -= CONFIG_VAL(SYS_MALLOC_F_LEN);栈顶向上生长，预留出malloc内存区域；

  top = rounddown(top-sizeof(struct global_data), 16);top继续预留出存放global_data结构体的区域；

```
void board_init_f_init_reserve(ulong base)

{

  struct global_data *gd_ptr;



  /*

   \* clear GD entirely and set it up.

   \* Use gd_ptr, as gd may not be properly set yet.

   */



  gd_ptr = (struct global_data *)base;

  /* zero the area */

  memset(gd_ptr, '\0', sizeof(*gd));

  /* set GD unless architecture did it already */

\#if !defined(CONFIG_ARM)

  arch_setup_gd(gd_ptr);

\#endif



  if (CONFIG_IS_ENABLED(SYS_REPORT_STACK_F_USAGE))

​    board_init_f_init_stack_protection_addr(base);



  /* next alloc will be higher by one GD plus 16-byte alignment */

  base += roundup(sizeof(struct global_data), 16);



  /*

   \* record early malloc arena start.

   \* Use gd as it is now properly set for all architectures.

   */



\#if CONFIG_VAL(SYS_MALLOC_F_LEN)

  /* go down one 'early malloc arena' */

  gd->malloc_base = base;

\#endif



  if (CONFIG_IS_ENABLED(SYS_REPORT_STACK_F_USAGE))

​    board_init_f_init_stack_protection();

}
```

初始化gd结构体，数据清零；gd_ptr = (struct global_data *)base;指向栈顶位置，向下248字节是预留的GD结构体，memset(gd_ptr, '\0', sizeof(*gd));这248B的数字清零；base += roundup(sizeof(struct global_data), 16);BASE地址对其到256;gd->malloc_base = base;将地址写入到全局变量结构体中；

