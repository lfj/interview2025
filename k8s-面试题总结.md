# Kubernetes 面试题总结（Markdown 版）

> 覆盖基础概念、核心对象、调度、网络、存储、安全、可观测性、扩缩容、发布策略、多集群与高可用、生态扩展、故障排查与实战题。按模块组织，含高频追问与作答要点。

## 目录
- [基础与架构](#基础与架构)
- [核心对象与工作负载](#核心对象与工作负载)
- [调度与资源管理](#调度与资源管理)
- [网络](#网络)
- [存储](#存储)
- [配置与密钥](#配置与密钥)
- [安全与合规](#安全与合规)
- [可观测性与故障排查](#可观测性与故障排查)
- [扩缩容与发布策略](#扩缩容与发布策略)
- [生态扩展与管理](#生态扩展与管理)
- [多集群与高可用](#多集群与高可用)
- [云原生与落地实践](#云原生与落地实践)
- [网络策略与流量治理](#网络策略与流量治理)
- [调试与性能优化](#调试与性能优化)
- [常见场景题](#常见场景题开放式)
- [实战动手题](#实战动手题)

---

## 基础与架构
1. 说说 Kubernetes 控制面与数据面组成
   - 要点：API Server、Controller Manager、Scheduler、etcd、kubelet、kube-proxy；控制循环与期望状态。
   - 追问：为什么 etcd 一致性重要；API Server 接入认证授权流程。

2. etcd 的角色与一致性模型
   - 要点：存储集群状态，Raft 保证线性一致；快照与备份策略。
   - 追问：写放大/读性能影响；灾备恢复步骤。

3. Kubernetes 的声明式管理
   - 要点：期望状态对比实际状态，控制器驱动；幂等与重试。
   - 追问：与命令式的区别；冲突解决（乐观并发，`resourceVersion`）。

4. Namespace 与资源隔离
   - 要点：逻辑边界；与 NetworkPolicy/ResourceQuota 联用实现多租户。
   - 追问：跨 Namespace 访问与权限控制。

---

## 核心对象与工作负载
1. Pod 生命周期与常见状态
   - 要点：`Pending`、`Running`、`CrashLoopBackOff`；`RestartPolicy`（Always/OnFailure/Never）。
   - 追问：容器退出码、探针对流量影响。

2. Deployment vs StatefulSet vs DaemonSet
   - 要点：有状态与无状态差异；唯一标识、顺序与持久卷；DaemonSet 用于节点级守护。
   - 追问：滚动更新与回滚；有序滚动与停机窗口。

3. ReplicaSet 与回滚机制
   - 要点：Deployment 管理 RS；`kubectl rollout undo`；变更记录。
   - 追问：不可变字段变更策略（创建新 RS）。

4. Job/CronJob 并发与重试
   - 要点：`completions`、`parallelism`；失败重试与时间窗口。
   - 追问：幂等任务设计；CronJob missed runs 处理。

5. InitContainer 与 Sidecar
   - 要点：初始化依赖、注入代理/日志；生命周期与重启。
   - 追问：热更新与 Sidecar 升级策略。

---

## 调度与资源管理
1. 调度流程（过滤/打分/绑定）
   - 要点：Predicates/Score；可插拔策略与调度扩展。
   - 追问：自定义调度器；多调度器并存。

2. `requests`/`limits` 与 OOM 排查
   - 要点：调度依据 requests；cgroup 限制与 OOMKill。
   - 追问：内存泄漏定位；`resourceQuota` 与 `limitRange`。

3. Affinity 与 Taints/Tolerations
   - 要点：Node/Pod Affinity/AntiAffinity；污点与容忍搭配精细调度。
   - 追问：稳定性与可用区分布；避免热点。

4. PriorityClass 与 Preemption
   - 要点：高优先级抢占资源；生产风险与保护。
   - 追问：禁止关键系统被抢占的配置。

5. PodTopologySpreadConstraints
   - 要点：在故障域内分散副本；可用性提升。
   - 追问：与 AntiAffinity 的取舍与组合。

---

## 网络
1. CNI 原理与实现对比（Calico/Flannel/Cilium）
   - 要点：覆盖网络/路由/BPF；性能与功能权衡。
   - 追问：网络策略能力；多集群网络。

2. Service 类型与选择
   - 要点：ClusterIP/NodePort/LoadBalancer/Headless；DNS 解析。
   - 追问：外部访问拓扑；Local vs ClusterIP 负载策略。

3. kube-proxy 模式
   - 要点：iptables vs ipvs；性能与可观测性差异。
   - 追问：大规模集群下的选择与调优。

4. Ingress 与 Gateway API
   - 要点：Ingress 控制器；Gateway API 更强表达能力。
   - 追问：灰度/金丝雀；TLS/HTTP2/gRPC 支持。

---

## 存储
1. PV/PVC/StorageClass 与动态供给
   - 要点：声明式申请存储；驱动匹配与回收策略。
   - 追问：绑定模式；不同 SC 的性能。

2. 访问模式与限制
   - 要点：RWO/ROX/RWX；有状态服务横向扩展。
   - 追问：NFS/Ceph 与云盘对比。

3. StatefulSet 与持久卷
   - 要点：基于模板生成有序卷；数据迁移与扩容。
   - 追问：卷快照与备份。

4. CSI 驱动
   - 要点：统一接口；常见驱动（EBS、Ceph、NFS）。
   - 追问：拓展卷与在线扩容。

---

## 配置与密钥
1. ConfigMap 与 Secret
   - 要点：用途与挂载方式（env/volume）；Secret 编码与类型。
   - 追问：热更新触发；安全审计。

2. 配置热更新
   - 要点：checksum 注入触发 Deployment 滚动；`subPath` 限制。
   - 追问：无损重载策略。

3. 多环境配置管理
   - 要点：Values 分层/覆盖；避免漂移。
   - 追问：Helm 与 Kustomize 对比。

---

## 安全与合规
1. 认证（AuthN）与授权（AuthZ）
   - 要点：证书、Bearer、OIDC；RBAC 角色与绑定。
   - 追问：最小权限落地；审计日志。

2. Admission 控制器
   - 要点：Mutating vs Validating；合规策略落地。
   - 追问：失败回滚与可用性影响。

3. Pod Security（PSA）
   - 要点：代替 PSP；baseline/restricted 等级。
   - 追问：rootfs 只读、非 root、能力裁剪。

---

## 可观测性与故障排查
1. 排查三件套
   - 要点：`kubectl describe`、`kubectl logs`、`kubectl get events`。
   - 追问：事件风暴抑制；日志侧链。

2. 探针设计
   - 要点：Liveness/Readiness/Startup；防止雪崩与误杀。
   - 追问：慢启动服务的探针参数。

3. 监控指标
   - 要点：Prometheus/Grafana；核心指标（CPU/Mem、QPS、错误率、延迟、容器重启）。
   - 追问：Exporter 选择与成本。

4. 节点健康与 NotReady
   - 要点：kubelet 心跳；磁盘/网络/内核问题。
   - 追问：驱逐与优雅停机。

---

## 扩缩容与发布策略
1. HPA/VPA/CA
   - 要点：HPA 按指标扩缩；VPA 资源建议；Cluster Autoscaler 节点层扩缩。
   - 追问：自定义指标（Prometheus Adapter）。

2. RollingUpdate 与灰度
   - 要点：`maxSurge`/`maxUnavailable`；有序与并发。
   - 追问：发布失败快速回滚。

3. 蓝绿/金丝雀发布
   - 要点：通过 Service/Ingress/Mesh 实现；逐步流量切换。
   - 追问：回滚与数据兼容。

---

## 生态扩展与管理
1. Helm
   - 要点：Chart 结构、Values 合并、依赖管理。
   - 追问：钩子与发布策略；模板可维护性。

2. Operator 与 CRD
   - 要点：声明式管理复杂系统；示例（DB/缓存）。
   - 追问：控制循环设计；错误处理。

3. GitOps（ArgoCD/Flux）
   - 要点：拉模式、漂移检测、审计可追溯。
   - 追问：多环境与多集群治理。

4. Service Mesh（Istio/Linkerd）
   - 要点：流量治理、可观测性、安全；成本与复杂度。
   - 追问：逐步引入策略。

---

## 多集群与高可用
1. 控制面高可用
   - 要点：多副本 API Server；etcd 集群部署与隔离。
   - 追问：升级与证书轮换。

2. 节点池与容量规划
   - 要点：按资源/工作负载类型分池；抢占风险控制。
   - 追问：Spot/Preemptible 权衡。

3. 多集群服务发现
   - 要点：MCS、Mesh、外部流量治理。
   - 追问：跨地域延迟与一致性。

4. 备份与灾备
   - 要点：etcd 备份、卷快照；恢复演练。
   - 追问：RPO/RTO 目标。

---

## 云原生与落地实践
1. 托管 K8s vs 自建
   - 要点：EKS/GKE/AKS 与自建的差异；控制面运维简化。
   - 追问：限制与可移植性。

2. 外部 LB 与存储集成
   - 要点：云 LB 特性；存储性能与一致性。
   - 追问：跨 AZ 架构与成本。

3. 成本优化
   - 要点：配额与利用率；按工作负载分层；Spot。
   - 追问：自动化扩缩与限流保护。

4. 供应链安全
   - 要点：镜像签名、扫描、SBOM；拉取策略与私有仓库。
   - 追问：漏洞响应流程。

---

## 网络策略与流量治理
1. NetworkPolicy 语义
   - 要点：默认拒绝、白名单、东西向隔离。
   - 追问：跨 Namespace 控制；与 CNI 能力匹配。

2. Ingress 控制器对比
   - 要点：Nginx/Traefik/Contour；生态与性能。
   - 追问：CRD 扩展能力。

3. TLS 与证书管理
   - 要点：终止、双向 TLS；证书轮换与自动化。
   - 追问：SNI 与多租户。

4. gRPC/HTTP2 注意点
   - 要点：负载均衡、连接复用、探针设计。
   - 追问：代理兼容性。

---

## 调试与性能优化
1. 容器启动慢/内存泄漏/CPU 飙升
   - 要点：冷启动优化；profiling；cgroup 限制。
   - 追问：镜像瘦身与分层。

2. 节点资源争抢与热点
   - 要点：NUMA/IO；调度分散；亲和与拓扑约束。
   - 追问：观测与缓解措施。

3. 密集部署上限与内核调优
   - 要点：proc/inotify/conntrack 限制；文件描述符。
   - 追问：kubelet 参数与垃圾回收。

4. 运行时问题（containerd/CRI-O）
   - 要点：镜像拉取、层缓存、日志。
   - 追问：拉取失败与重试策略。

---

## 常见场景题（开放式）
1. `CrashLoopBackOff` 排查路径
   - 要点：查看退出码与日志；探针是否误杀；依赖是否就绪；资源是否不足。
   - 方案：放宽 Startup/Readiness；增加资源；修复启动顺序。

2. Pod 长时间 `Pending`
   - 要点：调度约束过严、资源不足、准入失败。
   - 方案：检查 events；调整 requests/limits；放宽亲和/污点；扩容节点。

3. 新版本 5% 请求失败
   - 要点：快速回滚；定位配置差异与兼容问题。
   - 方案：`kubectl rollout undo`；比对 Config/Secret；逐步灰度。

4. 线上误删资源
   - 要点：最小化影响；恢复与审计。
   - 方案：从 GitOps/备份恢复；临时保护策略；复盘与预防。

5. 流量突增的自动扩容
   - 要点：HPA 指标与冷启动；限流与熔断。
   - 方案：预热副本，设置最大副本与速率限制。

---

## 实战动手题
1. 带 Readiness 探针的 Deployment（示例）
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
        - name: app
          image: your-registry/demo:1.0.0
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
            timeoutSeconds: 2
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /livez
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
```

2. HPA 基于自定义指标
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: demo-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: demo
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Pods
      pods:
        metric:
          name: requests_per_second
        target:
          type: AverageValue
          averageValue: "100"
```
> 需结合 Prometheus Adapter 暴露自定义指标。

3. Namespace 级默认拒绝与白名单 NetworkPolicy
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: prod
spec:
  podSelector: {}
  policyTypes: ["Ingress","Egress"]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-monitoring
  namespace: prod
spec:
  podSelector:
    matchLabels:
      app: demo
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: monitoring
```

4. Helm Chart 金丝雀发布要点
   - 分离 `stable` 与 `canary` 值文件；`weight`/`header` 路由；单独 Service/Ingress；回滚快捷。

5. ValidatingAdmissionWebhook 验证镜像来源
   - 校验 `image` 前缀、签名策略；失败拒绝创建；记录审计。

---

如果你需要我对内容进行精简版或加上示例答案，请告诉我你的偏好（例如“突出调度/网络”“附带简答参考”）。