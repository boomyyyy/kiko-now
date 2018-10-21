---
layout: post
title: "虚拟机字节码执行引擎2之方法调用"
date: 2018-10-21
tags: [JVM,类加载]
comments: true
share: true
---


## 概述
方法调用不等同于方法执行，方法调用阶段唯一的任务就是确定被调用方法的版本（即调用那一个方法），暂时还不涉及方法内部的具体运行过程。

## 解析
所有方法调用中的目标方法在 Class 文件里面都有一个常量池中的符号引用，在类加载解析阶段，会将其中一部分符号引用转化为直接引用，这种解析能成立的前提是：**方法在程序真正运行之前就有一个可确定的调用版本，并且这个方法的调用版本是运行期不可变的。话句话说，调动目标在程序代码写好、编译器进行编译时就必须确定下来。这类方法的调用称为解析（Resolution）。**

**在 Java 语言符合“编译期可知，运行期不可变”这个要求的方法，主要包括静态方法和私有方法两大类，前者与类型直接关联，后者在外部不可访问**，这两种方法各自的特点决定了他们都不可能通过继承或别的方式重写其他版本，因此它们都适合在类加载阶段进行解析。

与之相对应的是，在 Java 虚拟机里面提供了 5 条方法调用字节码指令，分别如下。

- invokestatic：调用静态方法
- invokespecial：调用实例构造器&lt;init>方法、私有方法和父类方法。
- invokevirtual：调用所有的虚方法。
- invokeinterface：调用接口方法，会在运行时在确定一个实现此接口的对象。
- invokedynamic：现在运行时动态解出调用点限定符所引用的方法，然后再执行该方法，在此之前的 4 条调用指令，分派逻辑是固化在 Java 虚拟机内部的，而 invokedynamic 指令的分派逻辑是由用户所设定的引导方法决定的。

**只要能被 invokestatic 和 invokespecial 指令调用的方法，都可以在解析阶段确定唯一的调用版本，符合这个条件的有静态方法、私有方法、实例构造器、父类方法 4 类，它们在类加载的时候就会把符号引用解析为该方法的直接引用。**这些方法可以称为非虚方法，与之相反，其他方法称为虚方法（除去 final 方法）。

```java
/**
* 方法静态解析演示
*/
public class StaticResolution {

    public static void sayHello() {
        System.out.println("hello world");
    }

    public static void main(String[] args) {
        StaticResolution.sayHello();
    }

}
```

使用 javap 命令查看这段程序的字节码，会发现的确是通过 invokestatic 命令来调用 sayHello() 方法的。

```
public static void main(java.lang.String[]);
    Code:
     Stack=0, Locals=1, Args_size=1
     0: invokestatic #31;           // Method sayHello:()V
     3: return
    LineNumberTable:
     line 15: 0
     line 16: 3
```

**Java 中的非虚方法除了使用 invokestatic、invokespecial 调用的方法之外还有一种，就是被 final 修饰的方法。虽然 final 方式是使用 invokevirtual 指令来调用的，但是由于它无法被覆盖，没有其他版本，所以也无需对方法接收者进行多态选择，又或者说多态的结果肯定是惟一的。在 Java 语言规范中明确说明了 final 方法是一种非虚方法。**

## 分派
Java 是一门面向对象的程序语言，因为 Java **具备面向对象的 3 个基本特征：继承、封装和多态**。本节讲解的分派调用过程会揭示多态性特征的一些最基本的体现，如“重载”和“重写”在 Java 虚拟机之中是如何实现的。

**分派（Dispatch）调用可能是静态的也可能是动态的（从确定方法调用版本的不同阶段角度来讲，编译时期、运行时期），根据分派依据的宗量数可分为单分派和多分派（从确定方法调用版本的选择依据来讲，单存依靠方法接收者还是需要同时依靠方法接收者和参数）。这两类分派方式的两两组合就构成了静态单分派、静态多分派、动态单分派、动态多分派4中分派情况。**这里仅仅是一个概述，具体请看下面几节内容。

### 静态分派

***静态分派是用于实现重载的！**

```java
/**
* 方法静态分派演示
*/
public class StaticDispatch {

    static abstract class Human {

    }

    static abstract class Man extends Human {

    }

    static abstract class Woman extends Human {

    }

    public void sayHello(Human human) {
        System.out.println("hello,guy!");
    }

    public void sayHello(Man man) {
        System.out.println("hello, gentleman!");
    }

    pubnlic void sayHello(Woman woman) {
        System.out.println("hello,lady!");
    } 

    public static void main(String[] args) {
        Human man = new Man();
        Human woman = new Woman();

        StatisDispatch sr = new StaticDispatch();
        sr.sayHello(man);
        sr.sayHello(woman);
    }

}
```

运行结果：

```
hello,guy!
hello,guy!
```

**虚拟机（准确的说是编译器）在重载时是通过参数的静态类型而不是实际类型作为判定依据的。并且静态类型是编译器可知的，因此在编译阶段，javac 编译器会根据参数的静态类型决定使用哪个重载版本**，所以选择了 sayHello(Human) 作为调用目标，并把这个方法的符号引用写到 main() 方法的两条 invokevirtual 指令的参数中。

**所有依赖静态类型来定位方法执行版本的分派动作称为静态分派。静态分派的典型应用是方法重载。静态分派发生在编译阶段，因此确定静态分派的动作实际上不是由虚拟机来执行的。编译器虽然能确定出方法的重载版本，但在很多情况下这个重载版本并不是“唯一的”，往往只能确定一个“更加合适的”版。产生这种模糊结论的主要原因是字面量不需要定义，所以字面量没有显式的静态类型，它的静态类型只能通过语言上的规则去理解和推断。**

### 动态分派

**动态分派和多态性的另外一个重要体现---重写（Override）有着很密切的关联。**

```java
public class DynamicDispatch {
    
    static abstract Class Human {
        protected abstract void sayHello();
    }

    static class Man extends Human {
        @OVerride
        protected void sayHello() {
            System.out.println("man say hello");
        }
    }

    static class Woman extends Human {
        @Overrid
        protected void sayHello() {
            System.out.println("woman say hello");
        }
    }

    public static void main(String[] args) {
        Human man = new Man();
        Human woman = new Woman();
        man.sayHello();
        woman.sayHello();
        man = new Woman();
        man.sayHello();
    }

}
```

运行结果：

```
man say hello
woman say hello
woman say hello
```

显然这里不可能在根据静态类型来决定，因为静态类型同样都是 Human 的两个变量 man 和 woman 在调用 sayHello() 方法时执行了不同的行为，并且变量 man 在两次调用中执行了不同的方法。导致这个现象的原因很明显，是这两个变量的实际类型不同，Java 虚拟机是如何根据实际类型来分派方法执行版本的呢？我们使用 javap 命令输出这段代码的字节码，尝试从中寻找答案。

 ```
 public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=3, args_size=1
         0: new           #2                  // class com/ys/sakalaka/jvm/DynamicDispatch$Man
         3: dup
         4: invokespecial #3                  // Method com/ys/sakalaka/jvm/DynamicDispatch$Man."<init>":()V
         7: astore_1
         8: new           #4                  // class com/ys/sakalaka/jvm/DynamicDispatch$Woman
        11: dup
        12: invokespecial #5                  // Method com/ys/sakalaka/jvm/DynamicDispatch$Woman."<init>":()V
        15: astore_2
        16: aload_1
        17: invokevirtual #6                  // Method com/ys/sakalaka/jvm/DynamicDispatch$Human.sayHello:()V
        20: aload_2
        21: invokevirtual #6                  // Method com/ys/sakalaka/jvm/DynamicDispatch$Human.sayHello:()V
        24: new           #4                  // class com/ys/sakalaka/jvm/DynamicDispatch$Woman
        27: dup
        28: invokespecial #5                  // Method com/ys/sakalaka/jvm/DynamicDispatch$Woman."<init>":()V
        31: astore_1
        32: aload_1
        33: invokevirtual #6                  // Method com/ys/sakalaka/jvm/DynamicDispatch$Human.sayHello:()V
        36: return

 ```

 0~15 行的字节码是准备动作，作用是建立 man 和 woman 的内存空间、调用 Man 和 Woman 类型的实例构造器，将这两个实例的引用存放在第 1、2 个局部变量表 Slot 之中。接下来的 16~21 句是关键部分，16、20 两句分别把刚刚创建的两个对象的引用压到栈顶，这两个对象就是将要执行的 sayHello() 方法的所有者，称为接收者（Receiver）；17 和 21 句是方法调用指令，这两条指令单从字节码的角度来看，无论是指令（都是 invokevirtual）还是参数（都是常量池中的第 6 项的常量，注释显示了这个常量是 Human.sayHello() 的符号引用）完全一样的，但是这两句指令最终执行的目标方法并不相同。原因就需要从 invokevirtual 指令的多态查找过程开始说起，invokevirtual 指令的运行时解析过程大概分为以下几个步骤：

1. 找到操作数栈顶的第一个元素所指向的对象的实例类型，记做 C。
2. 如果在类型 C 中找到与常量中的描述符和简单名称都相符的方法，则进行访问权限校验，如果通过则返回这个方法的直接引用，查找结束；如果不通过，则返回 java.lang.IllegalAccessError 异常。
3. 否则，按照继承关系从下往上一次对 C 的各个父类进行第 2 步的搜索和验证过程。
4. 如果始终没有找到合适的方法，则抛出 java.lang.AbstractMethodError 异常。

**由于 invokevirtual 指令执行的第一步就是在运行期确定接收者的实际类型，所以两次调用中的 invokevirtual 指令把常量池中的类方法符号引用解析到了不同的直接引用上，这个过程就是 Java 语言中方法重写的本质。我们把这种在运行期间根据实际类型确定方法执行版本的分派过程称为动态分派。**

### 单分派和多分派

**方法的接收者与方法的参数统称为方法的宗量。根据分派基于多少种宗量，可以将分派划分为单分派和多分派两种。单分派是根据一个宗量与目标方法进行选择，多分派则是根据多于一个宗量对目标方法进行选择。**

```java
/**
* 单分派、多分派演示
*/
public class Dispatch {
    
    static class QQ {}

    static class _360 {}

    public static class Father {
        public void hardChoice(QQ arg) {
            System.out.println("father choose qq");
        }

        public void hardChoice(_360 arg) {
            System.out.println("fatcher choose 360");
        }
    }

    public static class Son extends Father {
        public void hardChoice(QQ arg) {
            System.out.println("son choose qq");
        }

        public void hardChoice(_360 arg) {
            System.out.println("son choose 360");
        }
    }

    public static void main(String[] args) {
        Fathcer father = new Father();
        Father son = new Son();

        father.hardChoice(new _360());
        son.hardChoice(new QQ());
    }

}
```

执行结果

```
father choose 360
son choose qq
```

我们来看看编译阶段编译器的选择过程，也就是静态分派的过程。这是选择目标方法的依据有两点：一是静态类型是 Father 还是 Son，二是方法参数是 QQ 还是 _360。这次选择结果的最终产物是产生了两条 invokevirtual 指令，两条指令的参数分别为常量池中指向 Father.hardChoice(_360) 和 Father.hardChoice(QQ) 方法的符号引用。因此是根据两个宗量（静态类型和参数）进行选择，所以 **Java 语言的静态分派属于多分派类型。**

再看看运行阶段虚拟机的选择，也就是动态分派的过程。在执行 “son.hardChoice(new QQ)” 这句代码时，更准确的说是在执行方法调用指令 invokevirtual 时，由于编译器已经决定了目标方法的签名必须为 hardChoice(QQ)，因此此时虚拟机不会关心传递过来的参数 “QQ” 的实际类型是什么，因为这些都对方法的选择不够构成任何影响，唯一可以影响虚拟机选择的因素只有此方法的接受者的实际类型是 Father 还是 Son。因为只有一个宗量作为选择依据，所以 **Java 语言的动态分派属于单分派类型。**

### 虚拟机动态分派的实现
由于动态分派是非常频繁的动作，而且动态分派的方法版本选择过程需要运行时在类的方法元数据中搜索合适的目标方法，因此在虚拟机的实际实现中基于性能的考虑，大部分实现都不会真正的进行如此频繁的搜索。面对这种情况，**最常用的“稳定优化“手段就是为类在方法区中建立一个虚方法表（Virtural Method Table，也称为 vtable，与此对应的，在 invokeinterface 执行时也会用到接口方法表---Interface Method Table，简称 itable），使用虚方法表索引来代替元数据以提高性能。**

![方法表结构](/images/method_table.jpeg)

**虚方法表中存放着各个方法的实际入口地址。如果某个方法在子类中没有被重写，那子类的虚方法表里面的地址入口和父类相方法的地址入口是一致的，都指向父类的实现入口。如果子类重写了这个方法，子类方法表中的地址将会替换为指向子类实现版本的入口地址。**

**为了程序实现上的方便，具有相同签名的方法，在父类、子类的虚方法表中都有应当具有一样的索引序号，这样当类型变换时，仅需要变更查找的方法表，就可以从不同的虚方法表中按索引转换出所需的入口地址。**

**方法表一般都在类加载的连接阶段进行初始化，准备了子类的变量初始值后，虚拟机会把该类的方法表也初始化完毕。**
