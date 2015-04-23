title: Java系列笔记(1) - Java 类加载与初始化
categories: Java系列笔记
tags: 
- Java
- ClassLoader  
  
---

#类加载器

在了解Java的机制之前，需要先了解类在JVM（Java虚拟机）中是如何加载的，这对后面理解java其它机制将有重要作用。

每个类编译后产生一个Class对象，存储在.class文件中，JVM使用类加载器（Class Loader）来加载类的字节码文件（.class），类加载器实质上是一条类加载器链，一般的，我们只会用到一个原生的类加载器，它只加载Java API等可信类，通常只是在本地磁盘中加载，这些类一般就够我们使用了。如果我们需要从远程网络或数据库中下载.class字节码文件，那就需要我们来挂 载额外的类加载器。

一般来说，类加载器是按照树形的层次结构组织的，每个加载器都有一个父类加载器。另外，每个类加载器都支持代理模式，即可以自己完成Java类的加载工作，也可以代理给其它类加载器。

类加载器的加载顺序有两种，一种是父类优先策略，一种是是自己优先策略，父类优先策略是比较一般的情况（如JDK采用的就是这种方式），在这种策略 下，类在加载某个Java类之前，会尝试代理给其父类加载器，只有当父类加载器找不到时，才尝试自己去加载。自己优先的策略与父类优先相反，它会首先尝试 子经济加载，找不到的时候才要父类加载器去加载，这种在web容器（如tomcat）中比较常见。
<!--more-->

#动态加载

不管使用什么样的类加载器，类，都是在第一次被用到时，动态加载到JVM的。这句话有两层含义：

_1. Java程序在运行时并不一定被完整加载，只有当发现该类还没有加载时，才去本地或远程查找类的.class文件并验证和加载；_
_2. 当程序创建了第一个对类的静态成员的引用（如类的静态变量、静态方法、构造方法——构造方法也是静态的）时，才会加载该类。Java的这个特性叫做：动态加载。_

需要区分加载和初始化的区别，加载了一个类的.class文件，不意味着该Class对象被初始化，事实上，一个类的初始化包括3个步骤：

* 加载（Loading），由类加载器执行，查找字节码，并创建一个Class对象（只是创建）；
* 链接（Linking），验证字节码，为静态域分配存储空间（只是分配，并不初始化该存储空间），解析该类创建所需要的对其它类的应用；
* 初始化（Initialization），首先执行静态初始化块static{}，初始化静态变量，执行静态方法（如构造方法）。

#链接

Java在加载了类之后，需要进行链接的步骤，链接简单地说，就是将已经加载的java二进制代码组合到JVM运行状态中去。它包括3个步骤：

1. 验证（Verification），验证是保证二进制字节码在结构上的正确性，具体来说，工作包括检测类型正确性，接入属性正确性（public、private），检查final class 没有被继承，检查静态变量的正确性等。
2. 准备（Preparation），准备阶段主要是创建静态域，分配空间，给这些域设默认值，需要注意的是两点：一个是在准备阶段不会执行任何代码，仅仅是设置 默认值，二个是这些默认值是这样分配的，原生类型全部设为0，如：float:0f,int 0, long 0L, boolean:0（布尔类型也是0），其它引用类型为null。
3. 解析（Resolution），解析的过程就是对类中的接口、类、方法、变量的符号引用进行解析并定位，解析成直接引用（符号引用就是编码是用字符串表示某个 变量、接口的位置，直接引用就是根据符号引用翻译出来的地址），并保证这些类被正确的找到。解析的过程可能导致其它的类被加载。需要注意的是，根据不同的 解析策略，这一步不一定是必须的，有些解析策略在解析时递归的把所有引用解析，这是early resolution，要求所有引用都必须存在；还有一种策略是late resolution，这也是Oracle 的JDK所采取的策略，即在类只是被引用了，还没有被真正用到时，并不进行解析，只有当真正用到了，才去加载和解析这个类。


#初始化

注 意：在《Java编程思想》中，说static{}子句是在类第一次加载时执行且执行一次（可能是笔误或翻译错误，因为此书的例子显示static是在第 一次初始化时执行的），《Java深度历险》中说 static{}是在第一次实例化时执行且执行一次，这两种应该都是错误的，static{}是在第一次初始化时执行，且只执行一次；用下面的代码可以判 定出来：

    package myblog.classloader;
    
    /**
    * @project MyBlog
    * @create 2013年6月18日 下午7:00:45
    * @version 1.0.0
    * @author 张广
    */
    public class Toy {
        private String name;
    
        public static final int price = 10;
    
        static {
            System.out.println("Initializing");
        }
    
        Toy() {
            System.out.println("Building");
        }
    
        Toy(String name) {
            this.setName(name);
        }
    
        public static String playToy(String player) {
            String msg = buildMsg(player);
            System.out.println(msg);
            return msg;
        }
    
        private String buildMsg(String player) {
            String msg = player + " plays " + name;
            return msg;
        }
    }
    // 对上面的类，执行下面的代码：
    Class c = Class.forName("myblog.rtti.Toy");
    // c.newInstance();

可以看到，不实例化，只执行forName初始化时，仍然会执行static{}子句，但不执行构造方法，因此输出的只有Initializing，没有Building。
关于初始化，@阿春阿晓 在本文的评论中给出了很详细的场景，感谢@阿春阿晓：
根据java虚拟机规范，所有java虚拟机实现必须在每个类或接口被java程序首次主动使用时才初始化。
主动使用有以下6种：

1. 创建类的实例
2. 访问某个类或者接口的静态变量，或者对该静态变量赋值（如果访问静态编译时常量(即编译时可以确定值的常量)不会导致类的初始化）
3. 调用类的静态方法
4. 反射（Class.forName(xxx.xxx.xxx)）
5. 初始化一个类的子类（相当于对父类的主动使用），不过直接通过子类引用父类元素，不会引起子类的初始化（参见示例6）
6. Java虚拟机被标明为启动类的类（包含main方法的）

类与接口的初始化不同，如果一个类被初始化，则其父类或父接口也会被初始化，但如果一个接口初始化，则不会引起其父接口的初始化。

#示例

1，通过上面的讲解，将可以理解下面的程序（下面的程序部分来自于《Java编程思想》）：

    class Toy {
        static {
            System.out.println("Initializing");// 静态子句，只在类第一次被加载并初始化时执行一次，而且只执行一次    
        }
    
        Toy() {
            System.out.println("Building");// 构造方法，在每次声明新对象时加载   
        }
    }

对上面的程序段，第一次调用Class.forName("Toy")，将执行static子句；如果在之后执行new Toy()都只执行构造方法。

2，需要注意newInstance()方法

    Class cc = Class.forName("Toy");//获得类（注意，需要使用含包名的全限定名）
    Toy toy=(Toy)cc.newInstance(); //相当于new一个对象，但Gum类必须有默认构造方法（无参）

3，用类字面常量 .class和Class.forName都可以创建对类的应用，但是不同点在于，用Gum.class创建Class对象的应用时，不会自动初始化该Class对象（static子句不会执行）

    public class TestToy {
        public static void main(String[] args) {
            // try {
            // Class c = Class.forName("myblog.classloader.Toy");
            // } catch (ClassNotFoundException e) {
            // e.printStackTrace();
            // }
            Class c = Toy.class; // 不会输出任何值
        }
    }

使用Toy.class是在编译期执行的，因此在编译时必须已经有了Toy的.class文件，不然会编译失败，这与 Class.forName("myblog.classloader.Toy")不同，后者是运行时动态加载。
但是，如果该main方法是直接写在Toy类中，那么调用Toy.class，会引起初始化，并输出Initializing，原因并不是Toy.class引起的，而是该类中含有启动方法main，该方法会导致Toy的初始化。

 4，编译时常量。回到完整的类Toy，如果直接输出：System.out.println(Toy.price)，会发现static子句和构造方法都没有被执行，这是因为Toy中，常量price被static final限定，这样的常量叫做编译时常量，对于这种常量，不需要初始化就可以读取。
编译时常量必须满足3个条件：static的，final的，常量。
下面几种都不是编译时常量，对它们的应用，都会引起类的初始化：

        static int a;
        final  int b;
        static final  int c= ClassInitialization.rand.nextInt(100);
        static final  int d;
        static {
            d=5;
        }

5，static块的本质。注意下面的代码：

    class StaticBlock {
        static final int c = 3;
    
        static final int d;
    
        static int e = 5;
        static {
            d = 5;
            e = 10;
            System.out.println("Initializing");
        }
    
        StaticBlock() {
            System.out.println("Building");
        }
    }
    
    public class StaticBlockTest {
        public static void main(String[] args) {
            System.out.println(StaticBlock.c);
            System.out.println(StaticBlock.d);
            System.out.println(StaticBlock.e);
        }
    }

这段代码的输出是什么呢？Initialing在c、d、e之前输出，还是在之后？e输出的是5还是10？
执行一下，结果为：

    3
    Initializing
    5
    10

答案是3最先输出，Intializing随后输出，e输出的是10，为什么呢？
原因是这样的：输出c时，由于c是编译时常量，不会引起类初始化，因此直接输出，输出d时，d不是编译时常量，所以会引起初始化操作，即static块的执行，于是d被赋值为5，e被赋值为10，然后输出Initializing，之后输出d为5，e为10。
但e为什么是10呢？原来，JDK会自动为e的初始化创建一个static块（参考：http://www.java3z.com/cwbwebhome/article/article8/81101.html?id=2497），所以上面的代码等价于：

    class StaticBlock {
        static final int d;
    
        static int e;
    
        static {
            e = 5;
        }
    
        static {
            d = 5;
            e = 10;
            System.out.println("Initializing");
        }
    
        StaticBlock() {
            System.out.println("Building");
        }
    }  

可见，按顺序执行，e先被初始化为5，再被初始化为10，于是输出了10。
类似的，容易想到下面的代码：

    class StaticBlock {
        static {
            d = 5;
            e = 10;
            System.out.println("Initializing");
        }
    
        static final int d;
    
        static int e = 5;
    
        StaticBlock() {
            System.out.println("Building");
        }
    }

在这段代码中，对e的声明被放到static块后面，于是，e会先被初始化为10，再被初始化为5，所以这段代码中e会输出为5。

6，当访问一个Java类或接口的静态域时，只有真正声明这个域的类或接口才会被初始化（《Java深度历险》）
    
    /**
    * 例子来源于《Java深度历险》第二章
    * @author 张广
    *
    */
    class B {
        static int value = 100;
        static {
            System.out.println("Class B is initialized");// 输出       
        }
    }
    
    class A extends B {
        static {
            System.out.println("Class A is initialized"); // 不输出      
        }
    }
    
    public class SuperClassTest {
        public static void main(String[] args) {
            System.out.println(A.value);// 输出100        
        }
    }

在该例子中，虽然通过A来引用了value，但value是在父类B中声明的，所以只会初始化B，而不会引起A的初始化。

说明

笔者在开发过程中发现自己基础太薄弱，读书时除了系统学习了一下Java的基础语法和用法、一点简单的数据结构和设计模式之外，再无深入系统的学习，而工作中的学习也是东晃一枪西晃一枪，不够扎实和系统。想到一个学习方法：学到的东西能够系统的表达出来，才说明你学到手了；于是，笔者决定边学边写，将学到的东西以博客的形式表达出来。

本文档会因学习的深入或错误的订正而持续更新。

《Java系列笔记》是本人对Java的应用和学习过程中的笔记，按知识点分章节，写这一系列笔记的目的是学习，由于笔者是边学编写的，水平有限，文中必定有疏漏之处，欢迎斧正。

文中多有参考前辈的书籍和博客之处，原则上不会原文引用，而是加入自己的理解改造之，如果有引用，必会注明出处，如有疑问，请联系：daniel.zhguang@gmail.com

更新记录

* 2013年6月25日：发表；

* 2013年6月26日：加入@阿春阿晓的评论：引起初始化的6种场景；

* 2013年6月28日：加入链接和初始化的内容，加入示例6

本节参考资料

* JAVA编程思想，第14章

* Java深度历险

* Java中static块的本质：http://www.java3z.com/cwbwebhome/article/article8/81101.html?id=2497

* java类的装载(Loading)、链接(Linking)和初始化(Initialization)：http://blog.csdn.net/biaobiaoqi/article/details/6909141
