# class 笔记

class 加载顺序：加载 -> 验证 -> 准备 -> 解析 -> 初始化

- 什么是类的加载

类的加载是指将类的.class 文件中的二进制数据读入到内存中，将其放在运行时数据区的方法区内，然后在堆区创建一个 java.lang.class 对象，用来封装类在方法区内的数据结构。class 对象，class 对象封装了类在方法区内的数据结构，并且提供了访问方法区内的数据结构的接口

- 在什么时候启动类加载

类的加载并不需要某个类被“首次主动使用”时再加载它，JVM 规范允许类加载器在预料某个类将要被预先使用时就预先加载它，如果在预先加载过程中遇到了.class 文件缺失或存在错误，类加载器必须在程序首次主动使用该类时才报告错误，如果这个类一直没有被程序主动使用，那么类加载器就不会报告错误。

**加载**

加载主要是将.class 文件（也可以是 zip 包）通过二进制流读入到 jvm 中，在加载阶段 JVM 需要完成 3 件事情。

通过 classloader 在 classpath 中获取 XXX.class 文件，将其以二进制流的方式读入内存。
将字节流代表的静态存储结构，转化为方法区的运行时存储结构。
在内存中生成一个该类的 java.lang.class 对象，作为方法区这个类的各种数据的访问入口。

**连接**

1. 验证：主要是确保加载进来的字节流符合 JVM 规范，验证阶段会有 4 个检验动作：

- 文件格式验证
  验证.class 文件字节流是否符合 class 文件的格式的规范，并且能够被当前版本的虚拟机处理。这里主要被魔数、主版本号、常量池等等的校验。

- 元数据验证
  验证是否符合 java 语言规范，主要是对字节码描述的信息进行语义分析，以保证其描述的信息符合 java 语言规范的要求，比如说验证这个类是不是有父类，类中的字段方法是不是和父类冲突等等。

- 字节码验证
  确保程序语义合法，符合逻辑，是整个验证过程最复杂的阶段。主要是通过数据流和控制流分析，确保程序语义是合法的、符合逻辑。在元数据验证那个阶段对数据类型做出验证后，这个阶段主要对类的方法做出分析，保证类的方法在运行时不会做出危害虚拟机安全的事。

- 符号引用验证
  确保下一步的解析能正常执行，它是验证的最后一个阶段，发生在虚拟机将符号引用转化为直接引用的时候。主要是对类自身以外的信息进行校验。目的是确保解析动作能够完成。

对整个类加载机制而言，验证阶段是一个很重要但是非必需的阶段，如果我们的代码能够确保没有问题，那么我们就没有必要去验证，毕竟验证需要花费一定的的时间。当然我们可以使用-Xverfity:none 来关闭大部分的验证。

2. 准备：准备是连接阶段的第二步，主要为静态变量在方法区分配内存，并设置默认初始值。

类变量会分配内存，但是实例变量不会，实例变量主要随着对象的实例化一块分配到 java 堆中。
这里的初始值指的是数据类型默认值，而不是代码中被显式赋予的值，但是如果同时被 static 和 final 修饰准备阶段后就已经赋值了，普通赋值位于其他阶段。

3. 解析： 解析是连接阶段的第三步，是虚拟机将常量池内的符合引用替换为直接引用的过程。

符号引用：以一组符号来描述所引用的目标，可以是任何形式的字面量，只要是能无歧义的定位到目标就好。
直接引用：直接引用可以是指向目标的指针、相对偏移量或者是一个能直接引用或间接定位到目标的句柄。和虚拟机实现的内存有关，不同的虚拟机直接引用一般不同。

解析动作主要针对类或接口、字段、类方法、接口方法、方法类型、方法句柄和调用点限定符 7 类符号引用进行。

**初始化**

这是类加载机制的最后一步，在这个阶段，java 代码才开始真正执行。我们知道，在准备阶段已经为类变量赋过一次值，在初始化阶段，程序员可以根据自己的需求来赋值了。
在初始化阶段，主要为类的静态变量赋予正确的初始值，JVM 负责对类进行初始化，主要对类变量进行初始化。在 Java 中对类变量进行初始值设定有两种方式：

声明变量是指定初始值。
使用静态代码块为类变量指定初始值。
JVM 初始化步骤：
假如这个类还没有被加载和连接，则程序先加载并连接该类。
假如该类的直接父类还没有被初始化，则先初始化其直接父类。
假如类中有初始化语句，则系统依次执行这些初始化语句。

类的初始化时机：
只有对类的主动使用时才会导致类的初始化，主动使用包括以下 6 种：

1. 创建类的实例，也就是 new 的时候。
2. 访问某个类或接口的静态变量，或者对静态变量赋值。
3. 调用类的静态方法。
4. 反射操作。
5. 初始化某个类，则其父类也会被初始化。
6. 虚拟机启动时被标明为启动类的类，直接用 java.exe 来运行某个类

## 不会触发 Class Load 场景

1. 当在通过子类中调用父类的静态属性字段时，不会触发子类的 Class Load 操作。只会触发被调用父类以及它之上的父类进行 Class Load 操作
2. 定义数据类型时不会触发数组定义类的 Class Load
3. 当调用一个类的静态常量属性字段时，也不会触发该所属类的 Class Load（JVM 的常量传递优化，在类的编译阶段已经将调用类的常量放入到本类中的常量池中）

## Class 静态初始化顺序

Class 类在加载的过程中，会有对其进行 Class 对象的初始化操作[cinit()]。执行的代码主要由 Class 类中的静属性和静态代码块组成。

1. 静态字段和静态代码块的执行顺序按照在源代码中的先后顺序来执行，对于同个属性字段以最后的顺序赋值对象为准。并且位于静态代码之后的静态属性在静态代码块中只能进行赋值操作而不能进行调用操作。
2. 普通类的初始化操作会先对其父类进行初始化操作，只有先执行完父类的初始化才能执行子类的初始化。所以对于同个类型字段和名称的属性字段最终的初始化值已子类的赋值为最终结果。
3. 接口与普通类不用，对于实现类的初始化并不会先初始化接口本身。之后在使用是才会被执行初始化操作。

## Class 类加载器

Class 的加载器（1.8 以前）分为三层结构模式，Bootstarp ClassLoad、Extension ClassLoad、Application ClassLoad。

1. Bootstarp ClassLoad 负责加载 JAVA_HOME/lib 目录下的 jar 包类
2. Extension ClassLoad 负责加载 JAVA_HOME/lib/ext 目录下的 jar 包类
3. Application ClassLoad 负责加载应用程序目录下的 jar 包类

并且 JVM 通过双亲委派的模式来进行类的加载，对于一个类在加载时先交由父类进行加载父类加载不到时才会有自己进行加载。这样可以保证系统级的类不会被第三方随意的修改。

通常时候类型的加载都是通过双亲委派的方式进行加载，但是在 java 的历史上有 3 次破坏双亲委派加载模式的情况。

1. 1.2 版本之前的虚拟机系统，因为双亲委派模式是在 1.2 的版本中被加入
2. JNDI(Java Naming and Directory Interface,Java 命名和目录接口)。JNDI 提供统一的客户端 API，通过不同的访问提供者接口 JNDI 服务供应接口(SPI)的实现，由管理者将 JNDI API 映射为特定的命名服务和目录系统，使得 Java 应用程序可以和这些命名服务和目录服务之间进行交互。由于实现类是由第三方实现不在 lib 目录下，所以 API 对应的 ClassLoad 无法找到对应的类。对于该场景 jvm 通过线上下文类加载器（Thread 里的 contextClassLoad）来进行加载，线程的类加载器在创建时会通过父线程的加载器赋值给自己，并且可以手动设置。如果最后为空时就默认为应用程序的类加载器。这样在加载 SPI 实现类时通过线程加载器来进行具体的实现类，实现了逆向加载
3. OSGI 模块热部署，每个模块中都有自己的一个类加载器。在对给的类进行加载时，对于特定的类不会交与父类进行加载而是分配给特定的类加载器进行加载

## Class 方法栈帧

方法栈帧是 jvm 运行的最小单位，每个线程的执行过程就是对方法栈的执行过程，线程中的最顶部栈帧称为当期栈帧。 方法栈帧由: 局部变量表、操作数栈、动态链接、结果返回等组成。

1. 局部变量表存储方法的参数和方法中分配的局部变量数据，对于非静态方法。变量表中的下标 0 处存储的为当期对象引用，通过 this 进行使用，一个方法的局部变量表大小在编译的过程中已经确定。所以分配的内存大小也已固定
2. 操作数栈，方法执行中需要分配的栈大小也是已经确定的。操作数栈用来进行方法执行过程中的运算
3. 动态链接，每个方法都包含一个指向运行时常量池中该栈帧所属方法都引用。包含这个引用的目的是为了能够支持当期方法的动态链接。当一个方法调用另一个方法时，就是通过常量池中的方法符号引用来进行表示。动态链接的作用就是将符号引用转化为调用方法的直接引用
4. 结果返回，jvm 的方法结果返回分为两种：正常方法调用 return 和异常情况

方法调用不代表方法的执行，方法调用的唯一目标在于确认需要执行的方法版本。java 的方法引用都是在编译或者运行期间才能知道被调用方法的真实地址，对于在编译期间就能确定的地址引用方法都是一些在运行期间不会变动并且一开始就能确定一个唯一版本的方法

对于在编译期间就能确定方法引用地址的方法调用称之为解析。java 中能够在编译期可知，运行期不可变的方法主要有静态方法和私有方法，静态方法主要是跟类相关，而私有方法只能在对象中调用。因此他们都不可能被继承或被其它版本方法重写覆盖，所以它们都适合在类加载阶段进行解析
