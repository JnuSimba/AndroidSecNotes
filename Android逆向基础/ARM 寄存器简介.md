# ARM 寄存器简介
ARM处理器共有37个寄存器。其中包括：31个通用寄存器，包括程序计数器(PC)在内，这些寄存器都是32位寄存器；以及6个32位状态寄存器，但目前只使用了其中12位。ARM处理器共有7种不同的处理器模式，在每一种处理器模式中有一组相应的寄存器组。任意时刻(也就是任意的处理器模式下)，可见的寄存器包括15个通用寄存器(R0～R14)、一个或两个状态寄存器及程序计数器(PC)。在所有的寄存器中，有些是各模式共用的同一个物理寄存器，有一些寄存器是各模式自己拥有的独立的物理寄存器。下表列出了各处理器模式下可见的寄存器情况。  
<table style="width:100%;" border="1" bordercolor="#000000" cellpadding="1" cellspacing="0" height="100">
  <tbody>
    <tr>
      <td>User</td>

      <td>System</td>

      <td>Supervisor</td>

      <td>Abort</td>

      <td>Undefined</td>

      <td>IRQ</td>

      <td>FIQ</td>
    </tr>

    <tr>
      <td>R0</td>

      <td>R0</td>

      <td>R0</td>

      <td>R0</td>

      <td>R0</td>

      <td>R0</td>

      <td>R0</td>
    </tr>

    <tr>
      <td>R1</td>

      <td>R1</td>

      <td>R1</td>

      <td>R1</td>

      <td>R1</td>

      <td>R1</td>

      <td>R1</td>
    </tr>

    <tr>
      <td>R2</td>

      <td>R2</td>

      <td>R2</td>

      <td>R2</td>

      <td>R2</td>

      <td>R2</td>

      <td>R2</td>
    </tr>

    <tr>
      <td>R3</td>

      <td>R3</td>

      <td>R3</td>

      <td>R3</td>

      <td>R3</td>

      <td>R3</td>

      <td>R3</td>
    </tr>

    <tr>
      <td>R4</td>

      <td>R4</td>

      <td>R4</td>

      <td>R4</td>

      <td>R4</td>

      <td>R4</td>

      <td>R4</td>
    </tr>

    <tr>
      <td>R5</td>

      <td>R5</td>

      <td>R5</td>

      <td>R5</td>

      <td>R5</td>

      <td>R5</td>

      <td>R5</td>
    </tr>

    <tr>
      <td>R6</td>

      <td>R6</td>

      <td>R6</td>

      <td>R6</td>

      <td>R6</td>

      <td>R6</td>

      <td>R6</td>
    </tr>

    <tr>
      <td>R7</td>

      <td>R7</td>

      <td>R7</td>

      <td>R7</td>

      <td>R7</td>

      <td>R7</td>

      <td>R7</td>
    </tr>

    <tr>
      <td>R8</td>

      <td>R8</td>

      <td>R8</td>

      <td>R8</td>

      <td>R8</td>

      <td>R8</td>

      <td>R8_fiq</td>
    </tr>

    <tr>
      <td>R9</td>

      <td>R9</td>

      <td>R9</td>

      <td>R9</td>

      <td>R9</td>

      <td>R9</td>

      <td>R9_fiq</td>
    </tr>

    <tr>
      <td>R10</td>

      <td>R10</td>

      <td>R10</td>

      <td>R10</td>

      <td>R10</td>

      <td>R10</td>

      <td>R10_fiq</td>
    </tr>

    <tr>
      <td>R11</td>

      <td>R11</td>

      <td>R11</td>

      <td>R11</td>

      <td>R11</td>

      <td>R11</td>

      <td>R11_fiq</td>
    </tr>

    <tr>
      <td>R12</td>

      <td>R12</td>

      <td>R12</td>

      <td>R12</td>

      <td>R12</td>

      <td>R12</td>

      <td>R12_fiq</td>
    </tr>

    <tr>
      <td>R13</td>

      <td>R13</td>

      <td>R13_svc</td>

      <td>R13_abt</td>

      <td>R13_und</td>

      <td>R13_irq</td>

      <td>R13_fiq</td>
    </tr>

    <tr>
      <td>R14</td>

      <td>R14</td>

      <td>R14_svc</td>

      <td>R14_abt</td>

      <td>R14_und</td>

      <td>R14_irq</td>

      <td>R14_fiq</td>
    </tr>

    <tr>
      <td>PC</td>

      <td>PC</td>

      <td>PC</td>

      <td>PC</td>

      <td>PC</td>

      <td>PC</td>

      <td>PC</td>
    </tr>

    <tr>
      <td>CPSR</td>

      <td>CPSR</td>

      <td>CPSR</td>

      <td>CPSR</td>

      <td>CPSR</td>

      <td>CPSR</td>

      <td>CPSR</td>
    </tr>

    <tr>
      <td></td>

      <td></td>

      <td>SPSR_svc</td>

      <td>SPSR_abt</td>

      <td>SPSR_und</td>

      <td>SPSR_irq</td>

      <td>SPSR_fiq</td>
    </tr>
  </tbody>
</table>

## 1.通用寄存器的分类：
### a. 未备份寄存器，包括R0-R7  
   对每个未备份寄存器来说，在所有的模式下都是指同一个物理寄存器(例如：Usr下的R0与FIQ下的R0是同一个寄存器)。在异常程序中断造成模式切换时，由于不同模式使用的是相同的物理寄存器，这可能导致数据遭到破坏。未备份寄存器没有被系统作为别的用途，任何场合均可采用未备份寄存器。  
   R7对应于x86下的BP寄存器，相对与SP，R7就是栈底，在进入新一个栈帧之后先把原来的R7压栈，然后R7保存当前SP。  
   R0-R3用于传参数，调用函数的时候，参数先从R0依次传递，R0也用于返回值。  
   R4-R6 没有特殊规定，就是普通的通用寄存器。  
### b. 备份寄存器，包括R8-R14  
   对于备份寄存器R8-R12来说，除FIQ模式下其它模式均使用相同的物理寄存器。在FIQ模式下R8_fiq，R9_fiq，R10_fiq，R11_fiq，R12_fiq，它有自己的物理寄存器。对于R13和R14寄存器每种模式都有自己的物理寄存器(System与Usr的寄存器相同)，当异常中断发生时，系统使用相应模式下的物理寄存器，从而可以避免数据遭到破坏。
   R8，R10-R11 没有特殊规定，就是普通的通用寄存器。   
   R9 是操作系统保留。  
   R12 又叫IP(intra-procedure scratch )。  
   > Register r12 (IP) may be used by a linker as a scratch register between a routine and any subroutine it calls (for details, see §5.3.1.1, Use of IP by the linker). It can also be used within a routine to hold intermediate values between subroutine calls  
  > Both the ARM- and Thumb-state BL instructions are unable to address the full 32-bit address space, so it may be necessary for the linker to insert a veneer between the calling routine and the called subroutine. Veneers may also be needed to support ARM-Thumb inter-working or dynamic linking. Any veneer inserted must preserve the contents of all registers except IP (r12) and the condition code flags; a conforming program must assume that a veneer that alters IP may be inserted at any branch instruction that is exposed to a relocation that supports inter-working or long branches.  
  > 即是说现在如果汇编代码中存在bl指令，而r12又被用来作为通用寄存器，那么r12的值就很有可能会被链接器插入的veneer程序修改掉了。  
     
   R13也称为SP堆栈指针(stack pointer，用于存放栈顶指针)。    
   R14也称为LR寄存器(linked register，用于存放子程序的返回地址)。     
   R15也成为程序计数器(program counter，它的值是当前正在执行的指令在内存中的地址。      
### c.程序计数器，PC  
   PC寄存器存储指令地址，由于ARM采用流水机制执行指令，故PC寄存器总是存储下一条指令的地址。  
   由于ARM是按照字对齐，故PC被读取后的值的bit[1:0]总是0b00(thumb的bit[0]是0b0)。  

## 2.程序状态寄存器
程序状态寄存器包含当前程序状态寄存器和备份状态寄存器。  
### a.CPSR(程序状态寄存器，Current Program State Register)  
CPSR在任何处理器模式下都可以被访问。其结构如下：

  31 30 29 28  ---   7 &nbsp;  6 &nbsp;  5 &nbsp;  4 &nbsp;&nbsp;  3&nbsp; &nbsp;  2&nbsp; &nbsp;  1&nbsp; &nbsp;  0  
  N &nbsp;&nbsp;  Z&nbsp; &nbsp; C&nbsp; V   ---      I&nbsp;&nbsp;&nbsp;   F&nbsp;&nbsp;T &nbsp;      M4&nbsp;  M3 &nbsp; M2  M1  M0  

N(Negative)、Z(Zero)、C(Carry)以及V(oVerflow)称为条件标志位，ARM指令根据CPSR的条件标志位来选择地执行。

#### CPSR条件标志位

条件标志位 &nbsp;&nbsp;                  含义
N  &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;                        N=1 表示运算结果为负数，N=0 表示运算结果为正数。                 
Z  &nbsp;&nbsp;  &nbsp;&nbsp; &nbsp;&nbsp;                      Z=1 表示运算结果为0， Z=0 表示运算结果为非零。  
C   &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;                         C=1 表示运算结果产生了进位。  
V  &nbsp;&nbsp;  &nbsp;&nbsp;&nbsp;&nbsp;                      V=1 运算结果的符号位发生了溢出。  
Q  &nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp;                        在ARMv5 E系列版本中Q=1 表示DSP指令溢出，在ARMv5以前的版本中没有Q标志位。  

以下指令会影响CPSR的条件标志位  
(1) 比较指令，如: CMP、CMN、TEQ、TST等。  
(2) 当一些算术逻辑运算的目标寄存器不是PC时，这些指令会影响CPSR的条件标志位。  
(3) MSR与MRS指令可以对CPSR/SPSR进行操作。  
(4) LDM指令可以将SPSR复制到CPSR中。  

####  CPSR的控制位
控制位                        含义  
I&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;                           I=1 禁用IRO中断  
F&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;                           F=1 禁用FIQ中断  
T&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;                           ARMv4 以上T版本T=0 执行ARM指令，T=1执行Thumb指令，ARMv5以上非T版本T=0 执行ARM指令，T=1表示下一条指令产生未定义指令中断。  
M[4:0]&nbsp;&nbsp;                        控制处理器模式  
0b10000&nbsp;&nbsp;      User  
0b10001&nbsp;&nbsp;      FIQ  
0b10010 &nbsp;&nbsp;     IRQ  
0b10011&nbsp;&nbsp;      Supervisor  
0b10111&nbsp;&nbsp;      Abort  
0b11011&nbsp;&nbsp;      Undefined  
0b11111&nbsp;&nbsp;     System        

### b.SPSR(备份状态寄存器)
SPSR的结构与CPSR的结构相同，SPSR是用来备份CPSR的。