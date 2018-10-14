---
layout: post
title: "类加载器"
date: 2018-10-13
tags: [JVM,类加载]
comments: true
share: true
---

## 概述
上一篇文章学习了类加载的过程，并且只是简单的说到了使用类加载器加载类，而这一篇文件则详细的介绍一下类加载过程。

**虚拟机设计团队把类加载阶段中的“通过一个类的全限定名来获取此类的二进制字节流”这个动作放到 Java 虚拟机外部去实现，以便让应用程序自己决定如何去获取所需要的类，这个动作的代码模块称为 “类加载器”。类加载器在类层次划分，OSGi、热部署、代码加密等领域大放异彩，称为 Java 技术体系中重要的基石。**

## 类与类加载器

**比较两个类是否相等，只有在两个类是由通过一个类加载器的前提下才有意义，否则，即使这两个类来源于同一个 Class 文件，被同一个虚拟机加载，只要加载它们的类加载器不同，拿着两个类就必定不相等。**这里所指的“相等”，包括代表类的 Class 对象的 equals() 方法、isAssignableFrom() 方法、isInstantce() 方法的返回结果，也包括使用 instanceOf 关机子做对象所属关系判定等情况。

```java
/**
* 不同的类加载器对 instanceOf 关键字运算的结果的影响
*/ 
public class ClassLoaderTest {

    public static void main(String[] args) throws ClassNotFoundException, IllegalAccessException, InstantiationException {
        ClassLoader myLoader = new ClassLoader() {
            @Override
            public Class<?> loadClass(String name) throws ClassNotFoundException {
                try {
                    String fn = name.substring(name.lastIndexOf(".") + 1) + ".class";
                    InputStream in = getClass().getResourceAsStream(fn);
                    if (in == null) {
                        return super.loadClass(name);
                    }
                    byte[] b = new byte[in.available()];
                    in.read(b);
                    return defineClass(name, b, 0, b.length);
                } catch (IOException e) {
                    throw new ClassNotFoundException("name");
                }
            }
        };
        Object obj  = myLoader.loadClass("com.xxx.ClassLoaderTest").newInstance();
        System.out.println(obj.getClass());
        System.out.println(obj instanceof ClassLoaderTest);
    }

}
```

运行结果：<br/>
class com.xxx.ClassLoaderTest<br/>
false

## 双亲委派模型

**从 Java 虚拟机的角度来件，只存在两种不同的类加载器：一种是启动类加载器（Bootstrap ClassLoader），这个类加载器使用 C++ 语言实现，是虚拟机自身的一部分；另一种就是所有其他的类加载器，这些类加载器都由 Java 语言实现，独立于虚拟机外部，并且全都继承自抽象类 java.lang.ClassLoader。**

绝大部分 Java 程序都会使用到以下3种系统提供的类加载器：

- 启动类加载器（Bootstrap ClassLoader），这个类加载器负责将存放在 `<JAVA_HOME>\lib` 目录中的，或者被 -Xbootclasspath 参数所指定的路径中的，并且被虚拟机识别的（仅按照文件名识别，如 rt.jar，名字不符合的类库即使放在 lib 目录下也不会被加载）类库加载到虚拟机内存中。启动类加载器无法被 Java 程序直接引用，如果需要把加载请求为派给引导类加载器，纳智捷使用 null 代替即可。

- 扩展类加载器（Extension ClassLoader），这个类加载器由 sun.misc.Launcher$ExtClassLoader 实现，它负责加载 &lt;JAVA_HOME>\lib\ext 目录中的，或者被 java.ext.dirs 系统变量所指定路径中的所有类，开发者可以直接使用扩展类加载器。

- 应用程序类加载器（Application ClassLoader），这个类加载器由 sum.msic.Launcher$ApplicationLoader 实现。由于这个类加载器是 ClassLoader 中的 getSystemClassLoader() 方法的返回值，所以一般也称它为系统类加载器。它负责加载用户类路径（ClassPath）上所指定的类库，开发者可以直接使用这个类加载器，如果应用程序中没有自定义过自己的类加载器，一般情况下这个就是程序中默认的类加载器。

<center>
<img src="/images/parents-delegation-model.jpeg" alt="类加载器双亲委派模型" width="50%&quot;, :height=&quot;50%">
</center>

上图中展示的是类加载器之间的这种层次关系，称之为双亲委派模型（Parents Delegation Model）。**双亲委派模型要求除了顶层的启动类加载器外，其余的类加载器都应当有自己的父类加载器。这里类加载器之间的父子关系一般不会以继承（Inheritance）的关系来实现，而是都使用组合（Composition）关系来复用类加载器的代码。**双亲委派模型在 JDK1.2 期间被引用并被广泛应用于之后几乎所有的 Java 程序中，但他并**不是一个强制性的约束模型**，而是 Java 设计者推荐给开发者的一个类加载器实现方式。

双亲委派模型的工作过程是：**如果一个类加载器收到了类加载的请求，他首先不会自己去尝试加载这个类，而是把这个请求委派给父类加载器去完成，每一个层次的类加载器都是如此，因此所有的加载请求最终都应该传送到顶层的启动类加载器中，只有当父加载器反馈自己无法完成这个加载请求（它的搜索范围没有找到所需的类）时，子加载器才会尝试自己去加载。使用双亲委派模型来组织类加载器之间的关系，有一个显而易见的好处就是 Java 类随着它的类加载器一起具备了一种带有优先级的层次关系。**例如类 java.lang.Object，它存放在 rt.jar 之中，无论哪一个类加载器要加载这个类，最终都是委派给处于模型最顶端的启动类加载器进行加载，因此 Object 类在程序的各个类加载器环境中都是同一个类。相反，如果没有使用双亲委派模型，有各个类加载器自行去加载的话，如果用户自己编一个称为 java.lang.Object 的类，并放在程序的 ClassPath 中，那系统中就会出现多个不同的 Object 类，Java 类型体系中最基础的行为也就无法保证，应用程序也将会变得一片混乱。

双亲委派模型对于保证 Java 程序的稳定运作很重要，但它的实现却很简单，实现双亲委派模型都集中在 java.lang.ClassLoader 的 loadClass() 方法之中。代码示例如下：

```java
    protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // 首先，检查 class 是否已经被加载过了。
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                try {
                    // 如果父类加载器不为空，尝试使用父类加载器加载，否在尝试使用启动类加载器加载。
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // 抛出 ClassNotFoundException 说明父类加载器无法完成请求
                }

                // 在父类加载器或者启动类加载器无法完成请求的情况下，再调用自身的 findClass 来进行加载请求。
                if (c == null) {
                    c = findClass(name);

                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
```

## 破坏双亲委派模型

双亲委派模型并不是一个强制性的约束模型，而是 Java 设计者推荐给开发者的类加载器实现方式。在 Java 的世界中大部分的类加载器都遵循这个模型，但也有例外，到目前为止，双亲委派模型共遭到过3次大规模的“被破坏”情况。

- JDK 1.2 发布之前。由于双亲委派模型是在 JDK 1.2 之后才引入的，而类加载器和抽象类 java.lang.ClassLoader 在 JDK 1.0 时代就存在了，为了向前兼容，**JDK 1.2 之后的代码 java.lang.ClassLoader 添加了一个新的 protected 方法 findClass()，并且不再提倡重写 loadClass() 方法**，因为我们前面看到 loadClass() 方面里面实现了双亲委派的逻辑，应当把自己的类加载逻辑写到 findClass() 方法中，在 loadClass() 方法的逻辑里如果父类加载失败，则会调用自己的 findClass() 方法来完成类加载，这样就可以保证新写出来的类加载器是符合双亲委派规则的。

- 第二次是因为模型自身的缺陷导致的，双亲委派模型很好的解决了各个类加载器的基础类的统一问题（越基础的类由越上层的类加载器进行加载），基础类之所以成为“基础”，是因为它们总是作为被用户代码调用的 API，如果基础类又要回调用户代码，那该怎么办？典型的例子便是 JNDI（Java Naming and Directory Interface），JNDI 现在已经是 Java 的标准服务，它的代码由启动类加载器加载（在 JDK 1.3 时放进去的 rt.jar），但 JNDI 的目的就是对资源进行集中管理和查找，它需要调用独立厂商实现并部署在应用程序的 ClassPath 下的 JNDI 接口提供者（SPI，Service Provider Interface）的代码，但启动类不可能 “认识这些代码”。Java 设计团队只要引入了一个不太优雅的设计：**线程上下文类加载器（Thread Context ClassLoader）。这个类加载器可以通过 java.lang.Thread 类的setContextClassLoader() 方法进行设置，如果创建线程时还未设置，将会从父线程中集成一个，如果在应用程序的全局范围内都没有设置过的话，那这个类加载器默认就是应用程序类加载器。**有了线程上线文类加载器，就可以做一些“舞弊”的事情了，JNDI服务使用这个线程上线文类加载器去加载所有的 SPI 代码，也就是父类加载器请求子类加载器去完成类加载的动作，这种行为实际上就是打通了双亲委派模型的层次结构来逆向使用类加载器。

- 第三次“被破坏”是由于用户对应用程序动态性的追求而导致的，这里的“动态性”指的是一些非常“热门”的名词：代码热替换（HotSwap）、模块化部署（Hot Deployment）等。Sun 公司所提出的 JSR-294、JSR-277 规范在与 JCP 组织的模块化规范之争中落败给 JSR-291（即 OSGi R4.2），目前 OSGi 已经成为业界“事实上”的 Java 模块化标准。OSGi 实现模块化热部署的关键则是它自定义的类加载器机制的时间。每一个程序模块（OSGi 中成为 Bundle）都有一个自己的类加载器，当需要更换一个 Bundle 时，就把 Bundle 连同加载器一起换掉以实现代码的热替代。在 OSGi 环境下，类加载器不再是双亲委派模型中的树状结构，而是进一步发展为更加复杂的网状结构，当收到类加载请求时，OSGi 将按照下面的顺序进行搜索：
    1. 将以 java.* 开头的类委派给父类加载器加载。
    2. 否则，将委派给列表名单中的类委派给父类加载器记载。
    3. 否则，将 Import 列表中的类委派给 Export 这个类的 Bundle 的类加载器加载。
    4. 否则，查找当前 Bundle 的 ClassPath，使用自己的类加载器加载。
    5. 否则，查找类是否在自己的 Fragment Bundle 中，如果在，则委派给 Fragment Bundle 的类加载器加载。
    6. 否则，查找 Dynamic Import 列表的 Bundle，委派给对应的 Bundle 的类加载器加载。
    7. 否在，类查找失败。
    
    上面的查找顺序只有开头两点依然符合双亲委派机制，其余的类查找都是在平级的类加载器中进行的。

“被破坏”并不带有贬义的感情色彩，只要足够意义和理由，突破已有的原则就可认为是一种创新。