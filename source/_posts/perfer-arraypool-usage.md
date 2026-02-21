---
title: 尽量使用 ArrayPool 类
date: 2026-01-16 23:33:10
tags:
- C#
- Array
- ArrayPool
categories:
- C#
- Array
---

### **最后更新时间：2026-01-17 14:39:55**

---

## ArrayPool 概述

ArrayPool\<T\> 是 .NET 框架提供的数组对象池类，核心作用是通过复用数组减少内存分配与垃圾回收（GC）开销，提升高性能场景下的程序效率。

- **核心特性**

1. 避免重复创建 / 销毁数组：针对频繁使用短生命周期数组的场景（如网络传输、数据处理），从对象池获取数组，使用后归还，省去重复分配内存的成本；
2. 泛型适配：支持任意类型 \<T\>（如 ArrayPool\<int\>、ArrayPool\<byte\>），适配不同数据场景；
3. 平衡性能与内存：内置默认实现（ArrayPool\<T\>.Shared），也可自定义池大小、数组上限等配置，避免内存浪费；
4. 线程安全：默认实现支持多线程并发访问，无需额外加锁。

- **核心价值**

解决高频数组操作中“频繁分配 - 回收”导致的 GC 压力，尤其适合高并发、低延迟的应用（如 Web API、缓存服务），是 .NET 性能优化的关键工具之一。

ArrayPool 类源码如下：

```csharp
public abstract class ArrayPool<T>
{
    protected ArrayPool();
    public static ArrayPool<T> Shared { get; }
    public static ArrayPool<T> Create();
    public static ArrayPool<T> Create(int maxArrayLength, int maxArraysPerBucket);
    public abstract T[] Rent(int minimumLength);
    public abstract void Return(T[] array, bool clearArray = false);
}
```

## ArrayPool 使用教程

ArrayPool\<T\> 是.NET 中用于复用数组的工具类，核心只需掌握 Shared 实例、Rent（租借数组）、Return（归还数组） 这三个关键部分：

1. **核心对象与方法说明**

- ArrayPool\<T\>.Shared：

是框架提供的默认 ArrayPool 实例，无需手动创建，直接使用即可（也可自定义池配置，一般场景用 Shared 足够）。

- Rent(int minimumLength) 方法（租借数组）：
  - 作用：从对象池里获取一个 **长度 ≥ minimumLength** 的数组（实际长度不确定）；
  - 返回值：T[] 类型的数组，可直接用于数据操作。

- Return(T[] array) 方法（归还数组）：
  - 作用：将租借的数组还回对象池，供后续复用；
  - 关键参数 clearArray（默认 false）：
    - 设为 true：归还前会自动清空数组数据（用 Array.Clear 将所有元素置为默认值，比如 byte 数组会清为 0）；
    - 设为 false（默认）：数组数据会保留，后续租借者可能读取到旧数据，需自行注意清空。

2. 简单使用示例（以byte数组为例）

```csharp
// 1. 从默认池租借长度≥256的byte数组（实际返回256长度）
byte[] buffer = ArrayPool<byte>.Shared.Rent(256);

try
{
    // 2. 对数组进行业务操作
    buffer[0] = 123;
    buffer[1] = 234;
    buffer[2] = 255;
    // ... 其他数据处理逻辑
}
finally
{
    if (buffer != null)
    {
        // 3. 用完后归还数组（必须执行！否则池资源会泄漏）
        // 若需清空数据，加参数：ArrayPool<byte>.Shared.Return(buffer, clearArray: true)
        ArrayPool<byte>.Shared.Return(buffer);
    }
}
```

> 关于 Rent 需不需要 try-catch 的问题，具体请查看：[关于 ArrayPool 的你应该了解的内容](https://as436845345.github.io/2026/01/17/arraypool-what-you-should-know/)。

3. **关键注意事项**

- 必须归还数组：若租借后不调用 Return，对象池会逐渐耗尽可用数组，失去复用意义；
- 数组长度不固定：Rent 返回的数组长度可能大于请求的 minimumLength，操作时需以实际长度为准；
- 数据安全问题：默认 Return 不清理数据，若数组存敏感信息，务必加 clearArray: true 清空后再归还。

## 为什么尽量使用 ArrayPool 类

在.NET 中创建数组时，开发者通常会使用 new T[length] 语法，这种方式虽然简单，但在**高频创建 / 销毁短生命周期数组**的场景下，会显著影响程序性能 —— 而 ArrayPool\<T\> 正是为解决这一问题设计的内存复用工具类。

### 一、直接用 new 创建数组的性能痛点

使用 new 创建数组时，CLR（公共语言运行时）会在**托管堆（Managed Heap）**中申请一块连续的内存块，内存大小与数组长度匹配。数组不再被引用后，不会立即释放内存，需等待垃圾回收（GC）触发后才能清理。频繁创建数组时，会带来两大核心性能损耗：

1. **内存分配耗时增加**

每次用 new 创建数组，CLR 都需要在堆中查找并分配一块符合大小的连续空闲内存：

- 单次分配的耗时虽短，但高频操作（如网络 IO、数据流处理中每秒创建数百次数组）会让这部分耗时累积，成为性能瓶颈；
- 若堆中无足够连续内存，CLR 还会触发“内存压缩”（GC Compact），进一步阻塞程序执行。

2. **垃圾回收（GC）开销飙升**

- 频繁创建数组会快速消耗堆内存，导致 GC 更频繁触发（尤其是 Gen 0 代回收）；
- GC 执行时会暂停所有托管线程（STW，Stop-The-World），回收越多，程序的响应延迟和 CPU 占用率越高；
- 大数组还可能进入 Gen 1/Gen 2 代，增加全量垃圾回收的成本（Gen 2 回收耗时远高于 Gen 0）。

### 二、ArrayPool\<T\> 的核心价值：内存复用

ArrayPool\<T\> 是 .NET 提供的数组对象池，核心逻辑是“预分配一批数组存于池中，使用时租借（Rent），用完后归还（Return）”，从根本上解决上述问题：

- 避免重复分配：从池中租借数组时，直接复用已分配的内存块，无需每次在堆中查找 / 申请；
- 降低 GC 压力：数组归还后可被再次使用，不会被 GC 回收，减少堆内存占用和 GC 触发频率；
- 高性能设计：ArrayPool\<T\>.Shared 是框架内置的全局默认实例（底层为 SharedArrayPool 类），无需手动创建，开箱即用。

### 三、使用建议

1. 优先使用：在.NET Core 1.0+/.NET 5+ 等支持 ArrayPool\<T\> 的版本中，高频创建短生命周期数组（如字节数组、临时数据缓冲区）时，务必替换为 ArrayPool\<T\>；
2. 降级方案：若项目基于不支持 ArrayPool\<T\> 的旧版本（如.NET Framework 4.x），可参考 GitHub 上 .NET 官方源码，自行实现简化版的数组池（核心逻辑为“维护数组列表，租借时取可用数组，归还时回收”）。

## 外部链接

- [ArrayPool<T> 类 - Microsoft Learn](https://learn.microsoft.com/zh-cn/dotnet/api/system.buffers.arraypool-1?view=net-8.0)
- [ArrayList.cs - Github](https://github.com/dotnet/runtime/blob/main/src/libraries/System.Private.CoreLib/src/System/Buffers/ArrayPool.cs)
- [SharedArrayPool.cs - Github](https://github.com/dotnet/runtime/blob/main/src/libraries/System.Private.CoreLib/src/System/Buffers/SharedArrayPool.cs)