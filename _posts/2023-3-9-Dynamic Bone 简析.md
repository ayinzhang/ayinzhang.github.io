---
layout:     post
title:      Dynamic Bone 简析
subtitle:   经典正向运动学模拟算法
date:       2023-3-9
author:     Ayin
header-img: img/tag.jpg
catalog: 	 true
tags:
    - Unity
    - 物理模拟
---

### 前言：

&emsp;&emsp;最近接实习公司需求整Dynamic Bone，虽然因Dynamic Bone中节点距离恒定，布料模拟方面Magica Cloth效果远甚Dynamic Bone（然而性能消耗上也是），但Dynamic Bone的算法也算得上经典，也值得一写。

### 正文：

#### DataStruct

```c#
    public Transform m_Root = null;//裁切参照
    public List<Transform> m_Roots = null;//裁切距离

    public float m_UpdateRate = 60.0f;//更新速度

    public enum UpdateMode//更新模式
    {
        Normal,
        AnimatePhysics,
        UnscaledTime,
        Default
    }
    public UpdateMode m_UpdateMode = UpdateMode.Default;

    [Range(0, 1)]
    public float m_Damping = 0.1f;//阻尼
    public AnimationCurve m_DampingDistrib = null;

    [Range(0, 1)]
    public float m_Elasticity = 0.1f;//弹性
    public AnimationCurve m_ElasticityDistrib = null;

    [Range(0, 1)]
    public float m_Stiffness = 0.1f;//刚度
    public AnimationCurve m_StiffnessDistrib = null;

    [Range(0, 1)]
    public float m_Inert = 0;//惯性
    public AnimationCurve m_InertDistrib = null;

    public float m_Friction = 0;//摩擦
    public AnimationCurve m_FrictionDistrib = null;

    public float m_Radius = 0;//半径
    public AnimationCurve m_RadiusDistrib = null;

    public float m_EndLength = 0;//虚拟尾结点长度

    public Vector3 m_EndOffset = Vector3.zero;//虚拟尾结点偏移

    public Vector3 m_Gravity = Vector3.zero;//重力

    public Vector3 m_Force = Vector3.zero;//外力

    [Range(0, 1)]
    public float m_BlendWeight = 1.0f;//混合权重

    public List<DynamicBoneColliderBase> m_Colliders = null;//碰撞项集

    public List<Transform> m_Exclusions = null;//排除项集

    public enum FreezeAxis//锁定轴
    {
        None, X, Y, Z
    }
    public FreezeAxis m_FreezeAxis = FreezeAxis.None;

    public bool m_DistantDisable = false;//是否距离裁切
    public Transform m_ReferenceObject = null;//参照物
    public float m_DistanceToObject = 20;//参照物距

    [HideInInspector]
    public bool m_Multithread = true;//是否用多线程

    Vector3 m_ObjectMove;//移动方向
    Vector3 m_ObjectPrevPosition;//上帧位置
    float m_ObjectScale;//全局缩放

    float m_Time = 0;//计时器
    float m_Weight = 1.0f;//骨骼权重
    bool m_DistantDisabled = false;//是否距离裁切
    int m_PreUpdateCount = 0;//多线程计数器

    class Particle//骨骼节点
    {
        public Transform m_Transform;//节点Trans
        public int m_ParentIndex;//父节点编号
        public int m_ChildCount;///子节点个数
        public float m_Damping;//阻尼
        public float m_Elasticity;//弹性
        public float m_Stiffness;//刚度
        public float m_Inert;//惯性
        public float m_Friction;//摩擦
        public float m_Radius;//半径
        public float m_BoneLength;//骨骼长度
        public bool m_isCollide;//是否碰撞
        public bool m_TransformNotNull;//Trans是否为空

        public Vector3 m_Position;//当前位置
        public Vector3 m_PrevPosition;//上帧位置
        public Vector3 m_EndOffset;//虚拟尾节点偏移
        public Vector3 m_InitLocalPosition;//初始本地位置
        public Quaternion m_InitLocalRotation;//初始本地旋转

        //准备的数据
        public Vector3 m_TransformPosition;//Trans中位置
        public Vector3 m_TransformLocalPosition;//Trans中本地位置
        public Matrix4x4 m_TransformLocalToWorldMatrix;//Trans中本地空间到世界空间的变换矩阵
    }

    class ParticleTree//单根骨骼
    {
        public Transform m_Root;//根节点
        public Vector3 m_LocalGravity;//本地空间重力
        public Matrix4x4 m_RootWorldToLocalMatrix;//根节点世界空间到本地空间的变换矩阵
        public float m_BoneTotalLength;//骨骼总长
        public List<Particle> m_Particles = new List<Particle>();//骨骼中的节点列表

        //准备的数据
        public Vector3 m_RestGravity;//重力
    }

    List<ParticleTree> m_ParticleTrees = new List<ParticleTree>();//骨骼树

    //准备的数据
    float m_DeltaTime;//经典Δt
    List<DynamicBoneColliderBase> m_EffectiveColliders;//碰撞体列表

#if ENABLE_MULTITHREAD
    bool m_WorkAdded = false;//是否添加多线程任务
    static List<DynamicBone> s_PendingWorks = new List<DynamicBone>();//准备队列
    static List<DynamicBone> s_EffectiveWorks = new List<DynamicBone>();//就绪队列
    static AutoResetEvent s_AllWorksDoneEvent;//线程同步操作
    static int s_RemainWorkCount;//剩余工作数
    static Semaphore s_WorkQueueSemaphore;//工作队列信号量
    static int s_WorkQueueIndex;//工作队列索引
#endif

    static int s_UpdateCount;//更新数
    static int s_PrepareFrame;//准备帧
```

#### Start() 

```c#
    void Start()
    {
        SetupParticles();//设置节点
    }
```

##### SetupParticles()

```C#
    public void SetupParticles()
    {
        m_ParticleTrees.Clear();//清除骨骼树

        if (m_Root != null)//之前版本仅有m_Root没有m_Roots，此处为兼容
        {
            AppendParticleTree(m_Root);
        }

        if (m_Roots != null)//将m_Roots中根节点依次加入
        {
            for (int i = 0; i < m_Roots.Count; ++i)
            {
                Transform root = m_Roots[i];
                if (root == null)//root为空则不处理
                    continue;

                if (m_ParticleTrees.Exists(x => x.m_Root == root))//lambda去m_Roots中m_Root的重
                    continue;

                AppendParticleTree(root);//加入节点
            }
        }

        m_ObjectScale = Mathf.Abs(transform.lossyScale.x);//全局缩放
        m_ObjectPrevPosition = transform.position;//初始化上帧位置
        m_ObjectMove = Vector3.zero;//初始化移动方向

        for (int i = 0; i < m_ParticleTrees.Count; ++i)
        {
            ParticleTree pt = m_ParticleTrees[i];
            AppendParticles(pt, pt.m_Root, -1, 0);//加入子节点信息
        }

        UpdateParameters();//更新节点
    }
```

##### AppendParticleTree()

```C#
    void AppendParticleTree(Transform root)
    {
        if (root == null)
            return;

        var pt = new ParticleTree();
        pt.m_Root = root;
        pt.m_RootWorldToLocalMatrix = root.worldToLocalMatrix;
        m_ParticleTrees.Add(pt);//将各骨骼根节点加入骨骼树中
    }

    void AppendParticles(ParticleTree pt, Transform b, int parentIndex, float boneLength)
    {
        var p = new Particle();
        p.m_Transform = b;
        p.m_TransformNotNull = b != null;
        p.m_ParentIndex = parentIndex;

        if (b != null)//初始化正常节点
        {
            p.m_Position = p.m_PrevPosition = b.position;
            p.m_InitLocalPosition = b.localPosition;
            p.m_InitLocalRotation = b.localRotation;//保存本地坐标
        }
        else//创建虚拟尾节点
        {
            Transform pb = pt.m_Particles[parentIndex].m_Transform;
            if (m_EndLength > 0)
            {
                Transform ppb = pb.parent;
                if (ppb != null)
                {
                    p.m_EndOffset = pb.InverseTransformPoint((pb.position * 2 - ppb.position)) * m_EndLength;
                }
                else
                {
                    p.m_EndOffset = new Vector3(m_EndLength, 0, 0);
                }
            }
            else
            {
                p.m_EndOffset=pb.InverseTransformPoint(transform.TransformDirection(m_EndOffset)+pb.position);
            }
            p.m_Position = p.m_PrevPosition = pb.TransformPoint(p.m_EndOffset);
            p.m_InitLocalPosition = Vector3.zero;
            p.m_InitLocalRotation = Quaternion.identity;
        }

        if (parentIndex >= 0)//非根节点中骨骼长度和子节点数的计算
        {
            boneLength += (pt.m_Particles[parentIndex].m_Transform.position - p.m_Position).magnitude;
            p.m_BoneLength = boneLength;
            pt.m_BoneTotalLength = Mathf.Max(pt.m_BoneTotalLength, boneLength);
            ++pt.m_Particles[parentIndex].m_ChildCount;
        }

        int index = pt.m_Particles.Count;
        pt.m_Particles.Add(p);

        if (b != null)
        {
            for (int i = 0; i < b.childCount; ++i)
            {
                Transform child = b.GetChild(i);
                bool exclude = false;
                if (m_Exclusions != null)//遍历排除节点
                {
                    exclude = m_Exclusions.Contains(child);
                }
                if (!exclude)//无排除，递归添加质点
                {
                    AppendParticles(pt, child, index, boneLength);
                }
                else if (m_EndLength > 0 || m_EndOffset != Vector3.zero)//有排除，生成虚拟尾节点
                {
                    AppendParticles(pt, null, index, boneLength);
                }
            }

            if (b.childCount == 0 && (m_EndLength > 0 || m_EndOffset != Vector3.zero))//生成虚拟尾节点
            {
                AppendParticles(pt, null, index, boneLength);
            }
        }
    }
```

##### UpdateParameters()

```C#
    public void UpdateParameters()
    {
        SetWeight(m_BlendWeight);//设置动画物理混合参数，后续影响节点刚度

        for (int i = 0; i < m_ParticleTrees.Count; ++i)
        {
            UpdateParameters(m_ParticleTrees[i]);
        }
    }

    void UpdateParameters(ParticleTree pt)
    {
        //计算本地重力方向
        pt.m_LocalGravity=pt.m_RootWorldToLocalMatrix.MultiplyVector(m_Gravity).normalized*m_Gravity.magnitude;

        for (int i = 0; i < pt.m_Particles.Count; ++i)
        {
            Particle p = pt.m_Particles[i];
            p.m_Damping = m_Damping;
            p.m_Elasticity = m_Elasticity;
            p.m_Stiffness = m_Stiffness;
            p.m_Inert = m_Inert;
            p.m_Friction = m_Friction;
            p.m_Radius = m_Radius;

            if (pt.m_BoneTotalLength > 0)//由动画曲线和骨骼长度占比赋值
            {
                float a = p.m_BoneLength / pt.m_BoneTotalLength;
                if (m_DampingDistrib != null && m_DampingDistrib.keys.Length > 0)
                    p.m_Damping *= m_DampingDistrib.Evaluate(a);
                if (m_ElasticityDistrib != null && m_ElasticityDistrib.keys.Length > 0)
                    p.m_Elasticity *= m_ElasticityDistrib.Evaluate(a);
                if (m_StiffnessDistrib != null && m_StiffnessDistrib.keys.Length > 0)
                    p.m_Stiffness *= m_StiffnessDistrib.Evaluate(a);
                if (m_InertDistrib != null && m_InertDistrib.keys.Length > 0)
                    p.m_Inert *= m_InertDistrib.Evaluate(a);
                if (m_FrictionDistrib != null && m_FrictionDistrib.keys.Length > 0)
                    p.m_Friction *= m_FrictionDistrib.Evaluate(a);
                if (m_RadiusDistrib != null && m_RadiusDistrib.keys.Length > 0)
                    p.m_Radius *= m_RadiusDistrib.Evaluate(a);
            }
            //使参数符合物理
            p.m_Damping = Mathf.Clamp01(p.m_Damping);
            p.m_Elasticity = Mathf.Clamp01(p.m_Elasticity);
            p.m_Stiffness = Mathf.Clamp01(p.m_Stiffness);
            p.m_Inert = Mathf.Clamp01(p.m_Inert);
            p.m_Friction = Mathf.Clamp01(p.m_Friction);
            p.m_Radius = Mathf.Max(p.m_Radius, 0);
        }
    }
```

#### Update()

```C#
void Update()
    {
        if (m_UpdateMode != UpdateMode.AnimatePhysics)
        {
            PreUpdate();//初始化骨骼本地空间信息
        }

#if ENABLE_MULTITHREAD //如果开了多线程
        if (m_PreUpdateCount > 0 && m_Multithread) //需要用多线程更新
        {
            AddPendingWork(this);
            m_WorkAdded = true;
        }
#endif
        ++s_UpdateCount;
    }
```

##### PreUpdate()

```C#
    void PreUpdate()
    {
        if (IsNeedUpdate())
        {
            InitTransforms();//重置骨骼坐标信息
        }
        ++m_PreUpdateCount;
    }
```

##### InitTransforms()

```C#
void InitTransforms()
    {
        for (int i = 0; i < m_ParticleTrees.Count; ++i)
        {
            InitTransforms(m_ParticleTrees[i]);
        }
    }


void InitTransforms(ParticleTree pt)
    {
        for (int i = 0; i < pt.m_Particles.Count; ++i)
        {
            Particle p = pt.m_Particles[i];
            if (p.m_TransformNotNull)//将本地坐标改为初始值
            {
                p.m_Transform.localPosition = p.m_InitLocalPosition;
                p.m_Transform.localRotation = p.m_InitLocalRotation;
            }
        }
    }
```

##### AddPendingWork()

```C#
static void AddPendingWork(DynamicBone db)
    {
        s_PendingWorks.Add(db); //将骨骼加入等待队列
    }
```

#### LateUpdate()

```c#
void LateUpdate()
    {
        if (m_PreUpdateCount == 0)
            return;

        if (s_UpdateCount > 0)
        {
            s_UpdateCount = 0;
            ++s_PrepareFrame;
        }

        SetWeight(m_BlendWeight);//设置动画和物理的混合权重

#if ENABLE_MULTITHREAD
        if (m_WorkAdded) //如果之前加骨骼进多线程队列
        {
            m_WorkAdded = false;//置回原状态
            ExecuteWorks();//执行多线程
        }
        else
#endif
        {
            CheckDistance();//距离判断
            if (IsNeedUpdate())//是否需要物理计算
            {
                Prepare();////对数据结构中准备的数据的处理
                UpdateParticles();//对骨骼进行物理模拟
                ApplyParticlesToTransforms();//应用物理模拟结果
            }
        }

        m_PreUpdateCount = 0;
    }
```

##### SetWeight()

```C#
    public void SetWeight(float w)
    {
        if (m_Weight != w)
        {
            if (w == 0)
            {
                InitTransforms();//重置骨骼坐标信息
            }
            else if (m_Weight == 0)
            {
                ResetParticlesPosition();//重置当前位置坐标
            }
            m_Weight = m_BlendWeight = w;//m_Weight是之前版本留下的接口，此处仅是接入
        }
    }
```

##### ResetParticlesPosition()

```c#
    void ResetParticlesPosition()
    {
        for (int i = 0; i < m_ParticleTrees.Count; ++i)
        {
            ResetParticlesPosition(m_ParticleTrees[i]);
        }

        m_ObjectPrevPosition = transform.position;//留存上帧信息
    }

    void ResetParticlesPosition(ParticleTree pt)
    {
        for (int i = 0; i < pt.m_Particles.Count; ++i)
        {
            Particle p = pt.m_Particles[i];
            if (p.m_TransformNotNull)//当前位置和上帧位置都置为Trans值
            {
                p.m_Position = p.m_PrevPosition = p.m_Transform.position;
            }
            else //虚拟尾节点处理
            {
                Transform pb = pt.m_Particles[p.m_ParentIndex].m_Transform;
                p.m_Position = p.m_PrevPosition = pb.TransformPoint(p.m_EndOffset);
            }
            p.m_isCollide = false;
        }
    }
```

##### CheckDistance()

```c#
    void CheckDistance()
    {
        if (!m_DistantDisable)
            return;

        Transform rt = m_ReferenceObject;
        if (rt == null && Camera.main != null)
        {
            rt = Camera.main.transform;//没有参照物就设为相机Trans
        }

        if (rt != null)
        {
            float d2 = (rt.position - transform.position).sqrMagnitude;
            bool disable = d2 > m_DistanceToObject * m_DistanceToObject;//是否超出距离
            if (disable != m_DistantDisabled)
            {
                if (!disable)
                {
                    ResetParticlesPosition();//重置当前位置坐标
                }
                m_DistantDisabled = disable;
            }
        }
    }
```

##### IsNeedUpdate()

```c#
    bool IsNeedUpdate()
    {
        return m_Weight > 0 && !(m_DistantDisable && m_DistantDisabled);
    }
```

##### Prepare()

```c#
    void Prepare()
    {
        m_DeltaTime = Time.deltaTime;
#if UNITY_5_3_OR_NEWER
        if (m_UpdateMode == UpdateMode.UnscaledTime)
        {
            m_DeltaTime = Time.unscaledDeltaTime;
        }
        else if (m_UpdateMode == UpdateMode.AnimatePhysics)
        {
            m_DeltaTime = Time.fixedDeltaTime * m_PreUpdateCount;
        }
#endif

        m_ObjectScale = Mathf.Abs(transform.lossyScale.x);//scale
        m_ObjectMove = transform.position - m_ObjectPrevPosition;//移动向量
        m_ObjectPrevPosition = transform.position;//上帧位置

        for (int i = 0; i < m_ParticleTrees.Count; ++i)
        {
            ParticleTree pt = m_ParticleTrees[i];
            pt.m_RestGravity = pt.m_Root.TransformDirection(pt.m_LocalGravity);//重力由本地坐标转为世界坐标

            for (int j = 0; j < pt.m_Particles.Count; ++j)
            {
                Particle p = pt.m_Particles[j];
                if (p.m_TransformNotNull)//记录相关数据
                {
                    p.m_TransformPosition = p.m_Transform.position;
                    p.m_TransformLocalPosition = p.m_Transform.localPosition;
                    p.m_TransformLocalToWorldMatrix = p.m_Transform.localToWorldMatrix;
                }
            }
        }

        if (m_EffectiveColliders != null)//销毁碰撞体列表
        {
            m_EffectiveColliders.Clear();
        }

        if (m_Colliders != null)//新建碰撞体列表
        {
            for (int i = 0; i < m_Colliders.Count; ++i)
            {
                DynamicBoneColliderBase c = m_Colliders[i];
                if (c != null && c.enabled)
                {
                    if (m_EffectiveColliders == null)
                    {
                        m_EffectiveColliders = new List<DynamicBoneColliderBase>();
                    }
                    m_EffectiveColliders.Add(c);

                    if (c.PrepareFrame != s_PrepareFrame)//对碰撞体仅初始化一次
                    {
                        c.Prepare();
                        c.PrepareFrame = s_PrepareFrame;
                    }
                }
            }
        }
    }
```

##### UpdateParticles()

```c#
    void UpdateParticles()
    {
        if (m_ParticleTrees.Count <= 0)
            return;

        int loop = 1;
        float timeVar = 1;
        float dt = m_DeltaTime;

        if (m_UpdateMode == UpdateMode.Default)//帧率的处理
        {
            if (m_UpdateRate > 0)
            {
                timeVar = dt * m_UpdateRate;
            }
        }
        else
        {
            if (m_UpdateRate > 0)//模拟时步的调控
            {
                float frameTime = 1.0f / m_UpdateRate;
                m_Time += dt;
                loop = 0;

                while (m_Time >= frameTime)
                {
                    m_Time -= frameTime;
                    if (++loop >= 3)
                    {
                        m_Time = 0;
                        break;
                    }
                }
            }
        }

        if (loop > 0)//模拟迭代次数
        {
            for (int i = 0; i < loop; ++i)
            {
                UpdateParticles1(timeVar, i);//惯性及受力
                UpdateParticles2(timeVar);//弹性及刚度
            }
        }
        else
        {
            SkipUpdateParticles();//跳过模拟
        }
    }
```

##### UpdateParticles1()

```c#
    void UpdateParticles1(float timeVar, int loopIndex)
    {
        for (int i = 0; i < m_ParticleTrees.Count; ++i)
        {
            UpdateParticles1(m_ParticleTrees[i], timeVar, loopIndex);
        }
    }

    void UpdateParticles1(ParticleTree pt, float timeVar, int loopIndex)
    {
        Vector3 force = m_Gravity;//重力
        Vector3 fdir = m_Gravity.normalized;//重力单位向量
        Vector3 pf = fdir * Mathf.Max(Vector3.Dot(pt.m_RestGravity, fdir), 0);//投影重力
        force -= pf;//去除重力影响
        force = (force + m_Force) * (m_ObjectScale * timeVar);//应用time和scale

        Vector3 objectMove = loopIndex == 0 ? m_ObjectMove : Vector3.zero;//仅初次循环改变objectMove

        for (int i = 0; i < pt.m_Particles.Count; ++i)
        {
            Particle p = pt.m_Particles[i];
            if (p.m_ParentIndex >= 0)
            {
                //韦尔莱积分
                Vector3 v = p.m_Position - p.m_PrevPosition;//节点运动方向
                Vector3 rmove = objectMove * p.m_Inert;//根节点运动方向乘以惯性
                p.m_PrevPosition = p.m_Position + rmove;//原位置和根节点运动方向记为上一帧位置
                float damping = p.m_Damping;
                if (p.m_isCollide)//如果发生碰撞
                {
                    damping += p.m_Friction;//摩擦作用于阻尼
                    if (damping > 1)
                    {
                        damping = 1;
                    }
                    p.m_isCollide = false;//重置碰撞检测
                }
                p.m_Position += v * (1 - damping) + force + rmove;//计算节点计算完惯性和外力后的位置
            }
            else
            {
                p.m_PrevPosition = p.m_Position;
                p.m_Position = p.m_TransformPosition;
            }
        }
    }
```

##### UpdateParticles2()

```c#
    void UpdateParticles2(float timeVar)
    {
        for (int i = 0; i < m_ParticleTrees.Count; ++i)
        {
            UpdateParticles2(m_ParticleTrees[i], timeVar);
        }
    }

    void UpdateParticles2(ParticleTree pt, float timeVar)
    {
        var movePlane = new Plane();

        for (int i = 1; i < pt.m_Particles.Count; ++i)
        {
            Particle p = pt.m_Particles[i];//节点
            Particle p0 = pt.m_Particles[p.m_ParentIndex];//父节点

            float restLen;//父子节点骨骼长度
            if (p.m_TransformNotNull)//正常节点
            {
                restLen = (p0.m_TransformPosition - p.m_TransformPosition).magnitude;
            }
            else//虚拟尾节点
            {
                restLen = p0.m_TransformLocalToWorldMatrix.MultiplyVector(p.m_EndOffset).magnitude;
            }

            //形状保持
            float stiffness = Mathf.Lerp(1.0f, p.m_Stiffness, m_Weight);
            if (stiffness > 0 || p.m_Elasticity > 0)
            {
                Matrix4x4 m0 = p0.m_TransformLocalToWorldMatrix;//父节点局部坐标到世界坐标的矩阵
                m0.SetColumn(3, p0.m_Position);//m0即为节点局部坐标到世界坐标的矩阵
                Vector3 restPos;//回归坐标
                if (p.m_TransformNotNull)
                {
                    restPos = m0.MultiplyPoint3x4(p.m_TransformLocalPosition);
                }
                else
                {
                    restPos = m0.MultiplyPoint3x4(p.m_EndOffset);
                }

                Vector3 d = restPos - p.m_Position;//回归向量
                p.m_Position += d * (p.m_Elasticity * timeVar);//应用弹性

                if (stiffness > 0)//刚性运动
                {
                    d = restPos - p.m_Position;
                    float len = d.magnitude;
                    float maxlen = restLen * (1 - stiffness) * 2;//计算理论长度
                    if (len > maxlen)
                    {
                        p.m_Position += d * ((len - maxlen) / len);//坐标偏移
                    }
                }
            }

            //碰撞
            if (m_EffectiveColliders != null)
            {
                float particleRadius = p.m_Radius * m_ObjectScale;
                for (int j = 0; j < m_EffectiveColliders.Count; ++j)
                {
                    DynamicBoneColliderBase c = m_EffectiveColliders[j];
                    p.m_isCollide |= c.Collide(ref p.m_Position, particleRadius);
                }
            }

            //锁轴处理
            if (m_FreezeAxis != FreezeAxis.None)
            {
                Vector3 planeNormal=p0.m_TransformLocalToWorldMatrix.GetColumn((int)m_FreezeAxis-1).normalized;
                movePlane.SetNormalAndPosition(planeNormal, p0.m_Position);
                p.m_Position -= movePlane.normal * movePlane.GetDistanceToPoint(p.m_Position);
            }

            //长度保持
            Vector3 dd = p0.m_Position - p.m_Position;//子节点更新后的骨骼方向
            float leng = dd.magnitude;//更新后的骨骼长度
            if (leng > 0)
            {
                p.m_Position += dd * ((leng - restLen) / leng);//更新坐标，使骨骼长度一致
            }
        }
    }
```

##### ApplyParticlesToTransforms()

```c#
    void ApplyParticlesToTransforms()
    {
        Vector3 ax = Vector3.right;
        Vector3 ay = Vector3.up;
        Vector3 az = Vector3.forward;
        bool nx = false, ny = false, nz = false;

#if !UNITY_5_4_OR_NEWER
        Vector3 lossyScale = transform.lossyScale;
        if (lossyScale.x < 0 || lossyScale.y < 0 || lossyScale.z < 0)
        {
            Transform mirrorObject = transform;
            do
            {
                Vector3 ls = mirrorObject.localScale;
                nx = ls.x < 0;
                if (nx)
                    ax = mirrorObject.right;
                ny = ls.y < 0;
                if (ny)
                    ay = mirrorObject.up;
                nz = ls.z < 0;
                if (nz)
                    az = mirrorObject.forward;
                if (nx || ny || nz)
                    break;

                mirrorObject = mirrorObject.parent;
            }
            while (mirrorObject != null);
        }
#endif

        for (int i = 0; i < m_ParticleTrees.Count; ++i)
        {
            ApplyParticlesToTransforms(m_ParticleTrees[i], ax, ay, az, nx, ny, nz);
        }
    }

    void ApplyParticlesToTransforms(ParticleTree pt,Vector3 ax,Vector3 ay,Vector3 az,bool nx,bool ny,bool nz)
    {
        for (int i = 1; i < pt.m_Particles.Count; ++i)
        {
            Particle p = pt.m_Particles[i];
            Particle p0 = pt.m_Particles[p.m_ParentIndex];

            if (p0.m_ChildCount <= 1)//仅单个子节点时修改骨骼方向
            {
                Vector3 localPos;
                if (p.m_TransformNotNull)//正常节点
                {
                    localPos = p.m_Transform.localPosition;
                }
                else//虚拟尾节点
                {
                    localPos = p.m_EndOffset;
                }
                Vector3 v0 = p0.m_Transform.TransformDirection(localPos);
                Vector3 v1 = p.m_Position - p0.m_Position;
#if !UNITY_5_4_OR_NEWER
                if (nx)
                    v1 = MirrorVector(v1, ax);
                if (ny)
                    v1 = MirrorVector(v1, ay);
                if (nz)
                    v1 = MirrorVector(v1, az);
#endif
                Quaternion rot = Quaternion.FromToRotation(v0, v1);
                p0.m_Transform.rotation = rot * p0.m_Transform.rotation;//处理父节点旋转
            }

            if (p.m_TransformNotNull)//正常子节点
            {
                p.m_Transform.position = p.m_Position;//应用模拟结果
            }
        }
    }
```

### 总结：

&emsp;&emsp;该插件上次更新1.3.2版本还是一年多前，现在应该是入土了吧。解析Dynamic bone的文章前面已有许多（但没有新版），所以我务于全，展现了该脚本整个基本流程函数（其他未展现的流程函数基本也都调用前文出现过的子函数），希望对读者能有些许帮助。

&emsp;&emsp;总体而言，开始先构造了骨骼的链式结构，记录本地坐标，之后由根到尾一级一级的应用惯性，摩擦，外力，弹力，刚度处理更新位置。如果仍有疑惑，也可移步知乎其他文章。如核心代码的详解：[DynamicBone（动态骨骼）源码赏析](https://zhuanlan.zhihu.com/p/522873426)，模拟算法的讲解：[动态骨骼Dynamic Bone算法详解 ](https://zhuanlan.zhihu.com/p/49188230)。