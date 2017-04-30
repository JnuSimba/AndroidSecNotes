## synthetic的概念 

According to the JVM Spec: "**A class member that does not appear in the source code must be marked using a Synthetic attribute.**" Also, "The Synthetic attribute was introduced in JDK release 1.1 to support nested classes and interfaces."    

I know that nested classes are sometimes implemented using synthetic fields and synthetic contructors, e.g. **an inner class may use a synthetic field to save a reference to its outer class instance, and it may generate a synthetic contructor to set that field correctly**. I'm not sure if it Java still uses synthetic constructors or methods for this, but I'm pretty sure I did see them used in the past. I don't know why they might need synthetic classes here. On the other hand, something like RMI or java.lang.reflect.Proxy should probably create synthetic classes, since those classes don't actually appear in source code. I just ran a test where Proxy did not create a synthetic instance, but I believe that's probably a bug.   

Hmm, we discussed this some time ago back here. It seems like Sun is just ignoring this synthetic attribute, for classes at least, and we should too.   

## synthetic 实例
有synthetic标记的field 和method 是class 内部使用的，正常的源代码里不会出现synthetic field。  

下面的例子是最常见的synthetic field 
``` java
class parent {   
    public void foo() {   
    }   
  
    class inner {   
        inner() {   
      		foo();   
    	}   
    }   
}  
```
非static的inner class里面都会有一个`this$0`的字段保存它的父对象。编译后的inner class 就像下面这样：  
``` java
class parent$inner   
{   
	synthetic parent this$0;   
	parent$inner(parent this$0)   
	{   
		this.this$0 = this$0;   
		this$0.foo();   
	}   
}  
```
所有父对象的非私有成员都通过 `this$0`来访问，多层嵌套的情形如下所示。 
``` java
public class Outer { // this$0
	public class FirstInner { // this$1
		public class SecondInner { // this$2
			public class ThirdInner {
			}
		}
	}
}
```
还有许多用到synthetic 的地方，比如使用了assert 关键字的class会有一个 `synthetic static boolean $assertionsDisabled` 字段，`assert condition;` 在class里被编译成：  
``` java
if(!$assertionsDisabled && !condition)   
{   
	throw new AssertionError();   
}  
```

在jvm 里所有class的私有成员都不允许在其他类里访问，包括它的inner class。在java语言里inner class是可以访问父类的私有成员的，在class里是用如下的方法实现的：   
``` java
class parent   
{   
  private int value = 0;   
  synthetic static int access$000(parent obj)   
  {   
    return value;   
  }   
}  
```
在inner class里通过`access$000`来访问value字段。   

另外一个例子，外包类访问嵌套类私有属性。  
``` java
import java.lang.String;

public class A{
  private static class B{
    private String b1="b111";
    private String b2="b2222";
  }
  public static void main(String[] args){
    A.B b=new A.B();
    String tmp=b.b1;
            String tmp1=b.b2;
  }
}
```
运行javap -private A.B输出如下:
``` java
class A$B {
  private java.lang.String b1;
  private java.lang.String b2;
  private A$B();
  A$B(A$1);
  static java.lang.String access$100(A$B);
  static java.lang.String access$200(A$B);
}
``` 
生成了两个synthetic方法，分别对应于String tmp=b.b1;String tmp1=b.b2;访问两个私有属性。


