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

​	`. = ALIGN(4);`

​	`.text :` 

​	`{` 	

​		`*(.__image_copy_start)` 	

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

​	`}`

​		`. = ALIGN(4);`

​		`.rodata : { *(SORT_BY_ALIGNMENT(SORT_BY_NAME(.rodata*))) }`

​		`. = ALIGN(4);` 

​		`.data : {` 

​			`*(.data*)`

​		 `}`		

​		`.u_boot_list : {` 	

​				`KEEP(*(SORT(.u_boot_list*)));` 

​		`}`

<!--
  所有以u_boot_list名称开始的段都链接到这，主要是一些uboot命令相关的代码，KEEP是为了保证所有段都被加进来，SORT排序
-->

`. = ALIGN(4);`

`.image_copy_end : {` 

​	`*(.__image_copy_end)` 

`}`

<!--
拷贝结束时的位置标签
-->

 ``.rel_dyn_start :`
 `{`
  `*(.__rel_dyn_start)
 }`


 `.rel.dyn : {`
  `*(.rel*)`
 `}`
 `.rel_dyn_end :`
 `{`
  `*(.__rel_dyn_end)}.end :{*(.__end)}_image_binary_end = .;. = ALIGN(4096);.mmutable : {*(.mmutable)}`

​	`......`

`}`

<!--
位置标签，重定位时全局变量的LABLE的起始地址，下面end时结束地址
-->

##### 重定位理解

代码位于arch/arm/lib/relocate.S

`ENTRY(relocate_code)`

  `ldr r1, =__image_copy_start /* r1 <- SRC &__image_copy_start */`

  `subs  r4, r0, r1   /* r4 <- relocation offset */`

  `beq relocate_done    /* skip relocation */`

  `ldr r2, =__image_copy_end  /* r2 <- SRC &__image_copy_end */`

`copy_loop:`

  `ldmia  r1!, {r10-r11}   /* copy from source address [r1]  */`

  `stmia  r0!, {r10-r11}   /* copy to  target address [r0]  */`

  `cmp r1, r2     /* until source end address [r2]  */`

  `blo copy_loop`

