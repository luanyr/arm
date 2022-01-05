#### 冯诺依曼架构：

计算机硬件设备由存储器、运算器、控制器、输入设备和输出设备5部分组成；处理器使用同一个存储器，经由一个总线传输。

#### 哈弗架构：

哈弗架构是将程序指令存储和数据存储分开，处理器先到程序指令存储器中读取程序指令，解码后得到数据存储器地址，再到相应的数据存储器地址读数据，并进行下一步操作（通常是执行）。程序指令存储和数据存储分开，可以使**指令和数据有不同的数据宽度**。

![图片](https://mmbiz.qpic.cn/mmbiz_png/icRxcMBeJfc9ovpMhvqSpxxfcplZCVxd5lKmlCSww6ZoFgIObUuqULMnHgVSCTOafemgf59XZgzDKd6sUV1Lfmg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

​																			（ROM是程序指令存储器，RAM是数据存储器）

#### 冯诺依曼和哈弗架构的区别：

二者的区别是程序空间和数据空间是否是一体的。冯诺依曼架构是一体的，哈佛架构是分开的。

#### 哪些处理器是哈佛架构、冯诺依曼架构：

冯诺依曼：X86架构芯片，arm cortex-A系列芯片等

哈佛架构：arm cortex-M系列芯片

混合架构：现代的CPU（SOC）基本都不是纯粹的哈弗或者冯诺依曼



#### CPU工作原理

![图片](https://mmbiz.qpic.cn/mmbiz_png/icRxcMBeJfc9ovpMhvqSpxxfcplZCVxd5xnxEjrb0PyicXDn5gvYjeyJtktYQhqW6UWb83LSazfd9wCxzcXiaMTyQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

CPU内部包括运算器和控制器。

1）存储器

cache，主存储器，辅助存储器

![图片](https://mmbiz.qpic.cn/mmbiz_png/icRxcMBeJfc9ovpMhvqSpxxfcplZCVxd5Rs5Sbh3SSgJIO4E0vrW4raWX74lm0uyew7q1TF15XnEQHp3kkM8LDw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

2）运算器

3）控制器

4）CPU的运行原理

控制单元在时序脉冲的作用下，将指令计数器里所指向的指令地址(这个地址是在内存里的)送到地址总线上去，然后CPU将这个地址里的指令读到指令寄存器进行译码。

对于执行指令过程中所需要用到的数据，会将数据地址也送到地址总线，然后CPU把数据读到CPU的内部存储单元(就是内部寄存器)暂存起来，最后命令运算单元对数据进行处理加工。

周而复始，一直这样执行下去。

5）指令执行过程

取指，译指，执行，修改指令计数器

1、取指令：CPU的控制器从内存读取一条指令并放入指令寄存器。2、指令译码：指令寄存器中的指令经过译码，决定该指令应进行何种操作(就是指令里的操作码)、操作数在哪里(操作数的地址)。3、 执行指令，分两个阶段“取操作数”和“进行运算”。4、 修改指令计数器，决定下一条指令的地址。

![图片](https://mmbiz.qpic.cn/mmbiz_png/icRxcMBeJfc9ovpMhvqSpxxfcplZCVxd5tyGbARpg2RydOe2VufrLcVE98DK6lONw4iaXzM2fqnbeaahSPD1mqiaQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



