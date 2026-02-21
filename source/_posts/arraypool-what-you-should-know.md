---
title: 关于 ArrayPool 的你应该了解的内容
date: 2026-01-17 14:37:55
tags:
- C#
- Array
- ArrayPool
categories:
- C#
- Array
---

### **最后更新时间：2026-01-17 15:30:37**

---

## ArrayPool\<T\>.Rent 方法：是否需要 try-catch？源码视角的终极解答

在使用 ArrayPool\<T\>.Shared.Rent(int minimumLength) 时，很多开发者会纠结“是否需要用 try-catch 包裹”。以下结合 ArrayPool 源码，从“风险场景、报错条件、特殊情况”三方面，明确最佳实践和代码写法。

### 一、无源码认知时的安全写法

如果不了解 Rent 方法的底层逻辑，最稳妥的代码实现如下（核心是“确保数组最终归还”）：

```csharp
byte[]? buffer = null;

try
{
    // 尝试从对象池租借数组
    buffer = ArrayPool<byte>.Shared.Rent(256);

    // 业务逻辑：使用数组处理数据
    buffer[0] = 123;
    buffer[1] = 234;
    buffer[2] = 255;
    // ... 其他数据处理
}
finally
{
    // 关键：无论是否报错，都必须归还数组（避免池资源泄漏）
    if (buffer != null)
    {
        ArrayPool<byte>.Shared.Return(buffer);
    }
}
```

核心目的：不确定 Rent 是否会抛异常、业务逻辑是否会出错，用 try-finally 保证数组“租借-归还”的闭环，避免对象池资源耗尽。

### 二、源码视角：Rent 方法的核心逻辑（决定是否需要 try-catch）

通过分析 ArrayPool 源码，可明确 Rent 的报错场景、资源耗尽处理、特殊参数处理，进而判断 try-catch 的必要性。

1. **对象池“无可用数组”时：不会报错，自动新建数组**

很多人担心“频繁 Rent 不 Return，导致池为空时会报错”——但源码已处理此场景，不会抛异常：

```csharp
// 源码核心逻辑：池无可用数组时的降级处理
T[]? buffer;

// ... 尝试从对象池的对应桶中获取数组（省略池查找逻辑）

// 降级：直接新建数组（区分基元类型优化）
buffer = typeof(T).IsPrimitive && typeof(T) != typeof(bool) 
    ? GC.AllocateUninitializedArray<T>(minimumLength)  // 基元类型（除bool）：无零初始化，性能更优
    : new T[minimumLength];  // 其他类型：常规零初始化
```

结论：对象池耗尽时，Rent 会自动新建数组返回，不会抛异常。

**关键说明**：
- Type.IsPrimitive：判断类型是否为 .NET [基元类型](https://learn.microsoft.com/zh-cn/dotnet/api/system.type.isprimitive?view=net-9.0)；
- GC.AllocateUninitializedArray\<T\>：功能等价于 new T[length]，但对基元类型跳过“零初始化”（避免额外性能开销）。

2. **Rent 仅有的报错场景：minimumLength < 0**

源码明确限制“请求长度不能为负数”，否则直接抛异常：

```csharp
// 源码校验逻辑：负数长度直接报错
ArgumentOutOfRangeException.ThrowIfNegative(minimumLength);
```

**关键说明**：
- 只有当 minimumLength < 0 时，Rent 才会抛出 ArgumentOutOfRangeException；
- 无需用 try-catch 捕获此异常：调用前直接判断参数即可（异常捕获会增加性能损耗）。

3. **特殊情况：minimumLength = 0 时的处理**

当传入 minimumLength = 0（合法参数），源码会返回空数组单例，不会新建或从池获取：

```csharp
// 源码逻辑：处理 0 长度请求
else if (minimumLength == 0)
{
    // 允许 0 长度请求（虽无池复用价值，但保证 API 通用性）
    // 不会分配内存，直接返回空数组单例，归还时也不会存入池
    return Array.Empty<T>();
}
```

**关键说明**：
- 返回值是 Array.Empty\<T\>()（空数组单例），长度为 0；
- 使用时需先判断数组长度：if (buffer.Length == 0)，避免索引越界。

## 三、最终结论：是否需要 try-catch？

1. **针对 Rent 方法本身：不需要 try-catch**

- Rent 仅在 minimumLength < 0 时抛异常，可提前判断参数避免；
- 池资源耗尽时会自动新建数组，无其他报错场景。

2. **针对业务逻辑：建议用 try-finally（无需 catch）**

- 核心目的不是捕获 Rent 的异常，而是**确保数组必须归还**（即使业务逻辑报错，也不会导致池资源泄漏）；
- 优化后的最终写法（无多余 catch，仅用 try-finally 保证归还）：

```csharp
byte[]? buffer = null;

try
{
    // 提前校验参数：避免 Rent 抛出负数异常
    if (minimumLength < 0)
    {
        throw new ArgumentOutOfRangeException(nameof(minimumLength), "数组长度不能为负数");
    }

    buffer = ArrayPool<byte>.Shared.Rent(minimumLength);
    
    // 业务逻辑（可能报错，如索引越界、数据处理异常）
    if (buffer.Length > 0) // 处理 minimumLength=0 返回空数组的情况
    {
        buffer[0] = 123;
        // ... 其他逻辑
    }
}
finally
{
    // 无论业务是否报错，强制归还数组
    if (buffer != null)
    {
        ArrayPool<byte>.Shared.Return(buffer);
    }
}
```

## 四、关键补充：归还数组的注意事项

1. 即使 minimumLength=0 返回空数组，Return 也不会报错（源码会忽略空数组的归还，不会存入池）；
2. 数组归还后不可再使用：Return 后数组可能被池复用，再次操作会导致数据篡改；
3. 基元类型的性能优化：GC.AllocateUninitializedArray\<T\> 无零初始化，比 new T[length] 更快，无需手动优化。
