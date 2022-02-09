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

<!--__image_copy_start是在连接脚本中的uboot镜像起始位置标签，将起始位置标签中的地址赋给r1,r0为重定位后的地址，r0-r1的值就是搬运的偏移值存放于r4，判断r4是否为0，为0则跳转到重定位完成。image_copy_end是在连接脚本中的uboot镜像结束位置标签，将标签位置地址赋给r2。-->

<!--随后开始进入uboot镜像拷贝环节，ldmia:load memory  after increase,将r1指向的地址与r1指向的地址+4赋给r10-r11，随后stmid（store memory  after increase）r10-r11的值赋给r0指向的地址和r0指向的地址+4，r1!和r0!的表示每次地址+4字节。完成一次拷贝后比较一下r1和r2指向的地址是否相同，如果不同继续执行拷贝代码，如果地址相同则表示拷贝已经完成。-->

  `ldr r2, =__rel_dyn_start  /* r2 <- SRC &__rel_dyn_start */`

  `ldr r3, =__rel_dyn_end /* r3 <- SRC &__rel_dyn_end */`

`fixloop:`

  `ldmia  r2!, {r0-r1}    /* (r0,r1) <- (SRC location,fixup) */`

  `and r1, r1, #0xff`

  `cmp r1, #R_ARM_RELATIVE`

  `bne fixnext`

  `/* relative fix: increase location by offset */`

  `add r0, r0, r4`

  `ldr r1, [r0]`

  `add r1, r1, r4`

  `str r1, [r0]`

`fixnext:`

  `cmp r2, r3`

  `blo fixloop`

`relocate_done:`

<!--当为编译器指定编译选项-fpic火-fpie时会产生位置无关码。uboot中只指定了-pie给ld，但没有指定-fPIC给gcc。-->

<!--指定-pie后编译生成的uboot中就会有一个rel.dyn段，uboot通过rel.dyn段实现完美的重定位。rel_dyn_start和rel_dyn_end 是头部label和尾部label的地址。-->

<!--将头部label和尾部label的地址赋给r2,r3。然后将ldmia将r2!指向地址的值赋给r0,r1。r0储存变量地址，r1的固定值00000017，所以通过r1来验证rel.dyn段是否正确，R_ARM_RELATIVE的值就是17,如果r1不为17,比较r2,r3指向的地址是否一致判断是否完成拷贝label，如果r1为17，r0=r0+r4，r0存放label变量地址加上r4中存放的偏移地址量，就得到重定位后的label地址。然后将r0指向的地址存放的值（也就是变量的地址）赋给r1，r1=r1+r4，变量地址也同样加上偏移地址量得到重定位后的变量地址。最后将r1指向的地址存放到r0指向的地址的值中，继续重定位偏移知道覆盖整个rel.dyn段。-->



