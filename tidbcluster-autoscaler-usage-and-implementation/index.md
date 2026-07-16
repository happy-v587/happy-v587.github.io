# TiDB Operator 自动扩缩容：TidbClusterAutoScaler 使用与实现


## 前言

在 Kubernetes 上运行 TiDB 时，最常见的运维动作之一就是扩缩容。TiDB 计算节点 TiDB 是无状态的，扩缩容相对简单；TiKV 存储节点涉及数据重分布，需要谨慎处理。

TiDB Operator 里提供了一个叫 `TidbClusterAutoScaler` 的 CRD，试图让 TiDB 和 TiKV 的副本数根据负载自动调整。它支持两种决策方式：一种是把决策交给 PD，由 PD 根据集群负载给出扩缩容计划；另一种是调用外部服务，由用户自定义推荐副本数。

不过，这个特性长期处于实验状态，官方文档也比较少。本文基于 TiDB Operator `v1.6.0-alpha.7` 的代码，从使用到源码做一次完整梳理。

## 快速使用

### 前置条件

- 一个已经运行的 `TidbCluster`
- 一套 TiDB Monitor（PD 依赖它采集负载指标）
- TiDB Operator 版本支持 `TidbClusterAutoScaler`，本文使用 `v1.6.0-alpha.7`

### 一个最小示例

以下 YAML 会让 TiKV 在 CPU 使用率超过 80% 时自动扩容，低于 10% 时自动缩容：

```yaml
apiVersion: pingcap.com/v1alpha1
kind: TidbClusterAutoScaler
metadata:
  name: basic
spec:
  cluster:
    name: basic
  tikv:
    rules:
      cpu:
        max_threshold: 0.8
        min_threshold: 0.1
        resource_types:
        - default_tikv
    resources:
      default_tikv:
        cpu: "1000m"
        memory: "1Gi"
        storage: "10Gi"
```

执行：

```bash
kubectl apply -f tidbcluster-autoscaler.yaml
```

查看状态：

```bash
kubectl get ta basic -o yaml
```

注意：这个 CRD 不会直接修改原 `TidbCluster` 的 `replicas`。如果 PD 判断需要扩容，Operator 会创建一个新的 `TidbCluster` 资源来承载新的 TiKV group。

### 同时给 TiDB 也加上自动扩缩容

```yaml
spec:
  tidb:
    rules:
      cpu:
        max_threshold: 0.8
        min_threshold: 0.1
        resource_types:
        - default_tidb
    resources:
      default_tidb:
        cpu: "1000m"
        memory: "1Gi"
```

TiDB 计算节点无状态，扩缩容时只需要改副本数，不需要像 TiKV 那样考虑数据迁移。

### 调整扩缩容间隔

默认缩容间隔是 500 秒，扩容间隔是 300 秒。可以通过以下字段调整：

```yaml
spec:
  tikv:
    scaleInIntervalSeconds: 600
    scaleOutIntervalSeconds: 120
```

这个冷静期机制很重要，避免负载抖动导致反复扩缩容。

## CRD 结构解析

`TidbClusterAutoScaler` 的定义在 `pkg/apis/pingcap/v1alpha1/tidbclusterautoscaler_types.go`。

顶层结构很简单：

```go
type TidbClusterAutoScaler struct {
    metav1.TypeMeta
    metav1.ObjectMeta
    Spec   TidbClusterAutoScalerSpec
    Status TidbClusterAutoScalerStatus
}

type TidbClusterAutoScalerSpec struct {
    Cluster TidbClusterRef        // 目标 TidbCluster
    TiKV    *TikvAutoScalerSpec    // TiKV 自动扩缩容配置
    TiDB    *TidbAutoScalerSpec    // TiDB 自动扩缩容配置
}
```

`TikvAutoScalerSpec` 和 `TidbAutoScalerSpec` 都内嵌了 `BasicAutoScalerSpec`：

```go
type BasicAutoScalerSpec struct {
    Rules                   map[corev1.ResourceName]AutoRule
    ScaleInIntervalSeconds  *int32
    ScaleOutIntervalSeconds *int32
    External                *ExternalConfig
    Resources               map[string]AutoResource
}
```

### Rules

`Rules` 是触发扩缩容的判断条件。以 CPU 规则为例：

```go
type AutoRule struct {
    MaxThreshold  float64   // 扩容阈值
    MinThreshold  *float64  // 缩容阈值
    ResourceTypes []string  // 命中时使用的资源模板
}
```

当 CPU 使用率超过 `MaxThreshold` 时，PD 会建议扩容；低于 `MinThreshold` 时，会建议缩容。`ResourceTypes` 对应 `Resources` 中定义的 key。

### Resources

```go
type AutoResource struct {
    CPU     resource.Quantity
    Memory  resource.Quantity
    Storage resource.Quantity
    Count   *int32
}
```

这是扩缩容时使用的资源模板。PD 会从这个模板列表里选择合适的规格来生成扩缩容计划。

### External

如果不想用 PD 做决策，也可以接入外部推荐服务：

```go
type ExternalConfig struct {
    Endpoint    ExternalEndpoint
    MaxReplicas int32
}

type ExternalEndpoint struct {
    Host      string
    Port      int32
    Path      string
    TLSSecret *SecretRef
}
```

外部服务返回推荐副本数，Operator 会直接根据这个值创建或更新独立的 `TidbCluster`。

### Status

```go
type TidbClusterAutoScalerStatus struct {
    TiKV map[string]TikvAutoScalerStatus
    TiDB map[string]TidbAutoScalerStatus
}

type BasicAutoScalerStatus struct {
    LastAutoScalingTimestamp *metav1.Time
}
```

Status 只记录每个 group 最后一次扩缩容的时间戳，用于控制冷静期。

## 运行原理与架构

`TidbClusterAutoScaler` 的核心逻辑并不直接修改原 `TidbCluster` 的副本数，而是基于 PD 或外部服务的建议，创建、更新、删除额外的 `TidbCluster` 资源。

### 两种工作模式

**PD 计划模式**

这是默认模式。Operator 把 TAC 中配置的 rules 和 resources 转换成 `pdapi.Strategy`，调用 PD 的 `/pd/api/v1/autoscaling` 接口获取扩缩容计划。PD 根据集群负载返回一组 `Plan`，每个 Plan 包含：

- `Component`：tidb 或 tikv
- `Count`：建议副本数
- `ResourceType`：使用哪个资源模板
- `Labels`：group 标签

Operator 拿到这些 Plan 后，去创建、更新或删除对应的 autoscaling TC。

**外部服务模式**

如果 `spec.tikv.external` 或 `spec.tidb.external` 有配置，Operator 会跳过 PD，直接调用外部 HTTP 服务。外部服务返回一个推荐副本数，Operator 会创建或更新名为 `<原TC名>-<component>-external` 的独立 TC。

### 架构图

```
                +---------------------+
                | TidbClusterAutoScaler |
                +----------+----------+
                           |
                           v
              +------------------------+
              |  tidb-controller-manager |
              +----------+-------------+
                         |
         +---------------+---------------+
         |                               |
         v                               v
+--------+-------+              +-------+--------+
|       PD        |              | External Service |
| GetAutoscalingPlans |          | /recommend       |
+--------+-------+              +-------+--------+
         |                               |
         v                               v
+--------+--------------------------------+--------+
|              创建 / 更新 / 删除                     |
|         独立的 TidbCluster（autoscaling）          |
+--------------------------------------------------+
```

### 为什么生成独立 TC？

原 TC 是用户显式声明的期望状态。如果 AutoScaler 直接修改原 TC 的 `replicas`，相当于和用户配置打架。生成独立 TC 的好处是：

- 原 TC 保持稳定，用户仍然清楚自己的集群长什么样
- 不同 group 可以配置不同的资源规格
- AutoScaler 生成的 TC 可以设置 ownerReference，跟随 TAC 一起清理

但这也带来复杂度：集群被拆成了多个 TC，资源管理、监控、命名空间视角下都更混乱。

## 源码深入

### 控制器入口

控制器在 `pkg/controller/autoscaler/tidbcluster_autoscaler_controller.go`。

```go
func NewController(deps *controller.Dependencies) *Controller {
    t := &Controller{
        deps:    deps,
        control: NewDefaultAutoScalerControl(autoscaler.NewAutoScalerManager(deps)),
        queue: workqueue.NewNamedRateLimitingQueue(...),
    }
    tidbAutoScalerInformer := deps.InformerFactory.Pingcap().V1alpha1().TidbClusterAutoScalers()
    controller.WatchForObject(tidbAutoScalerInformer.Informer(), t.queue)
    return t
}
```

这是一个标准的 Kubernetes 控制器：监听 `TidbClusterAutoScaler` 变化，事件进入队列，worker 逐个处理。

`processNextWorkItem` 调用 `sync`：

```go
func (c *Controller) sync(key string) (err error) {
    ns, name, err := cache.SplitMetaNamespaceKey(key)
    ta, err := c.deps.TiDBClusterAutoScalerLister.TidbClusterAutoScalers(ns).Get(name)
    return c.control.ReconcileAutoScaler(ta)
}
```

### Sync 主流程

核心逻辑在 `pkg/autoscaler/autoscaler/autoscaler_manager.go` 的 `Sync` 方法：

```go
func (am *autoScalerManager) Sync(tac *v1alpha1.TidbClusterAutoScaler) error {
    // 1. 获取目标 TidbCluster
    tc, err := am.deps.TiDBClusterLister.TidbClusters(...).Get(tcName)

    // 2. 填充默认值
    defaultTAC(tac, tc)

    // 3. 校验
    if err := validateTAC(tac); err != nil {
        return nil
    }

    // 4. 执行扩缩容
    updatedTac := tac.DeepCopy()
    if err := am.syncAutoScaling(tc, updatedTac); err != nil {
        return err
    }

    // 5. 更新 TAC status
    return am.updateTidbClusterAutoScaler(updatedTac)
}
```

流程很清晰：找目标 TC → 默认值 → 校验 → 执行 → 更新状态。

`syncAutoScaling` 根据组件选择 PD 模式或外部模式：

```go
func (am *autoScalerManager) syncAutoScaling(tc, tac) error {
    if tac.Spec.TiDB != nil {
        if tac.Spec.TiDB.External != nil {
            am.syncExternal(...)
        } else {
            am.syncPD(...)
        }
    }
    if tac.Spec.TiKV != nil { ... }
}
```

### PD 模式实现

`syncPD` 在 `pkg/autoscaler/autoscaler/pdplan_autoscaler.go` 中：

```go
func (am *autoScalerManager) syncPD(tc, tac, component) error {
    strategy := autoscalerToStrategy(tac, component)
    plans, err := controller.GetPDClient(...).GetAutoscalingPlans(*strategy)
    if err != nil { return err }
    return am.syncPlans(tc, tac, plans, component)
}
```

`autoscalerToStrategy` 把 TAC 的 resources 和 rules 转成 PD 能理解的结构：

```go
type Strategy struct {
    Rules     []*Rule
    Resources []*Resource
}
```

拿到 PD 返回的 `Plan` 后，`syncPlans` 做三件事：

1. 找出当前已经存在的 autoscaling groups
2. 对比 PD 计划与当前 groups
3. 创建新增的、更新变更的、删除多余的

```go
toDelete := existedGroups.Difference(planGroups)
toUpdate := planGroups.Intersection(existedGroups)
toCreate := planGroups.Difference(existedGroups)
```

创建新 TC 时，代码会克隆原 TC 的 spec，但只保留一个组件：

```go
autoTc := newAutoScalingCluster(tc, tac, autoTcName, component)
autoTc.Spec.TiCDC = nil
autoTc.Spec.TiFlash = nil
autoTc.Spec.PD = nil
autoTc.Spec.Pump = nil
```

### 外部模式实现

`pkg/autoscaler/autoscaler/external_autoscaler.go` 更简单。Operator 调用外部服务拿到 `targetReplicas`，然后创建或更新名为 `<tcname>-<component>-external` 的 TC：

```go
externalTcName := fmt.Sprintf(externalTcNamePattern, tc.Name, component.String())
```

如果 `targetReplicas <= 0`，则优雅地删除这个 external TC。

### 工具函数

`pkg/autoscaler/autoscaler/util.go` 里有几个关键函数：

- `defaultTAC`：填充默认资源模板、默认间隔、默认阈值
- `validateTAC`：校验 rules、resources、thresholds 是否合法
- `autoscalerToStrategy`：把 CRD 转成 PD Strategy
- `newAutoScalingCluster`：克隆原 TC 生成 autoscaling TC
- `checkAutoScaling`：根据 `LastAutoScalingTimestamp` 判断是否在冷静期内

`checkAutoScaling` 逻辑：

```go
func checkAutoScaling(tac, memberType, group, beforeReplicas, afterReplicas) bool {
    if beforeReplicas > afterReplicas {
        // 缩容，检查 scaleInIntervalSeconds
    } else if beforeReplicas < afterReplicas {
        // 扩容，检查 scaleOutIntervalSeconds
    }
    return true
}
```

冷静期机制是这段代码里比较务实的部分，避免负载抖动导致反复扩缩容。

## 设计思考

### 优点

1. **与 PD 联动**：TiKV 数据分布的负载信息只有 PD 最清楚，把扩缩容决策交给 PD 是合理的。
2. **资源模板化**：通过 `resources` 定义多种规格，PD 可以按需选择不同资源类型的 group。
3. **独立 TC 不污染原集群**：用户配置和自动扩缩容产物分离，职责清晰。
4. **支持外部推荐**：如果内部规则不够，可以接入自己的决策服务。

### 缺点

1. **实验性，未 GA**：长期处于非稳定状态，官方文档和维护投入有限。
2. **生成多个 TC 管理复杂**：命名空间下会出现多个 `TidbCluster`，监控、运维、排查都会变复杂。
3. **TiKV 扩缩容仍然重**：创建新 TiKV group 涉及数据重分布，不是瞬间完成的，自动扩缩容的及时性有限。
4. **依赖 PD autoscaling API**：PD 侧的自动扩缩容计划本身成熟度有限，Operator 这边能做的只是执行者。
5. **缺少 min/max replicas 约束**：CRD 里没有直接限制总副本数上下限，完全依赖 PD 或外部服务返回合理值。

### 个人看法

`TidbClusterAutoScaler` 的设计思路是对的：把决策和 execution 分离，Operator 只做 Kubernetes 侧的资源编排，PD 或外部服务负责决策。但问题在于这个特性在 TiDB 生态里一直处于比较尴尬的位置——PD 的 autoscaling 计划不够成熟，用户侧真正用起来的场景也不多，导致代码长期缺乏维护。

对于 TiDB 计算层 TiDB，直接用 HPA 可能更自然；对于 TiKV 存储层，由于数据迁移成本，真正意义上的"自动弹性"在分布式数据库里本身就是一个难题。这个 CRD 更像是一个探索性实现，而不是一个成熟的生产特性。

## 总结

- `TidbClusterAutoScaler` 是 TiDB Operator 提供的自动扩缩容 CRD，支持 PD 决策和外部推荐两种模式
- 它不会直接修改原 TidbCluster，而是创建独立的 autoscaling TC
- 源码实现清晰，分为控制器、Sync 主流程、PD 计划执行、外部模式执行四部分
- 该特性长期处于实验状态，生产使用需谨慎

后续如果想继续深入，可以看看 PD 侧的 `autoscaling` API 实现，以及 `tipocket` 里相关的 autoscaling 测试 case。


