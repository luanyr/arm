#### 基于ARM中断详解

一般在系统中，中断控制分为三部分：模块，中断控制器和处理器；

其中模块通常由寄存器控制是否使能中断和中断触发条件；中断控制器可以管理中断的优先级，而处理器则由寄存器设置用来相应中断。

##### GIC

ARM系统中通用中断控制器是GIC（Generic Interrupt Controlle）

GIC通过AMBA（高级微控制器总线架构）片上总线连接到ARM处理器上。

![图片](https://mmbiz.qpic.cn/mmbiz_png/icRxcMBeJfc9ZxT0ayCRaKEQtnASgZrshS6DnK8VdkYOYZKNQb1ptibAJsEgzGypXDHUMQnBBnNMAqiaUV7qBkHfw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

从上图可以看到，GIC是联系外设中断和CPU的桥梁，也是各CPU之间中断互联的通道，负责检测，管理，分发中断，可以做到：

1.使能或禁止中断；（CPSR的I和F控制中断？）

ARM的CPSR（当前程序状态寄存器）中的I与F是ARM处理器内部控制所有的中断与快中断。

GIC使能禁止中断属于外部控制，可以针对具体中断。

对于各种中断都是通过GIC达到CPU的，所有的中断都使用同一个异常向量，即异常向量表的第七项IRQ异常，不同的中断通过软件区分。

2.把中断分组到GROUP0或者GROUP1（GROUP0作为安全模式使用连接FIQ，GROUP1作为非安全模式使用连接IRQ）；

3.多核系统将中断分配到不同处理器上；

4.设置电平触发还是边沿触发方式；（中断的触发方式？）

5.虚拟化扩展；（上图的virtual fiq和virtual irq）

ARM对外的中断连接只有：FIQ和IRQ，相对应的处理模式也就是FIQ模式和IRQ模式，所以GIC最后需要将中断汇集成FIQ和IRQ与CPU连接。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210117155217569.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4MTEzMDA2,size_16,color_FFFFFF,t_70)

##### GIC Distributor(分发器)

分发器的主要作用是检测各个中断源的状态，控制各个中断源的行为，分发各个中断源产生的中断事件到指定的CPU接口上。虽然分发器可以管理多个中断源，但是它总是把优先级最高的那个中断请求送往CPU接口。分发器对中断的控制包括：

1.中断使能或禁止。（通过启用或禁止向CPU接口转发中断功能，通过GICD寄存器控制。也可以针对单个中断源进行控制）

2.设置中断的优先级

3.设置每个外设中断的属性是边沿触发还是电平触发

4.设置中断组为0还是1

5.控制将当前优先级最高的中断事件分发到CPU接口

分发器可以管理若干中断源，这些中断源用ID来标识。

##### GIC CPU Interface(CPU接口)

用于与CPU接口

主要功能包括：

1.使能或者禁止CPU接口向连接的CPU提交中断事件，对于ARM，CPU接口和CPU之间的中断信号线是nIRQCPU和nFIQCPU。如果禁止了中断，那么即便是分发器分发了一个中断事件到CPU接口，但是也不会提交指定的nIRQ或者nFIQ通知CPU。

2.ackowledging中断

3.通知中断处理完毕

4.为处理器设置中断优先级掩码 ，通过优先级掩码可以屏蔽掉一些优先级比较低的中断，这些中断不会通知到CPU。

5.定义处理器的中断抢占策略  

6.在多个中断事件中选择一个优先级最高的通知CPU

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/icRxcMBeJfc9ZxT0ayCRaKEQtnASgZrshqzw0OWeicqu53rPG2GeI5gibqIrxUTcoSuDcjeflrN7S3aXicK0tCunyw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

上图表示一个中断到达CPU经过的流程:外设通过外部中断事件控制器到达GIC的分配器，分配器经过使能后，将中断信号分发给CPU接口，CPU接口使能会处理后会将中断信号送达CPU的IRQ接口，接着CPU会执行中断异常处理流程。

#### 中断分类

##### 硬件中断（Hardware Interrupt）

1. 可屏蔽中断（maskable interrupt）。硬件中断的一类，可通过在中断屏蔽寄存器中设定位掩码来关闭。
2. 非可屏蔽中断（non-maskable interrupt，NMI）。硬件中断的一类，无法通过在中断屏蔽寄存器中设定位掩码来关闭。典型例子是时钟中断（一个硬件时钟以恒定频率—如50Hz—发出的中断）。
3. 处理器间中断（interprocessor interrupt）。一种特殊的硬件中断。由处理器发出，被其它处理器接收。仅见于多处理器系统，以便于处理器间通信或同步。
4. 伪中断（spurious interrupt）。一类不希望被产生的硬件中断。发生的原因有很多种，如中断线路上电气信号异常，或是中断请求设备本身有问题。

##### 软件中断（Software Interrupt）

软件中断SWI,是一条CPU指令，用以自陷一个中断。由于软中断指令通常要运行一个切换CPU至内核态的子例程，它常被用作实现系统调用（System call）。

##### 外部中断

1. I/O设备：如显示器、键盘、打印机、A / D转换器等。
2. 数据通道：软盘、硬盘、光盘等。数据通道中断也称直接存储器存取（DMA）操作中断，如磁盘、磁带机或CRT等直接与存储器交换数据所要求的中断。
3. 实时时钟：如外部的定时电路等。在控制中遇到定时检测和控制，为此常采用一个外部时钟电路（可编程）控制其时间间隔。需要定时时，CPU发出命令使时钟电路开始工作，一旦到达规定时间，时钟电路发出中断请求，由CPU转去完成检测和控制工作。
4. 用户故障源：如掉电、奇偶校验错误、外部设备故障等。

##### 产生于CPU内部的中断源

1. 由CPU得运行结果产生：如除数为0、结果溢出、断点中断、单步中断、存储器读出出错等。
2. 执行中断指令swi
3. 非法操作或指令引起异常处理。

#### 中断类型

GIC 中断类型有3种：SGI(Software-generated interrupt)、PPI(Private peripheral interrupt )、SPI(Shared peripheral interrupt)。

1. SGI: SGI为软件可以触发的中断，统一编号为0~15(ID0-ID7是不安全中断，ID8-ID15是安全中断)，用于各个core之间的通信。该类中断通过相关联的中断号和产生该中断的处理器的CPUID来标识。通常为边沿触发。
2. PPI: PPI为每个 core 的私有外设中断，统一编号为 16-31 。例如每个 CPU 的 local timer 即 Arch Timer 产生的中断就是通过 PPI 发送给 CPU 的（安全为29，非安全为30）。

通常为边沿触发和电平触发。

1. SPI: SPI 是系统的外设产生的中断，为各个 core 公用的中断，统一编号为 32~1019 ，如 global timer 、uart 、gpio 产生的中断。通常为边沿触发和电平触发。

Note：电平触发是在高或低电平保持的时间内触发, 而边沿触发是由高到低或由低到高这一瞬间触发；在GIC中PPI和SGI类型的中断可以有相同的中断ID。

#### LINUX中断

##### 上半部和下半部

上半部就是中断处理函数，处理过程较快不占用长时间的处理可以放在上半部。

下半部是中断处理过程中处理比较耗时的代码放到下半部处理。

linux内核实现上半部和下半部的目的是实现中断处理函数的快进快出，对那些时间敏感，执行速度较快的操作放到中断处理函数中，也就是上半部。剩下的工作放到下半部执行。比如可以在中断处理函数中将数据拷贝到内存中，关于数据的具体的处理放到下半部。

关于上下部的区分可以参考:

①、如果要处理的内容不希望被其他中断打断，那么可以放到上半部。

②、如果要处理的任务对时间敏感，可以放到上半部。

③、如果要处理的任务与硬件有关，可以放到上半部

④、除了上述三点以外的其他任务，优先考虑放到下半部。

上半部只需要编写中断处理函数就行，linux内核提供以下多种下半部机制。

tasklet，softirq和workqueue。
