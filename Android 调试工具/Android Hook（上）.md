原文 by 蒸米  
## 0x00 序

随着移动安全越来越火，各种调试工具也都层出不穷，但因为环境和需求的不同，并没有工具是万能的。另外工具是死的，人是活的，如果能搞懂工具的原理再结合上自身的经验，你也可以创造出属于自己的调试武器。因此，笔者将会在这一系列文章中分享一些自己经常用或原创的调试工具以及手段，希望能对国内移动安全的研究起到一些催化剂的作用。  

文章中所有提到的代码和工具都可以在我的github下载到，地址是：  

https://github.com/zhengmin1989/TheSevenWeapons

## 0x01 离别钩

Hooking翻译成中文就是钩子的意思，所以正好配合这一章的名字《离别钩》。  
```
“我知道钩是种武器，在十八般兵器中名列第七，离别钩呢?”
“离别钩也是种武器，也是钩。”
“既然是钩，为什么要叫做离别?”
“因为这柄钩，无论钩住什么都会造成离别。如果它钩住你的手，你的手就要和腕离别；如果它钩住你的脚，你的脚就要和腿离别。”
“如果它钩住我的咽喉，我就和这个世界离别了?”
“是的，”
“你为什么要用如此残酷的武器?”
“因为我不愿被人强迫与我所爱的人离别。”
“我明白你的意思了。”
“你真的明白?”
“你用离别钩，只不过为了要相聚。”
“是的。”
```
一提到hooking，让我又回想起了2011年的时候。当时android才出来没多久，各大安全公司都在忙着研发自己的手机助手。当时手机上最泛滥的病毒就是短信扣费类的病毒，但仅仅是靠云端的病毒库扫描是远远不够的。而这时候”LBE安全大师”横空出世，提供了主动防御的技术，可以在病毒发送短信之前拦截下来，并让用户选择是否发送。 其实这个主动防御技术就是hooking。虽然在PC上hooking的技术已经很成熟了，但是在android上的资料却非常稀少，只有少数人掌握着android上hooking的技术，因此这些人也变成了各大公司争相抢夺的对象。但是没有什么东西是能够永久保密的，这些技术早晚会被大家研究出来并对外公开的。因此，到了2015年，android上的hook资料已经遍地都是了，各种开源的hook框架也层出不穷，使用这些hook工具就可以轻松的hook native，jni和java层的函数。但这同样也带来了一些问题，新手想研究hook的时候因为资料和工具太多往往不知道如何下手，并且就算使用了工具成功的hook，也根本不知道原理是什么。因此笔者准备从hook的原理开始，配合开源工具循序渐进的介绍native，jni和java层的hook，方便大家对hook进行系统的学习。  

## 0x02 Playing with Ptrace on Android

其实无论是hook还是调试都离不开ptrace这个system call，利用ptrace，我们可以跟踪目标进程，并且在目标进程暂停的时候对目标进程的内存进行读写。在linux上有一篇经典的文章叫《Playing with Ptrace》，简单介绍了如何玩转ptrace。在这里我们照猫画虎，来试一下playing with Ptrace on Android。PS：这里的一部分内容借鉴了harry大牛在百度Hi写的一篇文章，可惜后来百度Hi关了，就失传了。不过不用担心，我这篇比他写的还详细。：）  

首先来看我们要ptrace的目标程序，用来一直循环输出一句话”Hello,LiBieGou!”：  
``` c
#include <stdio.h>
int count = 0;

void sevenWeapons(int number)
{
    char* str = "Hello,LiBieGou!";
    printf("%s %d\n",str,number);
}

int main()
{
    while(1)
    {
        sevenWeapons(count);
        count++;
        sleep(1);
    }    
    return 0;
}
```
想要编译它非常简单，首先建立一个Android.mk文件，然后填入如下内容，让ndk将文件编译为elf可执行文件：  
```
LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)
LOCAL_MODULE    := target
LOCAL_SRC_FILES := target.c

include $(BUILD_EXECUTABLE)
```
接下来我们写出hook1.c程序来hook target程序的system call，main函数如下：  
``` c
int main(int argc, char *argv[])
{
    if(argc != 2) {
        printf("Usage: %s <pid to be traced>\n", argv[0]);
        return 1;
    }

    pid_t pid;
    int status;
    pid = atoi(argv[1]);

    if(0 != ptrace(PTRACE_ATTACH, pid, NULL, NULL))
    {
        printf("Trace process failed:%d.\n", errno);
        return 1;
    }

    ptrace(PTRACE_SYSCALL, pid, NULL, NULL);

    while(1)
    {
        wait(&status);
        hookSysCallBefore(pid);
        ptrace(PTRACE_SYSCALL, pid, NULL, NULL);

        wait(&status);
        hookSysCallAfter(pid);
        ptrace(PTRACE_SYSCALL, pid, NULL, NULL);
    }

    ptrace(PTRACE_DETACH, pid, NULL, NULL);
    return 0;
}
```
首先要知道hook的目标的pid，这个用ps命令就能获取到。然后我们使用ptrace(PTRACE_ATTACH, pid, NULL, NULL)这个函数对目标进程进行加载。加载成功后我们可以使用ptrace(PTRACE_SYSCALL, pid, NULL, NULL)这个函数来对目标程序下断点，每当目标程序调用system call前的时候，就会暂停下载。然后我们就可以读取寄存器的值来获取system call的各项信息。然后我们再一次使用ptrace(PTRACE_SYSCALL, pid, NULL, NULL)这个函数就可以让system call在调用完后再一次暂停下来，并获取system call的返回值。  

获取system call编号的函数如下：  
``` c
long getSysCallNo(int pid, struct pt_regs *regs)
{
    long scno = 0;
    scno = ptrace(PTRACE_PEEKTEXT, pid, (void *)(regs->ARM_pc - 4), NULL);
    if(scno == 0)
        return 0;

    if (scno == 0xef000000) {
        scno = regs->ARM_r7;
    } else {
        if ((scno & 0x0ff00000) != 0x0f900000) {
            return -1;
        }
        scno &= 0x000fffff;
    }
    return scno;    
}
```
ARM架构上，所有的系统调用都是通过SWI来实现的。并且在ARM 架构中有两个SWI指令，分别针对EABI和OABI：  

[EABI]  
机器码：`1110 1111 0000 0000` -- `SWI 0`  
具体的调用号存放在寄存器r7中.  

[OABI]  
机器码：`1101 1111 vvvv vvvv` -- `SWI immed_8`  
调用号进行转换以后得到指令中的立即数。立即数=调用号 | 0x900000  
既然需要兼容两种方式的调用，我们在代码上就要分开处理。首先要获取SWI指令判断是EABI还是OABI，如果是EABI，可从r7中获取调用号。如果是OABI，则从SWI指令中获取立即数，反向计算出调用号。  

我们接着看hook system call前的函数，和hook system call后的函数：  
``` c
void hookSysCallBefore(pid_t pid)
{
    struct pt_regs regs;
    int sysCallNo = 0;

    ptrace(PTRACE_GETREGS, pid, NULL, &regs);    
    sysCallNo = getSysCallNo(pid, &regs);
    printf("Before SysCallNo = %d\n",sysCallNo);

    if(sysCallNo == __NR_write)
    {
        printf("__NR_write: %ld %p %ld\n",regs.ARM_r0,(void*)regs.ARM_r1,regs.ARM_r2);
    }
}
```
``` c
void hookSysCallAfter(pid_t pid)
{
    struct pt_regs regs;
    int sysCallNo = 0;

    ptrace(PTRACE_GETREGS, pid, NULL, &regs);  
    sysCallNo = getSysCallNo(pid, &regs);

    printf("After SysCallNo = %d\n",sysCallNo);

    if(sysCallNo == __NR_write)
    {
        printf("__NR_write return: %ld\n",regs.ARM_r0);
    }

    printf("\n");
}
```
在获取了system call的number以后，我们可以进一步获取个个参数的值，比如说write这个system call。在arm上，如果形参个数少于或等于4，则形参由R0,R1,R2,R3四个寄存器进行传递。若形参个数大于4，大于4的部分必须通过堆栈进行传递。而执行完函数后，函数的返回值会保存在R0这个寄存器里。  

下面我们就来实际运行一下看看效果。我们先把target和hook1 push到 /data/local/tmp目录下，再chmod 777一下。接着运行target。  
``` bash
# ./target                                                                                                                                                              
Hello,LiBieGou! 0
Hello,LiBieGou! 1
Hello,LiBieGou! 2
Hello,LiBieGou! 3
Hello,LiBieGou! 4
Hello,LiBieGou! 5
Hello,LiBieGou! 6
….
```
我们随后再开一个shell，然后ps获取target的pid，然后使用hook1程序对target进行hook操作：  
``` bash
# ./hook1 23442
Before SysCallNo = 162
After SysCallNo = 162

Before SysCallNo = 4
__NR_write: 1 0xadf020 19
After SysCallNo = 4
__NR_write return: 19

Before SysCallNo = 162
After SysCallNo = 162

Before SysCallNo = 4
__NR_write: 1 0xadf020 19
After SysCallNo = 4
__NR_write return: 19
```
我们可以看到第一个SysCallNo是162，也就是sleep函数。第二个SysCallNo是4，也就是write函数，因为printf本质就是调用write这个系统调用来完成的。关于system call number对应的具体system call可以参考我在github上的reference文件夹中的systemcalllist.txt文件，里面有对应的列表。我们的hook1程序还对write的参数做了解析，比如1表示stdout，0xadf020表示字符串的地址，19代表字符串的长度。而返回值19表示write成功写入的长度，也就是字符串的长度。  

整个过程用图表达如下：    
![](../picutures/androidhook.jpg)  


## 0x03 利用Ptrace动态修改内存  

仅仅是用ptrace来获取system call的参数和返回值还不能体现出ptrace的强大，下面我们就来演示用ptrace读写内存。我们在hook1.c的基础上继续进行修改，在write被调用之前对要输出string进行翻转操作。  

我们在hookSysCallBefore()函数中加入modifyString(pid, regs.ARM_r1, regs.ARM_r2)这个函数：  
``` c
if(sysCallNo == __NR_write)
{
    printf("__NR_write: %ld %p %ld\n",regs.ARM_r0,(void*)regs.ARM_r1,regs.ARM_r2);
    modifyString(pid, regs.ARM_r1, regs.ARM_r2);
}
```
因为write的第二个参数是字符串的地址，第三个参数是字符串的长度，所以我们把R1和R2的值传给modifyString()这个函数：  
``` c
void modifyString(pid_t pid, long addr, long strlen)
{
    char* str;
    str = (char *)calloc((strlen+1) * sizeof(char), 1);
    getdata(pid, addr, str, strlen);
    reverse(str);
    putdata(pid, addr, str, strlen);
}
```
modifyString()首先获取在内存中的字符串，然后进行翻转操作，最后再把翻转后的字符串写入原来的地址。这些操作用到了getdata()和putdata()函数：  
``` c
void getdata(pid_t child, long addr,
             char *str, int len)
{   char *laddr;
    int i, j;
    union u {
            long val;
            char chars[long_size];
    }data;
    i = 0;
    j = len / long_size;
    laddr = str;
    while(i < j) {
        data.val = ptrace(PTRACE_PEEKDATA,
                          child, addr + i * 4,
                          NULL);
        memcpy(laddr, data.chars, long_size);
        ++i;
        laddr += long_size;
    }
    j = len % long_size;
    if(j != 0) {
        data.val = ptrace(PTRACE_PEEKDATA,
                          child, addr + i * 4,
                          NULL);
        memcpy(laddr, data.chars, j);
    }
    str[len] = '\0';
}

void putdata(pid_t child, long addr,
             char *str, int len)
{   char *laddr;
    int i, j;
    union u {
            long val;
            char chars[long_size];
    }data;
    i = 0;
    j = len / long_size;
    laddr = str;
    while(i < j) {
        memcpy(data.chars, laddr, long_size);
        ptrace(PTRACE_POKEDATA, child,
               addr + i * 4, data.val);
        ++i;
        laddr += long_size;
    }
    j = len % long_size;
    if(j != 0) {
        memcpy(data.chars, laddr, j);
        ptrace(PTRACE_POKEDATA, child,
               addr + i * 4, data.val);
    }
}
```
getdata()和putdata()分别使用PTRACE_PEEKDATA和PTRACE_POKEDATA对内存进行读写操作。因为ptrace的内存操作一次只能控制4个字节，所以如果修改比较长的内容需要进行多次操作。  

我们现在运行一下target，并且在运行中用hook2程序进行hook：  
``` bash
# ./target                                                                                                                                                              
Hello,LiBieGou! 0
Hello,LiBieGou! 1
Hello,LiBieGou! 2
Hello,LiBieGou! 3
Hello,LiBieGou! 4
Hello,LiBieGou! 5
Hello,LiBieGou! 6
Hello,LiBieGou! 7
8 !uoGeiBiL,olleH
9 !uoGeiBiL,olleH
01 !uoGeiBiL,olleH
11 !uoGeiBiL,olleH
21 !uoGeiBiL,olleH
31 !uoGeiBiL,olleH
Hello,LiBieGou! 14
Hello,LiBieGou! 15
Hello,LiBieGou! 16
```
哈哈，是不是看到字符串都被翻转了。如果我们退出hook2程序，字符串又会回到原来的样子。  

## 0x04 利用Ptrace动态执行sleep()函数

上一节中我们介绍了如何使用ptrace来修改内存，现在继续介绍如何用ptrace来执行libc .so中的sleep()函数。主要逻辑都在inject()这个函数中：  
``` c
void inject(pid_t pid)
{
    struct pt_regs old_regs,regs;
    long sleep_addr;

    //save old regs
    ptrace(PTRACE_GETREGS, pid, NULL, &old_regs);
    memcpy(&regs, &old_regs, sizeof(regs));

    printf("getting remote sleep_addr:\n");
    sleep_addr = get_remote_addr(pid, libc_path, (void *)sleep);

    long parameters[1];
    parameters[0] = 10;

    ptrace_call(pid, sleep_addr, parameters, 1, &regs);

    //restore old regs
    ptrace(PTRACE_SETREGS, pid, NULL, &old_regs);
}
```
首先我们用ptrace(PTRACE_GETREGS, pid, NULL, &old_regs)来保存一下寄存器的值，然后获取sleep()函数在目标进程中的地址，接着利用ptrace执行sleep()函数，最后在执行完sleep()函数后再用ptrace(PTRACE_SETREGS, pid, NULL, &old_regs)恢复寄存器原来值。  

下面是获取sleep()函数在目标进程中地址的代码：  
``` c
void* get_module_base(pid_t pid, const char* module_name)
{
    FILE *fp;
    long addr = 0;
    char *pch;
    char filename[32];
    char line[1024];
    if (pid == 0) {
        snprintf(filename, sizeof(filename), "/proc/self/maps");
    } else {
        snprintf(filename, sizeof(filename), "/proc/%d/maps", pid);
    }
    fp = fopen(filename, "r");

    if (fp != NULL) {
        while (fgets(line, sizeof(line), fp)) {
            if (strstr(line, module_name)) {
                pch = strtok( line, "-" );
                addr = strtoul( pch, NULL, 16 );
                if (addr == 0x8000)
                    addr = 0;
                break;
            }
        }
        fclose(fp) ;
    }
    return (void *)addr;
}

long get_remote_addr(pid_t target_pid, const char* module_name, void* local_addr)
{
    void* local_handle, *remote_handle;

    local_handle = get_module_base(0, module_name);
    remote_handle = get_module_base(target_pid, module_name);

    printf("module_base: local[%p], remote[%p]\n", local_handle, remote_handle);

    long ret_addr = (long)((uint32_t)local_addr + (uint32_t)remote_handle - (uint32_t)local_handle);

    printf("remote_addr: [%p]\n", (void*) ret_addr); 

    return ret_addr;
}
```
因为libc.so在内存中的地址是随机的，所以我们需要先获取目标进程的libc.so的加载地址，再获取自己进程的libc.so的加载地址和sleep()在内存中的地址。然后我们就能计算出sleep()函数在目标进程中的地址了。要注意的是获取目标进程和自己进程的libc.so的加载地址是通过解析/proc/[pid]/maps得到的。  

接下来是执行sleep()函数的代码：  
``` c
int ptrace_call(pid_t pid, long addr, long *params, uint32_t num_params, struct pt_regs* regs)
{
    uint32_t i;
    for (i = 0; i < num_params && i < 4; i ++) {
        regs->uregs[i] = params[i];
    }
    //
    // push remained params onto stack
    //
    if (i < num_params) {
        regs->ARM_sp -= (num_params - i) * sizeof(long) ;
        putdata(pid, (long)regs->ARM_sp, (char*)&params[i], (num_params - i) * sizeof(long));
    }

    regs->ARM_pc = addr;
    if (regs->ARM_pc & 1) {
        /* thumb */
        regs->ARM_pc &= (~1u);
        regs->ARM_cpsr |= CPSR_T_MASK;
    } else {
        /* arm */
        regs->ARM_cpsr &= ~CPSR_T_MASK;
    }

    regs->ARM_lr = 0;

    if (ptrace_setregs(pid, regs) == -1
            || ptrace_continue(pid) == -1) {
        printf("error\n");
        return -1;
    }

    int stat = 0;
    waitpid(pid, &stat, WUNTRACED);
    while (stat != 0xb7f) {
        if (ptrace_continue(pid) == -1) {
            printf("error\n");
            return -1;
        }
        waitpid(pid, &stat, WUNTRACED);
    }

    return 0;
}
```
首先是将参数赋值给R0-R3，如果参数大于四个的话，再使用putdata()将参数存放在栈上。然后我们将PC的值设置为函数地址。接着再根据是否是thumb指令设置ARM_cpsr寄存器的值。随后我们使用ptrace_setregs()将目标进程寄存器的值进行修改。最后使用waitpid()等待函数被执行。  

编译完后，我们使用hook3对target程序进行hook：  
``` bash
# ./target                                                                                                                                                              
Hello,LiBieGou! 0
Hello,LiBieGou! 1
Hello,LiBieGou! 2
Hello,LiBieGou! 3
[…sleep 10 seconds…]
Hello,LiBieGou! 4
Hello,LiBieGou! 5

# ./hook3 24835
getting remote sleep_addr:
module_base: local[0xb6f35000], remote[0xb6eec000]
remote_addr: [0xb6f1a24b]
```
正常的情况是target程序每秒输出一句话，但是用hook3程序hook后，就会暂停10秒钟的时间，因为我们利用ptrace运行了sleep(10)在目标程序中。  

## 0x05 利用Ptrace动态加载so并执行自定义函数  

仅仅是执行现有的libc函数是不能满足我们的需求的，接下来我们继续介绍如何动态的加载自定义so文件并且运行so文件中的函数。逻辑大概如下：  

保存当前寄存器的状态 -> 获取目标程序的mmap, dlopen, dlsym, dlclose 地址 -> 调用mmap分配一段内存空间用来保存参数信息 –> 调用dlopen加载so文件 -> 调用dlsym找到目标函数地址 -> 使用ptrace_call执行目标函数 -> 调用 dlclose 卸载so文件 -> 恢复寄存器的状态。  

实现整个逻辑的函数 injectSo()的代码如下：  
``` c
void injectSo(pid_t pid,char* so_path, char* function_name,char* parameter)
{
    struct pt_regs old_regs,regs;
    long mmap_addr, dlopen_addr, dlsym_addr, dlclose_addr;

//save old regs

    ptrace(PTRACE_GETREGS, pid, NULL, &old_regs);
    memcpy(&regs, &old_regs, sizeof(regs));

// get remote addres

    printf("getting remote addres:\n");
    mmap_addr = get_remote_addr(pid, libc_path, (void *)mmap);
    dlopen_addr = get_remote_addr( pid, libc_path, (void *)dlopen );
    dlsym_addr = get_remote_addr( pid, libc_path, (void *)dlsym );
    dlclose_addr = get_remote_addr( pid, libc_path, (void *)dlclose );

    printf("mmap_addr=%p dlopen_addr=%p dlsym_addr=%p dlclose_addr=%p\n",
    (void*)mmap_addr,(void*)dlopen_addr,(void*)dlsym_addr,(void*)dlclose_addr);

    long parameters[10];

//mmap

    parameters[0] = 0; //address
    parameters[1] = 0x4000; //size
    parameters[2] = PROT_READ | PROT_WRITE | PROT_EXEC; //WRX
    parameters[3] = MAP_ANONYMOUS | MAP_PRIVATE; //flag
    parameters[4] = 0; //fd
    parameters[5] = 0; //offset

ptrace_call(pid, mmap_addr, parameters, 6, &regs);

    ptrace(PTRACE_GETREGS, pid, NULL, &regs);
long map_base = regs.ARM_r0;

    printf("map_base = %p\n", (void*)map_base);

//dlopen

    printf("save so_path = %s to map_base = %p\n", so_path, (void*)map_base);
    putdata(pid, map_base, so_path, strlen(so_path) + 1);

    parameters[0] = map_base;
    parameters[1] = RTLD_NOW| RTLD_GLOBAL;

ptrace_call(pid, dlopen_addr, parameters, 2, &regs);

    ptrace(PTRACE_GETREGS, pid, NULL, &regs);
    long handle = regs.ARM_r0;

    printf("handle = %p\n",(void*) handle);

//dlsym

    printf("save function_name = %s to map_base = %p\n", function_name, (void*)map_base);
    putdata(pid, map_base, function_name, strlen(function_name) + 1);

    parameters[0] = handle;
    parameters[1] = map_base;

ptrace_call(pid, dlsym_addr, parameters, 2, &regs);

    ptrace(PTRACE_GETREGS, pid, NULL, &regs);
    long function_ptr = regs.ARM_r0;

    printf("function_ptr = %p\n", (void*)function_ptr);

//function_call

    printf("save parameter = %s to map_base = %p\n", parameter, (void*)map_base);
    putdata(pid, map_base, parameter, strlen(parameter) + 1);

    parameters[0] = map_base;

    ptrace_call(pid, function_ptr, parameters, 1, &regs);

//dlcose

    parameters[0] = handle;

    ptrace_call(pid, dlclose_addr, parameters, 1, &regs);

//restore old regs

    ptrace(PTRACE_SETREGS, pid, NULL, &old_regs);
}
```
mmap()可以用来将一个文件或者其它对象映射进内存，如果我们把flag设置为MAP_ANONYMOUS并且把参数fd设置为0的话就相当于直接映射一段内容为空的内存。mmap()的函数声明和参数如下：  

`void* mmap(void* start,size_t length,int prot,int flags,int fd,off_t offset);`
start：映射区的开始地址，设置为0时表示由系统决定映射区的起始地址。  
length：映射区的长度。  
prot：期望的内存保护标志，不能与文件的打开模式冲突。我们这里设置为RWX。  
flags：指定映射对象的类型，映射选项和映射页是否可以共享。我们这里设置为：MAP_ANONYMOUS(匿名映射，映射区不与任何文件关联)，MAP_PRIVATE(建立一个写入时拷贝的私有映射。内存区域的写入不会影响到原文件)。  
fd：有效的文件描述词。匿名映射设置为0。   
off_toffset：被映射对象内容的起点。设置为0。  
在我们使用ptrace_call(pid, mmap_addr, parameters, 6, &regs)调用完mmap()函数之后，要记得使用ptrace(PTRACE_GETREGS, pid, NULL, &regs); 用来获取保存返回值的regs.ARM_r0，这个返回值也就是映射的内存的起始地址。 

mmap()映射的内存主要用来保存我们传给其他函数的参数。比如接下来我们需要用dlopen()去加载”/data/local/tmp/libinject.so”这个文件，所以我们需要先用putdata()将”/data/local/tmp/libinject.so”这个字符串放置在mmap()所映射的内存中，然后就可以将这个映射的地址作为参数传递给dlopen()了。接下来的dlsym()，so中的目标函数，dlclose()都是相同调用的方式，这里就不一一赘述了。  

我们再来看一下被加载的so文件，里面的内容为：  
``` c
int mzhengHook(char * str){
    printf("mzheng Hook pid = %d\n", getpid());
    printf("Hello %s\n", str);
    LOGD("mzheng Hook pid = %d\n", getpid());
    LOGD("Hello %s\n", str);
    return 0;
}
```
这里我们不光使用printf()还使用了android debug的函数LOGD()用来输出调试结果。所以在编译时我们需要加上LOCAL_LDLIBS := -llog。  

编译完后，我们使用hook4对target程序进行hook：  
``` bash
# ./target                                                                                                         
Hello,LiBieGou! 0
Hello,LiBieGou! 1
Hello,LiBieGou! 2
mzheng Hook pid = 6043
Hello 7weapons
Hello,LiBieGou! 3
Hello,LiBieGou! 4
Hello,LiBieGou! 5
…
```
logcat:  
```
D/DEBUG   ( 6043): mzheng Hook pid = 6043
D/DEBUG   ( 6043): Hello 7weapons
```
``` bash
# ./hook4 6043
getting remote addres:
mmap_addr=0xb6f80c81 dlopen_addr=0xb6fd0f4d dlsym_addr=0xb6fd0e9d dlclose_addr=0xb6fd0e19
map_base = 0xb6f2f000
save so_path = /data/local/tmp/libinject.so to map_base = 0xb6f2f000
handle = 0xb6fcd494
save function_name = mzhengHook to map_base = 0xb6f2f000
function_ptr = 0xb6f29cbd
save parameter = 7weapons to map_base = 0xb6f2f000
```
可以看到无论是stdout还是logcat都成功的输出了我们的调试信息。这意味着我们可以通过注入让目标进程加载so文件并执行任意代码了。  

## 0x06 小结

现在我们已经可以做到hook system call以及动态的加载自定义so文件并且运行so文件中的函数了，但离执行以及hook java层的函数还有一定距离。因为篇幅原因，我们的hook之旅就先进行到这里，敬请期待一下篇《离别钩 - Hooking》。  

文章中所有提到的代码和工具都可以在我的github下载到，地址是：  

https://github.com/zhengmin1989/TheSevenWeapons  

## 0x07 参考资料

Playing with Ptrace http://www.linuxjournal.com/article/6100 
System Call Tracing using ptrace  
古河的Libinject http://bbs.pediy.com/showthread.php?t=141355  