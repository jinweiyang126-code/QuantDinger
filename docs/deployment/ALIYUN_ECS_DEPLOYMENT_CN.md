# QuantDinger 阿里云 ECS 部署指南（Fork + ACR）

本文面向在阿里云 ECS 上自托管 **本仓库 fork**（`jinweiyang126-code/QuantDinger`）的场景：

- **业务镜像**：本机构建后推到**阿里云 ACR**，ECS 只从 ACR `pull`
- **安装文件**：在 ECS 上从本 fork **clone / curl `install.sh`** 获取（不必本机打包）
- ACR **不能**代替 GitHub 提供 `install.sh`；两者职责不同

通用 Nginx / HTTPS 细节仍可参考 [云服务器部署指南](CLOUD_DEPLOYMENT_CN.md)。安装方式总览见 [安装与部署总览](../getting-started/INSTALLATION_CN.md)。

> 明确启用实盘后可能提交真实订单。请先从研究、回测和 `signal` 模式开始。本项目不提供投资建议。

## 先分清两件事

| 东西 | 放哪里 | 说明 |
| --- | --- | --- |
| **业务镜像**（backend / frontend / mobile） | **阿里云 ACR** | `docker push` / `docker pull` |
| **安装文件**（`install.sh`、compose、`.env` 模板） | **本 fork（GitHub）→ ECS** | 普通文本/配置；主路径用 clone 或 curl |

仅当 ECS **访问不了 GitHub** 时，才需要本机打包后 scp/OSS 上传（见第 5.5 节备用方案）。

## 部署架构

```text
开发机
  └─ docker build + push  →  阿里云 ACR（backend / frontend / mobile）
                                    ↓ docker pull
阿里云 ECS
  ├─ git clone 或 curl 本 fork 的 install.sh（拿 compose / .env）
  ├─ .env 中 *_IMAGE / *_TAG 指向 ACR
  ├─ docker login ACR
  └─ docker compose pull && up -d
```

| 组件 | 来源 |
| --- | --- |
| backend / frontend / mobile 镜像 | 本机构建 → **ACR** |
| Postgres / Redis | Docker Hub 或国内加速（`IMAGE_PREFIX`） |
| Kafka | Compose 默认镜像（可按需改 `KAFKA_IMAGE`） |
| `install.sh` / compose / `.env` | **本 fork**（ECS 上 clone/curl），不是 ACR |

本流程**不再依赖** `ghcr.io/openbyteinc/...` 业务镜像。

## 1. 推荐规格

| 场景 | CPU / 内存 | 系统盘 | 说明 |
| --- | --- | --- | --- |
| 最低可部署 | **4 核 8 GiB** | ≥ 80 GB SSD | 完整 Docker 栈下限 |
| 日常研究 + 回测 | **8 核 16 GiB** | ≥ 100 GB SSD | 更稳 |
| 不推荐 | 2 核 4 GiB | — | 极易 OOM |

安全组至少放行 `22` / `80` / `443`。测试期可临时放行 Web 端口（若宿主机 `8888` 已被占用，改用 `18888` 等空闲端口）。

**不要**把 `5000`、`5432`、`6379`、`29092` 暴露到公网。

## 2. 准备阿里云 ACR

1. 开通 [容器镜像服务 ACR](https://cr.console.aliyun.com/)，创建实例（个人版即可起步）。
2. 创建命名空间与**一个**仓库即可，例如命名空间 `quantdinger-acr`、仓库 `quantdinger-acr`（用不同 tag 区分 backend / frontend / mobile）。
3. 记录登录地址。个人版常见形如：

```text
crpi-xxxx.ap-southeast-1.personal.cr.aliyuncs.com
```

下文用占位符（与当前实测一致）：

```text
ACR_HOST=crpi-xxxx.ap-southeast-1.personal.cr.aliyuncs.com
ACR_REPO=quantdinger-acr/quantdinger-acr
# 单仓库多 tag
BACKEND_TAG=backend-1.0.0
FRONTEND_TAG=frontend-1.0.0
MOBILE_TAG=mobile-1.0.0
```

开发机与 ECS 都需要能登录 ACR：

```bash
docker login "${ACR_HOST}"
# 用户名一般为阿里云账号全名；密码用 ACR 访问凭证（不是登录密码）
```

## 3. 本机构建并推送到 ACR

在开发机准备三个源码目录（本 fork + Vue + Mobile）：

```bash
git clone https://github.com/jinweiyang126-code/QuantDinger.git
# Vue / Mobile 另 clone 到同级或指定路径
```

国内构建可设 `BUILD_REGION=cn`（backend Dockerfile 会走国内 apt 源）。

### 3.1 构建并推送（单仓库多 tag）

```bash
# Backend
docker build --build-arg BUILD_REGION=cn \
  -t "${ACR_HOST}/${ACR_REPO}:${BACKEND_TAG}" \
  -f QuantDinger/backend_api_python/Dockerfile \
  QuantDinger/backend_api_python
docker push "${ACR_HOST}/${ACR_REPO}:${BACKEND_TAG}"

# Frontend
docker build -t "${ACR_HOST}/${ACR_REPO}:${FRONTEND_TAG}" \
  -f QuantDinger-Vue/Dockerfile QuantDinger-Vue
docker push "${ACR_HOST}/${ACR_REPO}:${FRONTEND_TAG}"

# Mobile
docker build -t "${ACR_HOST}/${ACR_REPO}:${MOBILE_TAG}" \
  -f QuantDinger-Mobile/Dockerfile QuantDinger-Mobile
docker push "${ACR_HOST}/${ACR_REPO}:${MOBILE_TAG}"
```

也可先用 compose build 覆盖编本地 tag，再 `docker tag` / `push` 到 ACR。

> 三端都进 ACR 后再在 ECS 上把 `.env` 的 `*_IMAGE` / `*_TAG` 全部改掉。
## 4. ECS 环境准备（Alibaba Cloud Linux 3）

```bash
nproc && free -h && df -h
sudo dnf -y update
curl -fsSL https://get.docker.com | sh
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
newgrp docker
docker --version
docker compose version
docker login "${ACR_HOST}"
```

若宿主机已有服务，先查端口，避免冲突：

```bash
ss -tulnp | awk 'NR>1 {print $5}' | grep -oE '[0-9]+$' | sort -un
```

常见占用时建议：

| 默认 | 冲突时改为 |
| --- | --- |
| Web `8888` | `18888` |
| Redis 映射 `6379` | `16379`（`REDIS_BIND=127.0.0.1:16379`） |

## 5. 在 ECS 上安装（推荐：clone/curl 本 fork + 镜像走 ACR）

### 5.1 获取安装文件（主路径）

**方式 A：一键安装脚本（推荐）**

```bash
curl -fsSL https://raw.githubusercontent.com/jinweiyang126-code/QuantDinger/main/install.sh \
  | bash -s -- ~/quantdinger
```

按提示设置管理员账号与端口。脚本会从本 fork 下载 compose / `.env` 模板并尝试 `pull`；装完后按 **5.2** 把镜像改成 ACR，再重新 `pull`。

**方式 B：git clone**

```bash
git clone --depth 1 https://github.com/jinweiyang126-code/QuantDinger.git ~/quantdinger-src
mkdir -p ~/quantdinger/scripts
cp ~/quantdinger-src/docker-compose.ghcr.yml ~/quantdinger/docker-compose.yml
cp ~/quantdinger-src/.env.example ~/quantdinger/
cp ~/quantdinger-src/scripts/bootstrap_env.py ~/quantdinger/scripts/
cd ~/quantdinger
python3 scripts/bootstrap_env.py --compose-file docker-compose.yml
```

### 5.2 配置 `.env`：业务镜像全部指向 ACR

```bash
cd ~/quantdinger
cp -a .env .env.bak.$(date +%Y%m%d%H%M%S)
nano .env
```

单仓库多 tag 示例（与 `quantdinger-acr` 用法一致）：

```ini
# ---- 业务镜像：只从阿里云 ACR 拉取 ----
BACKEND_IMAGE=crpi-xxxx.ap-southeast-1.personal.cr.aliyuncs.com/quantdinger-acr/quantdinger-acr
FRONTEND_IMAGE=crpi-xxxx.ap-southeast-1.personal.cr.aliyuncs.com/quantdinger-acr/quantdinger-acr
MOBILE_IMAGE=crpi-xxxx.ap-southeast-1.personal.cr.aliyuncs.com/quantdinger-acr/quantdinger-acr
BACKEND_TAG=backend-1.0.0
FRONTEND_TAG=frontend-1.0.0
MOBILE_TAG=mobile-1.0.0

# ---- 基础镜像加速（Postgres / Redis）----
IMAGE_PREFIX=docker.m.daocloud.io/library/

# ---- 端口（8888/6379 被占用时用下面值）----
FRONTEND_HOST=0.0.0.0
FRONTEND_PORT=18888
MOBILE_HOST=127.0.0.1
MOBILE_PORT=8889
BACKEND_PORT=127.0.0.1:5000
REDIS_BIND=127.0.0.1:16379
FRONTEND_URL=http://你的公网IP:18888

ADMIN_USER=your_admin_user
ADMIN_PASSWORD=your_strong_password
# 已有数据卷后不要随意更换 POSTGRES_PASSWORD
```

### 5.3 从 ACR 拉镜像并启动

```bash
docker login crpi-xxxx.ap-southeast-1.personal.cr.aliyuncs.com
cd ~/quantdinger
docker compose -f docker-compose.yml pull
docker compose -f docker-compose.yml up -d
docker compose -f docker-compose.yml ps
curl -fsS http://127.0.0.1:5000/api/health
```

浏览器访问：`http://你的公网IP:18888`。安全组放行对应端口（可限制为办公 IP）。

### 5.4 说明

- **ACR**：只负责容器镜像  
- **GitHub fork**：负责安装脚本与 compose  
- 不必为本机再打安装 zip，除非走下面备用方案  

### 5.5 备用：ECS 访问不了 GitHub 时再本机打包

```bash
# 开发机
cd /path/to/QuantDinger
mkdir -p /tmp/quantdinger-install/scripts
cp install.sh docker-compose.ghcr.yml .env.example /tmp/quantdinger-install/
cp scripts/bootstrap_env.py /tmp/quantdinger-install/scripts/
cd /tmp && tar czf quantdinger-install.tgz quantdinger-install
scp quantdinger-install.tgz root@ECS公网IP:~/
```

ECS 解压到 `~/quantdinger` 后，仍按 **5.2 / 5.3** 配置 ACR 并启动。

## 6. 有域名后的生产配置

1. DNS：`app.example.com` → ECS 公网 IP  
2. `.env`：

```ini
FRONTEND_URL=https://app.example.com,https://m.example.com
FRONTEND_HOST=127.0.0.1
FRONTEND_PORT=8888
# 若 8888 冲突则保持 18888，Nginx 反代到该端口即可
BACKEND_PORT=127.0.0.1:5000
```

3. 安装 Nginx + Certbot，`/api/` → `127.0.0.1:5000`，页面 → 前端端口；完整 `server` 块见 [CLOUD_DEPLOYMENT_CN.md](CLOUD_DEPLOYMENT_CN.md)。  
4. HTTPS 生效后收回临时 Web 端口的公网放行，仅保留 `22/80/443`。

## 7. 登录后首次设置

1. 管理员登录  
2. Settings → AI / LLM 配置模型并测试  
3. 按 [30 分钟快速上手](../getting-started/QUICKSTART_30_MINUTES_CN.md) 跑通研究与回测  
4. 策略部署先用 **`signal`**，不要直接 **`live`**  
5. 备份 `~/quantdinger/.env` 与 Postgres 数据卷  

## 8. 升级（发新版 ACR 镜像）

本机推送新 tag，例如 `backend-1.0.1` / `frontend-1.0.1` / `mobile-1.0.1`。

ECS：

```bash
cd ~/quantdinger
# 修改 .env 中 BACKEND_TAG / FRONTEND_TAG / MOBILE_TAG
docker compose -f docker-compose.yml pull
docker compose -f docker-compose.yml up -d
```

需要同步更新 compose / 环境模板时，再执行一次本 fork 的 `install.sh`，或手动更新 `docker-compose.yml` 后运行：

```bash
python3 scripts/bootstrap_env.py --compose-file docker-compose.yml
```

## 9. 常见问题

| 现象 | 处理 |
| --- | --- |
| 以为要用 ACR 的 URL 跑 `curl install.sh` | ACR 只存镜像；安装文件用 **5.1 clone/curl 本 fork** |
| `pull` 业务镜像 401/403 | ECS 上重新 `docker login` ACR；检查仓库权限 |
| 拉到的仍是 GHCR 镜像 | `.env` 未设置 `BACKEND_IMAGE` / `*_TAG` 等，改完后 `up -d --force-recreate` |
| 端口被占 | 改 `FRONTEND_PORT` / `REDIS_BIND`，安全组同步放行 |
| 内存不足 | 停掉同机其他容器（如 nautilus、new-api）；或升到 8C16G |
| ECS 访问不了 GitHub | 用 **5.5** 本机打包 scp/OSS 备用方案 |

## 10. 检查清单

- [ ] 本机已将 backend / frontend / mobile 推送到 **ACR**  
- [ ] ECS 上已用 **clone 或 curl 本 fork** 取得安装文件  
- [ ] ECS ≥ 4C8G，Docker / Compose v2 可用  
- [ ] ECS 已 `docker login` ACR  
- [ ] `.env` 中 `*_IMAGE` / `*_TAG` 指向 ACR  
- [ ] 端口与安全组无冲突  
- [ ] `/api/health` 正常，Web 可登录  
- [ ] 已备份 `.env` 与数据卷  
- [ ] 未在未验证前启用实盘 `live`

## 相关文档

- [云服务器部署（通用）](CLOUD_DEPLOYMENT_CN.md)
- [配置参考](CONFIGURATION_REFERENCE_CN.md)
- [生产加固](PRODUCTION_HARDENING_CN.md)
- [第一次运行](../getting-started/FIRST_RUN_CN.md)
- [实盘安全清单](../trading/LIVE_TRADING_SAFETY_CN.md)
