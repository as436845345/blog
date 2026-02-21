---
title: 禁止使用 ArrayList 类
date: 2026-01-16 23:15:08
tags:
- C#
- Array
- ArrayList
categories:
- C#
- Array
---

## 深入理解 C# ArrayList

ArrayList 是 C# 中的一个动态列表类，其内部通过维护一个 object 类型的数组 _items 来存储数据。当你调用 Add 方法添加元素时，数据就被存入这个数组中。

它的存储字段如下：

```csharp
private object?[] _items;  // 存储元素的底层数组
```

- **Add 方法的工作原理**

Add 方法负责将元素追加到列表末尾，其参数类型为 object：

```csharp
public virtual int Add(object? value);
```

## 装箱

因为 ArrayList 内部存储的是 object 类型（引用类型），当你向其中添加值类型（如 int、char、double）时，会触发 **装箱（boxing）操作；当你从列表中取出值类型时，则会触发拆箱（unboxing）** 操作。

- ArrayList 测试代码

```csharp
public void M()
{
    ArrayList arrayList = new ArrayList();
    arrayList.Add(1);      // int 类型值 → 装箱为 object
    arrayList.Add('a');    // char 类型值 → 装箱为 object
    arrayList.Add(3.0);    // double 类型值 → 装箱为 object
}
```

- ArrayList 测试代码编译为 IL

上述 C# 代码编译为 IL 后，我们可以清晰看到装箱的过程：

![ArrayList 测试代码编译为 IL](../images/arraylist/arraylist_add_valuetype_il_code.png)

## 装箱的性能影响

**装箱**是指将值类型转换为引用类型（object）的过程。

在上面的例子中，添加值类型 1（int）时，IL 会先执行 box [System.Runtime]System.Int32 指令，再调用 Add(object) 方法。这个 box 指令本身会占用 5 个字节的 IL 代码空间（1 字节操作码 + 4 字节元数据令牌），而如果添加的是引用类型，则不会有这个额外开销。

频繁的装箱 / 拆箱会增加内存分配和类型转换的开销，从而影响程序性能。因此在实际开发中，应尽量避免不必要的装箱操作，推荐使用泛型集合（如 List\<T\>），通过指定具体的存储类型，从根本上避免装箱 / 拆箱。

## 外部链接

- [C# 类型系统 - Microsoft Learn](https://learn.microsoft.com/zh-cn/dotnet/csharp/fundamentals/types/)
- [装箱和取消装箱（C# 编程指南） - Microsoft Learn](https://learn.microsoft.com/zh-cn/dotnet/csharp/programming-guide/types/boxing-and-unboxing)
- [IL-box 字段 - Microsoft Learn](https://learn.microsoft.com/zh-cn/dotnet/api/system.reflection.emit.opcodes.box?view=net-9.0)
- [ArrayList.cs - Github](https://github.com/dotnet/runtime/blob/main/src/libraries/System.Private.CoreLib/src/System/Collections/ArrayList.cs)
- [测试代码 - sharplab](https://sharplab.io/#v2:C4LglgNgPgAgTARgLACgYAYAEMEDoDCA9hBAKYDGwYhAdgM4DcqMAzNnJvpgN6qb/Y2MACyYAsgAoAlDz4D5AQQBOSgIYBPADJg6wTKpUbtuzAF5MNUgHdMytVp3BpTFPPkH7x4LgUATXxIIUi5uAh5Gjj7+EgDkqjHBcqHhDrpRASy46ImuAgC+qHlAA===)
