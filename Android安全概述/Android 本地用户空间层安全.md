# 第三章 Android 本地用户空间层安全

> 来源：[Yury Zhauniarovich | Publications](http://www.zhauniarovich.com/pubs.html)

> 译者：[飞龙](https://github.com/)

> 协议：[CC BY-NC-SA 4.0](http://creativecommons.org/licenses/by-nc-sa/4.0/)

本地用户空间层在 Android 操作系统的安全配置中起到重要作用。 不理解在该层上发生了什么，就不可能理解在系统中如何实施安全架构决策。 在本章中，我们的主题是 Android 引导过程和文件系统特性的，并且描述了如何在本地用户空间层上保证安全性。

## 3.1 Android 引导过程

要了解在本地用户空间层上提供安全性的过程，首先应考虑 Android 设备的引导顺序。 要注意，在第一步中，这个顺序可能会因不同的设备而异，但是在 Linux 内核加载之后，过程通常是相同的。 引导过程的流程如图 3.1 所示。

![](../pictures/3-1.jpg)

图 3.1：Android 启动顺序

当用户打开智能手机时，设备的 CPU 处于未初始化状态。在这种情况下，处理器从硬连线地址开始执行命令。该地址指向 Boot ROM 所在的 CPU 的写保护存储器中的一段代码（参见图 3.1 中的步骤 1）。代码驻留在 Boot ROM 上的主要目的是检测 Boot Loader（引导加载程序）所在的介质[17]。检测完成后，Boot ROM 将引导加载程序加载到内存中（仅在设备通电后可用），并跳转到引导 Boot Loader 的加载代码。反过来，Boot Loader 建立了外部 RAM，文件系统和网络的支持。之后，它将 Linux 内核加载到内存中，并将控制权交给它。 Linux 内核初始化环境来运行 C 代码，激活中断控制器，设置内存管理单元，定义调度，加载驱动程序和挂载根文件系统。当内存管理单元初始化时，系统为使用虚拟内存以及运行用户空间进程[17]做准备。实际上，从这一步开始，该过程就和运行 Linux 的台式计算机上发生的过程没什么区别了。

第一个用户空间进程是`init`，它是 Android 中所有进程的祖先。 该程序的可执行文件位于 Android 文件系统的根目录中。 清单 3.1 包含此可执行文件的主要部分。 可以看出，`init`二进制负责创建文件系统基本条目（7 到 16 行）。 之后（第 18 行），程序解析`init.rc`配置文件并执行其中的命令。

```c
 1 int main( int argc, char **argv ) 
 2 { 
 3   ...
 4   if (!strcmp (basename( argv[0] ), "ueventd") ) 
 5     return ueventd_main ( argc, argv ) ; 
 6   ...
 7   mkdir("/dev", 0755) ; 
 8   mkdir("/proc", 0755) ; 
 9   mkdir("/sys", 0755) ; 
10 
11   mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755") ; 
12   mkdir("/dev/pts", 0755) ; 
13   mkdir("/dev/socket", 0755) ; 
14   mount("devpts", "/dev/pts", "devpts", 0, NULL) ; 
15   mount("proc", "/proc", "proc", 0, NULL) ; 
16   mount("sysfs", "/sys", "sysfs", 0, NULL) ; 
17   ... 
18   init_parseconfig_file("/init.rc") ; 
19   ... 
20 }
```

代码 3.1：`init`程序源码

`init.rc`配置文件使用一种称为 Android Init Language 的语言编写，位于根目录下。 这个配置文件可以被想象为一个动作列表（命令序列），其执行由预定义的事件触发。 例如，在清单 3.2 中，`fs`（行 1）是一个触发器，而第 4 - 7 行代表动作。 在`init.rc`配置文件中编写的命令定义系统全局变量，为内存管理设置基本内核参数，配置文件系统等。从安全角度来看，更重要的是它还负责基本文件系统结构的创建，并为创建的节点分配所有者和文件系统权限。

```
1 on fs 
2   # mount mtd partitions 
3   # Mount /system rw first to give the filesystem a chance to save a checkpoint 
4   mount yaffs2 mtd@system /system 
5   mount yaffs2 mtd@system /system ro remount 
6   mount yaffs2 mtd@userdata /data nosuid nodev 
7   mount yaffs2 mtd@cache /cache nosuid nodev
```

代码 3.2：模拟器中的`fs`触发器上执行的动作列表

此外，`init`程序负责在 Android 中启动几个基本的守护进程和进程（参见图 3.1 中的步骤 5），其参数也在`init.rc`文件中定义。 默认情况下，在 Linux 中执行的进程以与祖先相同的权限（在相同的 UID下）运行。 在 Android 中，`init`以 root 权限（`UID == 0`）启动。 这意味着所有后代进程应该使用相同的 UID 运行。 幸运的是，特权进程可以将其 UID 改变为较少特权的进程。 因此，`init`进程的所有后代可以使用该功能来指定派生进程的 UID 和 GID（所有者和组也在`init.rc`文件中定义）。

第一个守护进程派生于`init`进程，它是`ueventd`守护进程。 这个服务运行自己的`main`函数（参见清单 3.1 中的第 5 行），它读取`ueventd.rc`和`ueventd.[device name].rc`配置文件，并重放指定的内核`uevent_hotplug`事件。 这些事件设置了不同设备的所有者和权限（参见清单 3.3）。 例如，第 5 行显示了如何设置文件系统对`/ dev/cam`设备的权限，2.2 节中会涉及这个例子。 之后，守护进程等待监听所有未来的热插拔事件。

ueventd.rc

```
1 ...
2 /dev/ashmem 0666 root root 
3 /dev/binder 0666 root root 
4 ...
5 /dev/cam 0660 root camera 
6 ...
```

代码 3.3：`ueventd.rc`文件

由`init`程序启动的核心服务之一是`servicemanager`（请参阅图 3.1 中的步骤 5）。 此服务充当在 Android 中运行的所有服务的索引。 它必须在早期阶段可用，因为以后启动的所有系统服务都应该有可能注册自己，从而对操作系统的其余部分可见[19]。

`init`进程启动的另一个核心进程是 Zygote。 Zygote 是一个热身完毕的特殊进程。 这意味着该进程已经被初始化并且链接到核心库。 Zygote 是所有进程的祖先。 当一个新的应用启动时，Zygote 会派生自己。 之后，为派生子进程设置对应于新应用的参数，例如 UID，GID，`nice-name`等。 它能够加速新进程的创建，因为不需要将核心库复制到新进程中。 新进程的内存具有“写时复制"（COW）保护，这意味着只有当后者尝试写入受保护的内存时，数据才会从 zygote 进程复制到新进程。 从而，核心库不会改变，它们只保留在一个地方，减少内存消耗和应用启动时间。

使用 Zygote 运行的第一个进程是 System Server（图 3.1 中的步骤 6）。 这个进程首先运行本地服务，例如 SurfaceFlinger 和 SensorService。 在服务初始化之后，调用回调，启动剩余的服务。 所有这些服务之后使用 servicemanager 注册。

## 3.2 Android 文件系统

虽然 Android 基于 Linux 内核，它的文件系统层次不符合文件系统层次标准[10]，它了定义类 Unix 系统的文件系统布局（见清单 3.4）。 Android 和 Linux 中的某些目录是相同的，例如`/dev`，`/proc`，`/sys`，`/etc`，`/mnt`等。这些文件夹的用途与 Linux 中的相同。 同时，还有一些目录，如`/system`，`/data`和`/cache`，它们不存在于 Linux 系统中。这些文件夹是 Android 的核心部分。 在 Android 操作系统的构建期间，会创建三个映像文件：`system.img`，`userdata.img`和`cache.img`。 这些映像提供 Android 的核心功能，是在设备的闪存上存储的。 在系统引导期间，`init`程序将这些映像安装到预定义的安装点，如`/system`，`/data`和`/cache`（参见清单 3.2）。

```
 1 drwxr−xr−x root root 2013−04−10 08 : 13 acct 
 2 drwxrwx−−− system cache 2013−04−10 08 : 13 cache 
 3 dr−x−−−−−− root root 2013−04−10 08 : 13 config 
 4 lrwxrwxrwx root root 2013−04−10 08 : 13 d −> /sys/kernel/debug 
 5 drwxrwx−−x system system 2013−04−10 08 : 14 data 
 6 −rw−r−−r−− root root 116 1970−01−01 00 : 00 default . prop 
 7 drwxr−xr−x root root 2013−04−10 08 : 13 dev 
 8 lrwxrwxrwx root root 2013−04−10 08 : 13 etc −> /system/etc 
 9 −rwxr−x−−− root root 244536 1970−01−01 00 : 00 init 
10 −rwxr−x−−− root root 2487 1970−01−01 00 : 00 init . goldfish . rc 
11 −rwxr−x−−− root root 18247 1970−01−01 00 : 00 init . rc 
12 −rwxr−x−−− root root 1795 1970−01−01 00 : 00 init . trace . rc 
13 −rwxr−x−−− root root 3915 1970−01−01 00 : 00 init . usb . rc 
14 drwxrwxr−x root system 2013−04−10 08 : 13 mnt 
15 dr−xr−xr−x root root 2013−04−10 08 : 13 proc 
16 drwx−−−−−− root root 2012−11−15 05 : 31 root 
17 drwxr−x−−− root root 1970−01−01 00 : 00 sbin 
18 lrwxrwxrwx root root 2013−04−10 08 : 13 sdcard −> /mnt/sdcard 
19 d−−−r−x−−− root sdcard r 2013−04−10 08 : 13 storage 
20 drwxr−xr−x root root 2013−04−10 08 : 13 sys 
21 drwxr−xr−x root root 2012−12−31 03 : 20 system 
22 −rw−r−−r−− root root 272 1970−01−01 00 : 00 ueventd . goldfish . rc 
23 −rw−r−−r−− root root 4024 1970−01−01 00 : 00 ueventd . rc 
24 lrwxrwxrwx root root 2013−04−10 08 : 13 vendor −> /system/vendor
```

代码 3.4：Android 文件系统

`/system`分区包含整个 Android 操作系统，除了 Linux 内核，它本身位于`/boot`分区上。 此文件夹包含子目录`/system/bin`和`/system/lib`，它们相应包含核心本地可执行文件和共享库。 此外，此分区包含由系统映像预先构建的所有系统应用。 映像以只读模式安装（参见清单 3.2 中的第 5 行）。 因此，此分区的内容不能在运行时更改。

因此，`/system`分区被挂载为只读，它不能用于存储数据。 为此，单独的分区`/data`负责存储随时间改变的用户数据或信息。 例如，`/data/app`目录包含已安装应用程序的所有 apk 文件，而`/data/data`文件夹包含应用程序的`home`目录。

`/cache`分区负责存储经常访问的数据和应用程序组件。 此外，操作系统无线更新（卡刷）也在运行之前存储在此分区上。

因此，在 Android 的编译期间生成`/system`，`/data`和`/cache`，这些映像上包含的文件和文件夹的默认权限和所有者必须在编译时定义。 这意味着在编译此操作系统期间，用户和组 UID 和 GID 应该可用。 Android 文件系统配置文件（见清单 3.5）包含预定义的用户和组的列表。 应该提到的是，一些行中的值（例如，参见第 10 行）对应于在 Linux 内核层上定义的值，如第 2.2 节所述。

此外，文件和文件夹的默认权限，所有者和所有者组定义在该文件中（见清单 3.6）。 这些规则由`fs_config()`函数解析并应用，它在这个文件的末尾定义。 此函数在映像组装期间调用。

```c
 1 #define AID_ROOT 0 /* traditional unix root user */ 
 2 #define AID_SYSTEM 1000 /* system server */ 
 3 #define AID_RADIO 1001 /* telephony subsystem , RIL */ 
 4 #define AID_BLUETOOTH 1002 /* bluetooth subsystem */ 
 5 #define AID_GRAPHICS 1003 /* graphics devices */ 
 6 #define AID_INPUT 1004 /* input devices */ 
 7 #define AID_AUDIO 1005 /* audio devices */ 
 8 #define AID_CAMERA 1006 /* camera devices */ 
 9 ... 
10 #define AID_INET 3003 /* can create AF_INET and AF_INET6 sockets */ 
11 ... 
12 #define AID_APP 10000 /* first app user */ 
13 ... 
14 static const struct android_id_info android_ids [ ] = { 
15   { "root" , AID_ROOT, }, 
16   { "system" , AID_SYSTEM, }, 
17   { "radio" , AID_RADIO, }, 
18   { "bluetooth" , AID_BLUETOOTH, }, 
19   { "graphics" , AID_GRAPHICS, }, 
20   { "input" , AID_INPUT, }, 
21   { "audio" , AID_AUDIO, }, 
22   { "camera" , AID_CAMERA, }, 
23   ... 
24   { "inet" , AID_INET, }, 
25   ... 
26 }; 
```

代码 3.5：Android 中硬编码的 UID 和 GID，以及它们到用户名称的映射

### 3.2.1 本地可执行文件的保护

在清单 3.6 中可以看到一些二进制文件分配有`setuid`和`setgid`访问权限标志。例如，`su`程序设置了它们。这个众所周知的工具允许用户运行具有指定的 UID 和 GID 的程序。在 Linux 中，此功能通常用于运行具有超级用户权限的程序。根据列表 3.6，二进制`/system/xbin/su`的访问权限分配为“06755"（见第 21 行）。第一个非零数“6"意味着该二进制具有`setuid`和`setgid`（`4 + 2`）访问权限标志集。通常，在Linux中，可执行文件以与启动它的进程相同的权限运行。这些标签允许用户使用可执行所有者或组的权限运行程序[11]。因此，在我们的例子中，`binary/system/xbin/su`将以 root 用户身份运行。这些 root 权限允许程序将其 UID 和 GID 更改为用户指定的 UID 和 GID（见清单 3.7 中的第 15 行）。之后，`su`可以使用指定的 UID 和 GID 启动提供的程序（例如，参见行 22）。因此，程序将以所需的 UID 和 GID 启动。

在特权程序的情况下，需要限制可访问这些工具的应用程序的范围。 在我们的这里，没有这样的限制，任何应用程序可以运行`su`程序并获得 root 级别的权限。 在 Android 中，通过将调用程序的 UID 与允许运行它的 UID 列表进行比较，来对本地用户空间层实现这种限制。 因此，在第 9 行中，`su`可执行文件获得进程的当前 UID，它等于调用它的进程的 UID，在第`10`行，它将这个 UID 与允许的 UID 的预定列表进行比较。 因此，只有在调用进程的 UID 等于`AID_ROOT`或`AID_SHELL`时，`su`工具才会启动。 为了执行这样的检查，`su`导入在 Android 中定义的 UID 常量（见第 1 行）。

```c
 1 /* Rules for directories .*/
 2 static struct fs_path_config android_dirs [ ] = { 
 3   { 00770 , AID_SYSTEM, AID_CACHE, "cache" } , 
 4   { 00771 , AID_SYSTEM, AID_SYSTEM, "data/app" } , 
 5   ... 
 6   { 00777 , AID_ROOT, AID_ROOT, "sdcard" } , 
 7   { 00755 , AID_ROOT, AID_ROOT, 0 } , 
 8 }; 
 9 
10 /* Rules for files .*/ 
11 static struct fs_path_config android_files [ ] = { 
12   ... 
13   { 00644 , AID_SYSTEM, AID_SYSTEM, "data/app/*" } , 
14   { 00644 , AID_MEDIA_RW, AID_MEDIA_RW, "data/media/*" } , 
15   { 00644 , AID_SYSTEM, AID SYSTEM, "data/app−private /*" } , 
16   { 00644 , AID_APP, AID_APP, "data/data/*" } , 
17   ... 
18   { 02755 , AID_ROOT, AID_NET_RAW, "system/bin/ping" } , 
19   { 02750 , AID_ROOT, AID_INET, "system/bin/netcfg" } , 
20   ... 
21   { 06755 , AID_ROOT, AID_ROOT, "system/xbin/su" } , 
22   ... 
23   { 06750 , AID_ROOT, AID_SHELL, "system/bin/run−as" } , 
24   { 00755 , AID_ROOT, AID_SHELL, "system/bin/*" } , 
25   ... 
26   { 00644 , AID_ROOT, AID_ROOT, 0 } , 
27 };
```

代码 3.6：默认权限和所有者

此外，在较新的版本（从 4.3 开始），Android 核心开发人员开始使用 Capabilities Linux 内核系统[4]。 这允许它们额外限制需要以 root 权限运行的程序的权限。 例如，对于`su`程序来说，它不需要具有 root 用户的所有特权。 对于这个程序，它足以有能力修改当前的 UID 和 GID。 因此，此工具只需要`CAP_SETUID`和`CAP_SETGID` root 权限来正常运行。

```c
 1 #include <private/android_filesystem_config.h>
 2 ... 
 3 int main( int argc, char **argv ) 
 4 { 
 5   struct passwd *pw; 
 6   int uid, gid, myuid ; 
 7 
 8 /* Until we have something better , only root and the shell can use su . */ 
 9 myuid = getuid () ; 
10 if (myuid != AID_ROOT && myuid != AID_SHELL) { 
11   fprintf ( stderr, "su : uid %d not allowed to su\n", myuid) ; 
12   return 1; 
13 } 
14 ... 
15 if ( setgid ( gid ) || setuid ( uid ) ) { 
16   fprintf ( stderr, "su : permission denied\n") ; 
17   return 1; 
18 } 
19 
20 /* User specified command for exec . */ 
21 if ( argc == 3 ) { 
22   if ( execlp ( argv[2], argv[2], NULL) < 0) { 
23     fprintf ( stderr , "su : exec failed for %s Error:%s\n" , argv [2] , 
24     strerror ( errno ) ) ; 
25     return −errno ; 
26   } 
27   ... 
28 }
```

代码 3.7：`su`程序的源代码