# JVM

## 生命周期

### 线程分类

#### 普通线程 (no-daemon)

> 我们自己创建的线程,比如main方法

#### 守护线程(daemon)

> 由jvm虚拟机创建的线程,比如维护GC的线程,也可以通过setDaemon方法将线程设置成守护线程

### 开始

> 由Main()方法开始,启动一个线程运行Main方法,再由main方法运行其他线程,这些线程都属于普通线程

### 运行

> 通过main方法运行我们编写好的程序

### 结束

1. java.lang.system.exit()
2. 正常执行结束

> 指的是我们创建的所有普通线程都结束了之后jvm就会正常结束

3. 程序异常或者遇到错误

> 当程序出现异常没有被即使处理后,会一层一层向上抛,当在main方法中还没有被处理,jvm虚拟机会终止将异常抛出

4. 操作系统引起的异常

## jvm参数

### 三种情况

```bash
-XX:+<option> # 开启option选项
-XX:-<option> # 关闭option选项
-XX:<option>=<value> #设置option选项的值=value
```

### 助记符

> - ldc : 表示将 int,float或者是string类型的常量值从**常量池**中推送到**栈顶**
> - iconst_x[-1-5] : 将-1-5推送到栈顶,-1的时候助记符是iconst_m1
> - bipush [x] : 将一个数字推送到栈顶
> - anewarray [x]: 创建一个引用类型的数组,并将其压入栈顶
> - newarray [x]: 创建一个指定的基础类型(int,float,char等)数组,并将其引用值压入栈顶

```mermaid
graph LR
a[加载]
b[连接]
b1(验证)
b2(准备)
b3(解析)
c[初始化]
d[类实例化/使用]
e[垃圾回收和对象终结/卸载]
a --> b
b --> b1

b1 --> b2
b2 --> b3

b3 --> c
c --> d
d --> e
```

## 类加载

![](https://raw.githubusercontent.com/huwd5620125/my_pic_pool/master/img/20191217174820.png)

> 在JAVA代码中,类型的**加载**、**连接**与**初始化过程**都是在**程序运行期间**完成的
>
> 提供了更大的系统灵活性，更多的可能性

### 类的加载

> 读取类(.class)的二进制文件到内存中,将其放到**运行时数据区**的**方法区**内,然后在内存中建立一个**java.lang.class**对象(jvm规范没有指定准确位置,hotsport指定jvm放在方法区中),用来封装类在方法区中的数据结构。
>
> 类的加载最终产品就是位于内存中(**方法区**)中的class对象
>
> Class对象封装了类在方法区中的数据结构,并且向java程序员提供了访问方法区的数据结构的接口

#### 类加载器

> 类加载器并不需要等到某个类被"**首次主动使用**"时**加载**它
>
> - JVM规范允许类加载器在预料到某个类将要被使用的时候**预先加载它**,如果预先加载class文件缺失或存在错误,类加载器则必须要在程序**首次主动使用该类时**才会**报错**(**LinkageError**)
> - 如果这个错误的class一直没有被**主动使用**则JVM一直都不会报错

##### JVM自带的加载器

- 启动类加载器(Bootstrap)

> 该加载器没有父类,负责加载核心库文件

- 扩展类加载器(Extension)
- 系统(应用)类加载器(System)

##### 自定义加载器

- java.lang.ClassLoader的子类
- 用户可以定制类的加载方式

##### 双亲委派模型

> 从JDK1.2开始,java中类加载才用了双亲委派模型,除了jdk自带的根加载器外,其余的类加载器都有且只有一个父加载器,当java程序请求类加载器loader加载一个类的时候,loader会首先委派自己的父加载器尝试加载这个类,如果父加载器可以完成加载,那么则由父加载器加载这个类,否则才由当前的加载器加载

#### 加载方式

- 从本地磁盘中直接加载
- 从网络中下载class文件
- 从zip，jar中加载
- 从数据库加载
- 将java文件动态的编译为class文件（**动态代理**，jsp）

### 类的连接

#### 验证

> 确保被加载的类的正确性,验证字节码的**有效性**

#### 准备

> 为类的**静态变量**分配内存,并将其初始化为**默认值**

#### 解析

> 把类中的**符号引用**(类,接口,字段,方法)转换为**直接引用**

### 类的初始化

> 为类的**静态变量赋初始值**
>
> 类的静态字段以及静态代码块都被视作类的**初始化**语句
>
> 在**初始化阶段**会从**上而下**的执行初始化语句

#### 初始化步骤

- 假如这个类没有被加载和连接,先进行加载和连接
- 假如这个类存在父类,这个类没有被初始化,则先初始化父类
- 假如类中存在初始化语句,那么依次执行初始化语句

#### 注意

- !第二条规则并不适用于接口,在初始化一个类的时候,并不会先初始化它所实现的接口
- !在初始化一个接口的时候,并不会先初始化它的父接口
- !所以一个父接口并不会因为它的子接口或者实现类的初始化而初始化.只有当程序首次使用特定接口的静态变量的时候,才会导致该接口的初始化
- !!只有当程序访问的**静态变量**或者**静态方法**确实在**当前类或者当前接口的定义中**时,才可以认为是对类或者接口的主动使用
- !!调用ClassLoader类的loadClass方法加载一个类的时候,并不是对类的主动使用,不会导致类的初始化
- !!类加载器加载一个类的时候 并不属于对一个类的主动使用,反射属于对一个类的主动使用

```java
public class MyTest5 {
    public static void main(String[] args) {
		//System.out.println(MyChild51.a);//调用接口的变量,
        // 所有的class都不会被初始化,因为a是一个常量,编译器就会存在MyTest5的常量池中
        System.out.println(MyChild51.b);//调用子类的变量,
        // 只有MyChild51会被初始化,虽然调用了子类,但是子类初始化的时候,使用的接口不会被初始化
		//System.out.println(MyChild51.thread);//调用接口的变量
        // 接口中变量被直接调用的时候,接口就会被初始化
        // 因为接口的所有变量必然会被public static final标示
        // 就会存在和在普通类加载中一样的情况,若这个变量是一个可以确定的编译期常量,那么和类中一样,调用者会在编译器直接引用
        // 这个常量的值,运行期不会去加载这个变量所在的类,若不是一个编译器可以确认的值,那么还是会在运行期初始化这个类或接口
    }
}

interface Myparent5 {
    //没有初始化,因为没有打印
    public static Thread thread = new Thread() {
        {
            /*实例代码块*/
            System.out.println("MyParent5 start");
        }
    };
    public static final int a = 5;
}

interface MyChild5 extends Myparent5 {
    public static Thread childThread = new Thread() {
        {
            /*实例代码块*/
            System.out.println("MyChild5 start");
        }
    };
    public static final int b = 6;
}

class MyChild51 implements MyChild5 {
    static {
        System.out.println("MyChild51 starter");
    }

    public static int b = 6;
}
```

```java
// 反射和类加载器加载
public class MyTest12 {
    public static void main(String[] args) throws Exception {
        ClassLoader cl = ClassLoader.getSystemClassLoader();
        Class<?> clazz = cl.loadClass("top.cjsx.jvm.classloader.CL");//类加载器加载的时候并不会初始化
        System.out.println(clazz);
        System.out.println("============");
        Class<?> aClass = Class.forName("top.cjsx.jvm.classloader.CL");//反射的时候会初始化
        CL o= (CL) aClass.getDeclaredConstructor().newInstance();//这个时候会new
        System.out.println(aClass);
        System.out.println(o);
    }
}
```

### 类实例化

> 在堆上为对象分配内存
>
> 为实例变量赋正确的初始值
>
> java编译器会为每个类都**至少生成一个实例初始化**方法,在java的class文件中这个方法**被称作**<init>
>
> 针对源代码中**每一个构造器**,java**都会**生成一个<init>方法

### 类的使用

#### 主动使用

1. 创建类的实例(**new Object()**)
2. 访问某个类或者接口的**静态变量**，或者对静态变量赋值（对静态变量取值和赋值，getstatic，putstatic）(**只会加载这个静态变量所在的类**)
3. 调用类的静态方法（**invokestatic**）**(只会加载这个静态方法所在的类**)
4. 反射(**Class.forName("com.test.Test")**)
5. 初始化一个**类的子类**
6. jvm启动的时候,标明启动类的类(**设置为启动类的类**,包含main(String[] args)方法)
7. 1.7开始提供动态语言解析,java.lang.invoke.MethodHandle实例的解析结果(REF_getStatic,REF_putStatic,REF_invokeStatic)

#### 注意

> Java程序对类的使用方式,分为两种,一种是**主动使用**,一种是**被动使用**。
>
> JVM必须在每个类或者接口被JAVA程序**首次主动使用**时才会**初始化**他们。
>
> 

#### 被动使用

> 被动使用不会导致**初始化**这个动作
>
> 1. 创建一个对象的数组
>
> ```java
> Parent4[] parent4s = new Parent4[1];//new一个对象的时候
> Parent4[][] parent4ss = new Parent4[1][1];//new数组的话,jvm会使用一个特殊的对象存储数组,并没有创建
>  System.out.println(parent4s.getClass());// [Ltop.cjsx.jvm.classloader.Parent4
> //jvm运行期创建了一个数组类型的class,继承自Object
> //并不算是对parent4对象的主动对象
> //对于数组来说,JavaDoc经常将构成数组的元素为Component,实际上就是将数组降低一个纬度后的类型
> ```
>
> 2. 调用一个静态字段
>
> ```java
> class MyParent1 {
>     public static String str = "Hello World";
> 
>     static {
>         System.out.println("Parent initialize");
>     }
> }
> 
> class MyChild1 extends MyParent1 {
>     public static final String str2 = "Hello World";
>     static {
>         System.out.println("Child initialize");
>     }
> }
> //这个时候如果调用 MyChild1.str2 不会初始化MyChild1,因为对于静态字段来说,调用静态字段只会初始化这个字段所在的那个类
> ```
>
> 3. 调用接口的字段
>
> ```java
> public class MyTest5 {
>     public static void main(String[] args) {
>         System.out.println(MyChild5.a);//调用接口的变量
>         // 因为接口的所有变量必然会被public static final标示
>         // 就会存在和在普通类加载中一样的情况,若这个变量是一个可以确定的编译期常量,那么和类中一样,调用者会在编译器直接引用
>         // 这个常量的值,运行期不会去加载这个变量所在的类,若不是一个编译器可以确认的值,那么还是会在运行期初始化这个类或接口
>     }
> }
> 
> interface Myparent5 {
>     public static final int a = 5;
> }
> 
> interface MyChild5 extends Myparent5 {
>     public static final int b = 6;
> }
> ```
>
> 

##### 案例

```java
package top.cjsx.jvm.classloader;

/**
 * @author chengjs
 * @Date: 2019/11/06 17:10
 * @Desc: 主动使用和被动使用, 对于静态字段来说, 只有直接定义了该字段的类才会被初始化
 */
public class MyTest1 {
    public static void main(String[] args) {
         //System.out.println(MyChild1.str);
        /*
        如果调用的是基础类型或者静态字段,那么对于静态字段来说,被调用的时候只会初始化直接定义了该字段的类
        */
        System.out.println(MyChild1.str2);
        /*
        1.如果str2是静态字段,那么会初始化parent和child2个类,因为调用静态对象会初始化mychild,同时因为调用了parent的子类child,所以parent会首先初始化
        2.如果str2是编译期常量,那么常量字符串会在编译的时候从MyChild1中获取值,存放到调用常量的这个类的常量池中,本质上当前类并没有调用child中的常量,因此不会触发child1的初始化,甚至可以把child1的class文件删除
        */
    }
}

class MyParent1 {
    public static String str = "Hello World";

    static {
        System.out.println("Parent initialize");
    }
}

class MyChild1 extends MyParent1 {
    public static final String str2 = "Hello World";
    /*如果有final字段,则表示该字符串是一个常量,
    不可修改,则会在打包的时候就已经对变量赋值,不需要初始化时再赋值
    而是编译阶段存放到这个常量被调用的类所在的常量池中
    */
//    public static String str2 = "Hello World";
      /*如果没有final字段,则表示str2是一个静态字符串*/

    static {
        System.out.println("Child initialize");
    }
}

/*                 
输出结果:
Parent initialize
Hello World
*/
```

> -XX:+TraceClassLoading,用于追踪类的加载信息并打印出来
