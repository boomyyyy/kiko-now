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

