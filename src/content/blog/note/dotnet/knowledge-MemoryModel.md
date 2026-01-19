---
title: .NET 内存模型简介
date: 2026-01-19 00:25:01
link: dotnet/knowledge/MemoryModel
categories:
  - [笔记, .NET]
tags:
  - .NET
---

## 简介

本文记录 .NET 内存模型的基本概念，详细解释其中的**堆**与**栈**为何物及它们的具体作用。没有意外的话，这里介绍的内存模型应该与大多数语言基本一致，因为现代面向对象语言用的基本都是同一套东西，无非就是手动管理内存与自动管理内存的区别，有关这一点区别也会在后面说明。

若无特殊说明，本文所有代码示例均使用 C#。

.NET 托管内存主要分为两部分：负责存储代码执行各层逻辑中必要数据的**栈**，与用来丢各种杂七杂八东西的**堆**。这两部分共同构成了 CLR 程序运行过程主要的内存占用，但其存储内容和管理逻辑有显著的区别。

栈是典型的后进先出（LIFO）结构——没错就是数据结构经常提到的那个栈，它主要存储程序运行中的各类方法、调用层以及每层调用中局部的临时变量。每个线程都有自己的栈，每有一个方法调用，就会在最顶部新建一层，而每一层栈的生命周期通常都很短——只要对应的方法结束了，它对应的那一层栈空间就会被释放。

而堆是一种非线性的存储结构，它存的可就多了，什么东西都有。当你写一个 `var a = new StringBuilder();` 的时候，就有至少一个东西，也就是你 new 出来的那个对象，被丢到了堆里。当然远不止这些，因为一个 `StringBuilder` 在构造自己的时候还会去创建别的对象，它们也会被丢到堆里面。

## 内存分配的基本流程

让我们从程序的主方法说起。

我们知道，每个 .NET 程序都会从被 CLR（公共语言运行时）调用的主方法开始运行，也就是我们经常看到的：

```cs
[STAThread]
public static int Main()
{
    var a = "Hello, world!";
    Console.WriteLine(a);
    return 0;
}
```

这个示例写了一个最基本的程序，向控制台输出 `Hello, world!`。

首先，它是在程序的主线程启动的，主线程有自己的栈空间，在 `Main` 方法被调用时，主线程的栈里面就会被分配一块空间，用来存储 `Main` 方法本身定义、预留的返回值、方法参数以及方法中临时的局部变量。

然后，代码就开始执行了。第一行我们定义了一个变量 `a`，并将一个字符串赋给了它，这时，堆里面就会多一个字符串 `Hello, world!` 的对象，与此对应的，栈里面多了一个引用（也就是一个地址值，取决于操作系统，这个值可能是一个 32 位整形，也可能是 64 位），指向堆中的那个字符串。

> *在真实环境中，这里的字符串会被丢进常量池，但现在我们不考虑这一点优化，仅认为它就是一个正常的对象。*

那么这时候局部变量 `a` 的值是什么呢？答案很明显——就是一个引用。这个地址值会存在栈中，供方法在其逻辑中使用，比如下面一行调用了 `Console.WriteLine` 方法，将 `a` 传了进去。这个调用又会在栈顶部新建一层，给 `WriteLine` 方法使用，同时对于字符串 `Hello, world!` 的引用也会被传进去。在 `WriteLine` 方法中，这个传进来的引用会被用于在堆中寻找对应的对象，也就是地址值所对应的内存空间真实的数据。然后，这个数据，也就是 `Hello, world!` 这段内容，就被输出到控制台了。

紧接着，方法 `WriteLine` 结束，执行流程返回到了 `Main` 方法中，与 `WriteLine` 对应的栈空间——也就是栈顶那一部分空间，会被直接释放。然后，执行到了 `return 0` 语句，`Main` 方法也随之结束了，返回值 `0` 被写入栈中给返回值预留的空间，程序执行结束，最终 CLR 从最后的栈空间中读到 `Main` 方法的返回值 `0` 作为程序结束代码并退出。

现在梳理一下上面的流程，内存中存了这些东西：

```plain
infographic hierarchy-structure
data
  items
    - label 栈 (Stack)
      children
        - label Main
          children
            - label 方法参数: void
            - label 变量: a (string ref)
            - label WriteLine 参数/返回值
            - label 返回值: 0
        - label Console.WriteLine
          children
            - label 方法参数: 传入的 a
            - label WriteLine 内部逻辑
            - label 返回值: void
    - label 堆 (Heap)
      children
        - label 对象: "Hello, world!" (string)
        - label 其他在 Console.WriteLine 中创建的对象
```

可知 .NET 内存模型遵循的原则是：小数据和临时占用放到栈里，大数据和可能长期使用的内容放到堆里。与此同时，可见栈中存在一个现象——方法的调用处与被调用的方法需要共享一小部分连续内存，这是栈内存实现方式的一个先天优势。

这个原则在 C++ / Java 等等各种语言中也能见到，基本可以说是个计算机界的标准了。

## 栈与堆的使用策略

我们暂且忽略在具体分配时的方法签名及返回值等基本部分，栈通常存储**值类型本身**和局部变量中**指向引用类型的引用**，而堆通常存储各处的引用对应的**实际对象数据**。

说到这里，就不得不提一下 .NET CLR 中存在的两个重要概念：值类型、引用类型。

以 C# 为例，**值类型**就是使用 `struct` 声明的类型，而**引用类型**则是使用 `class` 及变体 `record` 声明的类型。特殊的 `enum` 在正常使用时是值类型，但实际上是引用类型（在某些情况下也会自动变为引用类型），这是一种性能优化，此处暂且不详细展开。

特别地，C# 中包含了很多基本数据类型关键字，例如 `int` `long` `short` `byte` `float` `double` 等。这些关键字实际上是指向 CLR 值类型的语法糖，例如 `int` 实际上是 `System.Int32`，`long` 实际上是 `System.Int64`，`double` 实际上是 `System.Double`。

来看一段具体的代码：

```cs
public struct ValueTypeTest
{
    public int Number;
    public ReferenceTypeTest TestRef;
}

public class ReferenceTypeTest
{
    public int Number;
    public ValueTypeTest TestVal;
    public ReferenceTypeTest TestRef;
}

public static class Program
{
    [STAThread]
    public static int Main()
    {
        var testRef1 = new ReferenceTypeTest();
        var testVal = new ValueTypeTest { TestRef = testRef1 };
        var testRef2 = new ReferenceTypeTest() { TestRef = testRef1, TestVal = testVal };
        return 0;
    }
}
```

在代码的 `Main` 方法中，我们声明了一个值类型和一个引用类型。

值类型包含另一个值类型 `int` 的数据和一个引用类型的引用，而引用类型包含值类型 `int` 的数据和值类型 `ValueTypeTest` 的数据，同时也包含了其本身类型的一个引用。

那这段代码的内存分配是什么样子的呢？

```plain
infographic hierarchy-structure
data
  desc 此处已省略方法本身在栈中占用的基本空间，VTT 与 RTT 为 ValueTypeTest 和 ReferenceTypeTest 的简写
  items
    - label 栈 (Stack)
      children
        - label Main 方法
          children
            - label RTT #1 引用
            - label VTT #1 本体
            - label RTT #2 引用
        - label Main - VTT #1
          children
            - label int #1 本体
            - label RTT #1 引用
    - label 堆 (Heap)
      children
        - label RTT #1 本体
          children
            - label int #2 本体
            - label VTT #2 本体
            - label RTT 空引用
        - label RTT #2 本体
          children
            - label int #3 本体
            - label VTT #3 本体
            - label RTT #1 引用
        - label RTT #1 - VTT #2
          children
            - label int #4 本体
            - label RTT 空引用
        - label RTT #2 - VTT #3
          children
            - label int #5 本体
            - label RTT #1 引用
```

可见，大多数数据都会被丢到堆里，栈里面只有各种引用和临时的值类型数据。由于平时我们写代码时使用**类**远多于使用**结构体**，因此堆内存占用通常应远大于栈内存。

## 为什么要这样设计？

从以上流程中我们不难得出，栈只负责存储引用和小数据，而大多数实际数据都存在堆里面。

这种设计实际上源自对性能和灵活性的取舍。

回想栈的主要作用，我们会发现它紧密伴随着程序的执行逻辑——于是，栈内存的管理效率直接决定了程序代码的运行效率。换句话说，在 CPU 速度足够的情况下，栈内存管理速度就决定了一段代码执行速度的上限。

由此可见，栈需要有极高的性能来支撑程序的运行。因此它采用了非常简单的存储逻辑，即一块连续地址配合一个指针来管理分配与释放：指针前移就分配，指针后移就释放。这样的逻辑就决定了栈的 LIFO 特色，也正合适满足了方法调用间共享一段连续内存的需求。

如此一来就圆满了对吧？但同时，这个逻辑也决定了栈只适合预先可知的固定内存分配逻辑，不适合存储复杂的、需要随时灵活分配和释放的内容。而由类初始化而来的对象正是这种灵活到极致的内容：它们可能一瞬间就释放，也可能连续存活好几天，更可能嵌套起多种不同的生命周期。

要应对这种内存需求，计算机需要在一块连续或甚至不连续的内存中用各种复杂的算法逻辑来管理分配与释放，以保证性能的同时尽可能提高内存空间的使用效率。这块内存就是堆，一个什么都能塞的存储池，它随时准备好接受新的分配，也随时准备把用完的内存空间还给操作系统供其他程序利用。

## 堆内存与内存管理

由上可知，栈内存管理非常简单，分配与释放都是伴随程序执行自动完成的。而堆内存具有非常灵活的特点，它随时都能接受内存分配和回收请求并完成对应操作。那么，什么时候应该分配，什么时候应该回收呢？

这就引出了一个老生常谈的问题——内存管理。

根据语言与运行时的区别，内存管理可分为两种主要流派：自动管理与手动管理。主流的面向对象语言如 C#、Java、JavaScript 等大多数都采用自动管理，而像 C++、Rust 这样较为原生的语言则采用手动管理。

手动管理即在代码中手动触发堆内存的分配与释放，而这引入了一个问题：一但管理不善，就会导致有些内存空间在用完后不会被释放，而是持续被占用。这种现象就叫**内存泄露**（Memory Leak），内存泄露一旦积攒起来，程序的内存占用可能不断膨胀，最后影响到其他程序的正常工作甚至是操作系统的稳定性。因此除了上述提到的较为原生的语言，几乎所有语言均考虑易用性和程序的稳健程度而选择了不同程度的自动管理。

这就得提到微软从 Java 码头叼来的经典自动内存管理机制——**垃圾回收器**（Garbage Collector，即 GC）。

## .NET 垃圾回收机制

.NET GC 基于**分代假设**：对象生存的时间越短，被回收的可能性就越大；而存活越久的对象，越倾向于继续存活。

于是堆内存被 GC 分为三代：

- Gen 0：最年轻的代，包含新分配的对象，绝大多数对象在这里就会被回收。
- Gen 1：作为 Gen 0 和 Gen 2 之间的缓冲区，存放从 Gen 0 幸存下来的对象。
- Gen 2：包含长期存在的对象，例如静态变量或在多次回收中幸存的对象。

其中 Gen 0 与 Gen 1 会共享一块叫做**临时段**（ephemeral segment）的内存空间，这块空间大小不固定，根据微软文档的描述，取决于运行环境，它通常具有从 256M 到 4G 不等的大小。

当临时段不足以分配新的对象，GC 便会被触发，在堆内存中寻找可以被回收的空间，同时使用一些算法来优化仍然存在的空间分配。

GC 触发时，首先从根域开始遍历所有引用，标记不会被回收的对象，然后根据剩余空间重定向原来的引用地址，最后移动仍然被使用的空间，以尽可能释放出连续内存，减少内存碎片。在一次标准的 GC 活动中，Gen 0 的内存会被优先释放，当仍然释放不出足够的空间时，就轮到了 Gen 1。Gen 2 在正常情况下不会被回收，除非实在没有其他内存可用（例如收到了系统的内存不足消息），这时 Full GC 会被触发，清理从 Gen 0 到 Gen 2 的整个堆内存区域。

此处仅简单介绍，若要了解详细的 GC 工作原理，请参阅文档 [Fundamentals of garbage collection](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/fundamentals)。
