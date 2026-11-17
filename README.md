# AgenticOS - LazyCat LPK

[AgenticOS](https://github.com/vstorm-co/agenticos)（Apache-2.0）的懒猫微服打包：开源、可自托管的 AI 智能体平台——共享聊天、RAG 知识库、技能、4700+ MCP 服务器、预算、审批与审计，基于 Pydantic AI。

发布到 **喵喵商店**（lazycat-contrib），镜像模式（`delivery.mode: mirror`），版本跟随上游 ghcr.io/vstorm-co/agenticos-backend 的 `1.x.y` tag（git tag `v1.x.y` → 镜像 tag `1.x.y`）。

## 服务结构

| 服务 | 镜像 | 说明 |
|------|------|------|
| `db` | pgvector/pgvector:pg16 | Postgres 16 + pgvector（RAG 向量存储必需） |
| `redis` | valkey/valkey:8-alpine | BSD 分支的 Redis 兼容缓存，不落盘（`--save ''`） |
| `app` | agenticos-backend | API + 启动前自动迁移（`agenticos db upgrade && exec ...`） |
| `worker` | agenticos-backend | 后台工作进程（触发器、摄取、同步、清理） |
| `frontend` | agenticos-frontend | Next.js 控制台（BFF：服务端代理 API + httpOnly Cookie 会话） |

路由（单域名两上游）：
- `/api/v1/` → `app:8000`（浏览器直连：聊天 WebSocket `/api/v1/ws/agent`、OAuth 回调、嵌入/工件页）
- 其余 → `frontend:3000`（控制台页面与 BFF 路由 `/api/<resource>`）

前端 BFF 路由全部位于 `/api/<name>`（无 `/api/v1` 前缀），与后端路由不冲突。

## 与上游 compose 的差异

- **migrate 一次性容器** → 并入 `app` 服务启动命令串行执行（Alembic 幂等，已到 head 时空操作），避免 LPK 平台对退出容器做常驻管理。
- **sandboxd / browser / browser-egress / minio / artifact-render 未打包**：均为上游 compose profile 可选项——sandboxd 需要 Docker 套接字（宿主 root 等价权限），browser 需要配套 squid 配置与专用网络；对象存储/渲染为可选后端。核心聊天/智能体/知识库/MCP 能力不受影响。
- 密钥（JWT `SECRET_KEY`、`VAULT_MASTER_KEY`、Postgres 密码）用平台稳定密钥注入，重装不变。
- `PUBLIC_*` / `FRONTEND_URL` / `CORS_ORIGINS` 全部指向应用域名。

## CI

`.github/workflows/lazycat.yml`（reusable `ca-x/lazycat-github-action@v1`）：
每 3 小时检查上游镜像新版本 → mirror 改写（ghcr.1ms.run / docker.1ms.run）→ 构建 LPK → GitHub Release 版本化资产 → 发布喵喵商店。凭据用组织级 `APPSTORE_URL` / `APPSTORE_TOKEN`。

首次发布需打 tag 触发（或手动 workflow_dispatch 直接发布）。
