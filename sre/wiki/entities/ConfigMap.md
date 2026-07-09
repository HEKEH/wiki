---
title: ConfigMap
date: 2026-06-30
tags: [kubernetes, configmap, 配置, 配置分离]
sources: [kubernetes/objects/configmap.md]
---

# ConfigMap

ConfigMap 保存**非敏感**配置的 key-value（单个属性或整份配置文件），实现**应用与配置分离**——改配置不必重建镜像、重打包。它与 [[entities/Secret]] 形态相同，区别是不针对敏感数据、不加密。

## 创建方式

```bash
kubectl create configmap special-config --from-literal=special.how=very   # 字面量
kubectl create configmap env-config      --from-env-file=config.env        # env 文件
kubectl create configmap special-config  --from-file=config/               # 目录（每文件一项）
```

也可直接写 YAML（`data:` 下列 key-value），既能存**单个属性**、也能存**整份配置文件**：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
data:
  # 1) 单个属性（key: value）
  log_level: "INFO"
  app.port: "8080"
  # 2) 整份配置文件（key 当文件名，value 用 | 保留多行内容）
  app.conf: |
    server.host=0.0.0.0
    server.timeout=30s
```

`kubectl apply -f app-config.yaml` 创建。（存二进制内容用 `binaryData` + base64。）

## 三种使用方式

| 方式 | 写法 | 说明 |
| --- | --- | --- |
| **环境变量** | `env.valueFrom.configMapKeyRef`（单 key）/ `envFrom.configMapRef`（整份） | 最常用；`envFrom` 自动忽略非法 key 名 |
| **命令行参数** | 先注入 env，再 `$(VAR_NAME)` 引用 | 间接经环境变量 |
| **Volume 挂载** | `volumes.configMap` | 每个 key → 一个文件（key=文件名、value=内容） |

以上文 `app-config` 为例，三种写法（放在 Pod 的 `spec.containers` 里）：

### ① 环境变量

```yaml
containers:
- name: app
  image: busybox
  env:                                  # 取单个 key，可改名
  - name: LOG_LEVEL
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: log_level
  envFrom:                              # 或整份注入（key 即变量名）
  - configMapRef:
      name: app-config
```

### ② 命令行参数（先注入 env，再 `$(VAR)` 引用）

```yaml
containers:
- name: app
  image: busybox
  env:
  - name: LOG_LEVEL
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: log_level
  command: ["/app"]
  args: ["--level=$(LOG_LEVEL)"]
```

### ③ Volume 挂载（每个 key 变成挂载目录下的一个文件）

```yaml
containers:
- name: app
  image: busybox
  volumeMounts:
  - name: cfg
    mountPath: /etc/app                 # 生成 /etc/app/log_level、/etc/app/app.conf …
volumes:
- name: cfg
  configMap:
    name: app-config
    # items: [{key: app.conf, path: app.conf}]   # 可选：只挑部分 key
```

读法：**`volumes`** 声明"卷 `cfg` 的内容来自 `app-config`"、**`volumeMounts`** 把 `cfg` 挂到 `/etc/app`，两者靠 **`name` 对接**（拆两半是因为同一卷可被多容器挂到不同路径）；挂进去后**每个 key 变成一个同名文件**（`/etc/app/log_level`=`INFO`、`/etc/app/app.conf`=那份多行配置），`items` 则只挑部分 key 并可改文件名。

> ⚠️ 两个坑：① `envFrom` 用 key 名当变量名，`app.port` 这类**带点的非法名会被跳过**（要用得靠 ① 单独取并改名）；② **env 是启动时一次性读入，改 ConfigMap 不会更新已运行容器**，需 `kubectl rollout restart`；而 **Volume 挂载的文件会自动更新**（应用需自己感知文件变化）。

挂载细节：
- `items` 可只挑特定 key 并自定义子路径（`path`）。
- 默认挂载会**覆盖**整个目标目录；用 **`subPath`** 只挂单个文件、保留目录原有内容（如往 `/etc/nginx/` 里塞一个文件）。

## 注意事项

- 必须**先于**引用它的 Pod 创建；Pod 只能引用**同 [[entities/Namespace]]** 内的 ConfigMap。
- **不可变 ConfigMap**（`immutable: true`，v1.21 GA）：禁止修改，K8s 关闭其 watch——大量 ConfigMap 时显著降 [[entities/kube-apiserver]] 压力，也防误更新。

## 相关

- 敏感数据用：[[entities/Secret]]
- 作为卷的一种：[[concepts/Volume存储]]（projected/subPath）
