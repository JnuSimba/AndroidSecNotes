
原文 by 瘦蛟舞  
官方教程: https://github.com/rovo89/XposedBridge/wiki/Development-tutorial  

官网: http://repo.xposed.info/module/de.robv.android.xposed.installer  

apk: http://dl-xda.xposed.info/modules/de.robv.android.xposed.installer_v33_36570c.apk  

源码: https://github.com/rovo89/XposedInstaller  

## 模块基本开发流程

1. 创建工程android4.0.3(api15,测试发现其他版本也可以),可以不用activity  

2. 修改AndroidManifest.xml
``` xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
package="de.robv.android.xposed.mods.tutorial"
android:versionCode="1"
android:versionName="1.0" >
<uses-sdk android:minSdkVersion="15" />
<application
android:icon="@drawable/ic_launcher"
android:label="@string/app_name" >
<meta-data
    android:name="xposedmodule"
    android:value="true" />
<meta-data
    android:name="xposeddescription"
    android:value="Easy example" />
<meta-data
    android:name="xposedminversion"
    android:value="54" />
</application>
</manifest>
```
3. 在工程目录下新建一个lib文件夹,将下载好的XposedBridgeApi-54.jar包放入其中.  

eclipse 在工程里 选中XposedBridgeApi-54.jar 右键–Build Path–Add to Build Path.  

IDEA 鼠标右键点击工程,选择Open Module Settings,在弹出的窗口中打开Dependencies选项卡.把XposedBridgeApi这个jar包后面的Scope属性改成provided.  

4. 模块实现接口  
``` java
package de.robv.android.xposed.mods.tutorial;

import de.robv.android.xposed.IXposedHookLoadPackage;
import de.robv.android.xposed.XposedBridge;
import de.robv.android.xposed.callbacks.XC_LoadPackage.LoadPackageParam;

public class Tutorial implements IXposedHookLoadPackage {
public void handleLoadPackage(final LoadPackageParam lpparam) throws Throwable {
XposedBridge.log("Loaded app: " + lpparam.packageName);
	}
}
```
5. 入口assets/xposed_init配置,声明需要加载到 XposedInstaller 的入口类:

`de.robv.android.xposed.mods.tutorial.Tutorial`  //完整类名:包名+类名  
6. 定位要hook的api 

反编译目标程序,查看Smali代码  
直接在AOSP(android源码)中查看  
7. XposedBridge to hook it  

指定要 hook 的包名  
判断当前加载的包是否是指定的包  
指定要 hook 的方法名  
实现beforeHookedMethod方法和afterHookedMethod方法  
示例如下:  
``` java
package de.robv.android.xposed.mods.tutorial;

import static de.robv.android.xposed.XposedHelpers.findAndHookMethod; 
import de.robv.android.xposed.IXposedHookLoadPackage; 
import de.robv.android.xposed.XC_MethodHook; 
import de.robv.android.xposed.callbacks.XC_LoadPackage.LoadPackageParam;

public class Tutorial implements IXposedHookLoadPackage { public void handleLoadPackage(final LoadPackageParam lpparam) throws Throwable { if (!lpparam.packageName.equals("com.android.systemui")) return;

findAndHookMethod("com.android.systemui.statusbar.policy.Clock", lpparam.classLoader, "updateClock", new XC_MethodHook() {
    @Override
    protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
        // this will be called before the clock was updated by the original method
    }
    @Override
    protected void afterHookedMethod(MethodHookParam param) throws Throwable {
        // this will be called after the clock was updated by the original method
    }
});
}


}
```
重写XC_MethodHook的两个方法beforeHookedMethod和afterHookedMethod，这两个方法会在原始的方法的之前和之后执行.您可以使用beforeHookedMethod 方法来打印/篡改方法调用的参数(通过param.args) ,甚至阻止调用原来的方法（发送自己的结果）.afterHookedMethod 方法可以用来做基于原始方法的结果的事情.您还可以用它来操纵结果 .当然，你可以添加自己的代码,它将会准确地在原始方法的前或后执行.  

## 关键API  

`IXposedHookLoadPackage`  

`handleLoadPackage` : 这个方法用于在加载应用程序的包的时候执行用户的操作  

调用示例  
``` java
public class XposedInterface implements IXposedHookLoadPackage {
public void handleLoadPackage(final LoadPackageParam lpparam) throws Throwable {
XposedBridge.log("Kevin-Loaded app: " + lpparam.packageName); }
}
```
参数说明|final LoadPackageParam lpparam 这个参数包含了加载的应用程序的一些基本信息。  

`XposedHelpers `

`findAndHookMethod` ;这是一个辅助方法,可以通过如下方式静态导入:  

`import static de.robv.android.xposed.XposedHelpers.findAndHookMethod;`  
使用示例  
``` java
findAndHookMethod("com.android.systemui.statusbar.policy.Clock", lpparam.classLoader, "handleUpdateClock", new XC_MethodHook() {
@Override
protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
// this will be called before the clock was updated by the original method }
@Override
protected void afterHookedMethod(MethodHookParam param) throws Throwable {
// this will be called after the clock was updated by the original method }
});
```
参数说明  

findAndHookMethod(Class<?>clazz, //需要Hook的类名
ClassLoader, //类加载器,可以设置为 null 
String methodName, //需要 Hook 的方法名 
Object... parameterTypesAndCallback
该函数的最后一个参数集,包含了:  

(1)Hook 的目标方法的参数,譬如:  

`"com.android.internal.policy.impl.PhoneWindow.DecorView"`  
是方法的参数的类。  

(2)回调方法:  

`a.XC_MethodHook`   
`b.XC_MethodReplacement`  

## 模块开发中的一些细节

1. Dalvik 孵化器 Zygote (Android系统中，所有的应用程序进程以及系统服务进程SystemServer都是由Zygote进程孕育/fork出来的)进程对应的程序是/system/bin/app_process. Xposed 框架中真正起作用的是对方法的 hook。  
因为 Xposed 工作原理是在/system/bin 目录下替换文件,在 install 的时候需要 root 权限,但是运行时不需要 root 权限。  

2. log 统一管理,tag 显示包名  
`Log.d(MYTAG+lpparam.packageName, "hello" + lpparam.packageName);`

3. 植入广播接收器,动态执行指令  

``` java
findAndHookMethod("android.app.Application", lpparam.classLoader, "onCreate", new XC_MethodHook() {
@Override
protected void beforeHookedMethod(MethodHookParam param) throws Throwable {

Context context = (Context) param.thisObject;
IntentFilter filter = new IntentFilter(myCast.myAction);
filter.addAction(myCast.myCmd);
context.registerReceiver(new myCast(), filter);

}

@Override
protected void afterHookedMethod(MethodHookParam param) throws Throwable {
super.afterHookedMethod(param);
}
});
```
4. context 获取

`fristApplication = (Application) param.thisObject;`   

5. 注入点选择 application oncreate 程序真正启动函数而是 MainActivity 的 onCreate (该类有可能被重写,所以通过反射得到 oncreate 方法)

``` java
String appClassName = this.getAppInfo().className;
if (appClassName == null) {
Method hookOncreateMethod = null;
try {
    hookOncreateMethod = Application.class.getDeclaredMethod("onCreate", new Class[] {});
} catch (NoSuchMethodException e) {
    e.printStackTrace();
}
hookhelper.hookMethod(hookOncreateMethod, new ApplicationOnCreateHook());
```
6. 排除系统 app,排除自身,确定主线程

``` java
if(lpparam.appInfo == null || 
    (lpparam.appInfo.flags & (ApplicationInfo.FLAG_SYSTEM | ApplicationInfo.FLAG_UPDATED_SYSTEM_APP)) !=0){
		return;
}else if(lpparam.isFirstApplication && !ZJDROID_PACKAGENAME.equals(lpparam.packageName)){
```
7. hook method
Only methods and constructors can be hooked,Cannot hook interfaces,Cannot hook abstract methods  
只能 hook 方法和构造方法,不能 hook 接口和抽象方法  
抽象类中的非抽象方法是可以 hook的, 接口中的方法不能 hook (接口中的method默认是public abstract 抽象的.field 必须是public static final)  

8. 参数中有 自定义类
`public void myMethod (String a, MyClass b) `
通过反射得到自定义类,也可以用[xposedhelpers](https://github.com/rovo89/XposedBridge/wiki/Helpers#class-xposedhelpers) 封装好的方法findMethod/findConstructor/callStaticMethod....  

9. 注入后反射自定义类

``` java
Class<?> hookMessageListenerClass = null;

hookMessageListenerClass = lpparam.classLoader.loadClass("org.jivesoftware.smack.MessageListener");

findAndHookMethod("org.jivesoftware.smack.ChatManager", lpparam.classLoader, "createChat", String.class , hookMessageListenerClass ,new XC_MethodHook() {
@Override
protected void beforeHookedMethod(MethodHookParam param) throws Throwable {

    String sendTo = (String) param.args[0];
    Log.i(tag , "sendTo : + " + sendTo );

}

@Override
protected void afterHookedMethod(MethodHookParam param) throws Throwable {
    super.afterHookedMethod(param);
}
});
```

10. hook 一个类的方法,该类是子类并且没有重写父类的方法,此时应该 hook 父类还是子类.(hook 父类方法后,子类若没重写,一样生效.子类重写方法需要另外 hook) (如果子类重写父类方法时候加上 spuer ,hook 父类依旧有效) 
例如 java.net.HttpURLConnection extends URLConnection  
方法在父类  

``` java
public OutputStream getOutputStream() throws IOException {
   throw new UnknownServiceException("protocol doesn't support output");
}


org.apache.http.impl.client.AbstractHttpClient extends CloseableHttpClient ,
方法在父类(注意,android的继承的 AbstractHttpClient implements org.apache.http.client.HttpClient)


public CloseableHttpResponse execute(
    final HttpHost target,
    final HttpRequest request,
    final HttpContext context) throws IOException, ClientProtocolException {
return doExecute(target, request, context);
}


android.async.http复写HttpGet导致zjdroid hook org.apache.http.impl.client.AbstractHttpClient execute 无法获取到请求 
url和method
```

11. hook 构造方法

``` java
public static XC_MethodHook.Unhook findAndHookConstructor(String className, ClassLoader classLoader, Object... parameterTypesAndCallback) {
return findAndHookConstructor(findClass(className, classLoader), parameterTypesAndCallback);
}
```

12. 承接4,application 的onCreate 方法被重写,例如阿里的壳,重写为原生 native 方法.   
解1:通过反射到 application 类重写后的 onCreate 方法再对该方法进行hook  
解2:hook 构造方法(构造方法被重写,继续解1)  

13. native 方法可以 hook,不过是在 java 层调用时hook而不是 hook 动态链接库.  

## 实战Hook android 中可能的出现 HTTP 请求  

首先确定http 请求的 api,大致分为:  
* apache 提供的 HttpClient
1) 创建 HttpClient 以及 GetMethod / PostMethod， HttpRequest等对象;
2) 设置连接参数;
3) 执行 HTTP 操作;
4) 处理服务器返回结果.
* java 提供的 HttpURLConnection
1) 创建 URL 以及 URLConnection / HttpURLConnection 对象
2) 设置连接参数
3) 连接到服务器
4) 向服务器写数据
5) 从服务器读取数据
* android 提供的 webview
* 第三方库:volley/android-async-http/xutils (本质是对前两种的方式的延伸,方法的重写可能对接下来的 hook 产生影响)
不太了解 java 的 hook 前可以先看下基础的代码。  
对 HttpClient的 hook 可以参考 贾志军大牛的Zjdroid
``` java
Method executeRequest = RefInvoke.findMethodExact("org.apache.http.impl.client.AbstractHttpClient", ClassLoader.getSystemClassLoader(),
    "execute", HttpHost.class, HttpRequest.class, HttpContext.class);

hookhelper.hookMethod(executeRequest, new AbstractBahaviorHookCallBack() {
@Override
public void descParam(HookParam param) {
    // TODO Auto-generated method stub
    Logger.log_behavior("Apache Connect to URL ->");
    HttpHost host = (HttpHost) param.args[0];

    HttpRequest request = (HttpRequest) param.args[1];
    if (request instanceof org.apache.http.client.methods.HttpGet) {
        org.apache.http.client.methods.HttpGet httpGet = (org.apache.http.client.methods.HttpGet) request;
        Logger.log_behavior("HTTP Method : " + httpGet.getMethod());
        Logger.log_behavior("HTTP GET URL : " + httpGet.getURI().toString());
        Header[] headers = request.getAllHeaders();
        if (headers != null) {
            for (int i = 0; i < headers.length; i++) {
                Logger.log_behavior(headers[i].getName() + ":" + headers[i].getName());
            }
        }
    } else if (request instanceof HttpPost) {
        HttpPost httpPost = (HttpPost) request;
        Logger.log_behavior("HTTP Method : " + httpPost.getMethod());
        Logger.log_behavior("HTTP URL : " + httpPost.getURI().toString());
        Header[] headers = request.getAllHeaders();
        if (headers != null) {
            for (int i = 0; i < headers.length; i++) {
                Logger.log_behavior(headers[i].getName() + ":" + headers[i].getValue());
            }
        }
        HttpEntity entity = httpPost.getEntity();
        String contentType = null;
        if (entity.getContentType() != null) {
            contentType = entity.getContentType().getValue();
            if (URLEncodedUtils.CONTENT_TYPE.equals(contentType)) {

                try {
                    byte[] data = new byte[(int) entity.getContentLength()];
                    entity.getContent().read(data);
                    String content = new String(data, HTTP.DEFAULT_CONTENT_CHARSET);
                    Logger.log_behavior("HTTP POST Content : " + content);
                } catch (IllegalStateException e) {
                    // TODO Auto-generated catch block
                    e.printStackTrace();
                } catch (IOException e) {
                    // TODO Auto-generated catch block
                    e.printStackTrace();
                }

            } else if (contentType.startsWith(HTTP.DEFAULT_CONTENT_TYPE)) {
                try {
                    byte[] data = new byte[(int) entity.getContentLength()];
                    entity.getContent().read(data);
                    String content = new String(data, contentType.substring(contentType.lastIndexOf("=") + 1));
                    Logger.log_behavior("HTTP POST Content : " + content);
                } catch (IllegalStateException e) {
                    // TODO Auto-generated catch block
                    e.printStackTrace();
                } catch (IOException e) {
                    // TODO Auto-generated catch block
                    e.printStackTrace();
                }

            }
        }else{
            byte[] data = new byte[(int) entity.getContentLength()];
            try {
                entity.getContent().read(data);
                String content = new String(data, HTTP.DEFAULT_CONTENT_CHARSET);
                Logger.log_behavior("HTTP POST Content : " + content);
            } catch (IllegalStateException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            } catch (IOException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            }

        }

    }
}

@Override
public void afterHookedMethod(HookParam param) {
    // TODO Auto-generated method stub
    super.afterHookedMethod(param);
    HttpResponse resp = (HttpResponse) param.getResult();
    if (resp != null) {
        Logger.log_behavior("Status Code = " + resp.getStatusLine().getStatusCode());
        Header[] headers = resp.getAllHeaders();
        if (headers != null) {
            for (int i = 0; i < headers.length; i++) {
                Logger.log_behavior(headers[i].getName() + ":" + headers[i].getValue());
            }
        }

    }
}
});
``` 
对 HttpURLConnection 的 hook Zjdroid 未能提供完美的解决方案,想要取得除了 URL 之外的 data 字段必须对I/O流操作.  
``` java
Method openConnectionMethod = RefInvoke.findMethodExact("java.net.URL", ClassLoader.getSystemClassLoader(), "openConnection");
hookhelper.hookMethod(openConnectionMethod, new AbstractBahaviorHookCallBack() {
@Override
public void descParam(HookParam param) {
    // TODO Auto-generated method stub
    URL url = (URL) param.thisObject;
    Logger.log_behavior("Connect to URL ->");
    Logger.log_behavior("The URL = " + url.toString());
}
});
```
我采取的临时解决方法是对I/O 进行正则匹配,类似 url 的 data 字段就打印出来,代码如下(这段代码只能解决前文 HttpUtils而且会有误报,大家有啥好想法欢迎指点一二)  
``` java
findAndHookMethod("java.io.PrintWriter", lpparam.classLoader, "print",String.class, new XC_MethodHook() {
@Override
protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
    String print = (String) param.args[0];
    Pattern pattern = Pattern.compile("(\\w+=.*)");
    Matcher matcher = pattern.matcher(print);
    if (matcher.matches())
        Log.i(tag+lpparam.packageName,"data : " + print);
    //Log.d(tag,"A :" + print);
}

});
```
因为Android-async-http重写了 HttpGet 导致 Zjdroidhook 失败(未进入 HttpGet 和 HttpPost 的判读),加入一个else 语句就可以解决这个问题  
``` java
else {
            HttpEntityEnclosingRequestBase httpGet = (HttpEntityEnclosingRequestBase) request;
            HttpEntity entity = httpGet.getEntity();
            Logger.log_behavior("HttpRequestBase URL : " + httpGet.getURI().toString());
            Header[] headers = request.getAllHeaders();
            if (headers != null) {
                for (int i = 0; i < headers.length; i++) {
                    Logger.log_behavior(headers[i].getName() + ":" + headers[i].getName());
                }
            }

            if(entity!= null){
                        try {
                            String content = EntityUtils
                                    .toString(entity);
                            Logger.log_behavior("HTTP entity Content : "
                                    + content);
                        } catch (IllegalStateException e) {
                            // TODO Auto-generated catch block
                            e.printStackTrace();
                        } catch (IOException e) {
                            // TODO Auto-generated catch block
                            e.printStackTrace();
                        }
            }

            }
```
## 一些常用工具

zjdroid:脱壳/api监控  
justTrustMe:忽略证书效验  
IntentMonitor:可以监控显/隐意图 intent  
Xinstaller:设置应用/设备属性...  
XPrivacy:权限管理  