---
title: Pod
date: 2026-06-23
tags: [kubernetes, pod, 容器, 生命周期, init-container, sidecar, ambassador]
sources: [kubernetes/kubernetes基本概念.md, kubernetes/kubernetes 101.md, kubernetes/objects/pod.md]
---

# Pod

Pod 是 Kubernetes **调度的基本单位**，是一组紧密关联的[[concepts/容器]]集合，每个 Pod 可包含一个或多个容器。

## 为什么需要 Pod 这一层

Pod 是介于"单个容器"与"整台 [[entities/Node]]"之间、有意设计的**最小调度与共享单位**。即使大多数 Pod 只含一个容器，这层抽象仍解决了直接在 Node 上跑容器无法解决的问题：

- **表达"一组容器是一伙的"**：紧密协作的辅助容器（sidecar，如日志收集、服务网格代理、配置同步）必须与主容器同机、共享网络和文件才能工作。只有"单容器"这一个单位时，无法声明这种归属关系。
- **原子调度**：Pod 内的容器保证被**一起调度到同一 Node、同生共死**，不会被分到不同机器而导致 localhost / 共享卷失效。
- **简化网络模型**：K8s 采用"**每个 Pod 一个 IP**"（而非每容器一个），Pod 内容器共享该 IP，避免端口冲突与 IP 数量爆炸；[[entities/Service]] 面向的也是 Pod IP。
- **统一挂载点**：[[concepts/Volume存储]]、生命周期、[[concepts/健康检查]] 都以 Pod 为单位统一管理，Pod 内多容器可共享同一卷。
- **与运行时解耦**：Pod 是运行时无关的抽象，K8s 只管调度/管理 Pod，容器具体用哪个运行时跑交给 CRI 插件（见 [[concepts/插件机制与可扩展性]]），换运行时不影响核心模型。

> 一句话：直接在 Node 上跑容器，缺的是"一组容器共享网络/存储/命运、并作为整体被调度"的能力，Pod 正是为此而生。

## 关键特性

- **共享命名空间**：Pod 内的多个容器共享 IPC 和 Network namespace，因此可通过进程间通信（IPC）和文件共享高效协作。
- **共享网络与文件系统**：同一 Pod 内的容器共享网络（同一 IP/端口空间）和挂载的 [[concepts/Volume存储]]。每个 Pod 有独立 IP（含同机 Pod），详见 [[concepts/Pod网络模型]]。
- **调度单位**：K8s 调度的是 Pod 整体，而非单个容器——同一 Pod 的容器总在同一 [[entities/Node]] 上运行。
- **生命周期短暂**：Pod 出现异常时会被销毁并由新 Pod 替代；其 IP 会随重启变化，因此不应直接用 Pod IP 交互（用 [[entities/Service]] 代替）。

## 定义方式

所有 K8s 对象都用 manifest（YAML 或 JSON）定义。一个最简 nginx Pod：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
```

通过 `kubectl create -f nginx.yaml` 创建。

> 生产环境**不直接建裸 Pod**：裸 Pod 一旦调度即与 Node 绑定，**Node 挂掉不会重新调度**（直接删除）。用控制器（[[entities/Deployment]]/[[entities/StatefulSet]]/[[entities/DaemonSet]]/[[entities/Job]]）托管才有容错（见 [[concepts/工作负载控制器]]）。

## 生命周期与重启

- **Phase**（`status.phase`，粗粒度状态）：`Pending`（已创建，镜像还在拉等）→ `Running` → `Succeeded` / `Failed`；`Unknown` 通常是 apiserver 连不上 kubelet。
  - `Running`：已绑定 Node、所有容器已创建，**至少一个容器在运行**（或正在启动/重启中）；非终态。
  - `Succeeded` / `Failed`：**所有容器都已终止且不再重启**，二者区别只在退出码——全部 exit 0 为 `Succeeded`，有非 0 退出或被系统 kill（如 OOMKilled）为 `Failed`。均为终态，不会再变回去。
  - `kubectl get pods` 的 STATUS 列**不等于** `status.phase`：`Succeeded` 显示为 `Completed`；而崩溃重启中的 Pod phase 仍是 `Running`，STATUS 却显示 `CrashLoopBackOff`。看真实 phase 用 `kubectl get pod <name> -o jsonpath='{.status.phase}'`。
  - 终态 Pod 不再占用 CPU/内存，但 API 对象仍留在 etcd 里（日志可查），需手动删或靠 Job 的 `ttlSecondsAfterFinished` 回收。
- **restartPolicy**：`Always`（默认）/ `OnFailure` / `Never`，是 **Pod 级**字段，对所有容器统一生效（唯一例外是下文的原生 sidecar）。**只在 Pod 所在 Node 本地重启容器，不会换节点**；换节点是上层控制器重建新 Pod 的事。核心区别就一条——**容器正常退出（exit 0）时要不要重启**：

  | | exit 0 | exit ≠ 0 / 被 kill | 能否进终态 |
  | --- | --- | --- | --- |
  | `Always` | 重启 | 重启 | **不能**，永远停在 `Running` 里循环 |
  | `OnFailure` | 不重启 → `Succeeded` | 重启 | 能 |
  | `Never` | 不重启 → `Succeeded` | 不重启 → `Failed` | 能 |

  - **取值受控制器强制**（apiserver 校验，写错直接拒绝）：[[entities/Deployment]]/[[entities/StatefulSet]]/[[entities/DaemonSet]]/[[entities/ReplicaSet]] 的 Pod 模板**只能 `Always`**——这些控制器靠 Pod 长期 `Running` 维持副本数，Pod 一旦能进 `Succeeded` 副本模型就崩了；[[entities/Job]]/[[entities/CronJob]] **只能 `OnFailure` 或 `Never`**——否则容器跑完自动重启，Job 永远判不了完成。
  - **Job 里 `OnFailure` vs `Never` 的重试粒度**：`OnFailure` 由 kubelet **在原 Pod 内就地重启容器**（同节点、Pod IP 不变、`emptyDir` 内容保留、`restartCount` 递增），开销小恢复快，适合瞬时故障；`Never` 则整个 Pod 标记 `Failed`、由 Job 控制器**新建 Pod** 重试（可能换节点、`emptyDir` 全新、失败 Pod 对象留存——现场可查但需清理），适合故障可能与节点相关或需保留每次现场。两者都受 `backoffLimit`（默认 6）约束。
  - **重启退避**：反复失败按指数退避 10s→20s→40s…上限 5min，容器成功运行满 10min 后计数器重置。退避期间即 `CrashLoopBackOff`。
- **优雅终止**：删除时 kubelet 先给容器发 `SIGTERM`，等 grace period（默认 30s）再 `SIGKILL`；可用 `preStop` 钩子做清理。

## Init 容器与 sidecar

- **Init 容器**：在应用容器前**按序 run-to-completion**（一个跑完退出才跑下一个），全部成功才启动应用容器；常用于初始化配置、等待依赖、跑迁移。镜像独立，可装应用镜像里不便包含的工具。

- **传统 sidecar 的痛点**：把辅助容器（日志/网格代理如 Envoy）放进 `containers` 与主容器**并列**——但 `containers` 里的容器**并行启动、无启停顺序**：① 代理没就绪主容器就发请求 → 早期请求失败；② 关闭时代理可能**先于主容器死** → 尾部请求丢；③ 在 [[entities/Job]] 里 sidecar 常驻不退 → **Job 永远判不了"完成"**。

- **原生 sidecar** 解决上述问题：把 sidecar 写成 **`initContainers` 里、单独设 `restartPolicy: Always` 的容器**。K8s 对它特殊对待——它常驻不退出，故 K8s 改为**等它"就绪"**（而非退出）就放行下一步。于是它同时获得：**先于主容器启动**（在 init 序列里）+ **贯穿 Pod 生命**（Always 常驻）+ **晚于主容器关闭**，且在 Job 里**不阻止完成**。复用 initContainers 是因其本就自带"排序/先启动"机制。（`v1.28` alpha → `v1.29` beta 默认开 → **`v1.33` GA**）

  ```yaml
  initContainers:
  - { name: setup, image: busybox, command: ["sh","-c","echo prepare"] }  # 普通 init:跑完退出
  - name: proxy
    image: envoy
    restartPolicy: Always     # ← 这一行使它成为"先启动+常驻"的 sidecar
  containers:
  - { name: app, image: myapp }
  # 顺序:setup 跑完 → proxy 就绪 → app 启动;proxy 全程在、且晚于 app 关闭
  ```

| | 普通 Init | 传统 sidecar(`containers`) | 原生 sidecar(init+`Always`) |
| --- | --- | --- | --- |
| 启动 | 主容器前、按序 | 与主容器并行、**无序** | 主容器前(**有序**) |
| 运行多久 | 跑完就退 | 贯穿 Pod 生命 | 贯穿 Pod 生命 |
| 关闭 | (已退出) | **无保证**(可能先死) | **晚于主容器** |
| Job 里 | 正常 | **阻止 Job 完成** | 不阻止 |

## 多容器协作模式

Pod 内多容器共享网络/卷，常见模式：**sidecar**（边车，加日志/监控/代理）、**ambassador**（大使，代理对外连接）、**adapter**（适配器，转换数据/协议格式）。这正是"为什么需要 Pod 这一层"的落地（见上文）。三者按 Pod 边界的方向划分：ambassador 管**出**、adapter 管**入**、sidecar 泛指**增强**。

三者在 K8s API 层**没有任何区别**——都只是 Pod 里多出来的一个容器（或 `initContainers` 里的原生 sidecar），区分的是**意图**而非字段。ambassador / adapter 可视为 sidecar 的特化。以最容易混淆的"sidecar 里的网格代理 vs ambassador"为例：

| | sidecar 里的网格代理（Envoy） | ambassador（大使） | adapter（适配器） |
| --- | --- | --- | --- |
| 流量方向 | 入站 + 出站，双向全接管 | **仅出站**（app → 外部依赖） | **仅入站**（外部来读 app 的数据） |
| 应用是否感知 | **无感**：iptables/eBPF 透明劫持，app 仍连真实服务名 | **有感**：app 被显式配置成连 `localhost:<port>` | 无感：app 只管写日志/暴露自有指标 |
| 代理对象 | 所有服务间流量，**通用**基础设施 | **某一个具体外部依赖**（Redis 分片、数据库、云 API） | app 自身的输出 |
| 解决什么 | 治理：mTLS、重试、熔断、流量切分、遥测 | 接入复杂度：分片、连接池、鉴权签名、端点差异、环境切换 | 格式统一：转成外部系统期望的协议/schema |
| 谁引入 | 平台/运维，准入 webhook 自动注入 | **应用作者**写进 manifest | 应用作者或监控方 |
| 典型实现 | Istio / Linkerd 的 Envoy、linkerd2-proxy | Cloud SQL Auth Proxy、PgBouncer、twemproxy、SigV4 签名代理 | Prometheus exporter、日志格式转换器 |

- **ambassador 的价值是"localhost 契约"**：应用代码里只写死 `localhost:5432`，不知道背后是单机库、连接池还是带 IAM 鉴权的云数据库。换环境、换分片策略、加认证只改 ambassador 容器的镜像与配置，应用代码不动——这里是**有意让应用感知"我在连本地"**，透明劫持反而破坏该契约。
- **网格代理的价值恰恰是"应用不知道它存在"**：它是平台统一下发、与业务无关的基础设施。佐证是 **Istio ambient 模式**能把代理整个**移出 Pod**（每节点 ztunnel + 按需 waypoint）而应用无感；ambassador 做不到——它与某个应用的具体依赖强绑定。
- 网格代理也能承担出站到集群外服务的职责（如 Istio `ServiceEntry`），此时功能与 ambassador 重叠，但接管方式（透明 vs 显式 localhost）依然不同。

### 为什么 `localhost` 能打到隔壁容器

因为同一 Pod 的所有容器**共享同一个 network namespace**：K8s 先起 `pause` 容器持有该 netns，其余容器以 `--net=container:pause` 加入，于是看到的是**同一张网卡、同一个 Pod IP、同一个 `lo` 回环、同一个端口空间**。A 容器 `connect("127.0.0.1", 6379)` 与 B 容器 `listen("127.0.0.1", 6379)` 走的是同一个 loopback，内核直接交付——**不出网卡、不经 CNI、不过 iptables**。这不是 K8s 做了转发，而是它们本就在同一台"虚拟主机"里（见 [[concepts/Pod网络模型]]）。

两个推论：

- **端口在 Pod 内是全局的**：ambassador 占了 6379，应用容器就不能再 `bind` 同一端口，否则启动失败。
- **`containerPort` 与此无关**：它纯是声明性文档（给人和工具看），删掉不影响 localhost 互通。

### ambassador 配置范式

场景：应用要连集群外的 Redis，对方要求 TLS + 凭据；用 HAProxy 当大使，应用只管明文连 `localhost:6379`。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-ambassador-cfg
data:
  haproxy.cfg: |
    defaults
      mode tcp
      timeout connect 5s
      timeout client  30s
      timeout server  30s
    frontend redis_in
      bind 127.0.0.1:6379          # 只绑回环，Pod 外部打不到
      default_backend redis_out
    backend redis_out
      server r1 redis.prod.example.com:6379 ssl verify required ca-file /etc/ssl/certs/ca-certificates.crt
---
# Deployment .spec.template.spec 片段
initContainers:
- name: redis-ambassador
  image: haproxy:2.9
  restartPolicy: Always            # 原生 sidecar:先于 app 启动、晚于 app 关闭
  args: ["-f", "/usr/local/etc/haproxy/haproxy.cfg"]
  volumeMounts:
  - { name: cfg, mountPath: /usr/local/etc/haproxy }
  readinessProbe:
    exec: { command: ["sh", "-c", "nc -z 127.0.0.1 6379"] }
containers:
- name: app
  image: myapp:1.0
  env:
  - { name: REDIS_ADDR, value: "localhost:6379" }   # 应用只知道这一行,永远不变
volumes:
- name: cfg
  configMap: { name: redis-ambassador-cfg }
```

应用代码里没有 TLS、没有密码、没有域名。**换环境只改 ConfigMap 里 backend 那一行**（开发指向明文 `redis:6379`，生产指向托管实例），应用镜像不重新构建——这就是 ambassador 的全部价值。同形状的官方实现如 Cloud SQL Auth Proxy：大使侧 `--address=127.0.0.1 --port=5432 <instance>`，应用侧 `postgres://appuser@localhost:5432/mydb?sslmode=disable`（`sslmode=disable` 并不危险，该连接不出 loopback，加密由大使到云端那一段负责）。

用 `initContainers` + `restartPolicy: Always` 而非并列写进 `containers`，是为了拿到启停顺序保证（理由见上文"Init 容器与 sidecar"）。

**三个坑**：

- **绑 `0.0.0.0` 会漏**：那样通过 Pod IP 也能访问，同集群任意 Pod 都能直连大使、白嫖其凭据。默认绑 `127.0.0.1` 锁成 Pod 私有。
- **回环-only 的监听不能用 `httpGet`/`tcpSocket` 探针**：这两类探针由 kubelet **从节点侧**发起，默认连 Pod IP；把 `host` 写成 `127.0.0.1` 指的是**节点自己**的回环。改用 `exec` 探针。
- **启动竞争**：应用起来时大使可能还没 listen，首批连接 `connection refused`。原生 sidecar 顺序 + readinessProbe 可解；集群低于 v1.29 则需应用自带连接重试。

## 镜像拉取策略（imagePullPolicy）

`IfNotPresent`（默认，本地有就不拉）/ `Always`（每次校验远端）/ `Never`（只用本地）。注意 **`:latest` 标签默认 `Always`**；生产应避免 `:latest`（不可复现）。与节点上的镜像管理/GC 相关，见 [[entities/kubelet]]。

## 私有镜像与身份相关字段

- **`imagePullSecrets`**：引用 `kubernetes.io/dockerconfigjson` 类型的 [[entities/Secret]] 拉私有仓库镜像；也可挂到 [[entities/ServiceAccount]] 上让其名下 Pod 自动继承。
- **`serviceAccountName`**：Pod 以哪个 [[entities/ServiceAccount]] 身份访问 API（默认 `default`，token 经投射卷注入）。
- **主机命名空间开关**：`hostNetwork` / `hostPID` / `hostIPC`（共享宿主对应命名空间，见 [[concepts/Pod网络模型]] 端口冲突）、`hostUsers: false`（启用用户命名空间隔离）——均属安全敏感项，详见 [[entities/SecurityContext]]。

## PodDisruptionBudget（PDB）

声明一组 Pod **同时可用的最小数量**（`minAvailable`/`maxUnavailable`），约束**自愿中断**（如 `kubectl drain` 节点维护、滚动升级）一次别端掉太多副本。作用于 Deployment/ReplicaSet/StatefulSet 管理的 Pod。

> 较新：**原地资源调整（In-Place Pod Resize，v1.33 Beta）** 可不重启容器改 CPU/内存（`kubectl ... --subresource resize`），并与 VPA 集成。

## 相关

- 通常不直接创建 Pod，而是由 [[entities/Deployment]] → ReplicaSet 来管理。
- 用 [[concepts/Label与Selector]] 被 Service / 控制器选中。
- 操作工具见 [[entities/kubectl]]。
