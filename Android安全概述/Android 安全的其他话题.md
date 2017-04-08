# 第六章 Android 安全的其它话题

> 来源：[Yury Zhauniarovich | Publications](http://www.zhauniarovich.com/pubs.html)

> 译者：[飞龙](https://github.com/)

> 协议：[CC BY-NC-SA 4.0](http://creativecommons.org/licenses/by-nc-sa/4.0/)

在本章中，我们会涉及到与 Android 安全相关的其他主题，这些主题不直接属于已经涉及的任何主题。

## 6.1 Android 签名过程

Android 应用程序以 Android 应用包文件（`.apk`文件）的形式分发到设备上。 由于这个平台的程序主要是用 Java 编写的，所以这种格式与 Java 包的格式 -- `jar`（Java Archive）有很多共同点，它用于将代码，资源和元数据（来自可选的`META-INF`目录 ）文件使用 zip 归档算法转换成一个文件。 `META-INF`目录存储软件包和扩展配置数据，包括安全性，版本控制，扩展和服务[5]。 基本上，在 Android 的情况中，`apkbuilder`工具将构建的项目文件压缩到一起[1]，使用标准的 Java 工具`jarsigner`对这个归档文件签名[6]。 在应用程序签名过程中，`jarsigner`创建`META-INF`目录，在 Android 中通常包含以下文件：清单文件（`MANIFEST.MF`），签名文件（扩展名为`.SF`）和签名块文件（`.RSA`或`.DSA`） 。

清单文件（`MANIFEST.MF`）由主属性部分和每个条目属性组成，每个包含在未签名的`apk`中文件拥有一个条目。 这些每个条目中的属性存储文件名称信息，以及使用 base64 格式编码的文件内容摘要。 在 Android 上，SHA1 算法用于计算摘要。 清单 6.1 中提供了清单文件的摘录。

```
1 Manifest−Version : 1.0 
2 Created−By: 1.6.0 41 (Sun Microsystems Inc. ) 
3 
4 Name: res/layout/main . xml 
5 SHA1−Digest : NJ1YLN3mBEKTPibVXbFO8eRCAr8= 
6 
7 Name: AndroidManifest . xml 
8 SHA1−Digest : wBoSXxhOQ2LR/pJY7Bczu1sWLy4=
```

代码 6.1：清单文件的摘录

包含被签名数据的签名文件（`.SF`）的内容类似于`MANIFEST.MF`的内容。 这个文件的一个例子如清单 6.2 所示。 主要部分包含清单文件的主要属性的摘要（`SHA1-Digest-Manifest-Main-Attributes`）和内容摘要（`SHA1-Digest-Manifest`）。 每个条目包含清单文件中的条目的摘要以及相应的文件名。

```
 1 Signature−Version : 1.0 
 2 SHA1−Digest−Manifest−Main−Attributes : nl/DtR972nRpjey6ocvNKvmjvw8= 
 3 Created−By: 1.6.0 41 (Sun Microsystems Inc. ) 
 4 SHA1−Digest−Manifest : Ej5guqx3DYaOLOm3Kh89ddgEJW4= 
 5 
 6 Name: res/layout/main.xml 
 7 SHA1−Digest : Z871jZHrhRKHDaGf2K4p4fKgztk= 
 8 
 9 Name: AndroidManifest.xml 
10 SHA1−Digest : hQtlGk+tKFLSXufjNaTwd9qd4Cw= 
11 ...
```

代码 6.2：签名文件的摘录

最后一部分是签名块文件（`.DSA`或`.RSA`）。 这个二进制文件包含签名文件的签名版本; 它与相应的`.SF`文件具有相同的名称。 根据所使用的算法（RSA 或 DSA），它有不同的扩展名。 

相同的apk文件有可能签署几个不同的证书。 在这种情况下，在`META-INF`目录中将有几个`.SF`和`.DSA`或`.RSA`文件（它们的数量将等于应用程序签名的次数）。

### 6.1.1 Android 中的应用签名检查

大多数 Android 应用程序都使用开发人员签名的证书（注意 Android 的“证书”和“签名”可以互换使用）。 此证书用于确保原始应用程序的代码及其更新来自同一位置，并在同一开发人员的应用程序之间建立信任关系。 为了执行这个检查，Android 只是比较证书的二进制表示，它用于签署一个应用程序及其更新（第一种情况）和协作应用程序（第二种情况）。

这种对证书的检查通过`PackageManagerService`中的方法`int compareSignatures(Signature[] s1，Signature[] s2)`来实现，代码如清单 6.3 所示。在上一节中，我们注意到在 Android 中，可以使用多个不同的证书签署相同的应用程序。这解释了为什么该方法使用两个签名数组作为参数。尽管该方法在 Android 安全规定中占有重要地位，但其行为强烈依赖于平台的版本。在较新版本中（从 Android 2.2 开始），此方法比较两个`Signature`数组，如果两个数组不等于`null`，并且如果所有`s2`签名都包含在`s1`中，则返回`SIGNATURE MATCH`值，否则为`SIGNATURE_NOT_MATCH`。在版本 2.2 之前，此方法检查数组`s1`是否包含在`s2`中。这种行为允许系统安装升级，即使它们已经使用原始应用程序的证书子集签名[2]。

在几种情况下，需要同一开发人员的应用程序之间的信任关系。 第一种情况与`signature`和`signatureOrSystem`的权限相关。 要使用受这些权限保护的功能，声明权限和请求它的包必须使用同一组证书签名。 第二种情况与 Android 运行具有相同 UID 或甚至在相同 Linux 进程中运行不同应用程序的能力有关。 在这种情况下，请求此类行为的应用程序必须使用相同的签名进行签名。

```java
 1 static int compareSignatures ( Signature[] s1 , Signature[] s2 ) { 
 2   if ( s1 == null ) { 
 3     return s2 == null 
 4       ? PackageManager.SIGNATURE_NEITHER_SIGNED 
 5       : PackageManager.SIGNATURE_FIRST_NOT_SIGNED; 
 6   } 
 7   if ( s2 == null ) { 
 8     return PackageManager.SIGNATURE_SECOND_NOT_SIGNED; 
 9   } 
10   HashSet<Signature> set1 = new HashSet<Signature>() ; 
11   for ( Signature sig : s1 ) { 
12     set1.add( sig ) ; 
13   } 
14   HashSet<Signature> set2 = new HashSet<Signature>() ; 
15   for ( Signature sig : s2 ) { 
16     set2.add( sig ) ; 
17   } 
18   // Make sure s2 contains all signatures in s1 . 
19   if ( set1.equals ( set2 ) ) { 
20     return PackageManager.SIGNATURE_MATCH; 
21   } 
22   return PackageManager.SIGNATURE_NO_MATCH; 
23 }
```

代码 6.3：`PackageManagerService`中的`compareSignatures`方法