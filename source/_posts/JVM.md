---
title: JVM内存结构、垃圾回收及参数配置 
date: 2020-01-10 10:01:17
tags: Java
---

------
### 什么是JVM？
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;JVM是Java Virtual Machine（Java虚拟机）的缩写，JVM屏蔽了与操作系统平台相关的信息，使得Java程序只需要生成在Java虚拟机上运行的目标代码（字节码），就可在多种平台上不加修改的运行，这也是Java能够“一次编译，到处运行的”原因。

- JDK（Java Development Kit，Java开发工具包）是用来编译、调试Java程序的开发工具包。包含java运行环境（JRE）和开发工具(编辑器，调试器，javadoc)。

- JRE（Java Runtime Environment， Java运行环境）是Java的运行环境，所有的程序都要在JRE下才能够运行。包括JVM和Java核心类库和支持文件。

- JVM（Java Virtual Machine， Java虚拟机）是JRE的一部分。JVM主要工作是解释自己的指令集（即字节码）并映射到本地的CPU指令集和OS的系统调用。包括类加载、jvm运行时数据区、执行引擎等。Java语言是跨平台运行的，不同的操作系统会有不同的JVM映射规则，使之与操作系统无关，完成跨平台性。

![cmd-markdown-logo](http://139.224.113.197/20200114135008.jpg)

概念范围：JDK > JRE > JVM 

JDK = JRE + Tools&Tool APIs

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;总结：使用JDK（调用JAVA API）开发JAVA程序后，通过JDK中的编译程序（javac）将Java程序编译为Java字节码，在JRE上运行这些字节码，JVM会解析并映射到真实操作系统的CPU指令集和OS的系统调用。

### Java体系
![cmd-markdown-logo](http://139.224.113.197/20200115104148.jpg)

- ClassLoader(类加载器)：用于装载.class文件
- Execution Engine(执行引擎)：用于执行字节码或者本地方法
- Runtime Data Areas(运行时方法区)：包括程序计数器、虚拟机栈、本地方法栈、方法区、堆。

#### JVM生命周期
Java实例对应一个独立运行的Java程序（进程级别）
1. 启动。启动一个Java程序，一个JVM实例就产生。拥有public static void main(String[] args)函数的class可以作为JVM实例运行的起点。
2. 运行。main()作为程序初始线程的起点，任何其他线程均可由该线程启动。JVM内部有两种线程：守护线程和非守护线程，main()属于非守护线程，守护线程通常由JVM使用，程序可以指定创建的线程为守护线程。
3. 消亡。当程序中的所有非守护线程都终止时，JVM才退出；若安全管理器允许，程序也可以使用Runtime类或者System.exit()来退出。

#### Java类的生命周期

类的生命周期包括加载、连接、初始化、使用和卸载，其中前三部分是类的加载过程，如下图：

![cmd-markdown-logo](http://139.224.113.197/20200114145443.jpg)
- 加载，查找并加载类的二进制字节码到JVM中，在JVM堆中创建一个java.lang.class类的对象。JVM通过类名、包名、类加载器ClassLoader完成类的加载。
- 连接，连接又包括验证、准备和解析。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 验证： 验证文件格式、元数据、字节码、符号引用验证<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 准备： 为类的静态属性分配内存，并初始化为默认值，这些内存将在方法区中进行分配<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 解析：把类中的符号引用转换为直接引用
- 初始化，执行类中的静态初始化代码、构造器代码以及静态属性的初始化。
- 使用，new出的对象在程序中使用
- 卸载，执行垃圾回收


#### Java类加载器
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;从Java虚拟机的角度来说，有两种不同的类加载器：启动类加载器和其它类加载器。启动类加载器在HotSpot虚拟机中使用C++语言实现，它是虚拟机的一部分；除了启动类加载器之外的其它类加载器都由Java语言实现，并且全部继承自java.lang.ClassLoader，它们是独立于虚拟机外部的。<br><br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;从程序开发人员的角度来说，类加载器分为四类：启动类加载器、扩展类加载器、应用类加载器和自定义类加载器。引导类加载器是扩展类加载器的父类，扩展类加载器是系统类加载器的父类。

1. **Bootstrap ClassLoader** 引导类加载器
主要负责加载jvm自身所需要的类，该加载器由C++实现，加载的是<JAVA_HOME>/lib下的class文件，或-Xbootclasspath参数指定的路径下且被虚拟机认可（按文件名识别，如rt.jar）的类的jar包加载到内存中。
2. **Extension ClassLoader** 扩展类加载器
扩展类加载器是指Sun公司(已被Oracle收购)实现的sun.misc.Launcher$ExtClassLoader类，由Java语言实现的，是Launcher的静态内部类，它负责加载<JAVA_HOME>/lib/ext目录下或者由系统变量-Djava.ext.dir指定位路径中的类库，开发者可以直接使用标准扩展类加载器。
3. **System  ClassLoader(App ClassLoader)** 系统类加载器
也称应用程序加载器是指 Sun公司实现的sun.misc.Launcher$AppClassLoader。它负责加载系统类路径java -classpath或-D java.class.path指定路径下的类库，也就是我们经常用到的classpath路径，开发者可以直接使用系统类加载器，一般情况下该类加载是程序中默认的类加载器，通过ClassLoade.getSystemClassLoader()方法可以获取到该类加载器。
4. **User-Defined ClassLoader** 自定义类加载器
User-Defined ClassLoader是Java开发人员继承ClassLoader抽象类实现的ClassLoader，基于自定义的ClassLoader可用于加载非ClassPath中的jar以及目录。

#### JVM类加载的顺序
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;当JVM加载一个类的时候，下层的加载器会将任务给上一层类加载器，上一层加载检查它的命名空间中是否已经加载这个类，如果已经加载，直接使用这个类。如果没有加载，继续往上委托直到顶部。检查之后，按照相反的顺序进行加载。如果Bootstrap加载器不到这个类，则往下委托，直到找到这个类，形成了一种双亲委派的模式。一个类可以被不同的类加载器加载。<br>

![cmd-markdown-logo](http://139.224.113.197/20200114165153.jpg)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;为什么要采用双亲委派的模式？<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;首先JVM判断一个类是否是同一个类有两个条件：一是看这个类的完整类名是否一样(包括包名)，二是看加载这个类的ClassLoader加载器是否是同一个。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;若不采用双亲委派机制，同一个类有可能被多个类加载器加载，这样该类会被识别为不同的类。双亲委派机制能保证多加载器加载某个类时，最终都是由一个加载器加载，确保最终加载结果都是同样的Object对象。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;双亲委派模型的好处：在于Java类随着它的类加载器一起具备了一种带有优先级的层次关系。例如类java.lang.Object，它存在在rt.jar中，无论哪一个类加载器要加载这个类，最终都是委派给处于模型最顶端的Bootstrap ClassLoader进行加载，因此Object类在程序的各种类加载器环境中都是同一个类。相反，如果没有双亲委派模型而是由各个类加载器自行加载的话，如果用户编写了一个java.lang.Object的同名类并放在ClassPath中，那系统中将会出现多个不同的Object类，程序将混乱。因此，如果开发者尝试编写一个与rt.jar类库中重名的Java类，
可以正常编译，但是永远无法被加载运行。

#### JVM执行引擎
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;类加载器将字节码载入内存后，执行引擎以java字节码为单元，读取java字节码。java字节码机器读不懂，必须将字节码转化为平台相关的机器码。这个过程就是由执行引擎完成的。
在执行方法时JVM提供了四种指令来执行：

- invokestatic:调用类的static方法。
- invokevirtual:调用对象实例的方法。
- invokeinterface：将属性定义为接口来进行调用。
- invokespecial：JVM对于初始化对象（Java构造器的方法为：）以及调用对象实例的私有方法时。

### Java运行时数据区

![cmd-markdown-logo](http://139.224.113.197/20200115142803.jpg)

#### 1、程序计数器（Program Counter Register）
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;当前线程所执行的字节码的行号指示器， 线程私有。字节码解释器工作的时候就是通过改变这个计数值来选取下一条要执行的字节码指令。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;由于Java虚拟机的多线程是通过线程轮流切换并分配处理器执行时间的方式来实现的，在任何一个确定的时刻，一个处理器（对于多核处理器来说是一个内核）只会执行一条线程中的指令。因此，为了线程切换后能恢复到正确的执行位置，每条线程都需要有一个独立的程序计数器，各条线程之间的计数器互不影响，独立存储，我们称这类内存区域为“线程私有”的内存。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;此内存区域是唯一一个在Java虚拟机规范中没有规定任何OutOfMemoryError情况的区域。

#### 2、虚拟机栈（java方法栈）
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;与程序计数器一样，Java虚拟机栈（Java Virtual Machine Stacks）也是线程私有的，它的生命周期与线程相同。虚拟机栈描述的是Java方法执行的内存模型：每个方法被执行的时候都会同时创建一个栈帧（Stack Frame）用于存储局部变量表、操作栈、动态链接、方法出口等信息。每一个方法被调用直至执行完成的过程，就对应着一个栈帧在虚拟机栈中从入栈到出栈的过程。<br>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;局部变量表存放了编译期可知的各种基本数据类型（boolean、byte、char、short、int、float、long、double）、对象引用（reference类型，它不等同于对象本身，根据不同的虚拟机实现，它可能是一个指向对象起始地址的引用指针，也可能指向一个代表对象的句柄或者其他与此对象相关的位置）和returnAddress类型（指向了一条字节码指令的地址）。<br>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;其中64位长度的long和double类型的数据会占用2个局部变量空间（Slot），其余的数据类型只占用1个。局部变量表所需的内存空间在编译期间完成分配，当进入一个方法时，这个方法需要在帧中分配多大的局部变量空间是完全确定的，在方法运行期间不会改变局部变量表的大小。<br>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在Java虚拟机规范中，对这个区域规定了两种异常状况：如果线程请求的栈深度大于虚拟机所允许的深度，将抛出StackOverflowError异常；如果虚拟机栈可以动态扩展（当前大部分的Java虚拟机都可动态扩展，只不过Java虚拟机规范中也允许固定长度的虚拟机栈），当扩展时无法申请到足够的内存时会抛出OutOfMemoryError异常。

#### 3、本地方法栈（Native Method Stacks）
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;本地方法栈（Native Method Stacks）与虚拟机栈所发挥的作用是非常相似的，不同于虚拟机栈为虚拟机执行Java方法（也就是字节码）服务，而本地方法栈则是为虚拟机使用到的Native方法服务。与虚拟机栈一样，本地方法栈区域也会抛出StackOverflowError和OutOfMemoryError异常。

#### 4、方法区
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;方法区（Method Area）与Java堆一样，是各个线程共享的内存区域，它用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。虽然Java虚拟机规范把方法区描述为堆的一个逻辑部分，但是它却有一个别名叫做Non-Heap（非堆），目的应该是与Java堆区分开来。<br>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;对于习惯在HotSpot虚拟机上开发和部署程序的开发者来说，很多人愿意把方法区称为“永久代”（Permanent Generation），本质上两者并不等价，仅仅是因为HotSpot虚拟机的设计团队选择把GC分代收集扩展至方法区，或者说使用永久代来实现方法区而已。<br>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Java虚拟机规范对这个区域的限制非常宽松，除了和Java堆一样不需要连续的内存和可以选择固定大小或者可扩展外，还可以选择不实现垃圾收集。相对而言，垃圾收集行为在这个区域是比较少出现的，但并非数据进入了方法区就如永久代的名字一样“永久”存在了。这个区域的内存回收目标主要是针对常量池的回收和对类型的卸载，一般来说这个区域的回收“成绩”比较难以令人满意，尤其是类型的卸载，条件相当苛刻，但是这部分区域的回收确实是有必要的。<br>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;根据Java虚拟机规范的规定，当方法区无法满足内存分配需求时，将抛出OutOfMemoryError异常。

#### 5、堆
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Java堆（Java Heap）是Java虚拟机所管理的内存中最大的一块。Java堆是被所有线程共享的一块内存区域，在虚拟机启动时创建。此内存区域的唯一目的就是存放对象实例，几乎所有的对象实例都在这里分配内存。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Java堆是垃圾收集器管理的主要区域，因此很多时候也被称做“GC堆”。如果从内存回收的角度看，由于现在收集器基本都是采用的分代收集算法，所以Java堆中还可以细分为：新生代和老年代；再细致一点的有Eden空间、From Survivor空间、To Survivor空间等。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;根据Java虚拟机规范的规定，Java堆可以处于物理上不连续的内存空间中，只要逻辑上是连续的即可。在实现时，既可以实现成固定大小的，也可以是可扩展的，不过当前主流的虚拟机都是按照可扩展来实现的（通过-Xmx和-Xms控制）。
如果在堆中没有内存完成实例分配，并且堆也无法再扩展时，将会抛出OutOfMemoryError异常。

#### 对象创建时内存分配过程
1.  JVM看到new指令，判断指令参数是否能在常量池中定位到这个类的符号引用（可以认为我们的类名），检查这个符号引用代表的类是否已被加载、解析和初始化过，没有的话需要进行class加载。
2. 为对象分配内存。一种办法“指针碰撞”、一种办法“空闲列表”，最终常用的办法“本地线程缓冲分配(TLAB)”
3. 除对象头以外的对象内存空间都初始化为零，保证对象的实例字段再Java代码中不赋初始值就能直接使用。
4. JVM对对象做一些设置。设置存放在对象头，对象头里主要包含一些实例是哪个类的实例、类的元信息数据、hashcode、GC分代年龄、锁等。
5. 执行init方法,此刻对于JVM一个对象已经创建完毕，接下来会按照程序做一些初始化动作。


#### 内存分配规则

- 对象优先分配在Eden区，如果Eden区没有足够的空间时，虚拟机执行一次Minor GC。

- 大对象直接进入老年代（大对象是指需要大量连续内存空间的对象）。这样做的目的是避免在Eden区和两个Survivor区之间发生大量的内存拷贝（新生代采用复制算法收集内存）。

- 长期存活的对象进入老年代。虚拟机为每个对象定义了一个年龄计数器，如果对象经过了1次Minor GC那么对象会进入Survivor区，之后每经过一次Minor GC那么对象的年龄加1，知道达到阀值对象进入老年区。

- 动态判断对象的年龄。如果Survivor区中相同年龄的所有对象大小的总和大于Survivor空间的一半，年龄大于或等于该年龄的对象可以直接进入老年代。

- 空间分配担保。每次进行Minor GC时，JVM会计算Survivor区移至老年区的对象的平均大小，如果这个值大于老年区的剩余值大小则进行一次Full GC，如果小于检查HandlePromotionFailure设置，如果true则只进行Monitor GC,如果false则进行Full GC。


### JVM垃圾回收

#### Java内存模型
![cmd-markdown-logo](http://139.224.113.197/20200115153338.jpg)

JVM内存结构主要有三大块：堆内存、方法区和栈。

- 堆内存是JVM中最大的一块，由年轻代和老年代组成，而年轻代内存又被分成三部分，Eden空间、From Survivor空间、To Survivor空间，默认情况下年轻代的这3种空间年轻代按照8:1:1的比例来分配

- 栈又分为java虚拟机栈和本地方法栈主要用于方法的执行

- 方法区存储类信息、常量、静态变量等数据，是线程共享的区域，为与Java堆区分，方法区还有一个别名Non-Heap(非堆)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;JDK1.8开始，用MetaSpace（元空间）替代了方法区，元空间不在虚拟机里面，而是直接使用本地内存。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;为什么要用元空间代替永久代？<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;(1) 类以及方法的信息比较难确定其大小，因此对于永久代的指定比较困难，太小容易导致永久代溢出，太大容易导致老年代溢出。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;(2) 永久代会给GC带来不需要的复杂度，并且回收效率偏低。<br>

#### java对象的定位

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;程序每执行一个方法，JVM都会为此创建一个栈帧，我们的一条调用链路，有一个一个的栈帧组成，最后形成一个线程栈，当栈的长度过长或者超过内存大小就好报栈溢出的异常。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;基本数据类型，String这些的指针指向的是常量池。而实例对象也就统称为reference类型，指向的是一个对象的引用，访问堆中的对象具体位置有两种方式，通过句柄池进行句柄访问，另一种直接通过指针。<br>

##### 句柄方式
Java堆中会划分一块内存用来作为句柄池，reference存储的就是句柄地址，而句柄包含了实例数据和类型数据各自的具体地址。具有稳定的句柄地址的优势，当对象移动，只会更改句柄中的实例数据指针。
![cmd-markdown-logo](http://139.224.113.197/20200114180303.jpg)
##### 指针方式
reference存储的直接就是对象地址，速度快，节省了一次指针定位的开销。Java中对象的访问特别多，所以在Java中该方式使用的最多。
![cmd-markdown-logo](http://139.224.113.197/20200114180304.jpg)

    
#### GC对象的判定方法
如何判断一个对象是否存活？
1. 引用计数法

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;引用计数法就是给每一个对象设置一个引用计数器，每当有一个地方引用这个对象时，就将计数器加1，引用失效时，计数器就减1。当一个对象的引用计数器为0时，说明此对象没有被引用，也就是“死对象”,将会被垃圾回收。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;引用计数法有一个缺陷就是无法解决循环引用问题，也就是说当对象 A 引用对象 B，对象 B 又引用者对象 A，那么此时 A、B 对象的引用计数器都不为零，也就造成无法完成垃圾回收，所以主流的虚拟机都没有采用这种算法。

2. 可到达性算法

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;从GC Roots开始向下搜索，搜索所走过的路径称为引用链。当一个对象到GC Roots没有任何引用链相连时，则证明此对象是不可用的，不可达对象即为已死对象。
![cmd-markdown-logo](http://139.224.113.197/20200115160722.jpg)<br>

在Java中，可以作为GC Roots的对象有以下几种
- 虚拟机栈中引用的对象
- 本地方法栈引用的对象
- 方法区类静态属性引用的对象
- 方法区常量池引用的对象

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;虽然这些算法可以判定一个对象是否能被回收，但是当满足上述条件时，一个对象比不一定会被回收。当一个对象不可达 GC Root时，这个对象并不会立马被回收，而是出于一个死缓的阶段，若要被真正的回收需要经历两次标记。

#### 回收算法
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;GC最基础的算法有三种：标记 -清除算法、复制算法、标记-压缩算法，我们常用的垃圾回收器一般都采用分代收集算法。

- 标记 -清除算法，“标记-清除”（Mark-Sweep）算法，如它的名字一样，算法分为“标记”和“清除”两个阶段：首先标记出所有需要回收的对象，在标记完成后统一回收掉所有被标记的对象。 缺点：会产生内存碎片。
- 复制算法，“复制”（Copying）的收集算法，它将可用内存按容量划分为大小相等的两块，每次只使用其中的一块。当这一块的内存用完了，就将还存活着的对象复制到另外一块上面，然后再把已使用过的内存空间一次清理掉。缺点：可使用内存只有一半，代价太大。
- 标记-压缩算法，标记过程仍然与“标记-清除”算法一样，但后续步骤不是直接对可回收对象进行清理，而是让所有存活的对象都向一端移动，然后直接清理掉端边界以外的内存。
- 分代收集算法，“分代收集”（Generational Collection）算法，把Java堆分为新生代和老年代，这样就可以根据各个年代的特点采用最适当的收集算法。

#### 收集器
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;JVM提供了很多个收集器，目前主要有Serial，Serial Old，ParNew, Paralle Scavenge，Paralle Scavenge Old，CMS，G1。当前用的最多的是 CMS和G1。 

- Serial收集器，串行收集器是最古老，最稳定以及效率高的收集器，只使用一个线程去回收，进行GC的时候会把所有工作线程停掉，可能会产生较长的停顿。Serial Old是专门回收老年代的。
- ParNew收集器，ParNew收集器其实就是Serial收集器的多线程并发版本。

- Parallel Scavenge收集器，Parallel Scavenge收集器类似ParNew收集器，Parallel收集器更关注系统的吞吐量。采用复制算法，追求吞吐量，可以高效的利用CPU时间，尽快完成程序的运算任务，适合在后台运算而不需要太多交互的程序。

- Parallel Scavenge Old 收集器，Parallel Old是Parallel Scavenge收集器的老年代版本，使用多线程和“标记－整理”算法。

- CMS收集器，CMS（Concurrent Mark Sweep）收集器是一种以获取最短回收停顿时间为目标的收集器。

- G1收集器，G1 (Garbage-First)是一款面向服务器的垃圾收集器,主要针对配备多颗处理器及大容量内存的机器. 以极高概率满足GC停顿时间要求的同时,还具备高吞吐量性能特征。

#### GC和对象引用

- 强引用：默认情况下，对象采用的均为强引用（这个对象的实例没有被其他对象引用时， GC时才会被回收）

- 软引用：软引用是Java中提供的一种比较适合于缓存场景的引用（只有内存不够的情况下才会被GC）

- 弱引用：在GC时一定会被GC回收

- 虚引用：虚引用只是用来得知对象是否被GC。

#### minor gc和full gc
- Minor GC：从新生代回收内存，关键是Eden区内存不足，造成不足的原因是Java对象大部分是朝生夕死(java局部对象)，而死掉的对象就需要在合适的时机被JVM回收。
- Major GC：从老年代回收内存，一般比Minor GC慢10倍以上。

- Full GC：对整个堆来说的，出现Full GC通常伴随至少一次Minor GC，但非绝对。Full GC被触发的时候：老年代内存不足；持久代内存不足；统计得到的Minor GC晋升到老年代平均大小大于老年代剩余空间。

### JVM调优

#### 调优工具

常用调优工具分为两类,jdk自带监控工具：jconsole和jvisualvm，第三方有：MAT(Memory Analyzer Tool)、GChisto。

- jconsole（Java Monitoring and Management Console）是从java5开始，在JDK中自带的java监控和管理控制台，用于对JVM中内存，线程和类等的监控

- jvisualvm，jdk自带全能工具，可以分析内存快照、线程快照；监控内存变化、GC变化等。

- MAT（Memory Analyzer Tool）是一个基于Eclipse的内存分析工具，是一个快速、功能丰富的Java heap分析工具，它可以帮助我们查找内存泄漏和减少内存消耗。

- GChisto，一款专业分析gc日志的工具。

#### 调优命令
Sun JDK监控和故障处理命令有jps jstat jmap jhat jstack jinfo。
- jps（JVM Process Status Tool）显示指定系统内所有的HotSpot虚拟机进程。

- jstat（JVM statistics Monitoring）是用于监视虚拟机运行时状态信息的命令，它可以显示出虚拟机进程中的类装载、内存、垃圾收集、JIT编译等运行数据。

- jmap（JVM Memory Map）命令用于生成heap dump文件

- jhat（JVM Heap Analysis Tool）命令是与jmap搭配使用，用来分析jmap生成的dump，jhat内置了一个微型的HTTP/HTML服务器，生成dump的分析结果后，可以在浏览器中查看

- jstack，用于生成java虚拟机当前时刻的线程快照。

- jinfo（JVM Configuration info） 这个命令作用是实时查看和调整虚拟机运行参数。


#### JVM参数调优

##### trace跟踪参数

参数 | 作用 | 示例 
---|---|---
-XX:+PrintGC/-verbose:gc| 打印GC的简要信息 |  
-XX:+PrintGCDetails| 打印GC的详细信息 |  
-XX:+PrintGCTimeStamps| 打印GC发生的时间戳 |  
-X:loggc:log/gc.log| 指定GC的log位置 | -Xloggc:$logs.basedir/gateway/gc.log 
-XX:+PrintHeapAtGC| 每次GC时都打印出堆的信息 |  
XX:+HeapDumpOnOutMemoryError| 当JVM发生OOM时，自动生成dump文件 |  
-XX:HeapDumpPath| 生成dump文件的路径 | -XX:HeapDumpPath=$logs.basedir/gateway/heapdump.hprof


![cmd-markdown-logo](http://139.224.113.197/20200115181602.jpg)<br>

##### 永久代（方法区）相关设置
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Java7及以前版本的Hotspot中方法区位于永久代中。同时，永久代和堆是相互隔离的，但它们使用的物理内存是连续的。永久代的垃圾收集是和老年代捆绑在一起的，因此无论谁满了，都会触发永久代和老年代的垃圾收集。
参数 | 作用
---|---
-XX:PermSize=64MB | 最小尺寸，初始分配
-XX:MaxPermSize=256MB | 最大允许分配尺寸，按需分配。-server选项下默64M, -client选项下默认为32M
-XX:+CMSClassUnloadingEnabled |
-XX:+CMSPermGenSweepingEnabled | 设置垃圾不回收

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在Java7中永久代中存储的部分数据已经开始转移到Java Heap或Native Memory中了。在Java8中Hotspot取消了永久代,换成了元空间。永久代的参数-XX:PermSize和-XX：MaxPermSize被MetaspaceSize/MaxMetaspaceSize取代。

参数 | 作用
---|---
-XX:MetaspaceSize | 元空间初始大小
-XX:MaxMetaspaceSize | 元空间最大内存，默认没有限制
-XX：MinMetaspaceFreeRatio | 在GC之后，最小的Metaspace剩余空间容量的百分比，减少为class metadata分配空间导致的垃圾收集
-XX:MaxMetaspaceFreeRatio | 在GC之后，最大的Metaspace剩余空间容量的百分比，减少为class metadata释放空间导致的垃圾收集

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;-XX:MetaspaceSize 达到该值就会触发垃圾收集进行类型卸载，同时GC会对该值进行调整：如果释放了大量的空间，就适当降低该值；如果释放了很少的空间，那么在不超过MaxMetaspaceSize时，适当提高该值。

![cmd-markdown-logo](http://139.224.113.197/20200115181603.jpg)<br>

##### 堆参数设置

参数 | 作用
---|---
-Xms | 设置堆的最小空间大小，默认物理内存的1/64
-Xmx | 设置堆的最大空间大小，默认物理内存的1/4
-Xmn | 设置年轻代大小，默认整个堆的3/8
-XX:NewSize | 设置新生代最小空间大小
-XX:MaxNewSize | 设置新生代最大空间大小
-XX:NewRatio | 设置年老代和年轻代的比例，比如3，则年轻代占年轻代和年老带和的1/4
-XX:SurvivorRatio |  设置年轻代中Eden和两个Survivor的比例，比如6，则一个Survivor占年轻代的1/8 
-Xss | 设置每个线程的堆栈大小
-XX:+UseParallelGC | 选择垃圾收集器为并行收集器。此配置仅对年轻代有效。即上述配置下,年轻代使用并发收集,而年老代仍旧使用串行收集。
-XX:ParallelGCThreads=20 | 配置并行收集器的线程数,即:同时多少个线程一起进行垃圾回收。此值最好配置与处理器数目相等
<br>

老年代空间的大小：没有直接设置老年代的参数，但是可以设置堆空间大小和新生代空间大小两个参数来间接控制。

```
老年代空间大小 = 堆空间大小 - 年轻代大空间大小
```
<br>

参考: <br>
[JVM原理分析，看了都说好](https://mp.weixin.qq.com/s/HCoMzvR-rRgj7LJ12MnlcA/) <br>
[JVM 内存区域与GC](https://mp.weixin.qq.com/s/bnYkfMfXrpDxOEFQ7QpXgw/) <br>
[JVM内存结构解析，看了都说好](https://mp.weixin.qq.com/s/BxobzO55ARK04FZfffBW9A/) <br>
[JVM内存结构、内存模型 、对象模型那些事](https://mp.weixin.qq.com/s/B2OYctnr8vPndhGgX2QD4A/) <br>
[Jvm面试题总结及答案分享](https://mp.weixin.qq.com/s/5vbB59kqmMHaMsd6XUcm8A/) <br>
[阿里等大厂21道Java虚拟机高频题及性能优化详细解析](https://mp.weixin.qq.com/s/R9r_wqcYMaPB9t4Ce0zOiQ/)<br>
[深入详解JVM内存模型与JVM参数详细配置](https://mp.weixin.qq.com/s/oOgdX7rlxCjr-8Kr74QkYg/) <br>