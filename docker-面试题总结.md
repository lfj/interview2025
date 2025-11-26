# Docker 面试题总结（Markdown 版）

> 覆盖架构原理、镜像构建、容器运行、网络、存储、Compose、仓库与供应链、安全、性能优化、故障排查、场景题与动手题。高频考点与追问路径，含示例与最佳实践提示。

## 目录
- [基础与架构](#基础与架构)
- [镜像与构建](#镜像与构建)
- [容器运行与资源](#容器运行与资源)
- [网络](#网络)
- [存储与数据管理](#存储与数据管理)
- [Compose 与编排](#compose-与编排)
- [仓库与供应链](#仓库与供应链)
- [安全](#安全)
- [性能与优化](#性能与优化)
- [故障排查](#故障排查)
- [常见场景题](#常见场景题)
- [实战动手题](#实战动手题)

---

## 基础与架构
- 组件链路：`docker cli → dockerd → containerd → runc`，遵循 OCI（镜像与运行时规范）。
- 内核隔离：Linux namespaces（pid/net/mnt/uts/ipc/user）、cgroups（资源约束）、capabilities（权限粒度）。
- 联合文件系统：分层镜像（AUFS/OverlayFS），只读层叠加 + 容器写层。
- 镜像结构：`config.json + manifest + layers`；多架构镜像（manifest list）。
- 与 Kubernetes 的关系：K8s 通过 CRI 使用 `containerd/CRI-O`，Docker 早期通过 dockershim。

## 镜像与构建
- Dockerfile 指令：`FROM`、`RUN`、`COPY`/`ADD`、`ENV`、`ARG`、`ENTRYPOINT`/`CMD`、`WORKDIR`、`HEALTHCHECK`。
- 构建上下文与缓存：`.dockerignore` 精简上下文；指令顺序与内容稳定性影响缓存命中。
- 多阶段构建：在构建阶段安装依赖和编译，最终镜像仅拷贝产物，显著减小体积。
- 标签与不可变性：避免 `latest`；语义化版本；`image:tag@sha256:digest` 双重固定。
- BuildKit 能力：并发、`--secret`、`--mount=type=cache`、内联缓存、`# syntax=docker/dockerfile:1.6`。
- 最佳实践：固定包源、删除构建产物与缓存、以非 root 用户运行、精简基础镜像。

### 示例：Node.js 多阶段构建
```dockerfile
# syntax=docker/dockerfile:1.6
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm npm ci

FROM node:20-alpine AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM node:20-alpine AS runtime
WORKDIR /app
ENV NODE_ENV=production
RUN adduser -D -u 10001 appuser
COPY --from=build /app/dist ./dist
COPY package.json ./
RUN npm prune --production
USER 10001
EXPOSE 8080
HEALTHCHECK --interval=10s --timeout=2s --retries=3 CMD node dist/healthcheck.js || exit 1
CMD ["node", "dist/server.js"]
```

## 容器运行与资源
- 资源限制：`--cpus`、`--memory`、`--memory-swap`、`--pids-limit`、`--cpuset-cpus`。
- 重启策略：`--restart=always|on-failure|unless-stopped`；结合健康检查与退出码管理。
- 健康检查：`HEALTHCHECK` 影响容器状态但不影响主进程；在 K8s 中用探针替代。
- 日志驱动：`json-file`（默认）、`local`、`syslog`、`journald`、`gelf`、`fluentd`；配置日志轮转与磁盘占用。
- UID/GID 与权限：容器内用户与宿主机文件权限匹配，卷挂载常见权限问题。

## 网络
- 网络驱动：`bridge`（默认）、`host`、`none`、`macvlan`、`overlay`（Swarm）。
- 端口发布：`-p 主机:容器`，区分 `EXPOSE`（声明）与 `-p`（实际映射）。
- 用户自定义 bridge：内置 DNS 与服务名解析，容器间互通与隔离。
- 常见问题：端口冲突、NAT/iptables 规则、DNS 解析异常、Hairpin NAT。

## 存储与数据管理
- 卷类型：`volume`（命名卷，Docker 管理）、`bind mount`（宿主机目录）、`tmpfs`。
- 只读根文件系统：`--read-only` 提升安全性；通过 `tmpfs`/指定写目录满足写需求。
- 备份与迁移：命名卷数据迁移与快照；权限与 SELinux/AppArmor 考量。
- Secrets：Swarm 支持 `secrets`；Compose 通过文件挂载模拟。

## Compose 与编排
- 关键段落：`services`、`volumes`、`networks`、`depends_on`、`healthcheck`、`profiles`。
- 多环境：`docker-compose.override.yml`、`.env`、`--profile`、`--env-file`。
- 典型用法：本地多服务集成、网络隔离、命名卷持久化、按需启停。
- 与 Swarm/K8s：Compose 偏本地与小规模；生产编排选 Swarm/K8s（自愈与扩缩容更强）。

### 示例：Compose 三服务
```yaml
version: "3.9"
services:
  api:
    build: ./api
    ports: ["8080:8080"]
    depends_on:
      db:
        condition: service_healthy
    environment:
      DATABASE_URL: "postgres://user:pass@db:5432/app"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/healthz"]
      interval: 10s
      timeout: 2s
      retries: 3
  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: app
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 10s
      timeout: 2s
      retries: 5
  cache:
    image: redis:7-alpine
    command: ["redis-server", "--appendonly", "yes"]
volumes:
  pgdata:
```

## 仓库与供应链
- 镜像仓库：Docker Hub、私有 Registry（Harbor、registry:2）；鉴权与速率限制。
- 内容信任：Docker Content Trust/Notary；现代推荐 Sigstore Cosign。
- 漏洞扫描：内置/外部（Trivy、Clair）；在 CI 阶段阻断不合规镜像。
- 多架构构建：`docker buildx` 生成 manifest list；`--platform=linux/amd64,linux/arm64`。

### 示例：buildx 多架构
```bash
docker buildx create --use
docker buildx build --platform linux/amd64,linux/arm64 -t yourrepo/app:1.0.0 --push .
```

## 安全
- 最小权限：非 root 用户、裁剪 capabilities（如移除 `NET_RAW`）、`seccomp`、`AppArmor`/`SELinux`。
- Rootless Docker：降低宿主机风险，但部分驱动/端口映射有限制。
- 文件系统与挂载：`read-only`、`no-new-privileges`、限制 `--privileged`、设备白名单。
- 供应链安全：固定基础镜像、SBOM、镜像签名与策略验证、私有仓库拉取策略。

## 性能与优化
- 镜像体积：精简基础镜像（alpine/distroless）、多阶段、删除包缓存、合并 `RUN` 指令。
- 构建性能：BuildKit 并行、缓存挂载（`--mount=type=cache`）、避免不稳定命令（时间/随机）。
- 运行性能：合理 CPU/内存限制、日志驱动选择、卷 I/O 优化（减少 overlay 写放大）。
- 网络性能：`host` 模式对吞吐友好但牺牲隔离；合理网络拓扑与连接复用。

## 故障排查
- 核心命令：`docker ps`、`docker logs`、`docker inspect`、`docker events`、`docker image history`、`docker network inspect`。
- 常见错误与思路：
  - `executable file not found`：入口命令路径错误或基础镜像缺少 shell。
  - `permission denied`：文件权限/UID 不匹配、SELinux/AppArmor 限制。
  - 端口冲突：宿主机端口被占用或重复映射。
  - DNS 问题：使用自定义网络或通过 `--dns`/daemon 配置。
  - `no space left on device`：清理未使用镜像/卷/容器；日志膨胀。
- 排查路径：先看 `logs` → `inspect` 配置/环境 → `events` 全局事件 → `network inspect`/`image history` 溯源。

## 常见场景题
- 镜像缓存失效：修改早期层导致后续层缓存失效；通过重排指令与锁定依赖恢复命中。
- 容器数据丢失：写入容器层随删除而消失；使用命名卷持久化与备份。
- 端口发布服务不可达：区分 `EXPOSE` 与 `-p`；服务监听 `127.0.0.1` 导致容器外不可达。
- 多服务依赖：`depends_on` 不等于“就绪”；使用 `healthcheck` 或应用级重试。
- 非 root 用户权限问题：构建时修正文件所有者与权限；卷挂载目录权限。

## 实战动手题
- 编写安全 Dockerfile：非 root、只拷贝产物、`HEALTHCHECK`、固定版本、清理缓存。
- 使用 Compose 运行三服务：API、Postgres、Redis；配置健康检查与命名卷。
- 启动时限制资源：`--cpus 1.5 --memory 512m --pids-limit 200 --ulimit nofile=65535:65535`。
- 用 BuildKit 管理敏感信息：`RUN --mount=type=secret,id=npm_token` 构建时读取密钥，避免进入镜像层。
- 多架构构建与推送：`buildx` + manifest list；用 `docker manifest inspect` 验证。

---

如果你需要我生成精简版或附带示例答案的版本，请说明你的偏好（例如“强调安全与供应链”“附简答参考”）。

---

## Dockerfile 指令速查
- `FROM <image> [AS name]`：指定基础镜像；多阶段构建用 `AS` 命名阶段，后续 `COPY --from=name` 引用。
- `ARG <name>[=<default>]`：构建期变量；只在构建阶段有效，常配合 `FROM` 与 `RUN`。
- `ENV <key>=<value>`：运行期环境变量，写入镜像层；可被 `docker run -e` 覆盖。
- `WORKDIR <path>`：设置工作目录；目录不存在时自动创建，影响后续 `RUN/COPY/CMD`。
- `COPY [--chown=user:group] <src>... <dest>`：复制文件到镜像；推荐优先于 `ADD`；支持 `--from=<stage>`。
- `ADD <src>... <dest>`：同 `COPY`，还支持解压本地 tar 与远程 URL；除非需要这些特性，否则不建议使用。
- `RUN <command>`：在构建时执行命令并生成新层；注意合并命令以减少层数与缓存失效。
- `CMD [","] | CMD <shell>`：容器默认启动命令；可被 `docker run` 参数覆盖；配合 `ENTRYPOINT` 提供默认参数。
- `ENTRYPOINT [":"] | ENTRYPOINT <shell>`：设定主进程；与 `CMD` 组合为“命令 + 默认参数”。
- `EXPOSE <port>[/protocol]`：文档化端口；不做实际映射，配合 `-p` 发布。
- `USER <user>[:group]`：设置后续层及运行时的用户；安全推荐非 root 用户。
- `HEALTHCHECK [options] CMD <command>`：健康检查；失败时容器状态为 unhealthy。
- `VOLUME <path>`：声明数据卷挂载点；实际挂载由 `docker run -v` 或编排工具决定。
- `LABEL <key>=<value>`：添加镜像元数据（作者、版本、来源等）；便于审计与管理。
- `ONBUILD <instruction>`：为子镜像定义触发的指令；避免在通用基础镜像中滥用。
- `SHELL ["/bin/sh","-c"]`：修改 `RUN` 等指令的默认 shell。
- `STOPSIGNAL <signal>`：定义停止容器时发送给主进程的信号。

**多阶段构建要点**
- 用 `FROM <builder> AS build` 进行编译，`FROM <runtime>` 仅复制产物与所需依赖，减小镜像体积。
- `COPY --from=build <src> <dest>` 从编译阶段拷贝产物；生产镜像不携带编译工具链。

**缓存与层优化**
- `.dockerignore` 排除无关文件，减少上下文体积与缓存失效概率。
- 先复制依赖清单（如 `package.json`/`go.mod`），再安装依赖，最后复制源码，以提升缓存命中。
- 合理合并 `RUN` 指令并清理包缓存；锁定版本与源，避免非确定性命令导致缓存失效。

**安全与最佳实践**
- 使用非 root 用户（`USER`）、设置只读根文件系统（运行时 `--read-only`）、限制能力与写入路径。
- 优先 `COPY` 而非 `ADD`；对敏感信息使用 BuildKit 的 `--mount=type=secret`。
- 提供 `HEALTHCHECK` 和最小化运行时依赖（`alpine` 或 `distroless`），降低攻击面与体积。

**示例骨架**
```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine AS runtime
WORKDIR /app
ENV NODE_ENV=production
RUN adduser -D -u 10001 appuser
COPY --from=build /app/dist ./dist
COPY package.json ./
RUN npm prune --production
USER 10001
EXPOSE 8080
HEALTHCHECK --interval=10s --timeout=2s --retries=3 CMD node dist/healthcheck.js || exit 1
CMD ["node", "dist/server.js"]
```