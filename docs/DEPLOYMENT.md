# 部署文档

**版本** v1

覆盖:本地开发、Docker 容器化、云端部署、CI/CD 自动发布、运维要点。

---

## 1. 部署形态总览

| 环境 | 形态 | 用途 |
|---|---|---|
| **本地开发** | docker-compose(部分服务本地进程) | 日常开发 |
| **本地完整** | docker-compose 全栈 | 集成验证、演示 |
| **云端演示** | 单机云主机 / PaaS | 公开可访问的演示环境 |
| **生产(未来)** | Kubernetes | 有多副本需求时(当前不做,见 §22) |

---

## 2. 本地运行

### 2.1 前置要求

| 依赖 | 版本 | 说明 |
|---|---|---|
| Docker Desktop | 最新稳定版 | Windows 需启用 WSL2 后端 |
| 内存 | ≥ 8 GB 可用 | 全栈约占 4–6 GB |
| 磁盘 | ≥ 20 GB | 镜像 + 数据卷 |

### 2.2 启动

```bash
cp .env.example .env
# 填入模型 API Key 等必需配置
docker compose up -d
```

```bash
docker compose ps
```

服务就绪后:

| 地址 | 服务 |
|---|---|
| http://localhost:3000 | 前端 |
| http://localhost:8000/docs | API 文档 |
| http://localhost:8000/healthz | 健康检查 |

### 2.3 profiles(按需启动)

```bash
docker compose up -d
```

```bash
docker compose --profile obs up -d
```

```bash
docker compose --profile eval run --rm harness run --suite factual
```

**分 profile 的意义**:开发时只起核心服务,启动时间从约 90 秒降到约 30 秒。可观测组件(Prometheus/Grafana)只在需要时拉起。

### 2.4 常用命令

```bash
make up
```

```bash
make logs
```

```bash
make migrate
```

```bash
make eval
```

```bash
make down
```

---

## 3. 容器化设计

### 3.1 镜像构建

采用多阶段构建,builder 阶段安装依赖并编译,runtime 阶段只保留运行期所需内容。

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /build
COPY pyproject.toml uv.lock ./
RUN pip install --no-cache-dir uv && uv sync --frozen --no-dev

FROM python:3.12-slim AS runtime
RUN groupadd -r app && useradd -r -g app -u 10001 app
WORKDIR /app
COPY --from=builder /build/.venv /app/.venv
COPY --chown=app:app . .
ENV PATH="/app/.venv/bin:$PATH" PYTHONUNBUFFERED=1
USER app
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=5s --start-period=40s --retries=3 \
  CMD python -c "import httpx;httpx.get('http://localhost:8000/healthz').raise_for_status()"
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**关键实践**

| 实践 | 理由 |
|---|---|
| 多阶段构建 | 运行镜像不含编译工具链,体积减小、攻击面减小 |
| 依赖层与代码层分离 | 改代码不触发依赖重装,构建缓存命中率高 |
| 非 root 用户 | 容器逃逸时限制权限 |
| 锁文件固定依赖 | 构建可复现 |
| HEALTHCHECK | 编排层可判断真实就绪状态 |
| slim 基础镜像 + 固定标签 | 体积与可复现性 |

### 3.2 服务清单

```yaml
services:
  frontend:        # Nginx + React 构建产物
  api:             # FastAPI (Uvicorn)
  worker:          # 队列消费者
  sandbox-runner:  # 受限代码执行
  postgres:        # PG16 + pgvector
  redis:           # Stream / 缓存 / 限流
  otel-collector:  # 追踪采集
  prometheus:      # [profile: obs]
  grafana:         # [profile: obs]
  harness:         # [profile: eval] 一次性任务
```

### 3.3 sandbox-runner 安全加固

```yaml
sandbox-runner:
  build: ./sandbox-runner
  network_mode: none                  # 无网络
  read_only: true                     # 只读根文件系统
  tmpfs:
    - /tmp:size=64m,noexec,nosuid
  mem_limit: 512m
  cpus: 1.0
  pids_limit: 64
  cap_drop: [ALL]
  security_opt:
    - no-new-privileges:true
  user: "10001:10001"
```

> `network_mode: none` 意味着 sandbox 无法被其他容器通过网络访问。调用方式为:worker 通过 Docker socket 按需创建一次性执行容器,或 sandbox 以独立网络暴露最小接口但禁止出站。**当前实现采用后者:sandbox 加入内部网络但通过 iptables 规则禁止出站流量。**

### 3.4 启动顺序与健康检查

```yaml
postgres:
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER -d $$POSTGRES_DB"]
    interval: 5s
    retries: 10

api:
  depends_on:
    postgres: {condition: service_healthy}
    redis:    {condition: service_healthy}
```

`/readyz` 检查数据库连通性、Redis 连通性、**迁移版本一致性**。迁移版本不匹配时返回未就绪,防止代码与 schema 不一致时接收流量。

### 3.5 数据卷

| 卷 | 内容 | 备份要求 |
|---|---|---|
| `pgdata` | 业务数据、向量、检查点 | **必须备份** |
| `redisdata` | 队列与缓存(AOF) | 可重建 |
| `prometheus_data` | 指标历史 | 可选 |
| `grafana_data` | 看板配置 | 建议(配置即代码则不需要) |

---

## 4. 配置管理

### 4.1 环境变量

遵循 12-factor,全部配置经环境变量注入,由 `pydantic-settings` 做类型化加载与**启动时校验**——缺失必填项时立即失败,而非运行到一半才报错。

`.env.example`(占位符,不含真实值):

```bash
# 数据库
POSTGRES_HOST=postgres
POSTGRES_DB=agentpro
POSTGRES_USER=agentpro
POSTGRES_PASSWORD=change_me

# Redis
REDIS_URL=redis://redis:6379/0

# 模型
ANTHROPIC_API_KEY=sk-ant-xxx
LLM_TIER_STRONG=claude-opus-x
LLM_TIER_MEDIUM=claude-sonnet-x
LLM_TIER_FAST=claude-haiku-x

# 认证
JWT_SECRET=change_me
JWT_ALGORITHM=HS256

# 预算
BUDGET_QUICK=60000
BUDGET_STANDARD=200000
BUDGET_DEEP=500000

# 可观测
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
LOG_LEVEL=INFO

# 金融数据源
MARKET_DATA_API_KEY=
NEWS_API_KEY=
```

### 4.2 密钥管理

| 环境 | 方式 |
|---|---|
| 本地 | `.env` 文件(已加入 `.gitignore`) |
| CI | GitHub Actions Secrets |
| 云端 | 平台密钥管理服务,或云主机上的受限权限 `.env` |

**约束**:密钥不进入版本库、不进入镜像、不出现在日志与追踪中(日志层对密钥字段做脱敏)。

---

## 5. 云端部署

### 5.1 方案对比

| 方案 | 月成本(估) | 部署复杂度 | 适用 |
|---|---|---|---|
| **A. 单台云主机 + docker compose** | 低 | 低 | **推荐:演示环境** |
| B. PaaS(Railway / Render / Fly.io) | 中 | 最低 | 快速上线,不想管服务器 |
| C. 托管数据库 + 容器服务 | 中高 | 中 | 需要数据库高可用 |
| D. Kubernetes | 高 | 高 | 有多副本与弹性需求(当前不做) |

### 5.2 推荐方案 A:单台云主机

**规格建议**

| 项 | 建议 |
|---|---|
| CPU / 内存 | 4 vCPU / 8 GB(全栈含可观测组件) |
| 磁盘 | 60 GB SSD |
| 系统 | Ubuntu 22.04 LTS |
| 网络 | 公网 IP + 域名 |

**部署步骤**

```bash
sudo apt update && sudo apt install -y docker.io docker-compose-plugin
```

```bash
sudo usermod -aG docker $USER && newgrp docker
```

```bash
git clone <repo> && cd AI_Agent_Pro
```

```bash
cp .env.example .env && vim .env
```

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

**`docker-compose.prod.yml` 的差异**

| 项 | 开发 | 生产 |
|---|---|---|
| 端口暴露 | 各服务直接暴露 | **仅 Caddy/Nginx 暴露 80/443**,其余仅内部网络 |
| 日志 | 文本 | JSON + 大小轮转 |
| 重启策略 | `no` | `unless-stopped` |
| 资源限额 | 无 | 各服务设 `mem_limit` |
| 调试端点 | 开放 | 关闭 |
| 数据库 | 容器内 | 容器内或托管实例 |

### 5.3 HTTPS 与反向代理

使用 Caddy 自动申请与续期证书:

```
your-domain.com {
    handle /api/* {
        reverse_proxy api:8000
    }
    handle {
        reverse_proxy frontend:80
    }
}
```

**SSE 注意事项**:反向代理必须关闭响应缓冲,否则事件会被攒着一起发,前端看到的进度是"卡很久然后全部出现"。Caddy 默认对 `text/event-stream` 处理正确;若用 Nginx 需显式配置:

```nginx
location /api/v1/research/ {
    proxy_pass http://api:8000;
    proxy_buffering off;
    proxy_cache off;
    proxy_read_timeout 3600s;
    proxy_set_header Connection '';
    proxy_http_version 1.1;
}
```

### 5.4 演示环境的额外约束

公开可访问的演示环境需要额外防护,否则会被刷爆模型额度:

| 措施 | 说明 |
|---|---|
| **强制登录** | 演示账号 + 密码,不开放匿名访问 |
| **严格限流** | 按 IP 与账号双限,单账号每日任务数上限 |
| **成本熔断** | 日成本超阈值自动停止接收新任务 |
| **只读数据集** | 预置固定的文档与行情数据,不允许上传 |
| **depth 限制** | 演示环境只允许 `quick` 与 `standard` |

**这不是可选项。** 一个无防护的公开 LLM 应用可能在数小时内消耗掉大量额度。

### 5.5 方案 B:PaaS 部署

适合不想维护服务器的情况。以 Railway / Render 为例:

| 组件 | 部署方式 |
|---|---|
| api | Web Service(Dockerfile) |
| worker | Background Worker(Dockerfile,无端口) |
| sandbox-runner | Private Service |
| postgres | 平台托管 PostgreSQL(需确认支持 pgvector 扩展) |
| redis | 平台托管 Redis |
| frontend | Static Site |

**注意事项**

- **确认托管 PostgreSQL 支持 pgvector**。部分平台默认不启用扩展,需要提前确认或改用支持的实例类型
- 平台的健康检查路径需配置为 `/healthz`
- worker 不暴露端口,配置为 background worker 类型
- sandbox-runner 在 PaaS 上的隔离能力弱于自管容器(无法完全控制 capabilities),**若安全要求高应选方案 A**

### 5.6 数据库迁移

容器启动时执行迁移检查:

| 环境 | 行为 |
|---|---|
| 开发 | 自动执行 `alembic upgrade head` |
| 生产 | **不自动迁移**,由发布流程显式执行;版本不一致时 `/readyz` 返回未就绪 |

```bash
docker compose exec api alembic upgrade head
```

生产环境不自动迁移的理由:自动迁移在多副本场景会并发执行,且不可控的 schema 变更难以回滚。显式执行使迁移成为发布流程中可审查的一步。

---

## 6. CI/CD

### 6.1 流水线

```
push / pull_request
  ├─ lint            ruff check + format check
  ├─ typecheck       mypy
  ├─ deps-check      模块依赖方向静态检查
  ├─ unit            pytest(无外部依赖)
  ├─ integration     testcontainers(PG + Redis)
  ├─ eval-gate       harness 评测子集 vs 基线      ← 关键门禁
  ├─ build           构建镜像
  └─ publish         [main] 推送镜像 + 部署
```

### 6.2 镜像发布

```yaml
- uses: docker/build-push-action@v5
  with:
    context: ./api
    push: true
    tags: |
      ghcr.io/${{ github.repository }}/api:latest
      ghcr.io/${{ github.repository }}/api:${{ github.sha }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

镜像同时打 `latest` 与 commit SHA 标签,**部署时使用 SHA 标签**——`latest` 不可追溯,出问题时无法确定线上跑的是哪个版本。

### 6.3 自动部署

main 分支构建成功后,通过 SSH 在云主机上拉取新镜像并滚动重启:

```bash
docker compose pull && docker compose up -d --no-deps api worker
```

`--no-deps` 避免连带重启数据库;逐个服务更新减少不可用时间。

### 6.4 评测门禁的成本控制

评测会产生真实的模型调用费用,需要控制:

| 措施 | 说明 |
|---|---|
| 子集抽样 | CI 只跑约 30% 样本(固定种子,保证可比) |
| 档位降级 | 门禁评测使用 fast 档位 |
| 触发条件 | 仅在 prompt / Agent 定义 / 检索参数 / 模型配置变更时触发 |
| 完整评测 | 放到定时任务(nightly),不阻塞 PR |

---

## 7. 运维

### 7.1 监控与告警

| 告警 | 条件 | 处置 |
|---|---|---|
| 成本异常 | 单位时间成本超基线倍数 | 检查是否有异常长任务或攻击 |
| 失败率上升 | run 失败率超阈值 | 查看 trace 定位失效环节 |
| 队列堆积 | `queue_depth` 持续超阈值 | 增加 worker 并发或副本 |
| 延迟劣化 | p95 超阈值 | 检查模型 API 延迟与检索延迟 |
| 磁盘 | 使用率 > 80% | 清理旧 trace 与日志分区 |

### 7.2 备份

```bash
docker compose exec -T postgres pg_dump -U agentpro agentpro | gzip > backup_$(date +%F).sql.gz
```

- 频率:每日
- 保留:7 日滚动 + 每月一份长期
- **定期验证恢复**:未验证过的备份等于没有备份

### 7.3 数据保留

`run_event` 与 `audit_log` 按时间分区,保留期可配置。超期分区归档或删除,防止表无限膨胀影响查询性能。

### 7.4 优雅关闭

| 服务 | 关闭行为 |
|---|---|
| api | 停止接收新请求,等待进行中请求完成(≤30s) |
| worker | **停止拉取新任务,当前任务执行到下一个检查点后退出** |
| — | 未 ack 的消息由其他 worker 认领 |

worker 的优雅关闭依赖检查点机制:即使被强制终止,任务也能从最近检查点恢复,不会丢失。

---

## 8. 故障排查

| 现象 | 排查方向 |
|---|---|
| 服务起不来 | `docker compose logs <service>`;检查 `.env` 必填项;检查端口占用 |
| `/readyz` 不通过 | 数据库/Redis 连通性;迁移版本是否一致 |
| SSE 无输出 | 反向代理是否关闭了缓冲;浏览器控制台是否有连接错误 |
| 任务一直 queued | worker 是否运行;`queue_depth` 指标;Redis 连通性 |
| 任务卡在 running | 查看该 run 的 trace 定位卡住的节点;检查是否等待 HITL 审批 |
| 成本异常 | 按 run 查 `llm_call` 表,定位高消耗 Agent;检查是否触发循环 |
| 检索结果差 | 先跑检索独立评测,区分是检索问题还是生成问题 |
| pgvector 报错 | 确认扩展已创建:`CREATE EXTENSION IF NOT EXISTS vector;` |

**排查起点始终是 `trace_id`**:从前端或日志拿到 trace_id,即可在追踪系统中看到完整执行链路。

---

## 9. 未来:Kubernetes 迁移准备

当前不上 K8s,但设计已满足云原生要求,迁移时不需要重构:

| 要求 | 当前状态 |
|---|---|
| 无状态服务 | api/worker 无本地状态,状态全在 PG/Redis |
| 健康探针 | `/healthz`(liveness)、`/readyz`(readiness)已实现 |
| 配置外部化 | 全部经环境变量,可直接映射 ConfigMap/Secret |
| 优雅关闭 | 已处理 SIGTERM |
| 水平扩展 | worker 通过 consumer group 天然支持多副本 |
| 日志 | 输出到 stdout,JSON 格式 |
| 镜像 | 已构建并推送到 registry |

迁移触发条件见 [ARCHITECTURE.md §22](./ARCHITECTURE.md#22-不引入的技术及触发条件)。

---

*文档版本 v1*
