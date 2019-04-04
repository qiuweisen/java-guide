## JVM类加载机制

Jvm类加载机制流程分为加载、验证、准备、初始化。下图是一个类从加载到使用及卸载的全部生命周期，图片来源于网络。

![image-20190325202215881](https://ws4.sinaimg.cn/large/006tKfTcgy1g1fb5j2ob9j30kw07iwfz.jpg)

### 加载

加载阶段是类加载过程的第一个阶段。在这个阶段，JVM 的主要目的是将字节码从各个位置（网络、磁盘等）转化为二进制字节流加载到内存中，接着会为这个类在 JVM 的方法区创建一个对应的 Class 对象，这个 Class 对象就是这个类各种数据的访问入口。

### 验证

这一阶段的主要目的是为了确保Class文件的字节流中包含的信息是否符合当前虚拟机的要求，并且不会危害虚拟机自身的安全。

### 准备

准备阶段是正式为类变量分配内存并设置类变量的初始值阶段。这里需要注意两个关键点，即内存分配的对象以及初始化的类型。

- 内存分配的对象

  Java中的变量有**类变量**和**类成员变量**两种类型，类变量指的是被static修饰的变量，而其它所有类型的变量都属于类成员变量。在准备阶段，JVM只会为类变量分配内存，而不会为类成员变量分配内存。类成员变量的内存分配需要等到初始化阶段才开始。

  例如以下代码在初始化阶段，只会为port属性分配内存，而不会为url属性分配内存。

  ```java
  public static int port = 8080;
  public String url = "http://www.baidu.com";
  ```

- 初始化的类型

  在准备阶段，JVM会为类变量分配内存并为其初始化。但是这里的初始化只是为变量赋予对应类型的零值，而不是用户代码里初始化的值。

  例如以下的代码在准备阶段只会为port赋值为0，而不是8080。

  ```java
  public static int port = 8080;
  ```

  但是如果一个变量是常量，在准备阶段属性便会被赋予用户希望的值。如以下代码在准备阶段port会被赋值为3。在编译阶段会为 port 生成 ConstantValue 属性，在准备阶段虚拟机会根据 ConstantValue 属性将 port
  赋值为 8080。

  ```java
  public static final int port = 8080;
  ```

  ### 解析

  解析阶段是指虚拟机将常量池中的符号引用替换成直接引用的过程。符号引用就是class文件中的CONSTANT_Class_info、CONSTANT_Field_info、CONSTANT_Method_info等类型的常量。

  ### 初始化

  初始化阶段时是类加载的最后一个阶段，用户定义的程序代码才真正开始执行。在这个阶段，JVM会根据语句执行顺序对类对象进行初始化。

  #### 类构造器

  初始化阶段是执行类构造器<client>方法的过程。<client>方法是由编译器自动收集类中的类变量的赋值操作和静态语句块中的语句合并而成的。虚拟机会保证子类<client>方法执行之前，父类的<client>方法已经执行完毕，如果一个类中没有静态变量赋值也没有静态语句块，那么编译器可以不为这个类生成<client>方法。

  注意以下几种情况不会执行类初始化：

  1. 通过子类引用父类的静态字段，只会触发父类的初始化，而不会触发子类的初始化；
  2. 定义对象数组，不会触发该类的初始化；
  3. 常量在编译期间会存入调用类的常量池中，本质上并没有直接引用定义常量的类，不会触发定义常量所在的类型；
  4. 通过类名获取Class对象，不会触发类的初始化；
  5. 通过Class.forName加载指定类时，如果指定参数initalize为false时，也不会触发类的初始化；
  6. 通过ClassLoader默认的loadClass方法也不会触发初始化操作；

  ### 类加载器

  JVM提供3种类加载器。

  #### 启动类加载器(Bootstrap ClassLoader)

  负责加载JAVA_HOME\lib目录中，或通过-Xbootclasspath参数指定路径中的，且被虚拟机认可（按文件名识别，如rt.jar）的类。

  #### 扩展类加载器(Extension ClassLoader)

  负责加载JAVA_HOME\lib\ext目录中的，或通过java.ext.dirs系统变量指定路径中的类库。

  #### 应用程序类加载器(Application ClassLoader)

  负责加载用户路径(classpath)上的类库。

  JVM通过双亲委派模型进行类的加载，我们也可以通过继承java.lang.ClassLoader实现自定义的类加载器。

  ![image-20190326201451543](https://ws1.sinaimg.cn/large/006tKfTcgy1g1ggk7m70fj30jg0dz3zr.jpg)

  

  ### 双亲委派

   当一个类收到了类加载请求，他首先不会尝试自己去加载这个类，而是把这个请求委派给父类去完成，每一个层次类加载器都是如此，因此所有的加载请求都应该传送到启动类加载其中。只有当父类加载器反馈自己无法完成这个请求的时候(在它的加载路径下没有找到所需加载的Class)，子类加载器才会尝试自己去加载。

  采用双亲委派的一个好处是比如加载位于rt.jar包中的java.lang.Object，不管是哪个加载器加载这个类，最终都是委托顶层的启动类加载器进行加载，这样就保证了使用不同的类加载器最终得到的都是同样一个Object对象。

  ### 案例分析

  ```java
  public class StaticTest
  {
     public static void main(String[] args)
     {
         staticFunction();
     }
  
     static StaticTest st = new StaticTest();
  
     static
     {
         System.out.println("1");
     }
  
     {
         System.out.println("2");
     }
  
     StaticTest()
     {
         System.out.println("3");
         System.out.println("a="+a+",b="+b);
     }
  
     public static void staticFunction(){
         System.out.println("4");
     }
  
     int a=110;
     static int b =112;
  }
  ```