---
title: .NET 新特性：基于代码生成的正则表达式
date: 2026-01-04 11:45:23
link: dotnet/feature/GeneratedRegex
catalog: false
categories:
  - [笔记, .NET]
tags:
  - .NET
  - C#
  - 正则表达式
  - 源生成器
---

说到**正则表达式**这一老生常谈的话题，最容易被提及的就是它的性能——由于需要将字符串形式的表达式即时翻译成逻辑结构，这东西在实际使用时经常会以各种姿态浪费性能，甚至于有人宁肯自己写一堆 if-else 都不肯使用正则表达式，就为了避免巨大的翻译性能代价。

那么，有没有什么办法可以优化正则表达式的性能嘞？

答案是有的，实际上 .NET 早就引入过一个特性 `RegexOptions.Compiled`，添加了这个 flag 的正则表达式会在初始化时预先编译为 CLR 动态代码，从而使匹配时的性能比不加这个 flag 高了好几倍，毕竟直接跑代码怎么也得比即时翻译更快吧。

但是这样做也引入了一个明显的缺点：编译为动态代码的过程，远比正常初始化一个正则表达式耗时。因此这个特性只适用于那种初始化一次但使用很频繁的场景，当一个正则表达式只需要使用一次或很少几次，这样反而会因为初始化代价过高，导致最终综合性能还不如不预先编译。

还有什么其他办法吗？

有的兄弟，有的。这种编译性能损耗问题最简单的解决方法就是预先生成——那既然 .NET 有个叫源生成器的好东西，为什么不直接在编译期把原本运行期干的事情给干了呢？

于是 .NET 7 引入了一个特性：基于代码生成的正则表达式。它可以在编译期根据你给出的正则表达式直接生成匹配逻辑代码，从而秒了一切优化——运行时根本就不存在表达式本身了，直接就是代码，还优化什么。

这个特性通过 `GeneratedRegex` 注解修饰一个 `partial` 方法来触发，使用方法很简单：

```cs
using System.Text.RegularExpressions;

namespace Your.Namespace;

public static partial class RegexCollect
{
    [GeneratedRegex(@"\r\n|\n|\r")]
    private static partial Regex _NewLine();
    public static readonly Regex NewLine = _NewLine();
}
```

在这个示例中，`partial` 方法 `_NewLine()` 被 `GeneratedRegex` 修饰，并传入了正则表达式。然后，我们通过一个 `readonly` 的字段 `NewLine` 来承接方法返回的内容，即一个包含预先生成代码的正则表达式对象，最后直接像调用正常的 `Regex` 对象一样使用它就好啦！

`GeneratedRegex` 注解的定义如下：

```cs
public GeneratedRegexAttribute([StringSyntax("Regex", new object[] {"options"})] string pattern, RegexOptions options);
```

观察可知，使用时只需传入与 `Regex` 的构造方法相同的两个参数 `pattern` `options` 即可。需要特别注意的是：这里的 `options` 传入 `RegexOptions.Compiled` 是没有任何作用的，因为它本身就已经是预编译了。

上述代码会触发源生成器生成以下内容：

<details>
<summary>由于内容很长，这里折叠起来了，点我展开</summary>

```cs
namespace Your.Namespace
{
    partial class RegexCollect
    {
        /// <remarks>
        /// Pattern:<br/>
        /// <code>\\r\\n|\\n|\\r</code><br/>
        /// Explanation:<br/>
        /// <code>
        /// ○ Match with 2 alternative expressions, atomically.<br/>
        ///     ○ Match the string "\r\n".<br/>
        ///     ○ Match a character in the set [\n\r].<br/>
        /// </code>
        /// </remarks>
        [global::System.CodeDom.Compiler.GeneratedCodeAttribute("System.Text.RegularExpressions.Generator", "8.0.13.2707")]
        private static partial global::System.Text.RegularExpressions.Regex _NewLine() => global::System.Text.RegularExpressions.Generated._NewLine_1.Instance;
    }
}
namespace System.Text.RegularExpressions.Generated
{
    using System;
    using System.Buffers;
    using System.CodeDom.Compiler;
    using System.Collections;
    using System.ComponentModel;
    using System.Globalization;
    using System.Runtime.CompilerServices;
    using System.Text.RegularExpressions;
    using System.Threading;

    /// <summary>Custom <see cref="Regex"/>-derived type for the _NewLine method.</summary>
    [GeneratedCodeAttribute("System.Text.RegularExpressions.Generator", "8.0.13.2707")]
    [SkipLocalsInit]
    file sealed class _NewLine_1 : Regex
    {
        /// <summary>Cached, thread-safe singleton instance.</summary>
        internal static readonly _NewLine_1 Instance = new();
    
        /// <summary>Initializes the instance.</summary>
        private _NewLine_1()
        {
            base.pattern = "\\r\\n|\\n|\\r";
            base.roptions = RegexOptions.None;
            ValidateMatchTimeout(Utilities.s_defaultTimeout);
            base.internalMatchTimeout = Utilities.s_defaultTimeout;
            base.factory = new RunnerFactory();
            base.capsize = 1;
        }
            
        /// <summary>Provides a factory for creating <see cref="RegexRunner"/> instances to be used by methods on <see cref="Regex"/>.</summary>
        private sealed class RunnerFactory : RegexRunnerFactory
        {
            /// <summary>Creates an instance of a <see cref="RegexRunner"/> used by methods on <see cref="Regex"/>.</summary>
            protected override RegexRunner CreateInstance() => new Runner();
        
            /// <summary>Provides the runner that contains the custom logic implementing the specified regular expression.</summary>
            private sealed class Runner : RegexRunner
            {
                /// <summary>Scan the <paramref name="inputSpan"/> starting from base.runtextstart for the next match.</summary>
                /// <param name="inputSpan">The text being scanned by the regular expression.</param>
                protected override void Scan(ReadOnlySpan<char> inputSpan)
                {
                    // Search until we can't find a valid starting position, we find a match, or we reach the end of the input.
                    while (TryFindNextPossibleStartingPosition(inputSpan) &&
                           !TryMatchAtCurrentPosition(inputSpan) &&
                           base.runtextpos != inputSpan.Length)
                    {
                        base.runtextpos++;
                        if (Utilities.s_hasTimeout)
                        {
                            base.CheckTimeout();
                        }
                    }
                }
        
                /// <summary>Search <paramref name="inputSpan"/> starting from base.runtextpos for the next location a match could possibly start.</summary>
                /// <param name="inputSpan">The text being scanned by the regular expression.</param>
                /// <returns>true if a possible match was found; false if no more matches are possible.</returns>
                private bool TryFindNextPossibleStartingPosition(ReadOnlySpan<char> inputSpan)
                {
                    int pos = base.runtextpos;
                    
                    // Empty matches aren't possible.
                    if ((uint)pos < (uint)inputSpan.Length)
                    {
                        // The pattern begins with a character in the set [\n\r].
                        // Find the next occurrence. If it can't be found, there's no match.
                        int i = inputSpan.Slice(pos).IndexOfAny('\n', '\r');
                        if (i >= 0)
                        {
                            base.runtextpos = pos + i;
                            return true;
                        }
                    }
                    
                    // No match found.
                    base.runtextpos = inputSpan.Length;
                    return false;
                }
        
                /// <summary>Determine whether <paramref name="inputSpan"/> at base.runtextpos is a match for the regular expression.</summary>
                /// <param name="inputSpan">The text being scanned by the regular expression.</param>
                /// <returns>true if the regular expression matches at the current position; otherwise, false.</returns>
                private bool TryMatchAtCurrentPosition(ReadOnlySpan<char> inputSpan)
                {
                    int pos = base.runtextpos;
                    int matchStart = pos;
                    char ch;
                    ReadOnlySpan<char> slice = inputSpan.Slice(pos);
                    
                    // Match with 2 alternative expressions, atomically.
                    {
                        int alternation_starting_pos = pos;
                        
                        // Branch 0
                        {
                            // Match the string "\r\n".
                            if (!slice.StartsWith("\r\n"))
                            {
                                goto AlternationBranch;
                            }
                            
                            pos += 2;
                            slice = inputSpan.Slice(pos);
                            goto AlternationMatch;
                            
                            AlternationBranch:
                            pos = alternation_starting_pos;
                            slice = inputSpan.Slice(pos);
                        }
                        
                        // Branch 1
                        {
                            // Match a character in the set [\n\r].
                            if (slice.IsEmpty || (((ch = slice[0]) != '\n') & (ch != '\r')))
                            {
                                return false; // The input didn't match.
                            }
                            
                            pos++;
                            slice = inputSpan.Slice(pos);
                        }
                        
                        AlternationMatch:;
                    }
                    
                    // The input matched.
                    base.runtextpos = pos;
                    base.Capture(0, matchStart, pos);
                    return true;
                }
            }
        }
    }
}
```
</details>

观察生成的内容可见，源生成器创建了一个定制化的 `Regex` 子类，并直接生成了正则表达式对应的匹配逻辑。这样，在运行时调用时，这段逻辑会被直接使用，彻底告别动态编译，极大程度上提升了正则表达式的性能。
