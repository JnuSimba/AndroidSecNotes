原文：http://d3adend.org/blog/?p=589  
## 0x00 前言

一个最近关于检测native hook框架的方法让我开始思考一个Android应用如何在Java层检测Cydia Substrate或者Xposed框架。  

声明: 

下文所有的anti-hooking技巧很容易就可以被有经验的逆向人员绕过，这里只是展示几个检测的方法。在最近DexGuard和GuardIT等工具中还没有这类anti-hooking检测功能，不过我相信不久就会增加这个功能。 
 
## 0x01 检测安装的应用

一个最直接的想法就是检测设备上有没有安装Substrate或者Xposed框架，可以直接调用PackageManager显示所有安装的应用，然后看是否安装了Substrate或者Xposed。  
``` java
PackageManager packageManager = context.getPackageManager();
List applicationInfoList  = packageManager.getInstalledApplications(PackageManager.GET_META_DATA);

for(ApplicationInfo applicationInfo : applicationInfoList) {
    if(applicationInfo.packageName.equals("de.robv.android.xposed.installer")) {
        Log.wtf("HookDetection", "Xposed found on the system.");
    }
    if(applicationInfo.packageName.equals("com.saurik.substrate")) {
        Log.wtf("HookDetection", "Substrate found on the system.");
    }
}     
```  

## 0x02 检查调用栈里的可疑方法
  
另一个想到的方法是检查Java调用栈里的可疑方法，主动抛出一个异常，然后打印方法的调用栈。代码如下：  
``` java
public class DoStuff {
    public static String getSecret() {
        try {
            throw new Exception("blah");
        }
        catch(Exception e) {
            for(StackTraceElement stackTraceElement : e.getStackTrace()) {
                Log.wtf("HookDetection", stackTraceElement.getClassName() + "->" + stackTraceElement.getMethodName());
            }
        }
        return "ChangeMePls!!!";
    }
}
```
当应用没有被hook的时候，正常的调用栈是这样的：  
``` 
com.example.hookdetection.DoStuff->getSecret
com.example.hookdetection.MainActivity->onCreate
android.app.Activity->performCreate
android.app.Instrumentation->callActivityOnCreate
android.app.ActivityThread->performLaunchActivity
android.app.ActivityThread->handleLaunchActivity
android.app.ActivityThread->access$800
android.app.ActivityThread$H->handleMessage
android.os.Handler->dispatchMessage
android.os.Looper->loop
android.app.ActivityThread->main
java.lang.reflect.Method->invokeNative
java.lang.reflect.Method->invoke
com.android.internal.os.ZygoteInit$MethodAndArgsCaller->run
com.android.internal.os.ZygoteInit->main
dalvik.system.NativeStart->main
``` 
但是假如有Xposed框架hook了com.example.hookdetection.DoStuff.getSecret方法，那么调用栈会有2个变化：  

在dalvik.system.NativeStart.main方法后出现de.robv.android.xposed.XposedBridge.main调用  
如果Xposed hook了调用栈里的一个方法，还会有de.robv.android.xposed.XposedBridge.handleHookedMethod 和  de.robv.android.xposed.XposedBridge.invokeOriginalMethodNative调用  
所以如果hook了getSecret方法，调用栈就会如下：  
```
com.example.hookdetection.DoStuff->getSecret

de.robv.android.xposed.XposedBridge->invokeOriginalMethodNative
de.robv.android.xposed.XposedBridge->handleHookedMethod

com.example.hookdetection.DoStuff->getSecret
com.example.hookdetection.MainActivity->onCreate
android.app.Activity->performCreate
android.app.Instrumentation->callActivityOnCreate
android.app.ActivityThread->performLaunchActivity
android.app.ActivityThread->handleLaunchActivity
android.app.ActivityThread->access$800
android.app.ActivityThread$H->handleMessage
android.os.Handler->dispatchMessage
android.os.Looper->loop
android.app.ActivityThread->main
java.lang.reflect.Method->invokeNative
java.lang.reflect.Method->invoke
com.android.internal.os.ZygoteInit$MethodAndArgsCaller->run
com.android.internal.os.ZygoteInit->main

de.robv.android.xposed.XposedBridge->main

dalvik.system.NativeStart->main
``` 
下面看下Substrate hook com.example.hookdetection.DoStuff.getSecret方法后，调用栈会有什么变化：  

dalvik.system.NativeStart.main调用后会出现2次com.android.internal.os.ZygoteInit.main，而不是一次。  
如果Substrate hook了调用栈里的一个方法，还会出现com.saurik.substrate.MS$2.invoked，com.saurik.substrate.MS  $MethodPointer.invoke还有跟Substrate扩展相关的方法（这里是com.cigital.freak.Freak$1$1.invoked）。  
所以如果hook了getSecret方法，调用栈就会如下：  
```
com.example.hookdetection.DoStuff->getSecret

com.saurik.substrate._MS$MethodPointer->invoke
com.saurik.substrate.MS$MethodPointer->invoke
com.cigital.freak.Freak$1$1->invoked
com.saurik.substrate.MS$2->invoked

com.example.hookdetection.DoStuff->getSecret
com.example.hookdetection.MainActivity->onCreate
android.app.Activity->performCreate
android.app.Instrumentation->callActivityOnCreate
android.app.ActivityThread->performLaunchActivity
android.app.ActivityThread->handleLaunchActivity
android.app.ActivityThread->access$800
android.app.ActivityThread$H->handleMessage
android.os.Handler->dispatchMessage
android.os.Looper->loop
android.app.ActivityThread->main
java.lang.reflect.Method->invokeNative
java.lang.reflect.Method->invoke
com.android.internal.os.ZygoteInit$MethodAndArgsCaller->run
com.android.internal.os.ZygoteInit->main

com.android.internal.os.ZygoteInit->main

dalvik.system.NativeStart->main
``` 
在知道了调用栈的变化之后，就可以在Java层写代码进行检测：  
``` java
try {
    throw new Exception("blah");
}
catch(Exception e) {
    int zygoteInitCallCount = 0;
    for(StackTraceElement stackTraceElement : e.getStackTrace()) {
        if(stackTraceElement.getClassName().equals("com.android.internal.os.ZygoteInit")) {
            zygoteInitCallCount++;
            if(zygoteInitCallCount == 2) {
                Log.wtf("HookDetection", "Substrate is active on the device.");
            }
        }
        if(stackTraceElement.getClassName().equals("com.saurik.substrate.MS$2") && 
                stackTraceElement.getMethodName().equals("invoked")) {
            Log.wtf("HookDetection", "A method on the stack trace has been hooked using Substrate.");
        }
        if(stackTraceElement.getClassName().equals("de.robv.android.xposed.XposedBridge") && 
                stackTraceElement.getMethodName().equals("main")) {
            Log.wtf("HookDetection", "Xposed is active on the device.");
        }
        if(stackTraceElement.getClassName().equals("de.robv.android.xposed.XposedBridge") && 
                stackTraceElement.getMethodName().equals("handleHookedMethod")) {
            Log.wtf("HookDetection", "A method on the stack trace has been hooked using Xposed.");
        }

    }
}
```

## 0x03 检测并不应该native的native方法

Xposed框架会把hook的Java方法类型改为"native"，然后把原来的方法替换成自己的代码（调用hookedMethodCallback）。可以查看XposedBridge_hookMethodNative的实现，是修改后app_process里的方法。  

利用Xposed改变hook方法的这个特性（Substrate也使用类似的原理），就可以用来检测是否被hook了。注意这不能用来检测ART运行时的Xposed，因为没必要把方法的类型改为native。  

假设有下面这个方法：  
``` java
public class DoStuff {
    public static String getSecret() {
        return "ChangeMePls!!!";
    }
}
``` 
如果getSecret方法被hook了，在运行的时候就会像下面的定义：  
``` java
public class DoStuff {
        // calls hookedMethodCallback if hooked using Xposed
    public native static String getSecret(); 
}
```
基于上面的原理，检测的步骤如下：  

定位到应用的DEX文件  
枚举所有的class  
通过反射机制判断运行时不应该是native的方法  
下面的Java展示了这个技巧。这里假设了应用本身没有通过JNI调用本地代码，大多数应用都不需要调用本地方法。不过如果有JNI调用的话，只需要把这些native方法添加到一个白名单中即可。理论上这个方法也可以用于检测Java库或者第三方库，不过需要把第三方库的native方法添加到一个白名单。检测代码如下：  
``` java
for (ApplicationInfo applicationInfo : applicationInfoList) {
    if (applicationInfo.processName.equals("com.example.hookdetection")) {      
        Set classes = new HashSet();
        DexFile dex;
        try {
            dex = new DexFile(applicationInfo.sourceDir);
            Enumeration entries = dex.entries();
            while(entries.hasMoreElements()) {
                String entry = entries.nextElement();
                classes.add(entry);
            }
            dex.close();
        } 
        catch (IOException e) {
            Log.e("HookDetection", e.toString());
        }
        for(String className : classes) {
            if(className.startsWith("com.example.hookdetection")) {
                try {
                    Class clazz = HookDetection.class.forName(className);
                    for(Method method : clazz.getDeclaredMethods()) {
                        if(Modifier.isNative(method.getModifiers())){
                            Log.wtf("HookDetection", "Native function found (could be hooked by Substrate or Xposed): " + clazz.getCanonicalName() + "->" + method.getName());
                        }
                    }
                }
                catch(ClassNotFoundException e) {
                    Log.wtf("HookDetection", e.toString());
                }
            }
        }
    }
}
```

## 0x04 通过/proc/[pid]/maps检测可疑的共享对象或者JAR

/proc/[pid]/maps记录了内存映射的区域和访问权限，首先查看Android应用的映像，第一列是起始地址和结束地址，第六列是映射文件的路径。  
``` bash
#cat /proc/5584/maps

40027000-4002c000 r-xp 00000000 103:06 2114      /system/bin/app_process
4002c000-4002d000 r--p 00004000 103:06 2114      /system/bin/app_process
4002d000-4002e000 rw-p 00005000 103:06 2114      /system/bin/app_process
4002e000-4003d000 r-xp 00000000 103:06 246       /system/bin/linker
4003d000-4003e000 r--p 0000e000 103:06 246       /system/bin/linker
4003e000-4003f000 rw-p 0000f000 103:06 246       /system/bin/linker
4003f000-40042000 rw-p 00000000 00:00 0 
40042000-40043000 r--p 00000000 00:00 0 
40043000-40044000 rw-p 00000000 00:00 0 
40044000-40047000 r-xp 00000000 103:06 1176      /system/lib/libNimsWrap.so
40047000-40048000 r--p 00002000 103:06 1176      /system/lib/libNimsWrap.so
40048000-40049000 rw-p 00003000 103:06 1176      /system/lib/libNimsWrap.so
40049000-40091000 r-xp 00000000 103:06 1237      /system/lib/libc.so
... Lots of other memory regions here ...
```
因此可以写代码检测加载到当前内存区域中的可疑文件：  
``` java
try {
    Set libraries = new HashSet();
    String mapsFilename = "/proc/" + android.os.Process.myPid() + "/maps";
    BufferedReader reader = new BufferedReader(new FileReader(mapsFilename));
    String line;
    while((line = reader.readLine()) != null) {
        if (line.endsWith(".so") || line.endsWith(".jar")) {
            int n = line.lastIndexOf(" ");
            libraries.add(line.substring(n + 1));
        }
    }
    for (String library : libraries) {
        if(library.contains("com.saurik.substrate")) {
            Log.wtf("HookDetection", "Substrate shared object found: " + library);
        }
        if(library.contains("XposedBridge.jar")) {
            Log.wtf("HookDetection", "Xposed JAR found: " + library);
        }
    }
    reader.close();
}
catch (Exception e) {
    Log.wtf("HookDetection", e.toString());
}
```
Substrate会用到几个so：  
```
Substrate shared object found: /data/app-lib/com.saurik.substrate-1/libAndroidBootstrap0.so
Substrate shared object found: /data/app-lib/com.saurik.substrate-1/libAndroidCydia.cy.so
Substrate shared object found: /data/app-lib/com.saurik.substrate-1/libDalvikLoader.cy.so
Substrate shared object found: /data/app-lib/com.saurik.substrate-1/libsubstrate.so
Substrate shared object found: /data/app-lib/com.saurik.substrate-1/libsubstrate-dvm.so
Substrate shared object found: /data/app-lib/com.saurik.substrate-1/libAndroidLoader.so
```
Xposed会用到一个Jar：  

Xposed JAR found: /data/data/de.robv.android.xposed.installer/bin/XposedBridge.jar  

## 0x05 绕过检测的方法

上面讨论了几个anti-hooking的方法，不过相信也会有人提出绕过的方法，这里对应每个检测方法如下：  

hook PackageManager的getInstalledApplications，把Xposed或者Substrate的包名去掉  
hook Exception的getStackTrace，把自己的方法去掉  
hook getModifiers，把flag改成看起来不是native 
hook 打开的文件的操作，返回/dev/null或者修改的map文件  