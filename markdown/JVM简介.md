[TOC]

## JVM概述

### JVM特点

* 一次编译，多处运行
* 自动内存管理
* 自动垃圾回收功能

### JVM的整体结构

![](https://tva1.sinaimg.cn/large/e6c9d24egy1gokj8v0yx8j20f40dm0wz.jpg)

### JVM的架构模型

#### 基于栈式架构

设计和实现简单，适用于资源受限的系统。不需要硬件支持，可移植性更好。

由于跨平台性，Java的指令都是根据栈设计的

特点：跨平台性，指令集小，指令多，执行性能比寄存器差

#### 基于寄存器架构

完全依赖于硬件，可移植性差

性能优秀，执行更高效



### 发展历程

#### Sun Classic VM

世界上第一款商用Java VM

只提供解释器（不包含后端编译器JIT，可以寻找热点代码，存入缓存，提高效率）

现在Hotspot内置了此VM

#### Sun Hotspot VM

Sun JDK，Open JDK默认VM

通过PC寄存器（程序计数器）找到最具有编译价值的代码，触发即时编译

通过编译器和解释器协同工作，在优化时间与执行性能上取得平衡



## 类加载器子系统（Class Loader）

### 作用

1. 负责从文件系统或网络中加载class文件，class文件在文件开头有特殊的文件标识**CAFE BABE**
2. ClassLoader只负责class文件的加载，至于它是否可以运行，由执行引擎决定

### 类加载过程

![](https://tva1.sinaimg.cn/large/e6c9d24egy1gokjps5i7ij20iz05y0x7.jpg)

#### 1.加载

在内存中生成一个代表这个类的java.lang.Class对象，作为方法区这个类的各种数据的访问入口

#### 2.链接

* 验证：确保class文件的字节流中包含的信息符合当前VM要求
* 准备：
  * 为类变量（static修饰）分配内存并设置该类变量的默认初始化值，即零值。在初始化时才赋值。
  * final static修饰的变量在编译时就会分配内存，准备阶段显式初始化
  * 不会为实例变量分配初始化，类变量会分配在方法区，实例变量会随着对象一起分配到Java堆中
* 解析：将常量池中的符号引用转换为直接引用的过程

#### 3.初始化

执行类的构造器方法clinit（）过程，javac编译器自动收集类中所有类变量的赋值动作和静态代码块



### 类加载器的分类

#### 引导类加载器（Bootstrap ClassLoader）

使用c/c++实现，嵌套在JVM内部，没有父加载器，只加载包名为java，javax，sun等开头的类

#### 自定义类加载器

指所有派生于抽象类ClassLoader的类加载器，由Java编写

![](https://tva1.sinaimg.cn/large/e6c9d24egy1gokk9p38c5j20nd0c3tck.jpg)

* 拓展类加载器：ExtensionClassLoader
* 系统类加载器：AppClassLoader

Example：String类使用**引导类加载器**进行加载，因为Java核心内库都是由BootstrapClassLoader负责加载

##### 为什么要自定义类加载器？

1. 隔离加载类
2. 修改类加载的方式
3. 拓展加载源
4. 防止源码泄露



### 获取ClassLoader的途径

* 获取当前类的ClassLoader：class.getClassLoader()
* 获取当前线程上下文的ClassLoader：Thread.currentThread().getContextClassLoader()
* 获取系统的ClassLoader：ClassLoader.getSystemClassLoader()
* 获取调用者的ClassLoader：DriverManager.getCallerClassLoader()



### 双亲委派机制

![](https://tva1.sinaimg.cn/large/e6c9d24egy1gokkk0gxwmj20ch0a0djy.jpg)

工作流程：

1. 如果一个类加载器收到了类加载请求，它并不会自己先去加载，而是把这个请求委托给父类的加载器去执行
2. 如果父类加载器还存在其父类加载器，则进一步向上委托，依次递归，请求最终到达顶层的引导类加载器
3. 如果父类加载器可以完成类加载任务，就成功返回；如无法完成，则子加载器才会尝试自己去加载

优势：

1. 避免类的重复加载
2. 保护程序安全，防止核心API被随意修改（沙箱安全机制）

Example：自定义类java.lang.String

