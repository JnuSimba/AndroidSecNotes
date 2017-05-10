原文 by 蒸米  
## 0x00 序

随着移动安全越来越火，各种调试工具也都层出不穷，但因为环境和需求的不同，并没有工具是万能的。另外工具是死的，人是活的，如果能搞懂工具的原理再结合上自身的经验，你也可以创造出属于自己的调试武器。因此，笔者将会在这一系列文章中分享一些自己经常用或原创的调试工具以及手段，希望能对国内移动安全的研究起到一些催化剂的作用。 

文章中所有提到的代码和工具都可以在我的github下载到，地址是： https://github.com/zhengmin1989/TheSevenWeapons  

## 0x01 利用函数挂钩实现native层的hook

我们在离别钩(上)中已经可以做到动态的加载自定义so文件并且运行so文件中的函数了，但还不能做到hook目标函数，这里我们需要用到函数挂钩的技术来做到这一点。函数挂钩的基本原理是先用mprotect()将原代码段改成可读可写可执行，然后修改原函数入口处的代码，让pc指针跳转到动态加载的so文件中的hook函数中，执行完hook函数以后再让pc指针跳转回原本的函数中。  

用来注入的程序hook5逻辑与之前的hook4相比并没有太大变化，仅仅少了“调用 dlclose 卸载so文件”这一个步骤，因为我们想要执行的hook后的函数在so中，所以并不需要调用dlclose进行卸载。基本步骤如下：  

保存当前寄存器的状态 -> 获取目标程序的mmap, dlopen, dlsym, dlclose 地址 -> 调用mmap分配一段内存空间用来保存参数信息 –> 调用dlopen加载so文件 -> 调用dlsym找到目标函数地址 -> 使用ptrace_call执行目标函数 -> 恢复寄存器的状态  

hook5的主要代码逻辑如下：  
``` c
void injectSo(pid_t pid,char* so_path, char* function_name,char* parameter)
{
    struct pt_regs old_regs,regs;
    long mmap_addr, dlopen_addr, dlsym_addr, dlclose_addr;

//save old regs

    ptrace(PTRACE_GETREGS, pid, NULL, &old_regs);
    memcpy(&regs, &old_regs, sizeof(regs));

//get remote addres

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

//restore old regs

    ptrace(PTRACE_SETREGS, pid, NULL, &old_regs);
}
```
我们知道arm处理器支持两种指令集，一种是arm指令集，另一种是thumb指令集。所以要hook的函数可能是被编译成arm指令集的，也有可能是被编译成thumb指令集的。Thumb指令集可以看作是arm指令压缩形式的子集，它是为减小代码量而提出，具有16bit的代码密度。Thumb指令体系并不完整，只支持通用功能，必要时仍需要使用ARM指令，如进入异常时。需要注意的一点是thumb指令的长度是不固定的，但arm指令是固定的32位长度。  

为了让大家更容易的理解hook的原理，我们先只考虑arm指令集，因为arm相比thumb要简单一点，不需要考虑指令长度的问题。所以我们需要将target和hook的so编译成arm指令集的形式。怎么做呢？很简单，只要在Android.mk中的文件名后面加上”.arm”即可 (真正的文件不用加)。  
```
include $(CLEAR_VARS)
LOCAL_MODULE    := target
LOCAL_SRC_FILES := target.c.arm
include $(BUILD_EXECUTABLE)

include $(CLEAR_VARS)
LOCAL_MODULE    := inject2
LOCAL_SRC_FILES := inject2.c.arm
LOCAL_LDLIBS := -llog 
include $(BUILD_SHARED_LIBRARY)
```
确定了指令集以后，我们来看实现挂钩最重要的逻辑，这个逻辑是在注入后的so里实现的。首先我们需要一个结构体保存汇编代码和hook地址：  
``` c
struct hook_t {
    unsigned int jump[3]; //保存跳转指令
    unsigned int store[3]; //保存原指令
    unsigned int orig; //保存原函数地址
    unsigned int patch; //保存hook函数地址
};
```
我们接着来看注入的逻辑，最重要的函数为hook_direct()，他有三个参数，一个参数是我们最开始定义的用来保存汇编代码和hook地址的结构体，第二个是我们要hook的原函数的地址，第三个是我们用来执行hook的函数。函数的源码如下：  
```
int hook_direct(struct hook_t *h, unsigned int addr, void *hookf)
{
    int i;

    printf("addr  = %x\n", addr);
    printf("hookf = %x\n", (unsigned int)hookf);

//mprotect
    mprotect((void*)0x8000, 0xa000-0x8000, PROT_READ|PROT_WRITE|PROT_EXEC);

//modify function entry 
    h->patch = (unsigned int)hookf;
    h->orig = addr;
    h->jump[0] = 0xe59ff000; // LDR pc, [pc, #0]
    h->jump[1] = h->patch;
    h->jump[2] = h->patch;
    for (i = 0; i < 3; i++)
        h->store[i] = ((int*)h->orig)[i];
    for (i = 0; i < 3; i++)
        ((int*)h->orig)[i] = h->jump[i];

//cacheflush    
    hook_cacheflush((unsigned int)h->orig, (unsigned int)h->orig+sizeof(h->jump));
    return 1;
}
```
虽然android有ASLR，但并没有PIE，所以program image是固定在0x8000这个地址的，因此我们用mprotect()函数将整个target代码段变成RWX，这样我们就能修改函数入口处的代码了。是否修改成功可以通过cat /proc/[pid]/maps查看：  
``` bash
# cat /proc/25298/maps
00008000-0000a000 rwxp 00000000 b3:1c 627105     /data/local/tmp/target
0000a000-0000b000 r--p 00001000 b3:1c 627105     /data/local/tmp/target
0000b000-0000c000 rw-p 00000000 00:00 0 
0017f000-00180000 rw-p 00000000 00:00 0          [heap]
……
```
随后我们需要确定目标函数的地址，这个有两种方法。如果目标程序本身没有被strip的话，那些symbol都是存在的，因此可以使用dlopen()和dlsym()等方法来获取目标函数地址。但很多情况，目标程序都会被strip，特别是可以直接运行的二进制文件默认都会被直接strip。比如target中的sevenWeapons()这个函数名会在编译的时候去掉，所以我们使用dlsym()的话是无法找到这个函数的。这时候我们就需要使用ida或者objdump来定位一下目标函数的地址。比如我们用objdump找一下target程序里面sevenWeapons(int number)这个函数的地址：  
``` bash
……
    84d4:       e1a01000        mov     r1, r0
    84d8:       e59f200c        ldr     r2, [pc, #12]   ; 84ec <__cxa_type_match@plt+0xe4>
    84dc:       e59f000c        ldr     r0, [pc, #12]   ; 84f0 <__cxa_type_match@plt+0xe8>
    84e0:       e08f2002        add     r2, pc, r2
    84e4:       e08f0000        add     r0, pc, r0
    84e8:       eaffffb1        b       83b4 <__cxa_atexit@plt>
    84ec:       00002b18        andeq   r2, r0, r8, lsl fp
    84f0:       ffffff58                        ; <UNDEFINED> instruction: 0xffffff58
    84f4:       e1a02000        mov     r2, r0
    84f8:       e59f100c        ldr     r1, [pc, #12]   ; 850c <__cxa_type_match@plt+0x104>
    84fc:       e59f000c        ldr     r0, [pc, #12]   ; 8510 <__cxa_type_match@plt+0x108>
    8500:       e08f1001        add     r1, pc, r1
    8504:       e08f0000        add     r0, pc, r0
    8508:       eaffffac        b       83c0 <printf@plt>
    850c:       00001080        andeq   r1, r0, r0, lsl #1
    8510:       00001074        andeq   r1, r0, r4, ror r0
    8514:       b5006803        strlt   r6, [r0, #-2051]        ; 0xfffff7fd
    8518:       d503005a        strle   r0, [r3, #-90]  ; 0xffffffa6
851c:       06122280        ldreq   r2, [r2], -r0, lsl #5
……
```
虽然target这个binary被strip了，但还是可以找到sevenWeapons()这个函数的起始地址是在0x84f4。因为”mov r2, r0”就是加载number这个参数的指令，随后调用了<printf@plt>用来输出结果。 最后一个参数也就是我们要执行的hook函数的地址。得到这个地址非常简单，因为是so中的函数，调用hook_direct()的时候直接写上函数名即可。  

`hook_direct(&eph,hookaddr,my_sevenWeapons);`  
接下来我们看如何修改函数入口（modify function entry），首先我们保存一下原函数的地址和那个函数的前三条指令。随后我们把目标函数的第一条指令修改为 LDR pc, [pc, #0]，这条指令的意思是跳转到PC指针所指的地址，由于pc寄存器读出的值实际上是当前指令地址加8，所以我们把后面两处指令都保存为hook函数的地址，这样的话，我们就能控制PC跳转到hook函数的地址了。  

最后我们再调用hook_cacheflush()这个函数来刷新一下指令的缓存。因为虽然前面的操作修改了内存中的指令，但有可能被修改的指令已经被缓存起来了，再执行的话，CPU可能会优先执行缓存中的指令，使得修改的指令得不到执行。所以我们需要使用一个隐藏的系统调用来刷新一下缓存。`hook_cacheflush()` 代码如下：  
``` c
void inline hook_cacheflush(unsigned int begin, unsigned int end)
{   
    const int syscall = 0xf0002;

    __asm __volatile (
        "mov     r0, %0\n"          
        "mov     r1, %1\n"
        "mov     r7, %2\n"
        "mov     r2, #0x0\n"
        "svc     0x00000000\n"
        :
        :   "r" (begin), "r" (end), "r" (syscall)
        :   "r0", "r1", "r7"
        );
}
```
刷新完缓存后，再执行到原函数的时候，pc指针就会跳转到我们自定义的hook函数中了，hook函数里的代码如下：  
``` c
void  __attribute__ ((noinline)) my_sevenWeapons(int number)
{
    printf("sevenWeapons() called, number = %d\n", number);
    number++;

    void (*orig_sevenWeapons)(int number);
    orig_sevenWeapons = (void*)eph.orig;

    hook_precall(&eph);
    orig_sevenWeapons(number);
    hook_postcall(&eph);

}
```
首先在hook函数中，我们可以获得原函数的参数，并且我们可以对原函数的参数进行修改，比如说将数字乘2。随后我们使用hook_precall(&eph);将原本函数的内容进行还原。`hook_precall()`内容如下：  
``` c
void hook_precall(struct hook_t *h)
{
    int i;
    for (i = 0; i < 3; i++)
        ((int*)h->orig)[i] = h->store[i];

    hook_cacheflush((unsigned int)h->orig, (unsigned int)h->orig+sizeof(h->jump)*10);
}
```
在`hook_precall()`中，我们先对原本的三条指令进行还原，然后使用`hook_cacheflush()`对内存进行刷新。经过处理之后，我们就可以执行原来的函数orig_sevenWeapons(number)了。执行完后，如果我们还想再次hook这个函数，就需要调用`hook_postcall(&eph)`将原本的三条指令再进行一次修改。  

下面我们来使用hook5和libinject2.so来注入一下target这个程序：  
``` bash
# ./target
Hello,LiBieGou! 0
Hello,LiBieGou! 1
Hello,LiBieGou! 2
Hello,LiBieGou! 3
Hello,LiBieGou! 4
Hello,LiBieGou! 5
Hello,LiBieGou! 6
mzheng Hook pid = 18962
Hello sevenWeapons
addr  = 84f4
hookf = b6e73e88
sevenWeapons() called, number = 7
Hello,LiBieGou! 14
sevenWeapons() called, number = 8
Hello,LiBieGou! 16
sevenWeapons() called, number = 9
Hello,LiBieGou! 18
sevenWeapons() called, number = 10
Hello,LiBieGou! 20
sevenWeapons() called, number = 11
Hello,LiBieGou! 22
sevenWeapons() called, number = 12
Hello,LiBieGou! 24
sevenWeapons() called, number = 13
```
``` bash
./hook5 28922                                                                                                                                                             
getting remote addres:
mmap_addr=0xb6f84c81 dlopen_addr=0xb6fd4f4d dlsym_addr=0xb6fd4e9d dlclose_addr=0xb6fd4e19
map_base = 0xb6f33000
save so_path = /data/local/tmp/libinject2.so to map_base = 0xb6f33000
handle = 0xb6fd1494
save function_name = mzhengHook to map_base = 0xb6f33000
function_ptr = 0xb6f2e368
save parameter = sevenWeapons to map_base = 0xb6f33000
```
可以看到经过注入后，我们成功的获取了参数number的值，并且将”Hello,LiBieGou!”后面的数字变成了原来的两倍。  

## 0x02 使用adbi实现JNI层的hook

我们在上一节中介绍了如何hook native层的函数。下面我们再来讲讲如何利用adbi来hook JNI层的函数。Adbi是一个android平台上的一个注入框架，本身是开源的。Hook的原理和我们之前介绍的技术是一样的，这个框架支持arm和thumb指令集，也支持通过字符串定位symbol函数的地址。  

首先我们需要一个例子用来讲解，所以我写了程序叫test2。  

![](../pictures/androidhook2.jpg) 

点击程序中的button后，程序会调用so中的`Java_com_mzheng_libiegou_test2_MainActivity_stringFromJNI(JNIEnv* env,jobject thiz,jint a,jint b)`函数用来计算a+b的结果。我们默认传的参数是a=1, b=1。接下来我就来教你如何利用adbi来hook这个JNI函数。  

因为adbi是一个注入框架，我们下载好源码后，只要对应着源码中给的example照猫画虎即可。Hijack那个文件夹是保存的用来注入的程序，和我们之前讲的hook5.c的原理是一样的，所以不用做任何修改。我们只需要修改example中的代码，也就是将要注入的so文件的源码。  

首先，我们在/adbi-master/instruments/example这个文件夹下新建两个文件”hookjni.c”和” hookjni_arm.c”。“hookjni_arm.c”其实只是一个壳，用来将hook函数的入口编译成arm指令集的，内容如下：  
``` c
extern jstring my_Java_com_mzheng_libiegou_test2_MainActivity_stringFromJNI(JNIEnv* env,jobject thiz,jint a,jint b);

jstring my_Java_com_mzheng_libiegou_test2_MainActivity_stringFromJNI_arm(JNIEnv* env,jobject thiz,jint a,jint b)
{
    return my_Java_com_mzheng_libiegou_test2_MainActivity_stringFromJNI(env, thiz, a, b);
}
```
这个文件的目的仅仅是为了用arm指令集进行编译，可以看到Android.mk中在”hookjni_arm.c”后面多了个”.arm”：  
```
include $(CLEAR_VARS)
LOCAL_MODULE    := libexample
LOCAL_SRC_FILES := ../hookjni.c  ../hookjni_arm.c.arm
LOCAL_CFLAGS := -g
LOCAL_SHARED_LIBRARIES := dl
LOCAL_STATIC_LIBRARIES := base
include $(BUILD_SHARED_LIBRARY)
```
下面我们来看”hookjni.c”:  
``` c
jstring my_Java_com_mzheng_libiegou_test2_MainActivity_stringFromJNI(JNIEnv* env,jobject thiz,jint a,jint b)
{
    jstring (*orig_stringFromJNI)(JNIEnv* env,jobject thiz,jint a,jint b);
    orig_stringFromJNI = (void*)eph.orig;

    a = 10;
    b = 10;

    hook_precall(&eph);
    jstring res = orig_stringFromJNI(env, thiz, a, b);
    if (counter) {
        hook_postcall(&eph);
        log("stringFromJNI() called\n");
        counter--;
        if (!counter)
            log("removing hook for stringFromJNI()\n");
    }

    return res;
}

void my_init(void)
{
    counter = 3;
    log("%s started\n", __FILE__)
    set_logfunction(my_log);

    hook(&eph, getpid(), "libhello-jni.", "Java_com_mzheng_libiegou_test2_MainActivity_stringFromJNI", my_Java_com_mzheng_libiegou_test2_MainActivity_stringFromJNI_arm, my_Java_com_mzheng_libiegou_test2_MainActivity_stringFromJNI);
}
```
这段代码和我上一节讲的代码非常像，my_init()用来进行hook操作，我们需要提供想要hook的so文件名和函数名，然后再提供thumb指令集和arm指令集的hook函数地址。  

my_Java_com_mzheng_libiegou_test2_MainActivity_stringFromJNI()就是我们提供的hook函数了。我们在这个hook函数中把a和b都改成了10。除此之外，我们还使用counter这个全局变量来控制hook的次数，这里我们把counter设置为3，当hook了3次以后，就不再进行hook操作了。  

编辑好代码后，我们只需要在adbi的根目录下执行” build.sh”进行编译：  
``` bash
adbi-master$ ./build.sh 
[armeabi] Compile arm    : hijack <= hijack.c
[armeabi] Executable     : hijack
[armeabi] Install        : hijack => libs/armeabi/hijack
[armeabi] Compile arm    : base <= util.c
[armeabi] Compile arm    : base <= hook.c
[armeabi] Compile arm    : base <= base.c
[armeabi] StaticLibrary  : libbase.a
[armeabi] Compile thumb  : example <= hookjni.c
[armeabi] Compile arm    : example <= hookjni_arm.c
[armeabi] SharedLibrary  : libexample.so
[armeabi] Install        : libexample.so => libs/armeabi/libexample.so
```
编译好后，我们把hijack和libexample.so拷贝到/data/local/tmp目录下。然后使用hijack进行注入：  
``` bash
#./hijack -d -p 21734 -l /data/local/tmp/libexample.so                                                                                                                     
mprotect: 0x4011c444
dlopen: 0x400d5f4d
pc=4011d6e0 lr=4018588b sp=bed65308 fp=bed6549c
r0=fffffffc r1=bed65328
r2=10 r3=ffffffff
stack: 0xbed45000-0xbed66000 leng = 135168
executing injection code at 0xbed652b8
calling mprotect
library injection completed!
```
然后我们再点击button就可以看到我们已经成功的修改了a和b的值为10了，最后显示1+1=20。  

![](../pictures/androidhook3.jpg)  

通过cat adbi_example.log我们可以看到hook过程中打印的log：  
``` bash
#cat adbi_example.log                                                                                                                                                      
/home/aliray/7weapons/libiegou/adbi-master/instruments/example/jni/../hookjni.c started
hooking:   Java_com_mzheng_libiegou_test2_MainActivity_stringFromJNI = 0x7538ecc5 THUMB using 0x763c9581
stringFromJNI() called
stringFromJNI() called
stringFromJNI() called
removing hook for stringFromJNI()
```
可以看到adbi是通过thumb指令集进行hook的，原因是test2程序是用thumb指令集进行编译的。其实hook thumb指令集和arm指令集差不多，在”/adbi-master/instruments/base”中可以找到hook thumb指令集的逻辑：  
``` c
if ((unsigned long int)hook_thumb % 4 == 0)
    log("warning hook is not thumb 0x%lx\n", (unsigned long)hook_thumb)
h->thumb = 1;
log("THUMB using 0x%lx\n", (unsigned long)hook_thumb)
h->patch = (unsigned int)hook_thumb;
h->orig = addr; 
h->jumpt[1] = 0xb4;
h->jumpt[0] = 0x60; // push {r5,r6}
h->jumpt[3] = 0xa5;
h->jumpt[2] = 0x03; // add r5, pc, #12
h->jumpt[5] = 0x68;
h->jumpt[4] = 0x2d; // ldr r5, [r5]
h->jumpt[7] = 0xb0;
h->jumpt[6] = 0x02; // add sp,sp,#8
h->jumpt[9] = 0xb4;
h->jumpt[8] = 0x20; // push {r5}
h->jumpt[11] = 0xb0;
h->jumpt[10] = 0x81; // sub sp,sp,#4
h->jumpt[13] = 0xbd;
h->jumpt[12] = 0x20; // pop {r5, pc}
h->jumpt[15] = 0x46;
h->jumpt[14] = 0xaf; // mov pc, r5 ; just to pad to 4 byte boundary
memcpy(&h->jumpt[16], (unsigned char*)&h->patch, sizeof(unsigned int));
unsigned int orig = addr - 1; // sub 1 to get real address
for (i = 0; i < 20; i++) {
    h->storet[i] = ((unsigned char*)orig)[i];
    //log("%0.2x ", h->storet[i])
}
//log("\n")
for (i = 0; i < 20; i++) {
    ((unsigned char*)orig)[i] = h->jumpt[i];
    //log("%0.2x ", ((unsigned char*)orig)[i])
}
```
其实h->jumpt[20]这个字符数组保存的就是thumb指令集下控制pc指针跳转到hook函数地址的代码。Hook完之后的流程就和arm指令集的hook一样了。  

## 0x03 使用Cydia或Xposed实现JAVA层的hook

关于Cydia和Xposed的文章和例子已经很多了，这里就不再重复的进行介绍了。这里推荐一下瘦蛟舞和我写的文章，基本上就知道怎么使用这两个框架了：  

Android.Hook框架xposed篇(Http流量监控)   

Android.Hook框架Cydia篇(脱壳机制作)   

个人感觉Xposed框架要做的更好一些，主要原因是Cydia的作者已经很久没有更新过Cydia框架了，不光有很多bug还不支持ART。但是有很多不错的调试软件/插件是基于两个框架制作的，所以有时候还是需要用到Cyida的。  

接下来就推荐几个很实用的基于Cydia和Xposed的插件：  

ZjDroid: ZjDroid是基于Xposed Framewrok的动态逆向分析模块，逆向分析者可以通过ZjDroid完成以下工作： 1、DEX文件的内存dump 2、基于Dalvik关键指针的内存BackSmali，有效破解主流加固方案 3、敏感API的动态监控 4、指定内存区域数据dump 5、获取应用加载DEX信息。 6、获取指定DEX文件加载类信息。 7、dump Dalvik java堆信息。 8、在目标进程动态运行lua脚本。 https://github.com/halfkiss/ZjDroid  

XPrivacy: XPrivacy是一款基于Xposed框架的模块应用，可以对所有应用可能泄露隐私的权限进行管理，对禁止可能会导致崩溃的应用采取欺骗策略，提供伪造信息，比如说可以伪造手机的IMEI号码等。 https://github.com/M66B/XPrivacy  

Introspy: Introspy是一款可以追踪分析移动应用的黑盒测试工具并且可以发现安全问题。这个工具支持很多密码库的hook，还支持自定义hook。 https://github.com/iSECPartners/Introspy-Android  

## 0x04 Introspy实战

我们使用alictf上的evilapk400作为例子讲解如何利用introspy来调试程序。Evilapk400使用了比较复杂的dex加壳技术，如果不利用基于自定义dalvik的脱壳工具来进行脱壳的话做起来会非常麻烦。但我们如果换一种思路，直接通过动态调试的方法来获取加密算法的字符串，key和IV等信息就可以直接获取答案了。  

首先我们安装cyida.apk，Introspy-Android Config.apk到手机上，然后用eclipse打开“Introspy-Android Core”的源码增加一个自定义的hook函数。虽然Introspy默认对密码库进行了hook，但却没有对一些strings的函数进行hook。所以我们手动添加一个对String.equals()的hook：  

![](../pictures/androidhook6.jpg)   

![](../pictures/androidhook10jpg)   
然后我们编译，生成并安装Introspy-Android Core.apk到手机上。然后我们安装上EvilApk400。然后打开Introspy-Android Config勾选com.ali.encryption。    

![](../pictures/androidhook4.jpg)   

接着打开evilapk400，然后随便输入点内容并点击登陆。  

![](../pictures/androidhook5.jpg)   

然后我们使用adb logcat就能看到Introspy输出的信息了：  

![](../pictures/androidhook7.jpg)   

通过log，很容易就能看出来evilapk400使用了DES加密。通过log我们获取了密文，Key以及IV，所以我们可以写一个python程序来计算出最后的答案：  
``` python
from M2Crypto.EVP import Cipher
from base64 import b64encode, b64decode

key = b64decode('H5jOqyCXcO+odcJFhT7Odh+Yzqsgl3Dv')
iv = b64decode('AAoKCgoCAqo=')
ciphertext = '5458d715704493d8e6b9bd38f8b6be0e'.decode('hex')
decipher = Cipher(alg='des_ede3_cbc', key=key, op=0, iv=iv)
plaintext = decipher.update(ciphertext)
plaintext += decipher.final()
print plaintext
```
```
$ python decrypt.py 
日天@土侸
```
![](../pictures/androidhook8.jpg)   
 
![](../pictures/androidhook9.jpg)   

## 0x05 总结

本篇介绍了native层，JNI层以及JAVA层的hook，基本上可以满足我们平时对于android上hook的需求了。 另外文章中所有提到的代码和工具都可以在我的github下载到，地址是： https://github.com/zhengmin1989/TheSevenWeapons  

## 0x06 参考资料

Android平台下hook框架adbi的研究（下）http://blog.csdn.net/roland_sun/article/details/36049307  
ALICTF Writeups from Dr. Mario  