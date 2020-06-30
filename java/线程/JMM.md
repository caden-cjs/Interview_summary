# JMM

## 虚拟机栈(线程私有)

### Stack Frame(栈帧)

> 每一个运行的方法 都会被JVM封装成一个栈帧对象

### 程序计数器(Program Counter)

> 记录当前执行的行数

### 本地方法栈(Native Stack)

> 主要用于执行本地方法,非JAVA代码

### 堆(Heap)

> 大部分对象实例存放的地方,通过引用(reference)的方式调用堆中的数据

### 方法区(Method Area)

> 存储元信息,**~~永久代~~**(Permanent Genertion),从JDK1.8开始,~~废弃~~,使用**元空间(meta space)**

#### 运行时常量池

### 直接内存(Direct Meory)

> 与Java NIO密切相关,JVM通过堆上的DirectByteBuffer来操作直接内存

## 关于java对象的创建过程

### new关键字创建对象的3个步骤

1. 在堆内存中创建出对象的实例
2. 为对象的实例成员变量赋值
3. 将对象的引用返回

- 指针碰撞

> 前提是堆中的内存通过一个指针进行分割,一侧是已经被占用的空间,另外一侧是未被占用的空间

- 空闲列表

> 前提是堆内存空间已被使用与未被使用的空间是交织在一起的,这时,虚拟机就需要通过一个列表来记录哪些空间是可以使用的,哪些空间已经被使用的,接下来找出可以容纳下新创建对象的且未被使用的空间,在此空间存放该对象,同时还要修改列表上的记录。

### 对象在内存中的布局

1. 对象头
2. 实例数据（就是我们在一个类中声明的各项信息）
3. 对齐填充（可选）

引用访问对象的方式:

1. 使用句柄的方式。
2. 使用直接指针的方式。

## 堆内存Demo

```java
/**
 * @author: Caden
 * @Date: 2019-12-17 20:58
 * @desc: -Xms1m -Xmx5m -XX:+HeapDumpOnOutOfMemoryError
 * 设置堆内存大小是5M
 * 使用java visualVM查看报错原因
 **/
public class MyTest1 {
    public static void main(String[] args) {
        ArrayList<MyTest1> objects = new ArrayList<>();
        while (true) {
            objects.add(new MyTest1());
        }
    }
}
```

