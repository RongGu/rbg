# RBG 架构图

本文档展示了 RoleBasedGroup (RBG) 项目的整体架构设计和核心组件。

## 整体架构

```mermaid
graph TB
    subgraph "Kubernetes 集群"
        subgraph "RBG 控制平面"
            Controller["RBG Controller<br/>核心编排控制器"]
            EngineRuntime["Engine Runtime Manager<br/>推理引擎运行时管理器"]
            
            Controller -->|管理| EngineRuntime
        end
        
        subgraph "RBG 核心功能"
            MultiRole["多角色编排<br/>Multi-Role Orchestration"]
            ServiceDiscovery["服务发现<br/>Service Discovery"]
            Scheduling["调度协同<br/>Gang Scheduling"]
            Update["滚动更新<br/>Rolling Update"]
            FailureHandling["故障恢复<br/>Failure Handling"]
            Scaling["弹性伸缩<br/>Elastic Scaling"]
        end
        
        subgraph "Kubernetes 资源"
            RBG["RoleBasedGroup<br/>CRD"]
            STS["StatefulSet"]
            LWS["LeaderWorkerSet"]
            Deploy["Deployment"]
            Pods["Pods"]
            ConfigMap["ConfigMap"]
            Service["Service"]
        end
        
        subgraph "调度插件"
            KubeScheduling["Kube Scheduling<br/>Plugin"]
            VolcanoScheduling["Volcano Scheduling<br/>Plugin"]
        end
        
        subgraph "LLM 推理场景 - PD 分离架构"
            Router["Router/Scheduler<br/>路由调度器"]
            Prefill["Prefill 角色<br/>预填充引擎"]
            Decode["Decode 角色<br/>解码引擎"]
            
            Router -->|请求路由| Prefill
            Router -->|请求路由| Decode
            Prefill -->|KV Cache 传输| Decode
        end
        
        Controller -->|创建和管理| RBG
        RBG -->|定义| MultiRole
        RBG -->|定义| ServiceDiscovery
        RBG -->|配置| Scheduling
        RBG -->|配置| Update
        RBG -->|配置| FailureHandling
        RBG -->|支持| Scaling
        
        MultiRole -->|创建| STS
        MultiRole -->|创建| LWS
        MultiRole -->|创建| Deploy
        STS -->|管理| Pods
        LWS -->|管理| Pods
        Deploy -->|管理| Pods
        
        ServiceDiscovery -->|注入配置| ConfigMap
        ServiceDiscovery -->|创建| Service
        ConfigMap -->|挂载到| Pods
        Service -->|服务发现| Pods
        
        Scheduling -->|集成| KubeScheduling
        Scheduling -->|集成| VolcanoScheduling
        
        EngineRuntime -->|注入容器| Pods
        EngineRuntime -->|管理| Router
        EngineRuntime -->|管理| Prefill
        EngineRuntime -->|管理| Decode
    end
    
    User["用户/应用"]
    User -->|创建 RBG 资源| RBG
    User -->|推理请求| Router

```

## 核心组件详解

### 1. RBG Controller (核心编排控制器)

RBG Controller 是项目的核心组件，负责：

- **多角色编排**: 管理复杂拓扑中的多个角色（如 Prefill、Decode、Router 等）
- **依赖关系管理**: 处理角色之间的启动顺序依赖
- **生命周期管理**: 协调多个角色的创建、更新、删除
- **配置管理**: 统一管理分布式系统的配置
- **调度协同**: 与 Kubernetes 调度器插件集成，支持 Gang Scheduling
- **滚动更新**: 以角色为单位进行原子化升级
- **故障恢复**: 触发角色级别的故障恢复机制

```mermaid
graph LR
    subgraph "RBG Controller 工作流程"
        Watch["监听 RBG CRD"]
        Reconcile["调谐循环"]
        Validate["验证配置"]
        CreateUpdate["创建/更新资源"]
        Status["更新状态"]
        
        Watch --> Reconcile
        Reconcile --> Validate
        Validate --> CreateUpdate
        CreateUpdate --> Status
        Status --> Reconcile
    end
```

### 2. Engine Runtime Manager (推理引擎运行时管理器)

Engine Runtime Manager 通过 `ClusterEngineRuntimeProfile` CRD 为大模型推理场景提供支持：

- **容器注入**: 向 Pod 中注入推理引擎相关的 sidecar 容器
- **模型切换**: 支持动态模型加载和切换
- **服务发现**: 在推理引擎组件之间提供服务发现机制
- **资源规划**: 为 PD 分离场景规划 Prefill 和 Decode 角色的资源配比
- **运行时配置**: 管理推理引擎的运行时参数和环境变量

```mermaid
graph TB
    subgraph "Engine Runtime Manager 功能"
        Profile["ClusterEngineRuntimeProfile"]
        Injector["容器注入器"]
        Discovery["服务发现"]
        Config["配置管理"]
        
        Profile --> Injector
        Profile --> Discovery
        Profile --> Config
        
        Injector -->|注入 InitContainers| Pod1["Pod"]
        Injector -->|注入 Containers| Pod1
        Injector -->|挂载 Volumes| Pod1
        
        Discovery -->|环境变量注入| Pod1
        Config -->|ConfigMap| Pod1
    end
```

## LLM 推理场景 - PD 分离架构

RBG 特别适用于大语言模型的 Prefill-Decode 分离部署场景：

```mermaid
graph TB
    subgraph "PD 分离推理架构"
        Client["客户端请求"]
        
        subgraph "RoleBasedGroup: sglang-pd"
            subgraph "Router 角色"
                RouterPod["Router Pod<br/>负载均衡和调度"]
            end
            
            subgraph "Prefill 角色 (高算力)"
                PrefillPod1["Prefill Pod 1<br/>首 Token 生成"]
                PrefillPod2["Prefill Pod 2<br/>首 Token 生成"]
            end
            
            subgraph "Decode 角色 (高吞吐)"
                DecodePod1["Decode Pod 1<br/>自回归解码"]
                DecodePod2["Decode Pod 2<br/>自回归解码"]
                DecodePod3["Decode Pod 3<br/>自回归解码"]
            end
        end
        
        Client -->|推理请求| RouterPod
        RouterPod -->|Prefill 任务| PrefillPod1
        RouterPod -->|Prefill 任务| PrefillPod2
        
        PrefillPod1 -->|KV Cache| DecodePod1
        PrefillPod1 -->|KV Cache| DecodePod2
        PrefillPod2 -->|KV Cache| DecodePod2
        PrefillPod2 -->|KV Cache| DecodePod3
        
        DecodePod1 -->|生成结果| Client
        DecodePod2 -->|生成结果| Client
        DecodePod3 -->|生成结果| Client
    end
    
    RBGController["RBG Controller"]
    EngineManager["Engine Runtime Manager"]
    
    RBGController -->|编排三个角色| RouterPod
    RBGController -->|管理依赖关系| PrefillPod1
    RBGController -->|自动服务发现| DecodePod1
    
    EngineManager -->|注入 Patio 容器| PrefillPod1
    EngineManager -->|注入 Patio 容器| DecodePod1
    EngineManager -->|配置推理引擎| RouterPod
```

### PD 分离的优势

1. **解耦 Prefill 和 Decode**: 分别优化两个阶段的资源配置和并行策略
2. **提高资源利用率**: Prefill 需要高算力，Decode 需要高吞吐，独立扩缩容
3. **降低干扰**: 避免 Prefill 和 Decode 的计算相互干扰
4. **灵活配比**: 根据业务特点调整 Prefill 和 Decode 的 Pod 数量比例

## 关键特性流程图

### 多角色启动顺序

```mermaid
sequenceDiagram
    participant User
    participant RBG as RBG Controller
    participant Router
    participant Prefill
    participant Decode
    
    User->>RBG: 创建 RoleBasedGroup
    RBG->>RBG: 解析角色依赖关系
    RBG->>Router: 1. 创建 Router 角色
    Router->>RBG: Router Ready
    RBG->>Prefill: 2. 创建 Prefill 角色<br/>(依赖 Router)
    Prefill->>RBG: Prefill Ready
    RBG->>Decode: 3. 创建 Decode 角色<br/>(依赖 Prefill)
    Decode->>RBG: Decode Ready
    RBG->>User: RoleBasedGroup Ready
```

### 服务发现机制

```mermaid
graph LR
    subgraph "服务发现流程"
        RBG["RoleBasedGroup"]
        Discovery["Discovery Injector"]
        
        ConfigBuilder["Config Builder"]
        EnvBuilder["Env Builder"]
        
        CM["ConfigMap"]
        Env["Environment Variables"]
        
        Pod["Pod"]
        
        RBG -->|拓扑信息| Discovery
        Discovery --> ConfigBuilder
        Discovery --> EnvBuilder
        
        ConfigBuilder -->|生成配置文件| CM
        EnvBuilder -->|生成环境变量| Env
        
        CM -->|挂载| Pod
        Env -->|注入| Pod
        
        Pod -->|读取拓扑信息| App["应用进程"]
    end
```

### 滚动更新策略

```mermaid
graph TB
    subgraph "角色级滚动更新"
        UpdateTrigger["更新触发"]
        
        SelectRole["选择待更新角色"]
        CheckDep["检查依赖关系"]
        
        UpdatePods["同时更新角色内所有 Pods"]
        WaitReady["等待 Pods Ready"]
        
        NextRole["下一个角色"]
        Complete["更新完成"]
        
        UpdateTrigger --> SelectRole
        SelectRole --> CheckDep
        CheckDep -->|依赖已更新| UpdatePods
        CheckDep -->|依赖未更新| SelectRole
        UpdatePods --> WaitReady
        WaitReady -->|Ready| NextRole
        WaitReady -->|失败| Rollback["回滚"]
        NextRole -->|有待更新角色| SelectRole
        NextRole -->|无待更新角色| Complete
    end
```

## 技术栈

- **Kubernetes**: 容器编排平台
- **Controller Runtime**: Kubernetes Controller 开发框架
- **StatefulSet/LeaderWorkerSet/Deployment**: 工作负载类型
- **Scheduler Plugins**: Gang Scheduling 支持
- **Custom Resource Definitions (CRDs)**: 
  - RoleBasedGroup
  - ClusterEngineRuntimeProfile
  - Instance
  - InstanceSet

## 适用场景

1. **LLM 推理服务**: Prefill-Decode 分离部署
2. **分布式数据库**: 多角色协同（如主从、分片）
3. **消息队列集群**: Broker、Controller、ZooKeeper 等角色
4. **微服务网格**: 需要严格启动顺序的服务组
5. **AI 训练框架**: Parameter Server、Worker 等角色

## 参考资料

- [RBG Quick Start](./quick_start.md)
- [Multi Roles Feature](./features/multiroles.md)
- [Gang Scheduling](./features/gang-scheduling.md)
- [Update Strategy](./features/update-strategy.md)
- [Failure Handling](./features/failure-handling.md)
