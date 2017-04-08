# 第四章 Android 框架层安全

> 来源：[Yury Zhauniarovich | Publications](http://www.zhauniarovich.com/pubs.html)

> 译者：[飞龙](https://github.com/)

> 协议：[CC BY-NC-SA 4.0](http://creativecommons.org/licenses/by-nc-sa/4.0/)

如我们在第1.2节中所描述的那样，应用程序框架级别上的安全性由 IPC 引用监视器实现。 在 4.1 节中，我们以 Android 中使用的进程间通信系统的描述开始，讲解这个级别上的安全机制。 之后，我们在 4.2 节中引入权限，而在 4.3 节中，我们描述了在此级别上实现的权限实施系统。

## 4.1 Android Binder 框架 

如 2.1 节所述，所有 Android 应用程序都在应用程序沙箱中运行。粗略地说，应用程序的沙箱通过在带有不同 Linux 身份的不同进程中运行所有应用程序来保证。此外，系统服务也在具有更多特权身份的单独进程中运行，允许它们使用 Linux Kernel DAC 功能，访问受保护的系统不同部分（参见第 2.1, 2.2 和 1.2 节）。因此，需要进程间通信（IPC）框架来管理不同进程之间的数据和信号交换。在 Android 中，一个称为 Binder 的特殊框架用于进程间通信[12]。标准的 Posix System V IPC 框架不支持由 Android 实现的 Bionic libc 库（参见[这里](https://android.googlesource.com/platform/ndk/+/android-4.2.2_r1.2/docs/system/libc/SYSV-IPC.html)）。此外，除了用于一些特殊情况的 Binder 框架，也会使用 Unix 域套接字（例如，用于与 Zygote 守护进程的通信），但是这些机制不在本文的考虑范围之内。

Binder 框架被特地重新开发来在 Android 中使用。 它提供了管理此操作系统中的进程之间的所有类型的通信所需的功能。 基本上，甚至应用程序开发人员熟知的机制，例如`Intents`和`ContentProvider`，都建立在 Binder 框架之上。 这个框架提供了多种功能，例如可以调用远程对象上的方法，就像本地对象那样，以及同步和异步方法调用，Link to Death（某个进程的 Binder 终止时的自动通知），跨进程发送文件描述符的能力等等[12,16] 。

根据由客户端 - 服务器同步模型组织的进程之间的通信。客户端发起连接并等待来自服务端的回复。 因此，客户端和服务器之间的通信可以被想象为在相同的进程线程中执行。 这为开发人员提供了调用远程对象上的方法的可能性，就像它们是本地的一样。 通过 Binder 的通信模型如图 4.1 所示。 在这个图中，客户端进程 A 中的应用程序想要使用进程 B  [12]中运行的服务的公开行为。

![](../pictures/4-1.jpg)

使用 Binder 框架的客户端和服务之间的所有通信，都通过 Linux 内核驱动程序`/dev/binder`进行。此设备驱动程序的权限设置为全局可读和可写（见 3.1 节中的清单 3.3 中的第 3 行）。因此，任何应用程序可以写入和读取此设备。为了隐藏 Binder 通信协议的特性，`libbinder`库在 Android 中使用。它提供了一种功能，使内核驱动程序的交互过程对应用程序开发人员透明。尤其是，客户端和服务器之间的所有通信通过客户端侧的代理和服务器侧的桩进行。代理和桩负责编码和解码数据和通过 Binder 驱动程序发送的命令。为了使用代理和桩，开发人员只需定义一个 AIDL 接口，在编译应用程序期间将其转换为代理和桩。在服务端，调用单独的 Binder 线程来处理客户端请求。

从技术上讲，使用Binder机制的每个公开服务（有时称为 Binder 服务）都分配有标识。内核驱动程序确保此 32 位值在系统中的所有进程中是唯一的。因此，此标识用作 Binder 服务的句柄。拥有此句柄可以与服务交互。然而，为了开始使用服务，客户端首先必须找到这个值。服务句柄的发现通过 Binder 的上下文管理器（`servicemanager`是 Android Binder 的上下文管理器的实现，在这里我们互换使用这些概念）来完成。上下文管理器是一个特殊的 Binder 服务，其预定义的句柄值等于 0（指代清单 4.1 的第 8 行中获得的东西）。因为它有一个固定的句柄值，任何一方都可以找到它并调用其方法。基本上，上下文管理器充当名称服务，通过服务的名称提供服务句柄。为了实现这个目的，每个服务必须注册上下文管理器（例如，使用第 26 行中的`ServiceManager`类的`addService`方法）。因此，客户端可以仅知道与其通信的服务名称。使用上下文管理器来解析此名称（请参阅`getService`第 12 行），客户端将收到稍后用于与服务交互的标识。 Binder 驱动程序只允许注册单个上下文管理器。因此，`servicemanager`是由 Android 启动的第一个服务之一（见第 3.1 节）。`servicemanager`组件确保了只允许特权系统标识注册服务。

Binder 框架本身不实施任何安全性。 同时，它提供了在 Android 中实施安全性的设施。 Binder 驱动程序将发送者进程的 UID 和 PID 添加到每个事务。 因此，由于系统中的每个应用具有其自己的 UID，所以该值可以用于识别调用方。 调用的接收者可以检查所获得的值并且决定是否应该完成事务。 接收者可以调用`android.os.Binder.getCallingUid()`和`android.os.Binder.getCallingPid()`[12]来获得发送者的 UID 和 PID。 另外，由于 Binder 句柄在所有进程中的唯一性和其值的模糊性[14]，它也可以用作安全标识。

```java
 1 public final class ServiceManager { 
 2   ... 
 3   private static IServiceManager getIServiceManager() { 
 4     if ( sServiceManager != null ) { 
 5       return sServiceManager ; 
 6     } 
 7     // Find the service manager 
 8     sServiceManager = ServiceManagerNative.asInterface( BinderInternal.getContextObject() ); 
 9     return sServiceManager ; 
10   } 
11 
12   public static IBinder getService ( String name) { 
13     try { 
14       IBinder service = sCache . get (name) ; 
15       if ( service != null ) { 
16         return service ; 
17       } else { 
18         return getIServiceManager().getService(name); 
19       } 
20     } catch (RemoteException e) { 
21       Log.e(TAG, "error in getService", e); 
22     } 
23     return null; 
24   } 
25 
26   public static void addService( String name, IBinder service, boolean allowIsolated ) { 
27     try { 
28       getIServiceManager().addService(name, service, allowIsolated ); 
29     } catch (RemoteException e) { 
30       Log.e(TAG, "error in addService" , e); 
31     } 
32   } 
33   ... 
34 }
```

代码 4.1：`ServiceManager`的源码

## 4.2 Android 权限

如我们在 2.1 节中所设计的那样，在 Android 中，每个应用程序默认获得其自己的 UID 和 GID 系统标识。 此外，在操作系统中还有一些硬编码的标识（参见清单 3.5）。 这些身份用于使用在 Linux 内核级别上实施的 DAC，分离 Android 操作系统的组件，从而提高操作系统的整体安全性。 在这些身份中，`AID SYSTEM`最为显著。 此 UID 用于运行系统服务器（`system server`），这个组件统一了由 Android 操作系统提供的服务。 系统服务器具有访问操作系统资源，以及在系统服务器内运行的每个服务的特权，这些服务提供对其他 OS 组件和应用的特定功能的受控访问。 此受控访问基于权限系统。

正如我们在 4.1 节中所提及的，Binder 框架向接收方提供了获得发送方 UID 和 PID 的能力。在一般情况下，该功能可以由服务利用来限制想要连接到服务的消费者。这可以通过将消费者的 UID 和 PID 与服务所允许的 UID 列表进行比较来实现。然而，在 Android 中，这种功能以略微不同的方式来实现。服务的每个关键功能（或简单来说是服务的方法）被称为权限的特殊标签保护。粗略地说，在执行这样的方法之前，会检查调用进程是否被分配了权限。如果调用进程具有所需权限，则允许调用服务。否则，将抛出安全检查异常（通常，`SecurityException`）。例如，如果开发者想要向其应用程序提供发送短信的功能，则必须将以下行添加到应用程序的`AndroidManifest.xml`文件中：`<uses-permission android:name ="android.permission.SEND_SMS"/>` 。Android 还提供了一组特殊调用，允许在运行时检查服务使用者是否已分配权限。

到目前为止所描述的权限模型提供了一种强化安全性的有效方法。 同时，这个模型是无效的，因为它认为所有的权限是相等的。 在移动操作系统的情况下，所提供的功能在安全意义上并不总是相等。 例如，安装应用程序的功能比发送 SMS 的功能更重要，相反，发送 SMS 的功能比设置警告或振动更危险。

这个问题在 Android 中通过引入权限的安全级别来解决。有四个可能的权限级别：`normal`，`dangerous`，`signature`和`signatureOrSystem`。权限级别要么硬编码到 Android 操作系统（对于系统权限），要么由自定义权限声明中的第三方应用程序的开发者分配。此级别影响是否决定向请求的应用程序授予权限。为了被授予权限，正常的权限可以只在应用程序的`AndroidManifest.xml`文件中请求。危险权限除了在清单文件中请求之外，还必须由用户批准。在这种情况下，安装应用程序期间，安装包所请求的权限集会显示给用户。如果用户批准它们，则安装应用程序。否则，安装将被取消。如果请求权限的应用与声明它的应用拥有相同签名，（6.1 中提到了 Android 中的应用程序签名的用法），系统将授予`signature`权限。如果请求的权限应用和声明权限的使用相同证书签名，或请求应用位于系统映像上，则授予`signatureOrSystem`权限。因此，对于我们的示例，振动功能被正常级别的权限保护，发送 SMS 的功能被危险级别的权限保护，以及软件包安装功能被`signatureOrSystem`权限级别保护。

### 4.2.1 系统权限定义

用于保护 Android 操作系统功能的系统权限在框架的`AndroidManifest.xml`文件中定义，位于 Android 源的`frameworks/base/core/res`文件夹中。 这个文件的一个摘录包含一些权限定义的例子，如代码清单 4.2 所示。 在这些示例中，展示了用于保护发送 SMS，振动器和包安装功能的权限声明。

```xml
 1 <manifest xmlns:android=" http://schemas.android.com/apk/res/android" 
 2   package="android" coreApp="true" android:sharedUserId="android.uid.system" 
 3   android:sharedUserLabel="@string/android_system_label "> 
 4   ... 
 5   <!−− Allows an application to send SMS messages. −−> 
 6   <permission android:name="android.permission.SEND_SMS" 
 7     android:permissionGroup="android.permission−group.MESSAGES" 
 8     android:protectionLevel="dangerous"
 9     android:permissionFlags="costsMoney" 
10     android:label="@string/permlab_sendSms" 
11     android:description="@string/permdesc _sendSms" /> 
12   ... 
13   <!−− Allows access to the vibrator −−> 
14   <permission android:name="android.permission.VIBRATE" 
15     android:permissionGroup="android.permission−group.AFFECTS_BATTERY" 
16     android:protectionLevel="normal" 
17     android:label="@string/permlab_vibrate" 
18     android:description="@string/permdesc_vibrate" /> 
19   ... 
20   <!−− Allows an application to install packages. −−> 
21   <permission android:name="android.permission.INSTALL_PACKAGES" 
22     android:label="@string/permlab_installPackages" 
23     android:description="@string/permdesc_installPackages" 
24     android:protectionLevel="signature|system" /> 
25   ... 
26 </manifest>
```

代码 4.2：系统权限的定义

默认情况下，第三方应用程序的开发人员无法访问受`signature`和`signatureOrSystem`级别的系统权限保护的功能。 这种行为以以下方式来保证：应用程序框架包使用平台证书签名。 因此，需要使用这些级别的权限保护的功能的应用程序必须使用相同的平台证书进行签名。 然而，仅有操作系统的构建者才可以访问该证书的私钥，通常是硬件生产者（他们自己定制 Android）或电信运营商（使用其修改的操作系统映像来分发设备）。

### 4.2.2 权限管理

系统服务`PackageManagerService`负责 Android 中的应用程序管理。 此服务有助于在操作系统中安装，卸载和更新应用程序。 此服务的另一个重要作用是权限管理。 基本上，它可以被认为是一个策略管理的要素。 它存储了用于检查 Android 包是否分配了特定权限的信息。 此外，在应用程序安装和升级期间，它执行一堆检查，来确保在这些过程中不违反权限模型的完整性。 此外，它还作为一个策略判定的要素。 此服务的方法（我们将在后面展示）是权限检查链中的最后一个元素。 我们不会在这里考虑`PackageManagerService`的操作。 然而，感兴趣的读者可以参考[15,19]来获得如何执行应用安装的更多细节。

`PackageManagerService`将所有第三方应用程序的权限的相关信息存储在`/data/system/packages.xml `[7]中。 该文件用作系统重新启动之间的永久存储器。 但是，在运行时，所有有关权限的信息都保存在 RAM 中，从而提高系统的响应速度。 在启动期间，此信息使用存储在用于第三方应用程序的`packages.xml`文件中的数据，以及通过解析系统应用程序来收集。

### 4.2.3 Android 框架层的权限实施

为了了解 Android 如何在应用程序框架层强制实施权限，我们考虑 Vibrator 服务用法。 在清单 4.3 的第 6 行中，展示了振动器服务如何保护其方法`vibrate `的示例。 这一行检查了调用组件是否分配有由常量`android.Manifest.permission.VIBRATE`定义的标签`android.permission.VIBRATE`。 Android 提供了几种方法来检查发送者（或服务使用者）是否已被分配了权限。 在我们这个库，这些设施由方法`checkCallingOrSelfPermission`表示。 除了这种方法，还有许多其他方法可以用于检查服务调用者的权限。

```java
 1 public class VibratorService extends IVibratorService.Stub 
 2   implements InputManager.InputDeviceListener { 
 3   ... 
 4   public void vibrate ( long milliseconds, IBinder token ) { 
 5     if (mContext.checkCallingOrSelfPermission(android.Manifest.permission.VIBRATE) 
 6             != PackageManager.PERMISSION_GRANTED) { 
 7       throw new SecurityException("Requires VIBRATE permission"); 
 8     } 
 9     ... 
10   } 
11   ... 
12 }
```

代码 4.3：权限的检查

方法`checkCallingOrSelfPermission`的实现如清单 4.4 所示。 在第 24 行中，方法`checkPermission`被调用。 它接收`uid`和`pid`作为 Binder 框架提供的参数。

```java
 1 class ContextImpl extends Context { 
 2   ... 
 3   @Override 
 4   public int checkPermission ( String permission, int pid, int uid ) { 
 5     if ( permission == null ) { 
 6       throw new IllegalArgumentException ("permission is null ") ; 
 7     } 
 8 
 9     try { 
10       return ActivityManagerNative.getDefault().checkPermission( 
11         permission, pid, uid ); 
12       } catch (RemoteException e) { 
13       return PackageManager.PERMISSION_DENIED; 
14     } 
15   } 
16 
17   @Override 
18   public int checkCallingOrSelfPermission ( String permission ) { 
19     if ( permission == null ) { 
20       throw new IllegalArgumentException("permission is null"); 
21     } 
22 
23     return checkPermission( permission, Binder. getCallingPid(), 
24       Binder.getCallingUid() ); 
25   } 
26   ... 
27 }
```

代码 4.4：`ContextImpl `类的摘录


在第 11 行中，检查被重定向到`ActivityManagerService`类，继而在`ActivityManager`组件的方法`checkComponentPermission`中执行实际检查。 此方法的代码如清单 4.5 所示。 在第 4 行中它检查调用者 UID 是否拥有特权。 具有 root 和系统 UID 的组件由具有所有权限的系统授予。

```java
 1 public static int checkComponentPermission ( String permission, int uid, 
 2   int owningUid, boolean exported ) { 
 3   // Root , system server get to do everything . 
 4   if ( uid == 0 || uid == Process.SYSTEM_UID) { 
 5     return PackageManager.PERMISSION_GRANTED; 
 6   } 
 7   // Isolated processes don ’ t get any permissions . 
 8   if ( UserId.isIsolated ( uid ) ) { 
 9     return PackageManager.PERMISSION_DENIED; 
10   } 
11   // If there is a uid that owns whatever is being accessed , it has 
12   // blanket access to it regardless of the permissions it requires . 
13   if (owningUid >= 0 && UserId.isSameApp(uid, owningUid) ) { 
14     return PackageManager.PERMISSION_GRANTED; 
15   } 
16   // If the target is not exported , then nobody else can get to it . 
17   if (!exported) { 
18     Slog.w(TAG, "Permission denied: checkComponentPermission() owningUid=" + owningUid) ; 
19     return PackageManager.PERMISSION_DENIED; 
20   } 
21   if ( permission == null ) { 
22     return PackageManager.PERMISSION_GRANTED; 
23   } 
24   try { 
25     return AppGlobals.getPackageManager() 
26       .checkUidPermission ( permission , uid ) ; 
27   } catch (RemoteException e) { 
28     // Should never happen , but if it does . . . deny !
29     Slog.e(TAG, "PackageManager is dead ?!?" , e) ; 
30   } 
31   return PackageManager.PERMISSION_DENIED;
32 }
```

代码 4.5：`ActivityManager`的`checkComponentPermission`方法。

在清单 4.5 的第 26 行中，权限检查被重定向到包管理器，将其转发到`PackageManagerService`。 正如我们前面解释的，这个服务知道分配给 Android 包的权限。 执行权限检查的`PackageManagerService`方法如清单 4.6 所示。 在第 7 行中，如果将权限授予由其 UID 定义的 Android 应用程序，则会执行精确检查。

```java
 1 public int checkUidPermission ( String permName, int uid ) { 
 2  final boolean enforcedDefault = isPermissionEnforcedDefault(permName); 
 3  synchronized (mPackages) { 
 4    Object obj = mSettings.getUserIdLPr( UserHandle.getAppId( uid ) ); 
 5    if ( obj != null ) { 
 6      GrantedPermissions gp = ( GrantedPermissions ) obj ; 
 7      if (gp.grantedPermissions.contains (permName) ) { 
 8        return PackageManager.PERMISSION_GRANTED; 
 9      } 
10    } else { 
11      HashSet<String> perms = mSystemPermissions.get ( uid ) ; 
12      if (perms != null && perms.contains (permName) ) { 
13        return PackageManager.PERMISSION_GRANTED; 
14      } 
15    } 
16    if (!isPermissionEnforcedLocked (permName, enforcedDefault ) ) { 
17      return PackageManager.PERMISSION_GRANTED; 
18    } 
19  } 
20  return PackageManager .PERMISSION DENIED; 
21 }
```

代码 4.6：`PackageManagerService`的`checkUidPermission `方法