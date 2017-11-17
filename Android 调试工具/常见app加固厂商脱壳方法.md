原文 by [mottoin](http://www.mottoin.com/89035.html)  

### Apk文件结构  
![](../pictures/androidshell1.png)  
### Dex文件结构
![](../pictures/androidshell2.png)   
## 壳史

### 第一代壳 Dex加密

1. Dex字符串加密
2. 资源加密
3. 对抗反编译
4. 反调试
5. 自定义DexClassLoader

### 第二代壳 Dex抽取与So加固

1. 对抗第一代壳常见的脱壳法
2. Dex Method代码抽取到外部（通常企业版）
3. Dex动态加载
4. So加

### 第三代壳 Dex动态解密与So混淆

1. Dex Method代码动态解密**
2. So代码膨胀混淆
3. 对抗之前出现的所有脱壳法

### 第四代壳 arm vmp（未来）

vmp

## 壳的识别

### 1.用加固厂商特征：  

* 娜迦： libchaosvmp.so , libddog.solibfdog.so  
* 爱加密：libexec.so, libexecmain.so  
* 梆梆： libsecexe.so, libsecmain.so , libDexHelper.so  
* 360：libprotectClass.so, libjiagu.so  
* 通付盾：libegis.so  
* 网秦：libnqshield.so *百度：libbaiduprotect.so  

### 2.基于特征的识别代码
![](../pictures/androidshell3.png)   


## 第一代壳

1. 内存Dump法  
2. 文件监视法
3. Hook法
4. 定制系统
5. 动态调试法

### 内存Dump法

内存中寻找dex.035或者dex.036    
/proc/xxx/maps中查找后，手动Dump    
![](../pictures/androidshell4.png)   

android-unpacker https://github.com/strazzere/android-unpacker  
![](../pictures/androidshell5.png)     

drizzleDumper https://github.com/DrizzleRisk/drizzleDumper  
升级版的android-unpacker，read和lseek64代替pread，匹配dex代替匹配odex  
![](../pictures/androidshell6.png)     

![](../pictures/androidshell7.png)     
### 文件监视法

Dex优化生成odex  
inotifywait-for-Android https://github.com/mkttanabe/inotifywait-for-Android  
监视文件变化  
![](../pictures/androidshell8.png)     

notifywait-for-Android https://github.com/mkttanabe/inotifywait-for-Android  
监视DexOpt输出  

![](../pictures/androidshell9.png)     

![](../pictures/androidshell10.png)   
### Hook法

Hook dvmDexFileOpenPartial  
http://androidxref.com/4.4_r1/xref/dalvik/vm/DvmDex.cpp  
![](../pictures/androidshell11.png)   
![](../pictures/androidshell12.png)    


### 定制系统

修改安卓源码并刷机  
![](../pictures/androidshell13.png)     

DumpApk https://github.com/CvvT/DumpApk  
只针对部分壳  
![](../pictures/androidshell14.png)     

### 动态调试法

IDA Pro  
![](../pictures/androidshell15.png)    
![](../pictures/androidshell16.png)   
![](../pictures/androidshell17.png)     

### gdb gcore法
```
.gdbserver :1234 –attach pid 
.gdb 
(gdb) target remote :1234 
(gdb) gcore
```
coredump文件中搜索“dex.035”  
![](../pictures/androidshell18.png)    


## 第二代壳

1. 内存重组法
2. Hook法
3. 动态调试
4. 定制系统
5. 静态脱壳机

### 内存重组法

#### Dex篇

ZjDroid http://bbs.pediy.com/showthread.php?t=190494  

对付一切内存中完整的dex，包括壳与动态加载的jar  

![](../pictures/androidshell19.png)   
![](../pictures/androidshell20.png)    



#### so篇

elfrebuild  

![](../pictures/androidshell21.png)   
![](../pictures/androidshell22.png)   


构造soinfo，然后对其进行重建  
![](../pictures/androidshell23.png)   
![](../pictures/androidshell24.png)    



### Hook法

针对无代码抽取且Hook dvmDexFileOpenPartial失败 

Hook dexFileParse  

http://androidxref.com/4.4_r1/xref/dalvik/vm/DvmDex.cpp  
![](../pictures/androidshell25.png)     


https://github.com/WooyunDota/DumpDex  
![](../pictures/androidshell26.png)    


针对无代码抽取且Hook dexFileParse失败  

Hook memcmp  

http://androidxref.com/4.4_r1/xref/dalvik/vm/DvmDex.cpp  

![](../pictures/androidshell27.png)    
![](../pictures/androidshell28.png)    



### 定制系统

修改安卓源码并刷机－针对无抽取代码  

https://github.com/bunnyblue/DexExtractor  
![](../pictures/androidshell29.png)     


Hook dexfileParse  
![](../pictures/androidshell30.png)   

![](../pictures/androidshell31.png)  


DexHunter-最强大的二代壳脱壳工具  

https://github.com/zyq8709/DexHunter  

DexHunter的工作流程：  
 
![](../pictures/androidshell32.png)  

DexHunter的工作原理：  

![](../pictures/androidshell33.png)   

绕过三进程反调试  

http://bbs.pediy.com/showthread.php?p=1439627  
![](../pictures/androidshell34.png)  

![](../pictures/androidshell35.png)    


修改系统源码后：  

![](../pictures/androidshell36.png)  

http://www.cnblogs.com/lvcha/p/3903669.html  

![](../pictures/androidshell37.png)  

ls /proc/345/task  
![](../pictures/androidshell38.png)  

./gdbserver :1234 --attach346 
... 
(gdb) gcore
gcore防Dump解决方案：  

http://bbs.pediy.com/showthread.php?t=198995  

断点mmap调试，针对Hook dexFileParse无效  

原理： dexopt优化时， dvmContinueOptimization()->mmap()  

![](../pictures/androidshell39.png)  

### 静态脱壳机

分析壳so逻辑并还原加密算法  

http://www.cnblogs.com/2014asm/p/4924342.html  

![](../pictures/androidshell40.png)  

自定义linker脱so壳  

https://github.com/devilogic/udog  
 
main() -> dump_file()  
![](../pictures/androidshell41.png)   

## 第三代壳

1. dex2oat法
2. 定制系统
3. dex2oat法

ART模式下，dex2oat生成oat时，内存中的DEX是完整的  

http://bbs.pediy.com/showthread.php?t=210532  
 ![](../pictures/androidshell42.png)  


### 定制系统

Hook Dalvik_dalvik_system_DexFile_defineClassNative  

枚举所有DexClassDef，对所有的class，调用dvmDefineClass进行强制加载  
![](../pictures/androidshell43.png)   


## 第N代壳

### so + vmp
### 动态调试 ＋ 人肉还原  