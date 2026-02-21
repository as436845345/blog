---
title: Span<T>：灵活高效的可写内存操作
date: 2026-01-19 10:19:09
tags:
- C#
- Span<T>
categories:
- C#
---

### **最后更新时间：2026-01-19 10:30:49**

本文通过豆包辅助生成！

---

<!-- 
这个是关于 ReadOnlySpan<T> 的文章，请生成一篇关于 Span<T> 的文章。因为 Span<T> 与 ReadOnlySpan<T> 大致相同，所以请描述 ReadOnlySpan<T> 没有的内容，讲解优势，并提供案例。
 -->

在 C# 高性能内存操作体系中，Span\<T\> 是 ReadOnlySpan\<T\> 的“可写版兄弟”——它继承了 ReadOnlySpan\<T\> 轻量、零拷贝、栈分配的核心优势，同时解锁了**修改内存内容**的能力，是高频读写场景下的核心工具。本文将从“与 ReadOnlySpan\<T\> 的差异、核心能力、使用教程、优势场景”四个维度，全面解析 Span\<T\> 的使用方式。

## 一、Span\<T\> 与 ReadOnlySpan\<T\> 核心差异

Span\<T\> 和 ReadOnlySpan\<T\> 同属“内存跨度类型”，底层均为栈分配的内存视图，但核心区别在于**可写性**，具体差异如下表：

| 特性                | ReadOnlySpan\<T\>                | Span\<T\>                        |
|---------------------|--------------------------------|--------------------------------|
| 内存操作权限        | 仅可读，无法修改内存内容       | 可读可写，支持修改内存原内容   |
| 类型继承/转换       | Span\<T\> 可隐式转为 ReadOnlySpan\<T\> | ReadOnlySpan\<T\> 无法转为 Span\<T\> |
| 核心适用场景        | 只读数据处理（如数据解析、读取） | 可写数据处理（如数据修改、组装） |
| 核心方法/属性       | 无写操作相关能力               | 包含写操作（如索引赋值、Fill） |

核心定义：  
Span\<T\> 是 .NET Core 2.1+/.NET 5+ 引入的**可写内存跨度类型**，本质是对连续内存（字符串、数组、非托管内存）的轻量可写视图，同样具备栈分配、零拷贝、低 GC 开销的特性，且支持直接修改指向的内存内容。

> 注意：与 ReadOnlySpan\<T\> 一致，Span\<T\> 同样是栈类型，无法用于异步方法返回值、类字段等跨上下文场景，跨上下文可写场景需改用 Memory\<T\>。

## 二、Span\<T\> 独有的核心能力（ReadOnlySpan\<T\> 无）

Span\<T\> 最核心的价值是**可修改内存内容**，同时扩展了一系列写操作相关的方法/属性，以下是其独有的关键能力：

### 1. 索引器可写：直接修改指定位置的元素

ReadOnlySpan\<T\> 的索引器仅支持“读”，而 Span\<T\> 支持通过索引直接修改原内存中的数据：

```csharp
int[] nums = { 1, 2, 3, 4, 5 };
// ReadOnlySpan<T> 仅可读
ReadOnlySpan<int> readOnlySpan = nums.AsSpan();
// readOnlySpan[0] = 100; // 编译报错：只读索引器无法赋值

// Span<T> 可写
Span<int> writableSpan = nums.AsSpan();
writableSpan[0] = 100; // 直接修改原数组的第一个元素
Console.WriteLine(string.Join(",", nums)); // 输出：100,2,3,4,5
```

### 2. 专属写操作方法

Span\<T\> 提供了 ReadOnlySpan\<T\> 没有的写操作方法，典型如下：

| 方法                | 作用                                                                 |
|---------------------|----------------------------------------------------------------------|
| Fill(T)             | 将 Span\<T\> 的所有元素填充为指定值                                     |
| Clear()             | 将 Span\<T\> 的所有元素重置为类型默认值（如 int 重置为 0，string 重置为 null） |
| CopyTo(Span\<T\>)     | （虽 ReadOnlySpan\<T\> 也有，但 Span\<T\> 可复制到可写目标后修改）|

#### 案例1：Fill 填充所有元素

```csharp
// 初始化 byte 数组
byte[] buffer = new byte[5];
Span<byte> bufferSpan = buffer.AsSpan();

// 填充所有元素为 0xFF
bufferSpan.Fill(0xFF);

// 输出：255,255,255,255,255
Console.WriteLine(string.Join(",", buffer));
```

#### 案例2：Clear 重置元素为默认值

```csharp
string[] strs = { "a", "b", "c", "d" };
Span<string> strSpan = strs.AsSpan();

// 清空 Span 内所有元素（重置为 null）
strSpan.Clear();

// 输出：, , , （所有元素为 null）
Console.WriteLine(string.Join(",", strs));
```

### 3. 支持隐式转换为 ReadOnlySpan\<T\>

Span\<T\> 可隐式转为 ReadOnlySpan\<T\>（因可写包含只读能力），反之则不行，这让 Span\<T\> 适配更多只读场景：

```csharp
int[] arr = { 10, 20, 30 };
Span<int> span = arr.AsSpan();

// 隐式转换为 ReadOnlySpan<T>
ReadOnlySpan<int> readOnlySpan = span;

// 正常读取
Console.WriteLine(readOnlySpan[1]); // 输出：20

// 仍无法修改（ReadOnlySpan<T> 特性）
// readOnlySpan[1] = 200; // 编译报错
```

## 三、如何创建/转换 Span\<T\>？

与 ReadOnlySpan\<T\> 类似，Span\<T\> 无公共构造函数，需通过 MemoryExtensions.AsSpan 方法转换常见类型，低版本 .NET 同样需安装 System.Memory NuGet 包：

```bash
# 低版本 .NET 安装依赖
Install-Package System.Memory
# 或 .NET CLI
dotnet add package System.Memory
```

### 1. 数组转 Span\<T\>（最常用）

任意数组可直接转为 Span\<T\>，转换后修改 Span\<T\> 会同步修改原数组：

```csharp
// 原始数组
char[] chars = { 'H', 'e', 'l', 'l', 'o' };
// 转为 Span<char>
Span<char> charSpan = chars.AsSpan();

// 修改 Span 内容（同步修改原数组）
charSpan[4] = 'O'; // 将最后一个 'l' 改为 'O'

Console.WriteLine(string.Join("", chars)); // 输出：HellO
```

### 2. 字符串转 Span<char>（特殊注意）

字符串是**不可变类型**，因此字符串转换的 Span<char> 本质仍是只读（底层做了保护），尝试修改会抛出 System.AccessViolationException：

```csharp
string str = "Hello";
// 字符串转 Span<char>（实际是只读封装）
Span<char> strSpan = str.AsSpan();

// 以下代码会运行时报错：尝试修改只读内存
// strSpan[0] = 'h'; 
```

> 若需修改字符串内容，需先将字符串转为 char 数组，再转 Span\<T\>：
> ```csharp
> string str = "Hello";
> char[] charArr = str.ToCharArray(); // 拷贝字符串到数组（仅此处有拷贝）
> Span<char> charSpan = charArr.AsSpan();
> charSpan[0] = 'h';
> Console.WriteLine(string.Join("", charArr)); // 输出：hello
> ```

### 3. List\<T\> 转 Span\<T\>

与 ReadOnlySpan\<T\> 一致，List\<T\> 需通过 CollectionsMarshal.AsSpan（.NET 5+）或先转数组再转 Span\<T\>：

```csharp
List<int> numList = new List<int> { 1, 2, 3 };
// .NET 5+ 直接转（无拷贝）
Span<int> listSpan = CollectionsMarshal.AsSpan(numList);

// 修改 Span 内容（同步修改 List）
listSpan[1] = 200;
Console.WriteLine(numList[1]); // 输出：200
```

## 四、Span\<T\> 的核心优势（对比 ReadOnlySpan\<T\> + 传统操作）

### 1. 保留零拷贝 + 栈分配优势，新增可写能力

Span\<T\> 继承了 ReadOnlySpan\<T\> 零拷贝、栈分配无 GC 开销的特性，同时解决了 ReadOnlySpan\<T\> 无法修改数据的痛点，无需为修改数据额外创建拷贝：

```csharp
// 传统方式：修改数组片段需拷贝
int[] source = { 1, 2, 3, 4, 5 };
int[] subCopy = new int[3];
Array.Copy(source, 1, subCopy, 0, 3); // 拷贝数据
subCopy[0] = 200; // 修改拷贝后的数组，原数组无变化
Console.WriteLine(source[1]); // 输出：2

// Span<T> 方式：零拷贝修改原数组片段
Span<int> sourceSpan = source.AsSpan();
Span<int> subSpan = sourceSpan.Slice(1, 3); // 零拷贝截取
subSpan[0] = 200; // 直接修改原数组
Console.WriteLine(source[1]); // 输出：200
```

### 2. 高性能修改连续内存，避免临时对象

传统修改数组/字符串的方式易产生大量临时对象（如 string.Substring、Array.Copy），而 Span\<T\> 直接操作原内存，无额外内存分配：

```csharp
// 场景：批量修改字节数组前10个元素为 0x01
byte[] data = new byte[1000];

// 传统方式：循环赋值（无拷贝，但语法繁琐）
for (int i = 0; i < 10; i++)
{
    data[i] = 0x01;
}

// Span<T> 方式：Slice + Fill 简洁高效
data.AsSpan(0, 10).Fill(0x01); // 截取前10个元素并填充，零拷贝
```

### 3. 内存安全：越界操作直接抛异常，避免内存越访问

Span\<T\> 会校验索引/长度的合法性，越界操作（如 Slice 超出范围、索引访问越界）会立即抛出 ArgumentOutOfRangeException，相比直接操作指针更安全：

```csharp
int[] nums = { 1, 2, 3 };
Span<int> span = nums.AsSpan();

// 越界访问索引，直接抛异常
// Console.WriteLine(span[3]); // 报错：索引超出范围

// 越界 Slice，直接抛异常
// span.Slice(1, 3); // 报错：长度超出范围
```

## 五、Span\<T\> 核心方法实战（对比 ReadOnlySpan\<T\> 补充）

除了 ReadOnlySpan\<T\> 也有的 CopyTo/TryCopyTo/Slice 方法，以下聚焦 Span\<T\> 独有的写操作方法案例：

### 1. Fill：批量填充元素

适用于初始化缓冲区、重置数据等场景，比循环赋值更简洁高效：

```csharp
// 场景：初始化 1024 字节的缓冲区为 0
byte[] buffer = new byte[1024];
Span<byte> bufferSpan = buffer.AsSpan();

// 批量填充
bufferSpan.Fill(0);

// 验证：前10个元素均为 0
Console.WriteLine(string.Join(",", bufferSpan.Slice(0, 10))); // 输出：0,0,0,0,0,0,0,0,0,0
```

### 2. 索引器写操作：精准修改单个元素

适用于按需修改内存中指定位置的数据，无拷贝开销：

```csharp
// 场景：修改数组中指定位置的数值
int[] scores = { 80, 85, 90, 95 };
Span<int> scoreSpan = scores.AsSpan();

// 修改第二个分数为 99
scoreSpan[1] = 99;

// 输出：80,99,90,95
Console.WriteLine(string.Join(",", scores));
```

### 3. Clear：快速重置内存内容

适用于数据脱敏、内存回收前的重置操作，比循环赋值默认值更高效：

```csharp
// 场景：重置敏感数据（如密码数组）
char[] password = { '1', '2', '3', '4', '5' };
Span<char> pwdSpan = password.AsSpan();

// 清空密码（重置为 '\0'）
pwdSpan.Clear();

// 输出：,,,,（所有元素为默认值）
Console.WriteLine(string.Join("", password));
```

## 六、Span\<T\> 适用场景 & 注意事项

### 适用场景

1. **高频数据修改**：如网络缓冲区读写、文件流数据处理、二进制数据解析/组装；
2. **内存敏感场景**：如低延迟服务、嵌入式开发，避免 GC 开销和临时对象；
3. **数组片段修改**：无需拷贝数组，直接修改指定范围的元素。

### 注意事项

1. 栈类型限制：无法用于异步方法返回值、类字段、闭包等场景，跨上下文需改用 Memory\<T\>；
2. 字符串不可变：直接转换字符串得到的 Span<char> 不可写，修改需先转 char 数组；
3. 生命周期：Span\<T\> 引用的内存需保证在 Span\<T\> 生命周期内有效（如避免引用已释放的非托管内存、已回收的数组）。

## 总结

1. Span\<T\> 是 ReadOnlySpan\<T\> 的可写版本，核心差异是支持修改内存内容，且可隐式转为 ReadOnlySpan\<T\>；
2. 独有能力：可写索引器、Fill/Clear 等写操作方法，零拷贝修改原内存；
3. 核心优势：保留栈分配、零拷贝、低 GC 开销的同时，解锁高效可写内存操作；
4. 适用场景：高频数据修改、内存敏感场景、数组片段操作。

Span\<T\> 与 ReadOnlySpan\<T\> 配合，覆盖了 C# 中“只读”和“可写”的高性能内存操作场景，是构建高性能 .NET 应用的核心工具。

## 外部链接
[Span\<T\> 结构 - Microsoft Learn](https://learn.microsoft.com/zh-cn/dotnet/api/system.span-1?view=net-8.0)
[Memory\<T\> 结构 - Microsoft Learn](https://learn.microsoft.com/zh-cn/dotnet/api/system.memory-1?view=net-8.0)
[CollectionsMarshal 类 - Microsoft Learn](https://learn.microsoft.com/zh-cn/dotnet/api/system.runtime.interopservices.collectionsmarshal?view=net-9.0)
[System.Memory - NuGet Gallery](https://www.nuget.org/packages/System.Memory)