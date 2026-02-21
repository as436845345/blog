---
title: ReadOnlySpan<T>：轻量高效的只读内存操作
date: 2026-01-18 14:07:14
tags:
- C#
- ReadOnlySpan
categories:
- C#
---

### **最后更新时间：2026-01-18 14:29:29**

本文通过豆包辅助生成！

---

<!--
帮我写一篇 markdown 文章，是关于 C# 的 ReadOnlySpan<T> 简单介绍+简单的使用教程+案例 的内容，需要把以下所有要求罗列出来：

1. 介绍 ReadOnlySpan<T> 是什么；
2. 在 string、数组、List<T>（其他列表不知道行不行）中，通过 MemoryExtensions.AsSpan 转化为 ReadOnlySpan<T> （如果当前 .net 版本不存在 MemoryExtensions 时，通过 NuGet 安装 System.Memory 库）；
3. 为什么使用 ReadOnlySpan<T>？有什么优势？
  1. ReadOnlySpan<T> 的优势；
  2. 相比直接操作 string 的方法，通过 MemoryExtensions.AsSpan 转换为 ReadOnlySpan<char> 时，那些方法的执行效率提高（高版本的 string 的方法内部实现也使用了 ReadOnlySpan，只有低版本没优化）；
  3. 也介绍 数组、列表 等其他。
4. ReadOnlySpan<T> 属性教程+案例（案例用 string），只介绍：
  - Item[Int32]：从指定从零开始的索引处的只读范围获取项。
  - Length：只读范围中的项数。
5. ReadOnlySpan<T> 方法教程+案例（案例用 数组），只介绍：
  - CopyTo(Span<T>)：将此 ReadOnlySpan<T> 的内容复制到目标 Span<T>。
  - TryCopyTo(Span<T>)：尝试将此 ReadOnlySpan<T> 的内容复制到 Span<T> 中，并返回一个值以指示操作是否成功。
  - Slice(Int32)：形成从当前只读范围开始的切片，该范围从指定的索引处开始。
  - Slice(Int32, Int32)：形成从指定长度的指定索引处开始的当前只读范围的切片。
  - 还有其他像 Contains、StartsWith、EndsWith、IndexOf 等，通过文档 / 网络教程查看。
-->

在 C# 中，ReadOnlySpan\<T\> 是处理内存片段的核心类型，尤其适合高频读写、低内存开销的场景。本文将从“是什么、怎么用、为什么好用”三个维度，带你快速掌握 ReadOnlySpan\<T\> 的基础用法和核心优势。

## 一、ReadOnlySpan\<T\> 是什么？

ReadOnlySpan\<T\> 是 .NET Core 2.1+/.NET 5+ 引入的**只读内存跨度类型**，本质是对一段连续内存（如字符串、数组、非托管内存）的“轻量视图”——它不分配新内存，仅记录内存的起始地址和长度，且**无法修改指向的内存内容**，因此兼具高性能和内存安全。

核心特点：
- **栈分配（stack-only）**：实例存储在栈上，无 GC 开销；
- 只读特性：仅能读取内存内容，无法修改，避免意外数据篡改；
- 零拷贝：操作内存时不复制数据，直接引用原内存区域。

> 注意：ReadOnlySpan\<T\> 不能用于异步方法的返回值、类的字段等场景（栈类型限制），若需跨上下文使用，可改用 ReadOnlyMemory\<T\>。

## 二、如何将常见类型转为 ReadOnlySpan\<T\>？

ReadOnlySpan\<T\> 无法直接创建（无公共构造函数），需通过 MemoryExtensions.AsSpan 方法将字符串、数组、列表等转为 ReadOnlySpan\<T\>。

### 前置条件：安装依赖（低版本 .NET 需处理）

若你的 .NET 版本（如 .NET Framework 4.x）未内置 MemoryExtensions，需先安装 NuGet 包：

```bash
Install-Package System.Memory
# 或 .NET CLI
dotnet add package System.Memory
```

### 1. 字符串转 ReadOnlySpan\<char\>

字符串本质是 char 数组，转换后可直接操作字符片段，且无字符串拷贝开销：

```csharp
// 原始字符串
string originalStr = "Hello ReadOnlySpan!";
// 转为 ReadOnlySpan<char>
ReadOnlySpan<char> strSpan = originalStr.AsSpan(); // 等价于 MemoryExtensions.AsSpan(originalStr)

Console.WriteLine(strSpan); // 输出：Hello ReadOnlySpan!
```

### 2. 数组转 ReadOnlySpan\<T\>

任意数组（如 int[]、byte[]）均可直接转换，适配任意值类型/引用类型：

```csharp
// 原始 int 数组
int[] numArray = new int[] { 1, 2, 3, 4, 5 };
// 转为 ReadOnlySpan<int>
ReadOnlySpan<int> arraySpan = numArray.AsSpan();

Console.WriteLine(arraySpan.Length); // 输出：5
```

### 3. List\<T\> 转 ReadOnlySpan\<T\>

若要将 List\<T\> 转为 ReadOnlySpan\<T\>，有两种方法可以实现：

1. 先将 List\<T\> 通过 ToArray() 转为数组（因 List 内存非绝对连续），再转 ReadOnlySpan\<T\>：

```csharp
// 原始 List<string>
List<string> strList = new List<string> { "a", "b", "c" };
// List → 数组 → ReadOnlySpan<T>
ReadOnlySpan<string> listSpan = strList.ToArray().AsSpan();

Console.WriteLine(listSpan[1]); // 输出：b
```

2. 通过 CollectionsMarshal.AsSpan\<T\>(List\<T\>) 转为 ReadOnlySpan\<T\>：

```csharp
// 原始 List<string>
List<string> strList = new List<string> { "a", "b", "c" };
// List → 数组 → ReadOnlySpan<T>
ReadOnlySpan<string> listSpan = CollectionsMarshal.AsSpan(strList);

Console.WriteLine(listSpan[1]); // 输出：b
```

> CollectionsMarshal 类需要 .NET 5+ 才能使用，或者通过 NuGet 安装包 System.Runtime.InteropServices（未测试过，不确定存不存在 CollectionsMarshal 类）。

## 三、为什么要使用 ReadOnlySpan\<T\>？核心优势

相比直接操作字符串、数组，ReadOnlySpan\<T\> 最大的价值是**零拷贝 + 低 GC 开销**，具体优势如下：

### 1. 核心优势总结

| 优势 | 说明 |
|------|------|
| 零内存拷贝 | 操作内存片段时（如截取子串、取数组片段），仅修改“视图范围”，不复制原数据 |
| 无 GC 压力 | 栈分配类型，实例无需 GC 回收；避免频繁创建临时字符串/数组导致的 GC 触发 |
| 内存安全 | 只读特性防止意外修改原数据，索引越界会直接抛出异常，避免内存越访问 |
| 高性能 | 直接操作内存地址，比传统字符串/数组方法（如 string.Substring）快数倍 |

### 2. 对比 string：效率大幅提升

传统 string 是不可变类型，调用 Substring、Split 等方法时，会创建**新字符串实例**（拷贝原字符数据），高频操作时会产生大量临时对象，触发频繁 GC。

而 ReadOnlySpan\<char\> 操作字符串时无拷贝：

```csharp
// 传统方式：创建新字符串（拷贝数据）
string original = "Hello World";
string subStr = original.Substring(0, 5); // 生成新字符串 "Hello"，拷贝5个字符

// ReadOnlySpan 方式：零拷贝，仅调整视图范围
ReadOnlySpan<char> span = original.AsSpan();
ReadOnlySpan<char> subSpan = span.Slice(0, 5); // 无拷贝，仅指向原字符串的前5个字符
```

> 补充：高版本 .NET 的 string 内置方法（如 Substring）已底层适配 ReadOnlySpan\<char\>，但低版本 .NET 仍为拷贝实现——因此低版本中手动用 ReadOnlySpan\<char\> 优化效果更显著。

### 3. 对比数组/列表：更轻量的内存操作

直接操作数组时，截取片段（如 Array.Copy）需拷贝数据；而 ReadOnlySpan\<T\> 仅通过 Slice 方法调整视图，无拷贝开销：

```csharp
int[] nums = { 1, 2, 3, 4, 5 };

// 传统方式：拷贝数组片段（创建新数组）
int[] subNums = new int[3];
Array.Copy(nums, 1, subNums, 0, 3); // 拷贝索引1-3的元素，生成新数组 [2,3,4]

// ReadOnlySpan 方式：零拷贝，仅定义视图
ReadOnlySpan<int> subSpan = nums.AsSpan().Slice(1, 3); // 直接指向原数组的索引1-3，无拷贝
```

## 四、ReadOnlySpan\<T\> 核心属性（案例：string）

仅介绍高频使用的 2 个核心属性，案例基于字符串场景：

### 1. Item[Int32]：获取指定索引的元素

通过索引访问 ReadOnlySpan\<T\> 中的元素，语法与数组一致，**只读不可改**：

```csharp
string text = "C# ReadOnlySpan Tutorial";
ReadOnlySpan<char> textSpan = text.AsSpan();

// 获取索引2的字符（索引从0开始）
char charAt2 = textSpan[2]; 
Console.WriteLine(charAt2); // 输出：空格（"C# " 的第三个字符）

// 尝试修改会编译报错：ReadOnlySpan 只读
// textSpan[2] = 'x'; // 错误：无法给只读索引器赋值
```

### 2. Length：获取只读范围的元素数量

返回 ReadOnlySpan\<T\> 包含的元素总数，等价于原数据的有效长度：

```csharp
string text = "Hello ReadOnlySpan";
ReadOnlySpan<char> textSpan = text.AsSpan();

// 获取长度
int length = textSpan.Length;
Console.WriteLine(length); // 输出：17（"Hello ReadOnlySpan" 共17个字符）

// 空 Span 的 Length 为 0
ReadOnlySpan<char> emptySpan = ReadOnlySpan<char>.Empty;
Console.WriteLine(emptySpan.Length); // 输出：0
```

## 五、ReadOnlySpan\<T\> 核心方法（案例：数组）

以下方法均基于 int[] 数组案例，聚焦高频使用的 4 个方法：

### 1. CopyTo(Span\<T\>)：复制内容到目标 Span\<T\>

将 ReadOnlySpan\<T\> 的内容复制到可写的 Span\<T\>（需保证目标 Span 长度足够，否则抛异常）：

```csharp
// 源数组 → ReadOnlySpan<int>
int[] source = { 10, 20, 30, 40 };
ReadOnlySpan<int> sourceSpan = source.AsSpan();

// 目标 Span（长度需 ≥ 源 Span）
int[] target = new int[4];
Span<int> targetSpan = target.AsSpan();

// 复制内容
sourceSpan.CopyTo(targetSpan);

// 输出目标数组：[10,20,30,40]
Console.WriteLine(string.Join(",", target)); 
```

### 2. TryCopyTo(Span\<T\>)：安全复制（返回操作结果）

与 CopyTo 功能一致，但不会抛异常——若目标 Span 长度不足，返回 false，否则返回 true：

```csharp
int[] source = { 10, 20, 30, 40 };
ReadOnlySpan<int> sourceSpan = source.AsSpan();

// 目标 Span 长度不足（仅3个元素）
int[] target = new int[3];
Span<int> targetSpan = target.AsSpan();

// 尝试复制
bool isSuccess = sourceSpan.TryCopyTo(targetSpan);
Console.WriteLine(isSuccess); // 输出：False（长度不足，复制失败）
Console.WriteLine(string.Join(",", target)); // 输出：0,0,0（无数据复制）

// 调整目标长度后重试
int[] target2 = new int[4];
bool isSuccess2 = sourceSpan.TryCopyTo(target2.AsSpan());
Console.WriteLine(isSuccess2); // 输出：True
Console.WriteLine(string.Join(",", target2)); // 输出：10,20,30,40
```

### 3. Slice(Int32)：从指定索引开始截取片段

截取从 startIndex 到末尾的所有元素，零拷贝仅调整视图：

```csharp
int[] nums = { 1, 2, 3, 4, 5 };
ReadOnlySpan<int> numSpan = nums.AsSpan();

// 从索引2开始截取（包含索引2）
ReadOnlySpan<int> sliceSpan = numSpan.Slice(2);

// 输出：3,4,5（索引2、3、4的元素）
Console.WriteLine(string.Join(",", sliceSpan)); 
```

### 4. Slice(Int32, Int32)：指定索引+长度截取片段

截取从 startIndex 开始、长度为 length 的片段，需保证 startIndex + length ≤ 原 Span 长度：

```csharp
int[] nums = { 1, 2, 3, 4, 5 };
ReadOnlySpan<int> numSpan = nums.AsSpan();

// 从索引1开始，截取3个元素
ReadOnlySpan<int> sliceSpan = numSpan.Slice(1, 3);

// 输出：2,3,4（索引1、2、3的元素）
Console.WriteLine(string.Join(",", sliceSpan)); 

// 索引越界会抛异常
// numSpan.Slice(1, 5); // 错误：长度超出范围
```

### 补充：其他常用方法

ReadOnlySpan\<T\> 还内置了大量实用方法，可参考官方文档/网络教程：
- Contains(T)：判断是否包含指定元素；
- StartsWith(ReadOnlySpan\<T\>)：判断是否以指定片段开头；
- EndsWith(ReadOnlySpan\<T\>)：判断是否以指定片段结尾；
- IndexOf(T)：查找指定元素的第一个索引；
- LastIndexOf(T)：查找指定元素的最后一个索引。

## 总结

1. ReadOnlySpan\<T\> 是只读的内存视图，通过 AsSpan() 从字符串/数组/列表转换，低版本需安装 System.Memory；
2. 核心优势是零拷贝、低 GC 开销，相比传统字符串/数组操作效率大幅提升；
3. 核心属性：Item[Int32]（索引访问）、Length（获取长度）；
4. 核心方法：CopyTo/TryCopyTo（复制）、Slice（截取片段），适配高性能内存操作场景。

ReadOnlySpan\<T\> 尤其适合高频处理字符串、字节数组的场景（如网络IO、数据解析），是 C# 高性能编程的必备工具。

## 外部链接

[System.Memory - NuGet Gallery](https://www.nuget.org/packages/System.Memory)
[System.Runtime.InteropServices - NuGet Gallery](https://www.nuget.org/packages/System.Runtime.InteropServices)
[ReadOnlySpan\<T\> 结构 - Microsoft Learn](https://learn.microsoft.com/zh-cn/dotnet/api/system.memoryextensions?view=net-8.0)
[MemoryExtensions 类 - Microsoft Learn](https://learn.microsoft.com/zh-cn/dotnet/api/system.memoryextensions?view=net-8.0)
[CollectionsMarshal 类 - Microsoft Learn](https://learn.microsoft.com/zh-cn/dotnet/api/system.runtime.interopservices.collectionsmarshal?view=net-9.0)