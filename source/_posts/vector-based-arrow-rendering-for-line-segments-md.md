---
title: 基于向量的线段箭头绘制详解
date: 2026-02-20 20:14:45
tags:
- C#
- SkiaSharp
- Vector
categories:
- C#
- Vector
---

### **最后更新时间：2026-02-20 21:08:05**

本文通过豆包辅助生成！

---

本文将详细解析如何通过向量运算在 SkiaSharp 中为线段绘制箭头，并系统讲解过程中涉及的向量数学原理与实现逻辑。

## 一、核心需求概述

你希望通过向量运算，在 SkiaSharp 的绘制场景中，为任意线段的端点（示例中为 startPos 到 endPos 的线段终点）绘制标准的箭头样式，核心是利用向量的减法、归一化、法向量、缩放、加法等运算实现箭头两个斜边的精准定位。

## 二、向量基础定义

### 1. 二维向量结构体（PEVector）

示例中定义了 PEVector 结构体来封装二维向量的属性与核心运算，核心属性和方法如下：

| 成员 | 说明 | 数学表达 |
|------|------|----------|
| X/Y | 向量的x、y分量 | 向量 $\vec{v} = (X, Y)$ |
| Length | 向量的模（长度） | $\|\vec{v}\| = \sqrt{X^2 + Y^2}$ |
| Normalize() | 向量归一化（转为单位向量） | $\hat{v} = \frac{\vec{v}}{\|\vec{v}\|}$ |
| GetNormal() | 获取向量的法向量 | 若 $\vec{v}=(x,y)$，则法向量为 $(y, -x)$ |
| Add(v1, v2) | 向量加法 | $\vec{v_1} + \vec{v_2} = (x_1+x_2, y_1+y_2)$ |
| Sub(v1, v2) | 向量减法 | $\vec{v_1} - \vec{v_2} = (x_1-x_2, y_1-y_2)$ |
| Scale(v, s) | 向量缩放 | $s \cdot \vec{v} = (s \cdot x, s \cdot y)$ |

### 2. 向量关键概念补充

- **单位向量**：模为 1 的向量，仅表示方向，无长度属性，公式：$\hat{v} = \frac{\vec{v}}{|\vec{v}|}$（要求 $|\vec{v}| \neq 0$）。
- **法向量**：与原向量垂直的向量，二维向量 $(x,y)$ 有两个正交法向量：$(y, -x)$ 和 $(-y, x)$（两者方向相反）。
- **向量减法几何意义**：$\vec{v_1} - \vec{v_2}$ 表示从 $v_2$ 指向 $v_1$ 的向量。

## 三、箭头绘制完整流程（附数学公式）

以下按代码执行顺序，拆解箭头绘制的每一步逻辑与对应的数学运算：

### 步骤1：定义基础参数与线段起点/终点

```csharp
const int Gap = 10; // 箭头斜边的长度（像素）
var startPos = new PEVector(200, 600); // 线段起点 S(200,600)
var endPos = new PEVector(600, 200);   // 线段终点 E(600,200)
canvas.DrawLine(startPos.ToSKPoint(), endPos.ToSKPoint(), linePaint); // 绘制基础线段
```

基础线段为：从点 $S(x_s, y_s)$ 到点 $E(x_e, y_e)$ 的直线段。

### 步骤2：计算线段的方向向量

```csharp
var direction = PEVector.Sub(endPos, startPos); // 从起点指向终点的方向向量
```

数学公式：
$$\vec{direction} = E - S = (x_e - x_s, y_e - y_s)$$
几何意义：该向量完全表示线段SE的方向和长度。

### 步骤3：将方向向量归一化（转为单位向量）

```csharp
direction.Normalize(); // 归一化得到单位方向向量
```

数学公式：
$$\hat{direction} = \frac{\vec{direction}}{|\vec{direction}|} = \frac{(x_e - x_s, y_e - y_s)}{\sqrt{(x_e - x_s)^2 + (y_e - y_s)^2}}$$
作用：消除长度影响，仅保留方向信息，方便后续按固定长度（Gap）偏移。

### 步骤4：计算箭头中心点（箭头斜边的交汇起点）

```csharp
var endHeadCenter = PEVector.Sub(endPos, PEVector.Scale(direction, Gap));
```

数学公式：
$$P_{center} = E - Gap \cdot \hat{direction}$$
几何意义：从线段终点 E，沿着线段反方向（-$\hat{direction}$）移动 Gap 像素，得到箭头的中心点 $P_{center}$，箭头的两个斜边将从该点向E的左右两侧延伸。

### 步骤5：计算左侧斜边的端点

```csharp
// 获取方向向量的法向量（左方向）
var directionToLeft = direction.GetNormal(); 
// 计算左侧端点：中心点 + 左法向量×Gap
var leftEndHeadPos = PEVector.Add(endHeadCenter, PEVector.Scale(directionToLeft, Gap));
// 绘制左侧斜边（从左端点到终点E）
canvas.DrawLine(leftEndHeadPos.ToSKPoint(), endPos.ToSKPoint(), linePaint);
```

数学公式：
$$\vec{n_{left}} = (y_{\hat{direction}}, -x_{\hat{direction}}) \quad \text{（左法向量）}$$
$$P_{left} = P_{center} + Gap \cdot \vec{n_{left}}$$
几何意义：左法向量与原方向向量垂直，沿该方向偏移 Gap 像素，得到箭头左侧斜边的端点 $P_{left}$，连接 $P_{left}$ 和 E 即箭头左斜边。

### 步骤6：计算右侧斜边的端点

```csharp
// 获取方向向量的右法向量（左法向量取反）
var directionToRight = PEVector.Scale(direction.GetNormal(), -1);
// 计算右侧端点：中心点 + 右法向量×Gap
var rightEndHeadPos = PEVector.Add(endHeadCenter, PEVector.Scale(directionToRight, Gap));
// 绘制右侧斜边（从右端点到终点E）
canvas.DrawLine(rightEndHeadPos.ToSKPoint(), endPos.ToSKPoint(), linePaint);
```

数学公式：
$$\vec{n_{right}} = -\vec{n_{left}} = (-y_{\hat{direction}}, x_{\hat{direction}}) \quad \text{（右法向量）}$$
$$P_{right} = P_{center} + Gap \cdot \vec{n_{right}}$$
几何意义：右法向量与左法向量方向相反，沿该方向偏移 Gap 像素，得到箭头右侧斜边的端点 $P_{right}$，连接 $P_{right}$ 和 E 即箭头右斜边。

### 步骤7（可选）：绘制辅助定位圆

通过绘制红/绿色小圆标记关键点位（$P_{center}$、$P_{left}$、$P_{right}$），便于调试箭头位置：
```csharp
if (ShowCircle)
{
    canvas.DrawCircle(endHeadCenter.ToSKPoint(), 3, new SKPaint { Color = SKColors.Red }); // 中心点（红）
    canvas.DrawCircle(leftEndHeadPos.ToSKPoint(), 3, new SKPaint { Color = SKColors.Green }); // 左端点（绿）
    canvas.DrawCircle(rightEndHeadPos.ToSKPoint(), 3, new SKPaint { Color = SKColors.Green }); // 右端点（绿）
}
```

## 四、完整代码整合
```csharp
using SkiaSharp;
using SkiaSharp.Views.Desktop;
using System;

public struct PEVector
{
    public double X;
    public double Y;

    public PEVector() { }

    public PEVector(double x, double y)
    {
        X = x;
        Y = y;
    }

    // 向量的模（长度）
    public readonly double Length => Math.Sqrt(X * X + Y * Y);

    /// <summary>
    /// 归一化：将向量转为单位向量（模≈1）
    /// </summary>
    public void Normalize()
    {
        var length = Length;
        if (length < 1e-9) return; // 避免除以0
        X /= length;
        Y /= length;
    }

    /// <summary>
    /// 获取向量的法向量（垂直向量）：(x,y) → (y, -x)
    /// </summary>
    public PEVector GetNormal()
    {
        return new PEVector { X = Y, Y = -X };
    }

    /// <summary>
    /// 向量加法
    /// </summary>
    public static PEVector Add(PEVector v1, PEVector v2)
    {
        return new PEVector { X = v1.X + v2.X, Y = v1.Y + v2.Y };
    }

    /// <summary>
    /// 向量减法
    /// </summary>
    public static PEVector Sub(PEVector v1, PEVector v2)
    {
        return new PEVector { X = v1.X - v2.X, Y = v1.Y - v2.Y };
    }

    /// <summary>
    /// 向量缩放
    /// </summary>
    public static PEVector Scale(PEVector v, double scale)
    {
        return new PEVector { X = v.X * scale, Y = v.Y * scale };
    }
}

public static class PEVectorExtensions
{
    /// <summary>
    /// 向量转为SkiaSharp的点
    /// </summary>
    public static SKPoint ToSKPoint(this PEVector v)
    {
        return new SKPoint((float)v.X, (float)v.Y);
    }
}

public class ArrowDrawing
{
    private void SKElement_PaintSurface(object sender, SKPaintSurfaceEventArgs e)
    {
        const int Gap = 10; // 箭头尺寸（像素）
        bool ShowCircle = false; // 是否显示辅助定位圆

        var canvas = e.Surface.Canvas;
        canvas.Clear();

        // 初始化画笔
        using var linePaint = new SKPaint { Color = SKColors.Black, StrokeWidth = 1 };

        // 1. 定义线段起点和终点
        var startPos = new PEVector(200, 600);
        var endPos = new PEVector(600, 200);
        // 绘制基础线段
        canvas.DrawLine(startPos.ToSKPoint(), endPos.ToSKPoint(), linePaint);

        // 2. 计算从起点到终点的方向向量
        var direction = PEVector.Sub(endPos, startPos);
        // 3. 归一化方向向量（转为单位向量）
        direction.Normalize();

        // 4. 计算箭头中心点（从终点反方向偏移Gap）
        var endHeadCenter = PEVector.Sub(endPos, PEVector.Scale(direction, Gap));

        // 5. 计算箭头左侧斜边端点并绘制
        var directionToLeft = direction.GetNormal(); // 左法向量
        var leftEndHeadPos = PEVector.Add(endHeadCenter, PEVector.Scale(directionToLeft, Gap));
        canvas.DrawLine(leftEndHeadPos.ToSKPoint(), endPos.ToSKPoint(), linePaint);

        // 6. 计算箭头右侧斜边端点并绘制
        var directionToRight = PEVector.Scale(direction.GetNormal(), -1); // 右法向量
        var rightEndHeadPos = PEVector.Add(endHeadCenter, PEVector.Scale(directionToRight, Gap));
        canvas.DrawLine(rightEndHeadPos.ToSKPoint(), endPos.ToSKPoint(), linePaint);

        // 可选：绘制辅助定位圆
        if (ShowCircle)
        {
            using var redPaint = new SKPaint { Color = SKColors.Red, StrokeWidth = 1, Style = SKPaintStyle.Fill };
            using var greenPaint = new SKPaint { Color = SKColors.Green, StrokeWidth = 1, Style = SKPaintStyle.Fill };
            canvas.DrawCircle(endHeadCenter.ToSKPoint(), 3, redPaint);
            canvas.DrawCircle(leftEndHeadPos.ToSKPoint(), 3, greenPaint);
            canvas.DrawCircle(rightEndHeadPos.ToSKPoint(), 3, greenPaint);
        }
    }
}
```

### 总结
1. **核心向量运算**：箭头绘制的关键是通过**向量减法**获取方向、**归一化**消除长度影响、**法向量**获取垂直方向、**缩放+加法**实现点位偏移；
2. **关键公式**：
   - 方向向量：$\vec{direction} = E - S$
   - 单位向量：$\hat{direction} = \vec{direction} / |\vec{direction}|$
   - 箭头中心点：$P_{center} = E - Gap \cdot \hat{direction}$
   - 左右端点：$P_{left/right} = P_{center} ± Gap \cdot (y_{\hat{direction}}, -x_{\hat{direction}})$；
3. **扩展建议**：可通过调整 Gap 参数控制箭头大小，若需为线段起点绘制箭头，仅需反转方向向量的计算逻辑（$\vec{direction} = S - E$）即可。