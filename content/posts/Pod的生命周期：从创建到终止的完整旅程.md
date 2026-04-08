+++
date = '2026-04-09T02:50:00+08:00'
draft = false
title = 'Pod 的生命周期：从创建到终止的完整旅程'
summary = '从凌晨三点的 CrashLoopBackOff 告警出发，深入拆解 Kubernetes Pod 的完整生命周期——五个阶段、容器状态、探针机制、重启策略、体面终止，附带调试速查表。'
tags = ['Kubernetes', 'Pod', '云原生', 'DevOps']
categories = ['技术']
+++

凌晨三点，你的手机响了。线上告警：某个服务挂了。你打开电脑，`kubectl get pods` 一敲，屏幕上赫然出现一个令人不安的状态——`CrashLoopBackOff`。

你开始排查：看日志、查事件、审视配置。十五分钟后，你发现是存活探针的超时时间设短了，容器启动慢了一拍，Kubernetes 就认为它死了，杀掉，重启，再杀掉，再重启……陷入了死循环。

如果你完全理解 Pod 的生命周期——它有哪些阶段、容器状态如何流转、探针的机制是什么、重启策略如何生效——你可能在三分钟内就定位到问题，而不是十五分钟。

**Pod 的生命周期不只是理论知识，它是你在 Kubernetes 世界里最实用的调试工具。** 这篇文章，我们从头到尾拆解它。

## Pod 是临时的

在理解 Pod 之前，先建立一个直觉：**把 Pod 想象成一个外卖骑手，把节点想象成配送站。**

配送站是固定的，但骑手来来去去。一个骑手接单出发（Pod 创建），送到目的地（Succeeded），或者半路出了问题（Failed）。如果骑手 A 今天不干了，平台不会把他"复活"——而是派一个新的骑手 B 来接手同样的订单。骑手 B 可能叫同样的名字，但他是个不同的人（不同的 UID）。

这个比喻揭示了 Kubernetes 中一个核心设计哲学：**Pod 是可替换的，不要对它产生感情。**

每个 Pod 在创建时会获得一个唯一的 UID，被调度到某个节点上运行，直到终止或被删除。如果节点挂了，运行在上面的 Pod 会被标记为失败并最终被删除。Pod 不会被"重新调度"到另一个节点——它是被一个全新的 Pod 替换掉，新 Pod 有自己的 UID。你的应用应该设计成：任何时候 Pod 被替换，系统都能继续正常工作。

那么，一个 Pod 从创建到消亡，到底经历了哪些阶段？

## 五个阶段：Pod 的宏观状态

Pod 的 `status.phase` 字段描述了它在生命周期中所处的宏观阶段。这个阶段不是完整的状态机，而是一个简单的高层概述。

| 阶段 | 含义 |
|------|------|
| `Pending` | 已被系统接受，但容器还没创建或运行。等待调度或拉取镜像 |
| `Running` | 已绑定到节点，所有容器已创建，至少一个在运行 |
| `Succeeded` | 所有容器成功结束，不会重启 |
| `Failed` | 所有容器已终止，至少一个以失败退出 |
| `Unknown` | 无法获取 Pod 状态，通常是与节点通信失败 |

这些东西光看表格可能没什么感觉。来看一个真实的例子——你刚创建了一个 Nginx Pod，用 `kubectl get pods -w`（`-w` 表示 watch，实时追踪变化）观察它：

```bash
# 你创建了 Pod，调度器还在选节点
NAME        READY   STATUS    RESTARTS   AGE
my-nginx    0/1     Pending   0          2s

# 调度成功，开始拉镜像
my-nginx    0/1     Pending   0          3s
my-nginx    0/1     ContainerCreating   0          4s

# 镜像拉取完成，容器启动成功
my-nginx    1/1     Running   0          12s

# 如果你手动删除它
my-nginx    1/1     Terminating   0          30s
```

你看，STATUS 列的变化（`Pending` → `ContainerCreating` → `Running` → `Terminating`）就是 Pod 生命周期在你眼前的真实投影。注意 `READY` 列从 `0/1` 变成 `1/1`——这意味着就绪探针通过了，Pod 可以接收流量了。

注意，`CrashLoopBackOff` 不是 Pod 的 phase。它是 kubectl 在 Status 列中显示的状态，表示容器反复崩溃并在指数退避中等待重启。如果你看到这样的输出：

```bash
NAME        READY   STATUS             RESTARTS   AGE
my-app      0/1     CrashLoopBackOff   5          8m
```

`RESTARTS: 5` 告诉你已经崩溃重启了 5 次，`CrashLoopBackOff` 说明 kubelet 正在用指数退避等待下一次重启尝试。

知道了 Pod 有哪些阶段之后，一个自然的问题是：在 Pod 内部，单个容器处于什么状态？Phase 告诉你 Pod 的宏观位置，但调试时你更需要知道的是每个容器的微观状态。

## 容器的三种状态

Phase 告诉你 Pod 在哪一层，但调试的时候你更关心的是每个容器到底怎么了。容器的状态只有三种，像一个迷你状态机：

**Waiting（等待）——"还没上场"**

容器正在后台做准备：拉镜像、挂载存储、注入配置……你可以用 `kubectl describe pod` 看到它为什么在等。最常见的等待原因是 `ImagePullBackOff`——镜像拉不下来，可能是镜像名拼错了，也可能是私有仓库的认证没配好。

**Running（运行中）——"正常干活"**

容器在执行，一切正常。如果配了 `postStart` 回调，此时已经跑完了。这是你最想看到的状态。

**Terminated（已终止）——"退场了"**

容器跑完了或者崩了。这时候关键是看**退出码**：0 表示正常结束，非 0 表示出了问题。`kubectl describe pod` 会给你退出码、起止时间和原因。如果配了 `preStop` 回调，它会在容器被标记为 Terminated 之前执行——这是你最后的清理机会。

## 重启策略：容器挂了怎么办

当容器失败时，Kubernetes 根据 `restartPolicy` 决定是否重启：

- **Always**：容器终止就自动重启（默认值）
- **OnFailure**：仅在错误退出（非零退出码）时重启
- **Never**：不自动重启

重启不是立即发生的。Kubernetes 使用**指数退避**机制：第一次等 10 秒，然后 20 秒、40 秒……最高到 300 秒（5 分钟）。如果容器成功运行了 10 分钟，退避计时器会被重置。这就是 `CrashLoopBackOff` 背后的机制。

容器挂了会重启，但 Kubernetes 怎么判断一个容器"挂了"？答案是探针。

## 探针：Kubernetes 的健康检查

探针是 kubelet 对容器执行的定期诊断，分为三种类型：

### 存活探针（livenessProbe）

回答一个问题：**容器还在正常运行吗？**

如果探测失败，kubelet 会杀死容器，根据重启策略决定是否重启。如果你的进程本身会在出问题时崩溃，可以不设存活探针。

### 就绪探针（readinessProbe）

回答一个问题：**容器准备好接收流量了吗？**

探测失败时，Pod 会从 Service 的 EndpointSlice 中移除，不再接收新请求。但容器不会被杀死。这对于应用启动时需要加载数据的场景特别有用。

### 启动探针（startupProbe）

回答一个问题：**容器内的应用已经启动了吗？**

启动探针成功之前，其他探针都被禁用。这解决了慢启动应用的难题——你不需要把存活探针的间隔设得很长，只需要单独给启动阶段一个更宽松的时间窗口。

探针可以使用四种检查机制：`exec`（执行命令）、`httpGet`（HTTP 请求）、`tcpSocket`（TCP 检查）和 `grpc`（gRPC 健康检查）。

说起来有点抽象，来看一个真实的配置。这是一个典型 Web 应用的 Pod，三种探针都用上了：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-web-app
spec:
  containers:
  - name: app
    image: my-app:latest
    ports:
    - containerPort: 8080

    # 启动探针：给应用 60 秒启动时间
    # failureThreshold 30 × periodSeconds 2s = 最多等 60 秒
    startupProbe:
      httpGet:
        path: /healthz
        port: 8080
      failureThreshold: 30
      periodSeconds: 2

    # 存活探针：启动通过后开始检查，挂了就重启
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10

    # 就绪探针：只有 /ready 返回 200 才接收流量
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
```

注意三个探针的分工：
- `startupProbe` 用宽松的参数（最多等 60 秒），专门应对启动慢的问题
- `livenessProbe` 检查 `/healthz`，如果应用死锁了就重启
- `readinessProbe` 检查 `/ready`（和 healthz 不同的端点），应用可能在运行但暂时不想接收流量（比如正在加载缓存）

这就是开头那个凌晨三点告警的正确配置方式。如果你当时这样配了 `startupProbe`，容器启动慢一点也不会被存活探针误杀。

探针解决了"容器是否健康"的问题。但在更上层，Kubernetes 如何判断一个 Pod 是否"准备好了"？这就是 Pod Conditions 的作用。

## Pod 状况：更细粒度的状态

除了 phase，Pod 还有一组 Conditions（状况），提供更详细的状态信息：

| 状况 | 含义 |
|------|------|
| `PodScheduled` | 已被调度到节点 |
| `Initialized` | 所有 Init 容器已成功完成 |
| `ContainersReady` | 所有容器都已就绪 |
| `Ready` | Pod 可以服务请求，应加入负载均衡池 |
| `DisruptionTarget` | Pod 即将因干扰被终止 |

Pod 有生就有死。一个常被忽视但极其重要的问题是：Pod 死的时候，你的应用有机会做清理吗？

## Pod 的终止：体面退出

当你删除一个 Pod 时，Kubernetes 不会立即杀死它，而是走一个体面终止的流程：

1. API 服务器更新 Pod 对象，设置 `deletionTimestamp`，Pod 显示为 "Terminating"
2. kubelet 开始本地关闭流程：
   - 如果容器定义了 `preStop` 回调，先执行回调
   - 发送 SIGTERM 信号给每个容器的 1 号进程
3. 控制平面将 Pod 从 EndpointSlice 中移除（`ready` 设为 false），负载均衡器不再发送新流量
4. 等待终止宽限期（默认 30 秒）
5. 宽限期过后，发送 SIGKILL 强制杀死所有剩余进程
6. API 服务器删除 Pod 对象

这个流程的关键点：你的应用需要正确处理 SIGTERM 信号，在收到信号后完成清理工作并退出。如果清理需要超过 30 秒，需要调大 `terminationGracePeriodSeconds`。

来看一个 Node.js 应用如何处理这个信号：

```javascript
const server = app.listen(8080);

// Kubernetes 发送 SIGTERM 时触发
process.on('SIGTERM', () => {
  console.log('收到 SIGTERM，开始优雅关闭...');

  // 1. 停止接收新请求
  server.close(() => {
    console.log('所有请求处理完毕，进程退出');
    process.exit(0);
  });

  // 2. 设置兜底超时（防止清理卡住）
  setTimeout(() => {
    console.error('优雅关闭超时，强制退出');
    process.exit(1);
  }, 25000); // 略小于 terminationGracePeriodSeconds
});
```

这段代码做了两件关键的事：先停止接收新连接，等已有请求处理完再退出；同时设了一个 25 秒的兜底超时——比 Kubernetes 默认的 30 秒宽限期短一点，确保进程自己退出而不是被 SIGKILL 强杀。

对于包含 Sidecar 容器的 Pod，kubelet 会延迟向 Sidecar 发送 TERM 信号，直到所有主容器都已终止，并且按定义的相反顺序终止 Sidecar。

## 完整的状态机：一图胜千言

把前面所有内容串起来，Pod 的完整生命周期其实是一个状态机。这张图建议你仔细看一遍，它把 Phase、容器状态、探针、重启策略、终止流程全部放在了一起：

```
                        ┌─────────────┐
                        │  用户创建 Pod │
                        └──────┬──────┘
                               │
                               ▼
                    ┌─────────────────────┐
          ┌────────│      Pending        │◄──── 调度失败重试
          │        │  · 等待调度          │
          │        │  · 拉取镜像          │
          │        └──────────┬──────────┘
          │                   │ 调度成功 + 镜像就绪
          │                   ▼
          │        ┌─────────────────────┐
          │        │     Container       │
          │        │      Waiting        │─────── 镜像拉取失败 ──→ Failed
          │        └──────────┬──────────┘
          │                   │ 容器启动
          │                   ▼
          │   ┌──────────────────────────────┐
          │   │          Running             │◄─────── 重启策略触发重启
          │   │  · postStart 回调执行         │               │
          │   │  · 探针开始工作               │               │
          │   │    ├─ livenessProbe ◄──失败──┤               │
          │   │    ├─ readinessProbe          │     指数退避   │
          │   │    └─ startupProbe            │   (10s→20s→   │
          │   └──────┬───────────┬───────────┘    40s→...     │
          │          │           │                 300s)       │
          │   所有容器正常退出    │ 容器失败退出                 │
          │          │           │                             │
          │          ▼           ├──── restartPolicy=Never ───►│
          │   ┌───────────┐     ├──── restartPolicy=OnFailure─►│
          │   │ Succeeded │     │  (仅非零退出码)              │
          │   │ (Job完成)  │     │                              │
          │   └───────────┘     │   ┌──────────────────────┐   │
          │                     └──►│    CrashLoopBackOff  │───┘
          │                         │  反复崩溃+指数退避     │
          │                         └──────────┬───────────┘
          │                                    │ 放弃治疗
          │                                    ▼
          │                             ┌───────────┐
          └─────────────────────────────│  Failed   │
                                        └───────────┘

                    ┌─────────────────────────────┐
                    │      用户删除 Pod            │
                    └──────────────┬──────────────┘
                                   ▼
                         ┌──────────────────┐
                         │   Terminating    │
                         │  · preStop 回调   │
                         │  · SIGTERM        │──── 30秒宽限期 ──→ SIGKILL
                         │  · 排空流量        │
                         └──────────────────┘
```

几个关键的观察：

1. **Running 是唯一可以"循环"的状态**——容器崩溃后根据重启策略可能回到 Running
2. **Pending 阶段包含两件不同的事**：等待调度 + 拉取镜像，它们的时间可以差很多
3. **CrashLoopBackOff 是一个"中间态"**——它不是 Phase，而是 Running 到 Running 之间的过渡状态
4. **一旦进入 Succeeded 或 Failed，就不再变了**——这是终态

## 调试速查表：根据状态定位问题

理论看完了，来点实战的。当你遇到 Pod 问题时，用这个流程快速定位：

**Pod 卡在 `Pending`**
```bash
kubectl describe pod <pod-name> | grep -A 5 Events
```
常见原因：资源不足（CPU/内存）、节点选择器不匹配、PVC 挂载失败。看 Events 部分，它会告诉你调度器拒绝调度的具体原因。

**Pod 卡在 `ContainerCreating`**
```bash
kubectl describe pod <pod-name> | grep -A 10 Events
```
常见原因：镜像拉取失败（镜像名错误、私有仓库认证问题）、ConfigMap/Secret 不存在。

**Pod 处于 `CrashLoopBackOff`**
```bash
kubectl logs <pod-name> --previous
```
注意 `--previous` 参数——它查看的是上一次崩溃的容器日志，而不是当前正在重启的容器。这是排查 CrashLoopBackOff 最关键的一个命令。

**Pod 是 `Running` 但 `READY 0/1`**
```bash
kubectl describe pod <pod-name> | grep -A 5 Readiness
```
容器在运行，但就绪探针没通过。检查探针配置是否正确，端点是否可达。

**删除 Pod 卡在 `Terminating`**
```bash
kubectl delete pod <pod-name> --force --grace-period=0
```
最后的手段。正常情况下应该检查为什么 `preStop` 回调或应用关闭流程卡住了。

## 回顾

回头看这篇文章开头的那个凌晨三点的告警场景。现在你应该能说清楚：

- `CrashLoopBackOff` 不是 Pod 的 Phase，它是容器反复崩溃时 kubelet 施加的指数退避状态
- 容器启动慢被存活探针杀死，是因为启动探针（`startupProbe`）没有配置——它本应在启动期间接管检查
- 重启间隔越来越长（10s → 20s → 40s → ...）是指数退避机制在工作
- 解决方案：加一个 `startupProbe` 给启动阶段更多时间，或者调大 `initialDelaySeconds`

这就是理解 Pod 生命周期的价值——它不只是理论，它是你排查问题时心里的那张地图。
