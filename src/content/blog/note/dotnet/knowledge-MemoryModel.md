---
title: .NET 托管内存模型简介
date: 2026-01-19 00:25:01
link: dotnet/knowledge/MemoryModel
categories:
  - [笔记, .NET]
tags:
  - .NET
---

## 简介

本文记录 .NET 托管内存模型的基本概念，详细解释其中的**堆**与**栈**为何物及它们的具体作用。没有意外的话，这里介绍的内存模型应该与大多数语言基本一致，因为现代面向对象语言用的基本都是同一套东西，无非就是手动管理内存与自动管理内存的区别，有关这一点区别也会在后面说明。

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

可知 .NET 内存模型遵循的原则是：小数据和临时占用放到栈里，大数据和可能长期使用的内容放到堆里。与此同时，可见栈中存在一个现象：方法的调用处与被调用的方法会共享一小部分栈内存。

这个原则在 C++ / Java 等等各种语言中也能见到，基本可以说是个计算机界的标准了。

## 栈与堆的使用策略

我们暂且忽略在具体分配时的方法签名及返回值等基本部分，栈通常存储**值类型本身**和**指向引用类型的引用**，而堆通常存储各处的引用对应的**实际对象数据**。

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

*未完待续*
