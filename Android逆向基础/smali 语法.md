## 0x01. smali概述
smali 最早起源于Jasmin,后被aosp采用为android虚拟机字节码,随后jesusfreke开发了最有名的smali和baksmali工具将其发扬光大，
他是一种jvm中的assembler语法,在art之前的davilik采用jit来即时翻译它运行.  
### 0x011. smali&java&dex之间的关系
摘自quora
> But their dex files are available which would be in a totally unreadable.  
> So to edit it we need to convert this .dex files to a more understandable form.  
> This is where smali comes in. To make it easier to understand I can represent it like this:  
> .dex <------------------> .smali <--------------------- java source code  
> Converting a .dex file to smali (called baksmaling) gives us readable code in smali language.
> Now if you wonder why can't smali be converted into java source, that's because java is a very developed language and smali is more of an assembly based language. And so while going from java source to smali information is lost and that's why smali can't be used to completely reconstruct java source code

## 0x02. smali的基本类型
Dalvik字节码只有两种类型:基本类型和引用类型.这两种类型就可以完整的表示java世界的所有类型.  
dalvik寄存器都是32位大小的,对于J D这种64位类型的需要用两个寄存器来存放,比如v0与v1 

## 0x03. smali的方法
方法格式如下:    
`Lpackage/name/ObjectName;->MethodName(III)Z`  
Lpackage/name/ObjectName; 表示ObjectName这个类型,MethodName是具体方法名,参数是int,int,int, Z表示返回值是boolean
另外一个复杂一点的例子:  
```
method(I[[IILjava/lang/String;[Ljava/lang/Object;)Ljava/lang/String
String method(int,int[][],int,String,Object[])  
```
构造方法  
 `.method public constructor <init>()V`  

构造调用    
 `Landroid/app/Activity;-><init>()V`  

一般的,invoke了的方法表示的都是super()  

## 0x04. smali的指令
指令集 : smali指令大全  
Dalvik 指令在调用格式上模仿了C语言的调用约定.Dalvik 指令的语法与助词符有如下特点:  
> * 参数采用从目标（ destination ）到源（ source ）的方式
> * 根据字节码的大小与类型不同，一些字节码添加了名称后缀以消除歧义 > > * 32 位常规类型的字节码未添加任何后缀。 > > * 64 位常规类型的字节码添加-wide后缀 > > * 特殊类型的字节码根据具体类型添加后缀。它们可以是-boolean、-byte、-char、-short、-int、-long、-float、-double、-object、-string、-class、-void 之一
> * 根据字节码的布局与选项不同，一些字节码添加了字节码后缀以消除歧义。这些后缀通过在字节码主名称后添加斜杠“ / ”来分隔开
> * 在指令集的描述中，宽度值中每个字母表示宽度为 4 位

例如这条指令
`move-wide/from16 vAA , vBBBB   ` 
> * move 为基础字节码（ base opcode ）。标识这是基本操作
> * wide 为名称后缀（ name suffix ）。标识指令操作的数据宽度（ 64 位）
> * from16 为字节码后缀（opcode suffix ）。标识源为一个 16 位的寄存器引用变量。
> * vAA 为目的寄存器。它始终在源的前面，取值范围为 vo-v255 。
> * vBBBB 为源寄存器。取值范围为 vo-v65535 。

Dalvik 指令集中大多数指令用到了寄存器作为目的操作数或源操作数，其中 A/B/C/ D/E/F/G/H 代表一个 4 位的数值，可用来表示 0 一15的数值或 vo 一 v15 的寄存器，而 AA / BB / CC / DD / EE / FF / GG / HH 代表一个 8 位的数值，可用来表示 0 一 255 的数位或 v0 一 v255 的寄存器， AAAA/BBBB / CCCC / DDDD / EEEE / FFFF / GGGG / HHHH 代表一个 8 位的数值，可用来表示 0 一 65535 的数值或 vo 一 v65535 的寄存器。  
### 0x041.空指令
nop 值为00,用来代码对齐,无用处  
### 0x042.数据操作指令
数据操作指令为move,move指令的原型为move destination,source 或 move destination , move 指令根据字节码的大小与类型不同,后面会跟上不同的后缀.  

> * move vA , vB 将 vB 寄存器的值赋给 vA 寄存器，源寄存器与目的寄存器都为 4 位.
> * move vA , vB 将 vB 寄存器的值赋给 vA 寄存器，源寄存器与目的寄存器都为 4 位.
> * move / from 16 vAA , vBBBB 将 vBBBB 寄存器的值赋给 vAA寄存器，源寄存器为16 位,目的寄存器为 8 位.
> * move / 16 vAAAA , vBBBB 将 vBBBB 寄存器的值赋给 vAAAA 寄存器，源寄存器与目的寄存器都为 16 位.
> * move-wide vA , vB 为 4 位的寄存器对赋值.源寄存器与目的寄存器都为 4 位.
> * move-wide/from16 vAA , vBBBB 与 move-wide/16 vAAAA , vBBBB 实现与move-wide相同
> * move-object vA , vB 为对象赋值.源寄存器与目的寄存器都为4位.
> * move-object/from16 vAA , vBBBB 为对象赋值,源寄存器为 16位，目的寄存器为8位.
> * move-object/16 vAAAA,vBBBB 为对象赋值源寄存器与目的寄存器都为 16 位.
> * move-result vAA 将上一个 invoke 类型指令操作的单字非对象结果赋给 vAA 寄存器.
> * move-result-wide vAA 将上一个invoke类型指令操作的双字非对象结果赋给 vAA 寄存器
> * move-result-object vAA 将上一个 invoke 类型指令操作的对象结果赋给 vAA 寄存器.
> * move-excecption vAA 保存一个运行时发生的异常到 vAA 寄存器.这条指令必须是异常发生时的异常处理器的一条指令.否则的话，指令无效.

### 0x043.返回指令
返回指令指的是函数结尾时运行的最后一条指令.它的基础字节码为retum,共有以下四条返回指令.  

> * return-void表示函数从一个void方法返回.
> * return vAA表示函数返回一个32位非对象类型的值，返回值寄存器为8位的寄存器vAA.
> * return-wide vAA表示函数返回一个64位非对象类型的值.返回值为8位的寄存器对vAA.
> * return-object vAA表示函数返回一个对象类型的值.返回值为8位的寄存器vAA.

### 0x044.数据定义指令
数据定义指令用来定义程序中用到的常量、字符串、类等数据.它的基础字节码为const.  
&#35;+X 表示它是一个常量数字，+X 表示它是一个相对指令的地址偏移，kind@X 表示它是一个常量池索引值。  

> * const/4 vA,#+B 将数值符号扩展为32位后赋给寄存器vA
> * const/16 vAA,#+BBBB将数值符号扩展为32位后赋给寄存器vAA.
> * const vAA,#+BBBBBBBB 将数值赋给寄存器vAA.
> * const/high16 vAA,#+BBBB0000 将数值右边零扩展为32位后赋给寄存器vAA
> * const-wide/16 vAA,#+BBBB 将数值符号扩展为64位后赋给寄存器对vAA
> * const-wide/32 vAA.#+BBBBBBBB 将数值符号扩展为64位后赋给寄存器对vAA
> * const-wide vAA,#+BBBBBBBBBBBBBBBB 将数值赋给寄存器对vAA.
> * const-wide/high16 vAA,#+BBBB000000000000 将数值右边零扩展为64位后赋给寄存器对vAA
> * const-string vAA,string@BBBB 通过字符串索引构造一个字符串并赋给寄存器vAA.
> * const-string/jumbo vAA,string@BBBBBBBB 通过字符串索引（较大）构造一个字符串并赋给寄存器vAA.
> * const-class vAA,type@BBBB 通过类型索引获取一个类引用并赋给寄存器vAA
> * const-class/jumbo vAAAA,type@BBBBBBBB 通过给定的类型索引获取一个类引用并赋给寄存器vAAAA.这条指令占用两个字节，值为ox00ff(Android4.0中新增的指令）

### 0x045.锁指令
锁指令多用在多线程程序中对同一对象的操作.Dalvik指令集中有两条锁指令.  

> * monitor-enter vAA 为指定的对象获取锁.
> * monitor-exit vAA 释放指定的对象的锁.

### 0x046.实例操作指令
与实例相关的操作包括实例的类型转换、检查及新建等  

> * check-cast vAA,type@BBBB 将vAA寄存器中的对象引用转换成指定的类型，如果失败会抛出ClassCastException异常.如果类型B指定的是基本类型，对于非基本类型的A来说，运行时始终会失败.
> * instance-of vA,vB,type@CCCC 判断vB寄存器中的对象引用是否可以转换成指定的类型，如果可以vA寄存器赋值为1，否则vA寄存器赋值为0.
> * new-instance vAA,type@BBBB 构造一个指定类型对象的新实例，并将对象引用赋值给vAA寄存器，类型符type指定的类型不能是数组类
> * check-cast/jumbo vAAAA,type@BBBBBBBB 指令功能与check-cast vAA,type@BBBB相同，只是寄存器值与指令的索引取值范围更大（Android4.0中新增的指令）
> * instance-of/jumbo vAAAA,vBBBB,type@CCCCCCCC 指令功能与 instance-of vA,vB,type@CCCC”相同，只是寄存器值与指令的索引取值范围更大（Android4.0中新增的指令）
> * new-instance/jumbo vAAAA,type@BBBBBBBB 指令功能与new-instance vAA,type@BBBB 相同，只是寄存器值与指令的索引取值范围更大（Android4.0中新增的指令）.

### 0x047.数组操作指令
数组操作包括读取数组长度、新建数组、数组赋值、数组元素取值与赋值等操作。  

> * array-length vA,vB 获取给定vB寄存器中数组的长度并将值赋给vA寄存器，数组长度指的是数组的条目个数。
> * new-array vA,vB,type@CCCC 构造指定类型（type@CCCC）与大小（vB）的数组，并将值赋给vA寄存器。
> * new-array/jumbo vAAAA,vBBBB,type@CCCCCCCC 指令功能与上一条指令相同，只是寄存器与指令的索引取值范围更大（Android4.0中新增的指令）
> * filled-new-array {vC,vD,vE,vF,vG},type@BBBB 构造指定类型（type@BBBB）与大小（vA）的数组并填充数组内容。vA寄存器是隐含使用的，除了指定数组的大小外还制订了参数的个数，vC~vG是使用到的参数寄存器序列
> * filled-new-array/range {vCCCC, … ,vNNNN},type@BBBB 指定功能与上一条指令相同，只是参数寄存器使用range字节码后缀指定了取值范围，vC是第一个参数寄存器， N=A+C-1。
> * filled-new-array/jumbo {vCCCC, … ,vNNNN},type@BBBBBBBB 指令功能与上一条指令相同，只是寄存器与指令的索引取值范围更大（Android4.0中新增的指令） fill-array-data vAA, +BBBBBBBB 用指定的数据来填充数组，vAA寄存器为数组引用，引用必须为基础类型的数组，在指令后面会紧跟一个数据表
> * arrayop vAA,vBB,vCC 对vBB寄存器指定的数组元素进入取值与赋值。vCC寄存器指定数组元素索引，vAA寄存器用来寄放读取的或需要设置的数组元素的值。读取元素使用 aget类指令，元素赋值使用aput指令，根据数组中存储的类型指令后面会紧跟不同的指令后缀，指令列表有aget、 aget-wide、aget-object、aget-boolean、aget-byte、aget-char、aget-short、aput、 aput-wide、aput-boolean、aput-byte、aput-char、aput-short。

### 0x048.异常指令
Dalvik指令集有一条指令用来抛出异常  

> * throw vAA 抛出vAA寄存器中指定类型的异常。


### 0x049.跳转指令

> * 跳转指令用于从当前地址跳转到偏移处。Dalvik指令集中有三种跳转指令：无条件跳转（goto）、分支跳转（switch）与条件跳转（if）。
> * goto +AA 无条件跳转到指定偏移处，偏移量AA不能为0
> * goto/16 +AAAA 无条件跳转到指定偏移处，偏移量AAAA不能为0。
> * goto/32 +AAAAAAAA 无条件跳转到指定偏移处。
> * packed-switch vAA,+BBBBBBBB 分支跳转指令。vAA寄存器为switch分支中需要判断的值，BBBBBBBB指向一个packed-switch-payload格式的偏移表，表中的值是有规律递增的。
> * sparse-switch vAA,+BBBBBBBB 分支跳转指令。vAA寄存器为switch分支中需要判断的值，BBBBBBBB指向一个sparse-switch-payload格式的偏移表，表中的值是无规律的偏移表，表中的值是无规律的偏移量。
> * if-test vA,vB,+CCCC 条件跳转指令。比较vA寄存器与vB寄存器的值，如果比较结果满足就跳转到CCCC指定的偏移处。偏移量CCCC不能为0。if-test类型的指令有以下几条： > > * if-eq 如果vA等于vB则跳转。Java语法表示为 if(vA == vB) > > * if-ne 如果vA不等于vB则跳转。Java语法表示为 if(vA != vB) > > * if-lt 如果vA小于vB则跳转。Java语法表示为 if(vA < vB) > > * if-le 如果vA小于等于vB则跳转。Java语法表示为 if(vA <= vB) > > * if-gt 如果vA大于vB则跳转。Java语法表示为 if(vA > vB) > > * if-ge 如果vA大于等于vB则跳转。Java语法表示为 if(vA >= vB)
> * if-testz vAA,+BBBB 条件跳转指令。拿vAA寄存器与 0 比较，如果比较结果满足或值为0时就跳转到BBBB指定的偏移处。偏移量BBBB不能为0。 if-testz类型的指令有一下几条： > > * if-eqz 如果vAA为 0 则跳转。Java语法表示为 if(vAA == 0) > > * if-nez 如果vAA不为 0 则跳转。Java语法表示为 if(vAA != 0) > > * if-ltz 如果vAA小于 0 则跳转。Java语法表示为 if(vAA < 0) > > * if-lez 如果vAA小于等于 0 则跳转。Java语法表示为 if(vAA <= 0) > > * if-gtz 如果vAA大于 0 则跳转。Java语法表示为 if(vAA > 0) > > * if-gez 如果vAA大于等于 0 则跳转。Java语法表示为 if(vAA >= 0)

### 0x0410.比较指令
比较指令用于两个寄存器的值（浮点型或长整型）进行比较。它的格式为 cmpkind vAA,vBB,vCC，其中vBB寄存器与vCC寄存器是需要比较的两个寄存器或者两个寄存器对，比较的结果放到vAA寄存器。Dalvik指令集中共有 5 条比较指令。  

> * cmpl-float 比较两个单精度浮点数。如果vBB寄存器小于vCC寄存器，则结果为1，相等则结果为0，大于的话结果为-1。
> * cmpg-float 比较两个单精度浮点数。如果vBB寄存器大于vCC寄存器，则结果为1，相等则结果为0，小于的话结果为-1。
> * cmpl-double 比较两个双精度浮点数。如果vBB寄存器小于vCC寄存器，则结果为1，相等则结果为0，大于的话结果为-1。
> * cmpg-double 比较两个双精度浮点数。如果vBB寄存器大于vCC寄存器，则结果为1，相等则结果为0，小于的话结果为-1。
> * cmp-long 比较两个长整型数。如果vBB寄存器大于vCC寄存器，则结果为1，相等则结果为0，小于的话结果为-1。


### 0x0411.字段操作指令

> * 字段操作指令用来对对象实例的字段进入读写操作。字段的类型那个可以是Java中有效的数据类型，对普通字段与静态字段操作有两种指令集，分别是iinstanceop vA,vB,field@CCCC 与 sstaticop vAA,field@BBBB
> * 普通字段指令的指令前缀为i，如对普通字段读操作使用iget指令，写操作使用iput指令；静态字段的指令前缀为s，如对静态字段读操作使用sget指令，写操作使用sput指令。
> * 根据访问的字段类型不同，字段操作指令后面会紧跟字段类型的后缀，如iget-byte指令表示读写实例字段的值类型为字节类型，iput-short指令表示设置实例字段的值类型为短整型。两类指令操作结果都是一样的，只是指令前缀与操作的字段类型不同。
> * 普通字段操作指令有：iget、iget-wide、iget-object、iget-boolean、iget-byte、iget-char、iget- short、iput、iput-wide、iput-object、iput-boolean、iput-byte、iput-char、iput-short。
> * 静态字段操作指令有：sget、sget-wide、sget-object、sget-boolean、sget-byte、sget-char、sget- short、sput、sput-wide、sput-object、sput-boolean、sput-byte、sput-char、sput- short。
> * 在Android4.0系统中，Dalvik指令集中增加了 instanceop/jumbo vAAAA,vBBBB,field@CCCCCCCC 与sstaticop/jumbo vAAAA,field@BBBBBBBB 两类指令，它们与上面介绍的两类指令作用相同，只是在指令中增加了jumbo字节码后缀，且寄存器值与指令的索引取值范围更大。

### 0x0412.方法调用指令

方法调用指令负责调用类实例的方法。它的基础指令为invoke，方法常用指令有 invoke-kind {vC,vD,vE,vF,vG},meth@BBBB 与 invoke-kind/range {vCCCC, … ,vNNNN},meth@BBBB 两类，两类指令在作用上并无不同，只是后则在设置参数寄存器时使用了range来指定寄存器的范围。根据方法类型的不同，共有如下 5 条方法调用指令：   

> * invoke-virtual 或 invoke-virtual/range 调用实例的虚方法
> * invoke-super 或 invoke-super/range 调用实例的父类方法
> * invoke-direct 或 invoke-direct/range 调用实例的直接方法
> * invoke-static 或 invoke-static/range 调用实例的静态方法
> * invoke-interface 或 invoke-interface/range 调用实例的接口方法

在 Android4.0系统中，Dalvik指令集中增加了 invoke-kind/jumbo {vCCCC, … ,vNNNN},meth@BBBBBBBB 这类指令，它与上面介绍的两类指令作用相同，只是在指令中增加了jumbo字节码后缀，且寄存器值与指令的索引取值范围更大。  
方法调用的指令的返回值必须使用move-result-> * 指令来获取。如下两条指令：  

> * invoke-static {},Landroid/os/Parcel;->obtain()Landroid/osParcel;
> * move-result-object v0

### 0x0413.数据转换指令
数据转换指令用于将一种类型的数值转换成另一种类型，它的格式为 unop vA,vB 。 vB寄存器或vB寄存器对存放需要转换的数据，转换后的结果保存在vA寄存器或vA寄存器对中。  

> * neg-int 对整型数求补
> * not-int 对整型数求反
> * neg-long 对长整型求补
> * not-long 对长整型求反
> * neg-float 对单精度浮点型数求补
> * neg-double 对双精度浮点型数求补
> * int-to-long 将整型数转换为长整型
> * int-to-float 将整型数转换为单精度浮点型
> * int-to-double 将整型数转换为双精度浮点型
> * long-to-int 将长整型数转换为整型
> * long-to-float 将长整型数转换为单精度浮点型
> * long-to-double 将长整型数转换为双精度浮点型
> * float-to-int 将单精度浮点型数转换为整型
> * float-to-long 将单精度浮点型数转换为长整型
> * float-to-double 将单精度浮点型数转换为双精度浮点型
> * double-to-int 将双精度浮点型数转换为整型
> * double-to-long 将双精度浮点型数转换为长整型
> * double-to-float 将双精度浮点型数转换为单精度浮点型
> * int-to-byte 将整型转换为字节型
> * int-to-char 将整型转换为字符串
> * int-to-short 将整型转换为短整型

### 0x0414.数据运算指令
数据运算指令包括算术运算指令与逻辑运算指令。算术运算指令主要进行数值间如加、减、乘、除、模、移位等运算，逻辑运算主要进行数值间与、或、非、异或等运算。数据运算指令有如下四类（数据运算时可能在寄存器或寄存器对间进行，下面的指令作用讲解时使用寄存器来描述）：  

> * binop vAA,vBB,vCC 将vBB寄存器与vCC寄存器进行运算，结果保存到vAA寄存器
> * binop/2addr vA,vB 将vA寄存器与vB寄存器进行运算，结果保存到vA寄存器
> * binop/lit16 vA,vB,#+CCCC 将vB寄存器与常量CCCC进行运算，结果保存到vA寄存器
> * binop/lit8 vAA,vBB,#+CC 将vBB寄存器与常量CC进行运算，结果保存到vAA寄存器

后面3类指令比第1类指令分别多了addr、lit16、lit8等指令后缀。四类指令中基础字节码后面加上数据类型后缀，如-int或-long分别表示操作的数据类型那个为整型与长整型。第1类指令可归类如下：  


> * add-type vBB寄存器与vCC寄存器值进行加法运算（vBB + vCC）
> * sub-type vBB寄存器与vCC寄存器值进行减法运算（vBB - vCC）
> * mul-type vBB寄存器与vCC寄存器值进行乘法运算（vBB > * vCC）
> * div-type vBB寄存器与vCC寄存器值进除法运算（vBB / vCC）
> * rem-type vBB寄存器与vCC寄存器值进行模运算（vBB % vCC）
> * and-type vBB寄存器与vCC寄存器值进行与运算（vBB & vCC）
> * xor-type vBB寄存器与vCC寄存器值进行异或运算（vBB ^ vCC）
> * shl-type vBB寄存器（有符号数）左移vCC位（vBB « vCC）
> * shr-type vBB寄存器（有符号数）右移vCC位（vBB » vCC）
> * ushr-type vBB寄存器（无符号数）右移vCC位（vBB » vCC）
> * or-type vBB 寄存器与vCC寄存器 值进行或运算（vBB | vCC）



其中基础字节码后面的-type可以是-int、-long、-float、-double。后面3类指令与之类似。    
至此Dalvik虚拟机支持的所有指令都介绍完了。在Android4.0系统以前，每个指令的字节码只在用一个字节，取值范围是0×0~-0x0ff，在 Android4.0系统中，有扩充了一部分指令，这些指令被成为扩展指令，如果指令后添加了jumbo后缀，增加了寄存器与常量的取值范围。    

## 0x05.类与包
### 0x051.类与继承与包
一般的smali文件都遵循了一套语法规范.在smali文件的头3行描述了当前类的一些信息.格式如下  
```
.class <访问权限>  [修辞关键字] <包名/类名>   
.super <包名/类名>  
.source "<原java类名>"  
```

比如  
```
.class public Lnet/smalinuxer/sdktest/MainActivity;
.super Landroid/app/Activity;
.source "MainActivity.java"
``` 
注:.source可能为空   
### 0x052.接口
如果一个类实现了一个接口将会以# interfaces开头  
`.implements <接口名>`

例如:  
```
# interfaces
.implements Ljava/lang/Thread
```
### 0x053.注解与泛型
```
.annotation [注解属性] <注解类名>
 [注解字段 = 值]
.end annotation
```
```
.field private infos:Ljava/util/Map;
.annotation system Ldalvik/annotation/Signature;
    value = {
        "Ljava/util/Map",
        "<",
        "Ljava/lang/String;",
        "Ljava/lang/String;",
        ">;"
    }
.end annotation
.end field
```
原：`private Map<String, String> infos = new HashMap<String, String>();` 这里表示泛型  
```
# instance fields
.field public sayWhat:Ljava/lang/String;
 .annotation runtime Lcom/droider/anno/MyAnnoField;
      info = "Hello World"
 .end annotation
.end field
```
原 : `@com.droider.anno.MyAnnoField(info = "Hello World")`

### 0x054.内部类
内部类将会成为另外一个smali文件 文件格式 : [外部类]$[内部类].smali 例如:Manifest$permission.smali  
在small中,内部类会自动保存外部类的引用,引用层数向下则指针标识加一  
## 0x06.属性
### 0x061.静态属性
一般的静态属性以 #static fields开头,#为注释进行标注 有如下格式: 
`.field <访问权限> static [修饰关键字] <字段名>:<字段类型>` 
例如:  
`.field private static final CONTENT_DISPOSITION_ATTRIBUTE_PATTERN:Ljava/util/regex/Pattern;`

### 0x062.实体属性  
一般的静态属性以 #instance fields开头 有如下格式:   
`.field <访问权限> [修饰关键字] <字段名>:<字段类型>`    

例如:  
`.field protected asyncRunner:Lnet/smalinuxer/mopp/httpd/NanoHTTPD$AsyncRunner;`  

## 0x07.方法
### 0x071.直接方法
一般的静态属性以 #direct methods开头 有如下格式:  
```
.method <访问权限>[修饰关键字]<方法原型>
<.locals>                    # 指定了使用的局部变量个数
[.parameter]                 # 指定了方法的参数,如果有三个参数就有三个.parameter
[.prologue]                  # 指定了代码开始段,混淆过的代码可能去掉了该段落
[.line]                      # 指定了该处指令在源代码中的行数,混淆过的代码可能会去掉
<代码体>
.end method
```
例如:  
```
.method public static makeSSLSocketFactory(Ljava/lang/String;[C)Ljavax/net/ssl/SSLServerSocketFactory;
.locals 10
.param p0, "keyAndTrustStoreClasspathPath"    # Ljava/lang/String;
.param p1, "passphrase"    # [C
.annotation system Ldalvik/annotation/Throws;
    value = {
        Ljava/io/IOException;
    }
.end annotation

.prologue
.line 1566
const/4 v5, 0x0

.line 1568
.local v5, "res":Ljavax/net/ssl/SSLServerSocketFactory;
:try_start_0
invoke-static {}, Ljava/security/KeyStore;->getDefaultType()Ljava/lang/String;

move-result-object v7

invoke-static {v7}, Ljava/security/KeyStore;->getInstance(Ljava/lang/String;)Ljava/security/KeyStore;

move-result-object v3

.line 1569
.local v3, "keystore":Ljava/security/KeyStore;
const-class v7, Lnet/smalinuxer/mopp/httpd/NanoHTTPD;

invoke-virtual {v7, p0}, Ljava/lang/Class;->getResourceAsStream(Ljava/lang/String;)Ljava/io/InputStream;

move-result-object v4

.line 1570
.local v4, "keystoreStream":Ljava/io/InputStream;
invoke-virtual {v3, v4, p1}, Ljava/security/KeyStore;->load(Ljava/io/InputStream;[C)V

.line 1571
invoke-static {}, Ljavax/net/ssl/TrustManagerFactory;->getDefaultAlgorithm()Ljava/lang/String;

move-result-object v7

invoke-static {v7}, Ljavax/net/ssl/TrustManagerFactory;->getInstance(Ljava/lang/String;)Ljavax/net/ssl/TrustManagerFactory;

move-result-object v6

.line 1572
.local v6, "trustManagerFactory":Ljavax/net/ssl/TrustManagerFactory;
invoke-virtual {v6, v3}, Ljavax/net/ssl/TrustManagerFactory;->init(Ljava/security/KeyStore;)V

.line 1573
invoke-static {}, Ljavax/net/ssl/KeyManagerFactory;->getDefaultAlgorithm()Ljava/lang/String;

move-result-object v7

invoke-static {v7}, Ljavax/net/ssl/KeyManagerFactory;->getInstance(Ljava/lang/String;)Ljavax/net/ssl/KeyManagerFactory;

move-result-object v2

.line 1574
.local v2, "keyManagerFactory":Ljavax/net/ssl/KeyManagerFactory;
invoke-virtual {v2, v3, p1}, Ljavax/net/ssl/KeyManagerFactory;->init(Ljava/security/KeyStore;[C)V

.line 1575
const-string v7, "TLS"

invoke-static {v7}, Ljavax/net/ssl/SSLContext;->getInstance(Ljava/lang/String;)Ljavax/net/ssl/SSLContext;

move-result-object v0

.line 1576
.local v0, "ctx":Ljavax/net/ssl/SSLContext;
invoke-virtual {v2}, Ljavax/net/ssl/KeyManagerFactory;->getKeyManagers()[Ljavax/net/ssl/KeyManager;

move-result-object v7

invoke-virtual {v6}, Ljavax/net/ssl/TrustManagerFactory;->getTrustManagers()[Ljavax/net/ssl/TrustManager;

move-result-object v8

const/4 v9, 0x0

invoke-virtual {v0, v7, v8, v9}, Ljavax/net/ssl/SSLContext;->init([Ljavax/net/ssl/KeyManager;[Ljavax/net/ssl/TrustManager;
                                                                       Ljava/security/SecureRandom;)V

.line 1577
invoke-virtual {v0}, Ljavax/net/ssl/SSLContext;->getServerSocketFactory()Ljavax/net/ssl/SSLServerSocketFactory;
:try_end_0
.catch Ljava/lang/Exception; {:try_start_0 .. :try_end_0} :catch_0

move-result-object v5

.line 1581
return-object v5

.line 1578
.end local v0    # "ctx":Ljavax/net/ssl/SSLContext;
.end local v2    # "keyManagerFactory":Ljavax/net/ssl/KeyManagerFactory;
.end local v3    # "keystore":Ljava/security/KeyStore;
.end local v4    # "keystoreStream":Ljava/io/InputStream;
.end local v6    # "trustManagerFactory":Ljavax/net/ssl/TrustManagerFactory;
:catch_0
move-exception v1

.line 1579
.local v1, "e":Ljava/lang/Exception;
new-instance v7, Ljava/io/IOException;

invoke-virtual {v1}, Ljava/lang/Exception;->getMessage()Ljava/lang/String;

move-result-object v8

invoke-direct {v7, v8}, Ljava/io/IOException;-><init>(Ljava/lang/String;)V

throw v7
.end method
```
## 0x08.注解
### 0x081.内部类注解
当产生一个内部类一定会存在EnclosingMethod的注解,这个注解是用来标注内部类范围,还有一个InnerClass的注解,表明该类是内部类, 例:  
```
# annotations
.annotation system Ldalvik/annotation/EnclosingMethod;
value = Lcom/example/atest/MainActivity;->onCreate(Landroid/os/Bundle;)V
.end annotation

.annotation system Ldalvik/annotation/InnerClass;
accessFlags = 0x0
name = null
.end annotation
```
标注了内部类范围为oncreate

### 0x082.其他注解
没什么用处  

## 0x09.R文件
R为自动生成的文件,包括了R.smali,R$attr.smali,R$dimen.smali,R$drawable.smali,R$id.smali,R$layout.smali,R$menu.smali,R$string.smali,R$style.smali 其中还有BuildConfig.smali,这个也是自动生成的文件。  



