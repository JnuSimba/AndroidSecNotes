# AndroidSecNotes
此系列文章是本人关于学习 Android 安全时记录的一些笔记，部分原创，部分是对网上文章的理解整理。如果可以找到原始参考链接时则会在文末贴出（如 乌云很多链接已失效，或者记不起当时存档时的链接），或者在文章开头写上 by xx，如有侵权请联系我删除或加上reference，感谢在网上共享知识的师傅们。
# 目录
* Java语言基础
	- [Java Basic](./Java语言基础/Java%20Basic.md)
	- [Java synthetic](./Java语言基础/Java%20synthetic.md)
	- [Java 反射机制到Android 的注解](./Java语言基础/Java%20反射机制到Android%20的注解.md)
	
* Android开发基础
	- [Android开发基础知识](./Android开发基础/Android开发基础知识.md)
	- [AndroidManifest.xml](./Android开发基础/AndroidManifest.xml.md)
	- [Android 工程相关文件说明](./Android开发基础/Android%20工程相关文件说明.md)
	- [Android NDK 开发基础](./Android开发基础/Android%20NDK%20开发基础.md)
	- [Android 基于监听的事件处理机制](./Android开发基础/Android%20基于监听的事件处理机制.md)
	- [Android Handler消息传递机制](./Android开发基础/Android%20Handler消息传递机制.md)
	- [Android 基于回调的事件处理机制](./Android开发基础/Android%20基于回调的事件处理机制.md)

* Android 安全概述
	- [Android 安全概述](./Android安全概述/Android%20安全概述.md)
	- [Android Linux 内核层安全](./Android安全概述/Android%20Linux%20内核层安全.md)
	- [Android 本地用户空间层安全](./Android安全概述/Android%20本地用户空间层安全.md)
	- [Android 框架层安全](./Android安全概述/Android%20框架层安全.md)
	- [Android 应用层安全](./Android安全概述/Android%20应用层安全.md)
	- [Android 安全的其他话题](./Android安全概述/Android%20安全的其他话题.md)

* Android 应用安全
	- [Android Application Security](./Android应用安全/Android%20Application%20Security.md)
	- [OWASP Mobile top 10_2014](./Android应用安全/OWASP%20Mobile%20top%2010_2014.md)
	- [Android Activity Security](./Android应用安全/Android%20Activity%20Security.md)
	- [Android Broadcast Security](./Android应用安全/Android%20Broadcast%20Security.md)
	- [Android Content Provider Security](./Android应用安全/Android%20Content%20Provider%20Security.md)
	- [Android Logcat Security](./Android应用安全/Android%20Logcat%20Security.md)
	- [Android Service Security](./Android应用安全/Android%20Service%20Security.md)
* Android逆向基础
	- [apk 反编译基础](./Android逆向基础/apk%20反编译基础.md)
	- [smali 语法](./Android逆向基础/smali%20语法.md)
	- [ARM 寄存器简介](./Android逆向基础/ARM%20寄存器简介.md)
	- [ARM 汇编伪指令简介](./Android逆向基础/ARM%20汇编伪指令简介.md)
	- [ARM 汇编指令简介](./Android逆向基础/ARM%20汇编指令简介.md)

* Android 调试工具
	- [Smali Instrumentation](./Android%20调试工具/Smali%20Instrumentation.md)
	- [Ida Pro.md](./Android%20调试工具/Ida%20Pro.md)
	- [Android Hook（上）](./Android%20调试工具/Android%20Hook（上）.md)
	- [Android Hook（下）](./Android%20调试工具/Android%20Hook（下）.md)
	- [Android Hook 框架（Cydia篇）](./Android%20调试工具/Android%20Hook%20框架（Cydia篇）.md)
	- [Android Hook 框架（XPosed篇）](./Android%20调试工具/Android%20Hook%20框架（XPosed篇）.md)
	- [Android Java 层的anti-Hook 技巧](./Android%20调试工具/Android%20Java%20层的anti-Hook%20技巧.md)
	- [Android 应用程序通用脱壳方法研究](./Android%20调试工具/Android%20应用程序通用脱壳方法研究.md)
	- [Android ART 运行时（上）](./Android%20调试工具/Android%20ART%20运行时（上）.md)
	- [Android ART 运行时（下）](./Android%20调试工具/Android%20ART%20运行时（下）.md)

## 学习方向
作者：Flanker Edward  
链接：https://www.zhihu.com/question/45548548/answer/105379921  
来源：知乎  
著作权归作者所有，转载请联系作者获得授权。    

Andriod平台漏洞也有很多种，分为不同的方向，比如内核漏洞挖掘/利用，用户态高权限进程漏洞挖掘/利用，浏览器漏洞挖掘/沙箱逃逸和针对Andriod自身结构的漏洞（权限泄漏、敏感信息泄漏等）。  

Android是一个基于Linux的复杂系统，想要熟练挖掘Android漏洞必须要了解Linux系统本身（内核，驱动），以及Android在Linux架构上添加的各式各样的组件和功能（binder，media/graphics subsystem，zygote, chromium/skia和最上层的四大金刚等）。从问题中的角度来讲，对于不熟悉Windows平台的初学者，不建议将大量时间投入到Windows平台上的漏洞利用学习上，因为两个平台的漏洞虽然核心思想相通，但具体到利用的细节不太相同。这些精力时间投入到Linux平台和Android自身的研究学习上对于Android漏洞挖掘这个目的意义更大。并且从实际的角度出发，业界目前需要更多的也是Mobile系统方向上的人才。    

基础的漏洞类型例如栈溢出、堆溢出、未初始化、OOB、UAF、条件竞争/TOCTOU漏洞等必须要熟练理解掌握，这些在各种CTF平台上在Linux平台下都可以练习到。但是需要注意的一点是CTF还是和实际有些区别。例如CTF上的堆溢出题目绝大部分都基于dlmalloc，而Android早在5.0就切换到了jemalloc。现代软件中最常见的条件竞争漏洞因为比赛平台的限制也很少在CTF中看到（利用比较耗费资源，很难支撑比赛中的大规模并发需求）。   

在了解了基础知识之后，可以进一步学习Andriod自身的体系结构。    
* Binder方面可以学习调试几个经典漏洞（[flankerhqd/mediacodecoob](https://link.zhihu.com/?target=https%3A//github.com/flankerhqd/mediacodecoob)， [blackhat](https://www.blackhat.com/asia-16/briefings.html#hey-your-parcel-looks-bad-fuzzing-and-exploiting-parcel-ization-vulnerabilities-in-android)）
* 文件格式漏洞自然是经典的[stagefright](https://www.exploit-db.com/docs/39527.pdf) 
* [驱动/内核漏洞3636和1805](https://www.blackhat.com/docs/us-15/materials/us-15-Xu-Ah-Universal-Android-Rooting-Is-Back.pdf)
* [Chrome V8相关](https://github.com/4B5F5F4B/Exploits)  
* 关注每个月的android security bulletin, 尝试从diff反推漏洞研究如何触发和利用  
* 代码审计：经典书籍 The art of software security assessment  
* fuzzing  
 
当然，只要掌握了计算机体系结构，熟悉了程序的运作方式，到达一定境界之后那么各种漏洞、各种系统之间的界限也就渐渐模糊了。我所知道的业界人士中有不少从Windows大牛跨界为Android大牛，例如Fireeye的王宇王老师，pjf大牛。毕竟触类旁通，运用之妙，存乎一心，与君共勉。	  
	
