## 一. 类与文件

1. 一个 java 文件可以写多个类，每个类里面可以有main函数，一个java文件里面只能有一个 public 类，此时 java 文件的命名只能是public类名.java。使用 javac 编译一个 java 文件时，如果有多个类，会生成多个 类名.class 文件，java 类名 执行程序（单元测试）。
2. 多个class 文件可以打包成一个 jar 文件，java -jar test.jar 执行前需要设置一下程序入口，即在MANIFEST.MF 里面添加如下一句话：Main-Class: test.someClassName
3. 在Java中，为了组织代码的方便，可以将功能相似的类放到一个文件夹内，这个文件夹，就叫做包。包不但可以包含类，还可以包含接口和其他的包。
目录以"\"来表示层级关系，例如 E:\Java\workspace\Demo\bin\p1\p2\Test.java。
包以"."来表示层级关系，例如 p1.p2.Test 表示的目录为 \p1\p2\Test.class。
4. import 只能导入包所包含的类，而不能导入包。为方便起见，我们一般不导入单独的类，而是导入包下所有的类，例如 import java.util.*;

## 二. 关键字

### final
可以修饰类，方法和成员变量
final修饰的类不能被继承
final修饰的方法不能被覆盖
final修饰的变量是常量，只能赋值一次

覆盖注意事项：
1. 子类方法覆盖父类方法时，子类方法的权限要>=父类
2. 静态方法只能覆盖静态方法
3. 如果父类方法添加final, 则子类重新定义此方法会编译出错
4. 在子类方法中可以通过super.method 调用父类方法，当然如果父类方法是private，也是不能调用的（实际上是子类重新定义method，并没
有覆盖父类method，可以认为父类method被隐藏了）


### static
1. 用于修饰成员（成员变量和成员函数），被修饰后的成员具备以下特点：
  * 随着类的加载而加载，随着类的消失而消失
  * 优先于对象而存在
  * 被所有对象所共享
  * 可以直接用类名调用如类名.成员

2. 用于修饰静态代码块： static {...}
  * 随着类的加载而执行，而且只执行一次，可以用于给类进行初始化
  * 注：构造代码块{...}随着对象的构造而执行，而且创建几次就执行几次，可以用于给所有对象进行初始化
  * 静态代码块-->构造函数{super()-->成员初始化-->构造代码块-->后续语句}

3. 使用注意：
  * 静态方法只能访问静态成员
  * 静态方法中不可以出现this, super等关键字
  * 主函数是静态的

### this & super
* this代表本类对象的引用
* super代表一个父类空间
* 当本类的成员和局部变量同名用this区分
* 当子父类的成员变量同名用super区分父类

### interface 
* 当一个抽象类中的方法都是抽象的时候，这时可以将该抽象类用另一种形式定义和表示，就是接口interface. 
* 对于接口中的常见成员都有固定的修饰符。
  全局常量: public static final
  抽象方法： public abstract
* 类与类之间是继承extends关系；类与接口之间是实现implements关系；接口与接口之间是继承关系，而且接口可以多继承
* 类可以在继承一个类的同时实现多个接口
* 抽象类的继承，是is a关系，在定义该体系的基本共性内容，接口的实现是like a关系，在定义体系额外功能
* 接口类型的引用，用于指向接口的子类对象
* 抽象类可以为部分方法提供实现，避免了在子类中重复实现这些方法，提高了代码的可重用性，这是抽象类的优势；而接口中只能包含抽象方法，不能包含任何实现。  

## 三. 继承

在子类的构造函数中第一行有一个默认的隐式语句 super(); 子类中所有的构造函数默认都会访问父类中的空参数的构造函
数。如果父类中没有定义空参数构造函数，那么子类的构造函数必须用super(...)明确要调用父类中哪个构造函数。同时子类构造函数中如果使用this调用了本类构造函数时，那么super语句就没有了，因为super和this都只能定义在第一行，所有只能有一个。
但是可以保证的是，子类中肯定会有其他的构造函数访问父类的构造函数。  

## 四. 多态

1. 成员变量：
   编译时：参考引用型变量所属的类中是否有调用的成员变量，如果没有则编译失败
   运行时：参考引用型变量所属的类中是否有调用的成员变量，并运行该所属类中的成员变量

2. 成员函数：
   编译时：参考引用类型变量所属的类中是否有调用的函数，如果没有则编译失败
   运行时：参考的是对象所属的类中是否有调用的函数

3. 静态函数：
   编译时：参考引用类型变量所属的类中是否有调用的静态方法
   运行时：参考引用类型变量所属的类中是否有调用的静态方法
   其实对于静态方法，是不需要对象的，直接用类名调用


## 五. 内部类

内部类可以直接访问外部类中的成员
外部类要访问内部类，必须建立内部类的对象
   
// 直接访问外部类中的内部类中的成员
``` java
outer.inner in = new outer().new inner();
in.show();
```
// 如果内部类是静态的，相当于一个外部类
```
outer.inner in = new outer.inner();
in.show(); 
```
//如果内部类是静态的，而且成员是静态的 
`outer.inner.function();`

// 如果内部类中定义了静态成员，该内部类必须也是静态的

``` java

class Outer
{
    int num = 3;
    class Inner
    {
        int num = 4;
        void show()
        {
            int num = 5;
            System.out.println(Outer.this.num);
        }
    }
    void method()
    {
        new Inner().show();
    }
}
```    
* 局部内部类
  内部类可以放在局部位置上
  内部类在局部位置上只能访问局部中被final修饰的局部变量

     
* 匿名内部类

   前提：内部类必须继承或者实现一个外部类或者接口。
   匿名内部类其实就是一个匿名子类对象。
   格式：new 父类or接口(){子类内容}

``` java
abstract class Demo
{
    abstract void show();
}

class Outer
{
    int num = 4;
    /*
    class Inner extends Demo
    {
        void show()
        {
            System.out.println("show ..."+num);
        }
    }
    */
    public void method()
    {
        //new Inner().show();
        /* Demo de = */ new Demo()//匿名内部类。
        {
            void show()
            {
                System.out.println("show ........" + num);
            }
        } .show();
        //      de.show();
    }
}


class InnerClassDemo4
{
    public static void main(String[] args)
    {
        new Outer().method();
    }
}
```

注意：如下做法是错误的

``` java
Object obj = new Object()
{
    public void show()
    {
        System.out.println("show run");
    }

};
obj.show();
```
因为匿名内部类这个子类对象被向上转型为了Object类型，而Object类并没有show()的实现

通常的使用场景之一：
当函数参数是接口类型时，而且接口中的方法不超过三个，可以用匿名内部类作为实际参数进行传递
``` java
interface Inter
{
    void show1();
    void show2();
}

class InnerClassDemo5
{
    public static void main(String[] args)
    {
        System.out.println("Hello World!");

        show(new Inter()
        {
            public void show1() {}
            public void show2() {}
        });

    }

    public static void show(Inter in)
    {
        in.show1();
        in.show2();
    }
}
```  

## 六. 异常

函数内容如果抛出需要检测的异常，那么函数必须要声明异常，否则必须在函数内用try catch捕捉，否则编译失败
如果调用到了声明异常的函数，要么try catch 要么throws， 否则编译失败
功能内容可以解决用catch，解决不了用throws告诉调用者，由调用者解决
一个功能如果抛出了多个异常，那么调用时必须有对应多个catch进行针对性的处理

自定义异常时，继承Exception类(编译时异常），或者RuntimeException类(运行时异常）

子类在覆盖父类方法时，父类的方法如果抛出了异常，那么子类的方法只能抛出父类的异常或者该异常的子类

如果父类抛出多个异常，那么子类只能抛出父类异常的子集。如果父类方法没有抛出异常，那么子类覆盖时绝对不能抛


## 七.访问权限

包与包之间的类进行访问，被访问的包中的类必须是public的，被访问的包中的类的方法也必须是public的。
<table border="1">
  <tr>
<th></th>
   <th>public</th>
   <th>protected</th>
<th>default</th>
   <th>private</th>
  </tr>
  <tr>
<td>同一类中</td>
   <td>Y</td>
   <td>Y</td>
<td>Y</td>
   <td>Y</td>
  </tr>
  <tr>
<td>同一包中</td>
   <td>Y</td>
   <td>Y</td>
<td>Y</td>
   <td>N</td>
  </tr>
<tr>
<td>子类中</td>
   <td>Y</td>
   <td>Y</td>
<td>N</td>
<td>N</td>
  </tr>
<tr>
<td>不同包中</td>
   <td>Y</td>
   <td>N</td>
<td>N</td>
   <td>N</td>
  </tr>
</table>


## 八.线程

创建线程的第一种方式:继承Thread类。

创建线程的第二种方式：实现Runnable接口。

1. 定义类实现Runnable接口。
2. 覆盖接口中的run方法，将线程的任务代码封装到run方法中。
3. 通过Thread类创建线程对象，并将Runnable接口的子类对象作为Thread类的构造函数的参数进行传递。
为什么？因为线程的任务都封装在Runnable接口子类对象的run方法中。
所以要在线程对象创建时就必须明确要运行的任务。
4. 调用线程对象的start方法开启线程。

实现Runnable接口的好处：
1. 将线程的任务从线程的子类中分离出来，进行了单独的封装。按照面向对象的思想将任务的封装成对象。
2. 避免了java单继承的局限性。

所以，创建线程的第二种方式较为常用。

``` java
class Thread
{
    private Runnable r;
    Thread()
    {

    }
    Thread(Runnable r)
    {
        this.r  = r;
    }

    public void run()
    {
        if(r != null)
            r.run();
    }

    public void start()
    {
        run();
    }
}
class ThreadImpl implements Runnable
{
    public void run()
    {
        System.out.println("runnable run");
    }
}
ThreadImpl i = new ThreadImpl();
Thread t = new Thread(i);
t.start();




class SubThread extends Thread
{
    public void run()
    {
        System.out.println("hahah");
    }
}
SubThread s = new SubThread();
s.start();
```


