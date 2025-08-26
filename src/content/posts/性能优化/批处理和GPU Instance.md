---
title: 批处理和GPU Instance
published: 2025-08-25
category: 性能优化
tags: [性能优化,批处理,GPU Instance]
draft: false
---



# 🎯 Unity 批处理与渲染优化方案详解

Unity 提供了多种 **减少 Draw Call 开销** 的方案，它们分别作用于不同层面：  
- **静态合批（Static Batching）**  
- **动态合批（Dynamic Batching）**  
- **GPU Instancing**  
- **SRP Batcher**  

下面分别详解。

---

## 1. 静态合批（Static Batching）

### ✅ 原理
- Unity 在构建或运行时，把 **勾选 Static 的物体 Mesh 数据合并**成一个大 Mesh。  
- 渲染时，它们被当作一个整体绘制，从而减少 **Draw Call**。  

### ⚙️ 条件
- 物体必须勾选 **Static（Batching Static）**。  
- 使用相同材质。  
- Mesh 合并后顶点数有限制（老版本 65k，新版支持 32bit Index 更大）。  
- 只允许 **等比缩放**，非等比缩放会打破合批。  
- Shader 必须支持合批。  

### 👍 优点
- 大幅减少 Draw Call。  
- 适合大量静态物体。  

### 👎 缺点
- 会占用 **额外内存**（复制 Mesh 数据）。  
- 物体不能移动/旋转，否则破坏合批。  

### 🔥 应用场景
- 建筑、地形碎块、树木、石头（不会动）。  

---

## 2. 动态合批（Dynamic Batching）

### ✅ 原理
- Unity 在 **CPU 端运行时**，把 **相同材质的小型动态物体** 合并成一个批次提交给 GPU。  

### ⚙️ 条件
- 使用相同材质。  
- Mesh 必须很小：  
  - 有顶点色时，每个物体顶点数 < **300**。  
  - 无顶点色时，每个物体顶点数 < **900**。  
- Shader 必须支持动态合批。  

### 👍 优点
- 对小型动态物体友好。  
- 无需勾选 Static。  

### 👎 缺点
- **CPU 开销高**（运行时需要合并顶点数据）。  
- 物体一旦多或复杂，会拖慢 CPU。  
- 只适合小 Mesh。  

### 🔥 应用场景
- UI 特效的小物件。  
- 子弹、粒子碎片（顶点少）。  

---

## 3. GPU Instancing

### ✅ 原理
- 让 GPU **一次性绘制多个相同 Mesh & 材质的物体**。  
- 通过 **InstanceID** 给每个实例传递不同属性（位置、颜色等）。  

### ⚙️ 条件
- 使用相同材质，并在材质上勾选 **Enable GPU Instancing**。  
- Shader 必须支持 Instancing（`#pragma multi_compile_instancing`）。  
- Mesh 大小不限。  
- 动态物体也可使用（移动、旋转、缩放都行）。  

### 👍 优点
- **高效**，特别适合大量重复物体。  
- 几乎没有 CPU 负担。  
- 动态物体也能合批。  

### 👎 缺点
- **材质属性不能完全独立**（不同贴图会破坏合批）。  
- 每个实例差异大时需要额外传 buffer。  
- 部分旧移动 GPU 支持较差。  

### 🔥 应用场景
- 大量动态重复物体：树木、草丛、敌人、特效粒子。  
- 《塞尔达荒野之息》那种成千上万棵树草。  

---

## 4. SRP Batcher

### ✅ 原理
- 作用于 **Scriptable Render Pipeline (SRP)**（URP/HDRP）。  
- 把 **材质和 Shader 的常量数据** 打包到统一的 **GPU 缓冲区**。  
- CPU 绘制时，只需告诉 GPU 使用缓冲区的哪一段数据，而不是每次重新上传全部参数。  
- **降低 CPU → GPU 通信开销**，让 Draw Call 更便宜。  

### ⚙️ 条件
- 必须使用 **SRP（URP/HDRP）**，内置管线不支持。  
- Shader 必须兼容 SRP Batcher：  
  - 使用 `CBUFFER_START(UnityPerMaterial)` / `CBUFFER_END`。  
  - Unity 官方 SRP Shader 默认支持。  
- 支持 float、vector、matrix，不支持数组/结构体。  

### 👍 优点
- **大幅降低 CPU 开销**（尤其几千上万个 Renderer 时）。  
- 适用范围广（静态 + 动态物体都能受益）。  
- 自动生效，无需手动管理。  

### 👎 缺点
- 仅支持 SRP（URP/HDRP），内置管线不可用。  
- 自定义 Shader 需要正确写 CBUFFER。  
- 材质属性支持有限（不支持复杂数组）。  

### 🔥 应用场景
- URP/HDRP 项目中所有物体（普遍受益）。  

---

## 📊 四者对比表

| 特性 | 静态合批 (Static) | 动态合批 (Dynamic) | GPU Instancing | SRP Batcher |
|------|-------------------|---------------------|----------------|-------------|
| 是否需要勾选 Static | ✅ 必须 | ❌ 不需要 | ❌ 不需要 | ❌ 不需要 |
| 物体是否可动 | ❌ 不可动（仅等比缩放） | ✅ 可动 | ✅ 可动 | ✅ 可动 |
| Mesh 大小限制 | 有（顶点数） | 严格（300/900 顶点） | 无限制 | 无限制 |
| 材质要求 | 相同材质 | 相同材质 | 相同材质（开启 Instancing） | Shader 支持 SRP Batcher |
| CPU 开销 | 中等（合并 Mesh） | 高（运行时合并） | 低（GPU 执行） | **最低（CPU 优化）** |
| GPU 开销 | 低 | 较低 | 最低 | 较低 |
| 内存占用 | 高（复制 Mesh） | 低 | 低 | 低 |
| 支持管线 | 内置 / URP / HDRP | 内置 / URP / HDRP | 内置 / URP / HDRP | **仅 URP/HDRP** |
| 典型用途 | 静态场景物体 | 小动态物体 | 大量重复物体 | SRP 项目的通用优化 |

有了 SRP Batcher ≠ 完全不用 静态合批 / 动态合批 / GPU Instancing。
它们优化的 方向不同，有 叠加关系，而不是谁替代谁。
- SRP Batcher 不减少 Draw Call 数量。  
- 静态合批依然能减少调用次数，**两者可以叠加使用**。  
- 动态合批的收益变低，反而增加 CPU 负担。  
- **URP/HDRP 官方推荐：关闭动态合批**。  

👉 总结：  
**SRP Batcher 并不是替代，而是 CPU 优化器。**  
在 SRP 项目里，最佳组合是：  
- **静态物体 → 静态合批 + SRP Batcher**  
- **动态重复物体 → GPU Instancing + SRP Batcher**

---

## ⚡ 最佳实践总结
- **静态物体（房子、树、石头） → 静态合批**  
- **少量小动态物体（粒子碎片、道具） → 动态合批**  
- **大量重复动态物体（树海、草丛、敌人） → GPU Instancing**  
- **URP/HDRP 项目 → 默认开启 SRP Batcher**  
