## Android 源码下载和编译
Android Jelly Bean(Android 4.1)的编译依赖Sun JDK 1.6，由于Ubuntu默认使用Open JDK，所以需要首先安装JDK1.6。  

## 安装JDK

首先从oracle下载[JDK1.6](http://www.oracle.com/technetwork/java/javase/downloads/jdk6-jsp-136632.html)，得到文件jdk-6u45-linux-x64.bin 运行直接运行解包，得到文件夹jdk1.6.0_45。  

设置环境变量，编辑文件gedit /etc/profile  
```
export JAVA_HOME=/home/monkey/Documents/jdk1.6.0_45
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
export PATH=$PATH:${JAVA_HOME}/bin:${JRE_HOME}/bin
```
## 安装依赖软件包

`sudo apt-get install git-core gnupg flex bison gperf build-essential zip curl zlib1g-dev gcc-multilib g++-multilib libc6-dev-i386 lib32ncurses5-dev x11proto-core-dev libx11-dev lib32z-dev ccache libgl1-mesa-dev libxml2-utils xsltproc unzip bison flex xsltproc gperf`

## 下载Android代码
``` bash
//建立repo工作目录
mkdir ~/bin 
PATH=~/bin:$PATH
//下载repo脚本
$ curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
$ chmod a+x ~/bin/repo
//建立Android源码目录
mkdir -p ~/android/jellybean
cd ~/android/jellybean
//初始化repo
repo init -u https://android.googlesource.com/platform/manifest -b android-4.1.1_r3
//查看分支情况
git ls-remote --tags https://android.googlesource.com/platform/manifest
//下载Android源代码
repo sync
```

## 下载指定模块源码
`repo manifest -o -`  
其中，name表示项目模块名称以及在源码服务器上的相对路径，path表示项目的本地路径。  
 
repo manifest -o - 命令读取的是本地源码目录(~/android/jellybean)下的.repo/manifest/default/xml文件。  

知道了有哪些项目可以单独下载，只要将项目模块名指定给repo sync即可。  

`repo sync platfoem/system/core`  

## 下载Android Linux Kernel源码

Kernel部分的源码没有采用repo工具管理，可以直接通过git下载。  
``` bash
cd ~/android/jellybean
mkdir kernel
cd kernel
```
下载通用版，其余是针对特定处理器的版本，执行第一条指令即可。  

``` bash
git clone https://android.googlesource.com/kernel/common.git
git clone https://android.googlesource.com/kernel/goldfish.git
git clone https://android.googlesource.com/kernel/msm.git
git clone https://android.googlesource.com/kernel/omap.git
git clone https://android.googlesource.com/kernel/samsung.git
git clone https://android.googlesource.com/kernel/tegra.git
```
由于Android JellyBean使用的是Linux 3.0内核，还需要切换到Kernel 3.0分支。  
``` bash
cd common
git branch -a
git checkout remotes/origin/Android-3.0
```
## 编译Android上层系统源码

导入预设脚本：  
`. build/envsetup.sh` 
或者  
`source build/envsetup.sh` 
指定产品名和编译变量  

```
lunch
//选择full-eng 模拟器设备
```
编译源码：  
`make -j8`  
Mac环境配置：  

```
hdiutil create -type SPARSE -fs 'Case-sensitive Journaled HFS+' -size 40g ~/android.dmg
hdiutil resize -size <new-size-you-want>g ~/android.dmg.sparseimage
hdiutil attach ~/android.dmg -mountpoint /Volumes/android
```

## 编译指定模块源码  

* make 模块名
* mm 来自于envsetup.sh脚本中注册的函数
* mmm 来自于envsetup.sh脚本中注册的函数

### make 模块名

适合第一次编译，会把依赖块一并编译。 
编译应用层源码，查看Android.mk文件的LOCAL_PACKAGE_NAME。  

``` bash
cat packages/apps/Phone/Android.mk
...
LOCAL_PACKAGE_NAME := Phone
...
make Phone
```
编译框架层和系统运行库源码，查看LOCAL_MODULE变量：  

``` bash
find frameworks -name Android.mk
cat frameworks/base/cmds/app_process/Android.mk
...
LOCAL_MODULE := app_process
...
make app_process
```
### mmm命令

用于在源码根目录编译指定模块，参数为模块的相对路径。只能在第一编译后使用，比如要编译Phone部分源码：  

`mmm packages/apps/phone`  
### mm命令

用于在模块根目录编译这个模块，只能在第一次编译后使用。比如要编译Phone部分源码：  
```
cd packages/apps/phone
mm
```
mmm和mm命令必须在执行. build/envsetup.sh之后才能使用，并且只编译发生变化的文件，如果需要编译模块的所有文件，需要加-B。如：mm -B  

## Android 源码结构
<table>
<thead>
<tr>
<th>包名</th>
<th>内容</th>
</tr>
</thead>
<tbody>
<tr>
<td>abi</td>
<td>二进制兼容性检查</td>
</tr>
<tr>
<td>bionic</td>
<td>Bionic C库实现代码</td>
</tr>
<tr>
<td>boottable</td>
<td>启动引导程序的源码，包含bootloader，diskinstall和recovery</td>
</tr>
<tr>
<td>build</td>
<td>编译系统，包含各种make和shell脚本</td>
</tr>
<tr>
<td>cts</td>
<td>兼容性检测源码，Android手机如果需要Google认证，就需要通过Google的兼容性检测，目的是确保该手机系统具备标准的SDK API接口</td>
</tr>
<tr>
<td>dalvik</td>
<td>Dalvik虚拟机源码</td>
</tr>
<tr>
<td>development</td>
<td>Android开发所使用的一些配置文件</td>
</tr>
<tr>
<td>device</td>
<td>不同厂商设备相关的编译脚本，包含三星和摩托罗拉等</td>
</tr>
<tr>
<td>docs</td>
<td>source.android.com文档</td>
</tr>
<tr>
<td>external</td>
<td>Android依赖的扩展库，包括bluetooth、skia、sqlite、webkit、wpa_supplicant等功能库和一些工具库，如oprofile用于JNI层的性能调试。系统运行库层大部分代码位于这里</td>
</tr>
<tr>
<td>frameworks</td>
<td>框架层源码，应用框架层位于这里</td>
</tr>
<tr>
<td>gdk</td>
<td>提供NDK build的封装脚本</td>
</tr>
<tr>
<td>hardware</td>
<td>硬件抽象层相关源码</td>
</tr>
<tr>
<td>libcore</td>
<td>核心Java库。Android2.3 以前位于/dalvik/libcore目录下</td>
</tr>
<tr>
<td>libnativehelper</td>
<td>JNI的一些头文件</td>
</tr>
<tr>
<td>Makefile</td>
<td>编译入口，指向/build/main.mk</td>
</tr>
<tr>
<td>ndk</td>
<td>NDK(Native Development Kit)开发环境相关源码</td>
</tr>
<tr>
<td>out</td>
<td>编译输出目录，编译后的所有输出都在这个目录，分为主机部分和目标机部分</td>
</tr>
<tr>
<td>packages</td>
<td>包含各种内置应用程序、内容提供器、输入法等。应用层开发主要集中在这部分</td>
</tr>
<tr>
<td>prebuilt</td>
<td>编译所需的程序文件，主要包含不同平台下的ARM编译器</td>
</tr>
<tr>
<td>sdk</td>
<td>编译SDK工具所需的文件，包含hierachyviewer、eclipse插件、emulator、raceview等主要工具</td>
</tr>
<tr>
<td>system</td>
<td>Linux所需的一些系统工具程序，比如adb、debuggerd、fastboot、logcat等。</td>
</tr>
</tbody>
</table>

## 编译遇到的问题
出现错误：  
```
make: *** Waiting for unfinished jobs....
make: *** [out/host/linux-x86/obj/STATIC_LIBRARIES/libhost_intermediates/CopyFile.o] Error 127
host C++: libandroidfw <= frameworks/base/libs/androidfw/Asset.cpp
prebuilts/tools/gcc-sdk/g++: line 40: prebuilts/tools/gcc-sdk/../../gcc/linux-x86/host/i686-linux-glibc2.7-4.6/bin/i686-linux-g++: No such file or directory
make: *** [out/host/linux-x86/obj/STATIC_LIBRARIES/libandroidfw_intermediates/Asset.o] Error 127
Note: Some input files use or override a deprecated API.
Note: Recompile with -Xlint:deprecation for details.
```
解决方案：  
`sudo apt-get install gcc-multilib`  
出现错误：  

`/home/monkey/android/4.1.1/prebuilts/gcc/linux-x86/host/i686-linux-glibc2.7-4.6/bin/../lib/gcc/i686-linux/4.6.x-google/../../../../i686-linux/bin/as: error while loading shared libraries: libz.so.1: cannot open shared object file: No such file or directory`
解决方案：    

`sudo apt-get install zlib1g:i386`
出现错误：   

`gcc: error trying to exec 'cc1plus': execvp: No such file or directory` 
解决方案：  
```
gcc和g++版本不匹配
$ sudo apt-get install gcc-4.4 g++-4.4
$ sudo rm -f /usr/bin/gcc /usr/bin/g++
$ sudo ln -s /usr/bin/gcc-4.4 /usr/bin/gcc
$ sudo ln -s /usr/bin/g++-4.4 /usr/bin/g++
```
出现错误：  

`libstdc++.so.6: cannot open shared object file: No such file or directory`  
解决方案：  

`sudo apt-get install lib32stdc++6`  
出现错误：  

`Can't locate Switch.pm in @INC (you may need to install the Switch module)`  
解决方案：  

`sudo apt-get install libswitch-perl`  
出现错误：  

`/bin/bash: xmllint: command not found`  
解决方案：  
`sudo apt-get install libxml2-utils`