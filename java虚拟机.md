## Java 内存区域                                                                                                                                                                                                                              

参考链接：https://blog.csdn.net/qq_41701956/article/details/81664921

根据《Java 虚拟机规范(Java SE 7 版)》规定，Java虚拟机所管理的内存如下图所示



<img src="https://imgconvert.csdnimg.cn/aHR0cHM6Ly91c2VyLWdvbGQtY2RuLnhpdHUuaW8vMjAxNy85LzQvZGQzYjE1YjNkODgyNmZhZWFlMjA2Mzk3NmZiOTkyMTM_aW1hZ2VWaWV3Mi8wL3cvMTI4MC9oLzk2MC9mb3JtYXQvd2VicC9pZ25vcmUtZXJyb3IvMQ" alt="img" style="zoom: 50%;" />

### 

### Java虚拟机栈

线程私有，生命周期和线程一致。描述的是Java方法执行的内存模型：

每个方法在执行时，都会先创建一个栈帧（Stack Frame)用于存储**局部变量表、操作数栈、动态链接、方法出口**等信息。每一个方法从调用直至执行结束，就对应着一个栈帧从虚拟机栈中入栈到出栈的过程。

存储当前线程运行方法所需的数据，指令、返回地址。

#### 局部变量表

存放了编译期可知的各种基本数据类型（boolean、byte、char、short、int、float、double、long)、对象引用、返回地址类型（指向了一条字节码指令的地址）

StackOverflowError：线程请求的栈深度大于虚拟机所允许的深度。
OutOfMemoryError：如果虚拟机栈可以动态扩展，而扩展时无法申请到足够的内存。

#### 操作数栈

#### 动态链接

#### 方法出口



### 本地方法栈

区别于Java虚拟机栈的是：

Java虚拟机栈为虚拟机执行Java方法（也就是字节码）服务，

而本地方法栈则为虚拟机使用到的Native方法服务。也会有 StackOverflowError 和 OutOfMemoryError 异常。

### 程序计数器

内存空间小，线程私有。字节码解释器 工作就是通过改变这个计数器的值来选取下一条需要执行的字节码指令，分支、跳转、异常处理、线程恢复等基础功能都需要依赖计数器来完成。

如果线程正在执行一个Java方法，这个计数器记录的是正在执行的虚拟机字节码指令的地址；

如果正在执行的是Native方法，这个计数器的值为 `Undefined`。

此内存区域是唯一一个在Java 虚拟机规范中没有规定任何 OutOfMemoryError 情况的区域。

### 堆

对大多数应用来说，这块区域是JVM所管理的内存中最大的一块。

线程共享。主要存放对象实例和数组。

### 方法区

线程共享。存储已被虚拟机加载的类信息、常量、静态变量、即使编译器编译后的代码等数据



现在用一张图来介绍每个区域存储的内容。

### ![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91c2VyLWdvbGQtY2RuLnhpdHUuaW8vMjAxNy85LzQvZGE3N2Q5MDE0Njc4NmMwY2IzZTE3MGI5YzkzNzZhZTQ_aW1hZ2VWaWV3Mi8wL3cvMTI4MC9oLzk2MC9mb3JtYXQvd2VicC9pZ25vcmUtZXJyb3IvMQ)

#### 运行时常量池

属于方法区的一部分，用来存放编译期生成的各种字面量和符号引用。

编译期和运行期都可以将常量放入池中，内存有限，无法申请时抛出OutOfMemoryError。

## 虚拟机类加载机制

虚拟机把描述类的数据从 Class 文件加载到内存，并对数据进行校验、装换解析和初始化，最终形成可以被虚拟机直接使用的 Java 类型。

### 类加载时机

类的生命周期

<img src="https://imgconvert.csdnimg.cn/aHR0cHM6Ly91c2VyLWdvbGQtY2RuLnhpdHUuaW8vMjAxNy85LzQvMjdhYzg3ZjQzOTJmMGFiOTllNGM2NWMyM2NjNzE5NDU_aW1hZ2VWaWV3Mi8wL3cvMTI4MC9oLzk2MC9mb3JtYXQvd2VicC9pZ25vcmUtZXJyb3IvMQ" alt="img" style="zoom:50%;" />

以下几种情况必须对类进行初始化

- 1、遇到new 、getStatic、putStatic或invokeStatic 这四条字节码指令时，没初始化触发初始化。

  使用场景使用new 关键字实例化对象时、读取一个类的静态字段、调用一个类的静态方法时。

- 使用 java.lang.reflect 包中的方法对类进行反射调用的时候。

- 当初始化一个类的时候，如果发现其父类还没有进行初始化，则先需要触发其父类的初始化。

- 当虚拟机启动时，用户指定一个需要加载的主类（包含main()方法的那个类），虚拟机会先初始化这个主类。

  

### 类的加载过程

#### 加载

#### 验证

#### 准备

#### 解析

#### 初始化