---
layout:     post
title:      Unity Burst 初探
subtitle:   多线程物理模拟
date:       2023-5-20
author:     Ayin
header-img: img/tag.jpg
catalog: 	 true
tags:
    - Unity
    - 多线程
---

###  前言

&emsp;&emsp;Unity DOTS是Unity官方基于ECS架构开发的一套包含Burst Complier技术和JobSystem技术面向数据的技术栈，它旨在充分利用SIMD，多线程操作充分发挥ECS的优势。ECS本人暂且没这个能力，与之关系相对没这么密的Burst和Job倒是可以单拆下来把玩把玩。

### 准备

#### 资源准备

&emsp;&emsp;在Unity的Package Manager中开启显示Preview的选项，之后安装Jobs，Burst会被一并装上。

![img](https://pic4.zhimg.com/80/v2-cd54169e0d1133ac07444a0f36656bb7_720w.webp)

#### 大致用法

```csharp
using Unity.Jobs; //定义【IJob】【IJobParallelFor】
using Unity.Burst; //定义【BurstCompile】
using Unity.Collections; //定义【NativeArray】等容器
using UnityEngine.Jobs; //定义【IJobParallelForTransform】
using Unity.Mathematics; //SIMD数学库

[BurstCompile] //Burst加速，但要求数据结构非委托
struct xx : IJob
{
    public void Execute(){}
}

[BurstCompile] //大致就是不能有GameObject，Transform等Unity的结构或套娃Native容器
struct xx : IJobParallelFor
{
    public void Execute(int i) {}
}

[BurstCompile] //用数学库的float3，float4x4即可
struct xx : IJobParallelForTransform
{
    public void Execute(int i, TransformAccess t) {}
}
```

### 小试

&emsp;&emsp;既然都并行化了，那先处理下数据，顺便和传统主线程比较下

```csharp
a = new int3[dataCount];
time = Time.realtimeSinceStartup;
for (int i = 0; i < dataCount; ++i)
    a[i] = new int3(i, i, i);
Debug.Log("顺序直接赋值" + dataCount + "个用时" + (Time.realtimeSinceStartup - time) + "秒");

b = new NativeArray<int3>(dataCount, Allocator.TempJob);
JobHandle orderHandle = new CountInOrder() { data = b }.Schedule(dataCount, 64);
time = Time.realtimeSinceStartup;
orderHandle.Complete();
Debug.Log("并行直接赋值" + dataCount + "个用时" + (Time.realtimeSinceStartup - time) + "秒");
```

&emsp;&emsp;数据开到了1e7，用时从0.03秒降到了0.01秒甚至不到，效果拔群（配置为2020年最低配游戏本，12线程）

![img](https://pic1.zhimg.com/80/v2-7a5115d456a4624937f83b90505e9280_720w.webp)

![img](https://pic3.zhimg.com/80/v2-6cb12469d56048744933174f29f70c02_720w.webp)

### 进阶

&emsp;&emsp;看起来多线程很厉害的样子，那......再玩把大的，之前有说过DynamicBone[^1]非常适合多线程化，择日不如撞日。之前1.2版本时，已有很多前人苦于其性能而将其多线程化，之后很久官方的1.3多线程版才姗姗来迟。自己写的ToyDynamicBone肯定不能和前辈以及官方的相比较，姑且以效果一致为主要目标，后续优化的话......我隐约记得标题是初探来着。

#### InitTransforms() + Prepare()

```csharp
[BurstCompile]
struct Prepare : IJobParallelForTransform
{
    public NativeArray<Particle> ps;
    public void Execute(int i, TransformAccess t)
    {
        Particle p = ps[i];
        t.localPosition = p.m_InitLocalPosition;
        t.localRotation = p.m_InitLocalRotation;
        p.m_TransformPosition = t.position;
        p.m_TransformLocalPosition = t.localPosition;
        p.m_TransformLocalToWorldMatrix = t.localToWorldMatrix;
        ps[i] = p;
    }
}
```

#### UpdateParticles1() + UpdateParticles2()

```csharp
[BurstCompile]
struct UpdateParticles : IJobParallelFor
{
    public NativeArray<Particle> ps;
    public void Execute(int i)
    {
        Particle p = ps[i];
        if (p.m_ParentIndex == -1)
        {
            p.m_PrevPosition = p.m_Position;
            p.m_Position = p.m_TransformPosition;
            return; 
        }
        Particle p0 = ps[p.m_ParentIndex];
        // verlet integration
        float3 v = p.m_Position - p.m_PrevPosition;
        p.m_PrevPosition = p.m_Position;
        p.m_Position += v * (1 - p.m_Damping);
        float restLen;
        restLen = math.length(p0.m_TransformPosition - p.m_TransformPosition);
        // keep shape
        float4x4 m0 = p.m_TransformLocalToWorldMatrix;
        m0.c3.xyz = p0.m_Position;
        float3 restPos = math.mul(m0, new float4(p.m_TransformLocalPosition, 1)).xyz;
        float3 d = restPos - p.m_Position;
        p.m_Position += d * p.m_Elasticity;
        float len = math.length(d);
        float maxlen = restLen * (1 - p.m_Stiffness) * 2;
        if (len > maxlen)
            p.m_Position += d * ((len - maxlen) / len);
        // keep length
        float3 dd = p0.m_Position - p.m_Position;
        float leng = math.length(dd);
        if (leng > 0)
            p.m_Position += dd * ((leng - restLen) / leng);
        ps[i] = p;
    }
}
```

#### SkipUpdateParticles

```csharp
[BurstCompile]
struct SkipUpdateParticles : IJobParallelFor
{
    public NativeArray<Particle> ps;
    
    public void Execute(int i)
    {
        Particle p = ps[i];
        if (p.m_ParentIndex >= 0)
        {
            Particle p0 = ps[p.m_ParentIndex];
            float restLen;
            restLen = math.length(p0.m_TransformPosition - p.m_TransformPosition);
            // keep shape
            float4x4 m0 = p.m_TransformLocalToWorldMatrix;
            m0.c3.xyz = p0.m_Position;
            float3 restPos;
            restPos = math.mul(m0, new float4(p.m_TransformLocalPosition, 1)).xyz;
            float3 d = restPos - p.m_Position;
            p.m_Position += d * p.m_Elasticity;
            d = restPos - p.m_Position;
            float len = math.length(d);
            float maxlen = restLen * (1 - p.m_Stiffness) * 2;
            if (len > maxlen)
                p.m_Position += d * ((len - maxlen) / len);
            // keep length
            float3 dd = p0.m_Position - p.m_Position;
            float leng = math.length(dd);
            if (leng > 0)
                p.m_Position += dd * ((leng - restLen) / leng);
        }
        else
        {
            p.m_PrevPosition = p.m_Position;
            p.m_Position = p.m_TransformPosition;
        }
        ps[i] = p;
    }
}
```

#### 效果对照

&emsp;&emsp;实习期间断断续续写了几天，改又用了更多的天数。在各种简化压缩下终于完成了多线程版的DynamicBone。简单搭个场景甩一甩看看效果。符合预期，毕竟也是以1.3版为对照写的。

<div align=center>
<img src="https://pic2.zhimg.com/v2-cb0461fdfd7db337c8b45849c84516f1_b.webp"/>
<center><p>左1.2版DynammicBone，中为1.3版DynamicBone，右为自己的DynamicBone</p></center>
</div> 


#### 性能对照

&emsp;&emsp;虽然不大会改（也不怎么想改），但性能对比还是很有必要的，正好见识下Job在各自为政的调度下的性能消耗。这边以脚本创建121个这种tail，然后用Unity Profiler进行分析。

|                 | CPU Consume(ms per frame) | GPU Consume(ms per frame) |
| --------------- | ------------------------- | ------------------------- |
| DynamicBone 1.2 | 9.4                       | 4.4                       |
| DynamicBone 1.3 | 9.1                       | 4.6                       |
| DynamicBone toy | 11.3                      | 3.9                       |

差距果然明显，按前辈们的做法应该再批处理调度所有动态骨骼[^2]。本来打算开摆的，良心有些看不下去，于是乎就按此继续优化，从toy版迭代到了plus版，效果拔群。

|                  | CPU Consume(ms per frame) | GPU Consume(ms per frame) |
| ---------------- | ------------------------- | ------------------------- |
| DynamicBone toy  | 11.3                      | 3.9                       |
| DynamicBone plus | 7.0                       | 4.5                       |

### 小结

&emsp;&emsp;总的来说，Unity的Job和Burst的确是性能优化的利器，并且上手不算很难。只是使用时需要用“并行化”的思维修改数据结构，同时留意计算中的小细节就好（迫真.jpg）。

### 参考

[^1]: https://zhuanlan.zhihu.com/p/49188230
[^2]: https://github.com/ldh/UnityHighPerformanceDynamicBone

