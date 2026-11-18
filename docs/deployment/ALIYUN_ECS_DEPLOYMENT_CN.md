# QuantDinger 阿里云 ECS 部署指南（Fork + ACR）

本文面向在阿里云 ECS 上自托管 **本仓库 fork**（`jinweiyang126-code/QuantDinger`）的场景：本机构建业务镜像并推送到**阿里云容器镜像服务 ACR**，再在 ECS 上用本 fork 的 `install.sh` 拉起 Compose 栈。

通用 Nginx / HTTPS 细节仍可参考 [云服务器部署指南](CLOUD_DEPLOYMENT_CN.md)。安装方式总览见 [安装与部署总览](../getting-started/INSTALLATION_CN.md)。

> 明确启用实盘后可能提交真实订单。请先从研究、回测和 `signal` 模式开始。本项目不提供投资建议。

## 部署架构

```text
开发机（本地）
  ├─ 构建 backend / frontend / mobile 镜像
  └─ docker push → 阿里云 ACR
        ↓
阿里云 ECS
  ├─ curl 本 fork 的 install.sh（下载 compose + .env 模板）
  ├─ .env 中 BACKEND_IMAGE / FRONTEND_IMAGE / MOBILE_IMAGE 指向 ACR
  └─ docker compose pull && up -d
```

| 组件 | 镜像来源 |
| --- | --- |
| `quantdinger-backend` | 本机构建 → 推送到 ACR |
| `quantdinger-frontend` | 本机构建 → 推送到 ACR（源码在独立 Vue 仓库） |
| `quantdinger-mobile` | 本机构建 → 推送到 ACR（源码在独立 Mobile 仓库） |
| Postgres / Redis | Docker Hub 或国内镜像加速（`IMAGE_PREFIX`） |
| Kafka | Compose 默认镜像（可按需改 `KAFKA_IMAGE`） |

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
2. 创建命名空间，例如：`quantdinger`。
3. 创建仓库（或推送时自动创建）：
   - `quantdinger-backend`
   - `quantdinger-frontend`
   - `quantdinger-mobile`
4. 记录登录地址，形如：

```text
<registry-id>-registry.cn-<region>.cr.aliyuncs.com
```

下文用占位符：

```text
ACR_HOST=<registry-id>-registry.cn-<region>.cr.aliyuncs.com
ACR_NS=quantdinger
IMAGE_TAG=1.0.0
```

开发机与 ECS 都需要能登录 ACR：

```bash
docker login "${ACR_HOST}"
# 用户名一般为阿里云账号全名；密码用 ACR 访问凭证（不是登录密码）
```

## 3. 本机构建并推送到 ACR

在开发机克隆本 fork（已含后端与 Compose）：

```bash
git clone https://github.com/jinweiyang126-code/QuantDinger.git
cd QuantDinger
```

### 3.1 构建 Backend

```bash
docker build -t "${ACR_HOST}/${ACR_NS}/quantdinger-backend:${IMAGE_TAG}" \
  -f backend_api_python/Dockerfile \
  backend_api_python

docker push "${ACR_HOST}/${ACR_NS}/quantdinger-backend:${IMAGE_TAG}"
```

### 3.2 构建 Frontend / Mobile

前端与移动端源码不在本仓库内，需另备 `QuantDinger-Vue`、`QuantDinger-Mobile`（或你们自己的前端工程），目录中需有 `Dockerfile`。

```bash
# 将 Vue / Mobile 源码放到同级或指定路径后：
git clone <你的-Vue仓库> ./QuantDinger-Vue
git clone <你的-Mobile仓库> ./QuantDinger-Mobile

docker build -t "${ACR_HOST}/${ACR_NS}/quantdinger-frontend:${IMAGE_TAG}" \
  -f QuantDinger-Vue/Dockerfile \
  QuantDinger-Vue

docker build -t "${ACR_HOST}/${ACR_NS}/quantdinger-mobile:${IMAGE_TAG}" \
  -f QuantDinger-Mobile/Dockerfile \
  QuantDinger-Mobile

docker push "${ACR_HOST}/${ACR_NS}/quantdinger-frontend:${IMAGE_TAG}"
docker push "${ACR_HOST}/${ACR_NS}/quantdinger-mobile:${IMAGE_TAG}"
```

也可使用仓库自带的 build 覆盖文件在本地先编再打 tag：

```bash
# 需已存在 ./QuantDinger-Vue 与 ./QuantDinger-Mobile
docker compose -f docker-compose.yml -f docker-compose.build.yml build frontend mobile
docker tag quantdinger-frontend:local "${ACR_HOST}/${ACR_NS}/quantdinger-frontend:${IMAGE_TAG}"
docker tag quantdinger-mobile:local   "${ACR_HOST}/${ACR_NS}/quantdinger-mobile:${IMAGE_TAG}"
docker push "${ACR_HOST}/${ACR_NS}/quantdinger-frontend:${IMAGE_TAG}"
docker push "${ACR_HOST}/${ACR_NS}/quantdinger-mobile:${IMAGE_TAG}"
```

> 若暂时没有前端源码：可先仅推送 backend，ECS 上临时仍指向可访问的 frontend/mobile 镜像；生产环境建议三端都进 ACR。

### 3.3 建议同时打 `latest`（可选）

```bash
for name in quantdinger-backend quantdinger-frontend quantdinger-mobile; do
  docker tag  "${ACR_HOST}/${ACR_NS}/${name}:${IMAGE_TAG}" "${ACR_HOST}/${ACR_NS}/${name}:latest"
  docker push "${ACR_HOST}/${ACR_NS}/${name}:latest"
done
```

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

## 5. 使用本 Fork 的 install.sh 一键安装

**使用 `jinweiyang126-code/QuantDinger` 的安装脚本**（不要用 OpenByteInc 官方 raw 地址）：

```bash
curl -fsSL https://raw.githubusercontent.com/jinweiyang126-code/QuantDinger/main/install.sh \
  | bash -s -- ~/quantdinger
```

脚本会：

1. 从本 fork 下载 `docker-compose.ghcr.yml`（保存为安装目录下的 `docker-compose.yml`）与 `.env.example`
2. 交互设置管理员账号、端口等
3. 执行 `docker compose pull` / `up -d`

也可显式指定 raw 源：

```bash
export QUANTDINGER_GITHUB_RAW=https://raw.githubusercontent.com/jinweiyang126-code/QuantDinger/main
curl -fsSL "${QUANTDINGER_GITHUB_RAW}/install.sh" | bash -s -- ~/quantdinger
```

> 若 GitHub raw 在 ECS 上超时：在能访问 GitHub 的机器下载 `install.sh`、`docker-compose.ghcr.yml`、`.env.example`、`scripts/bootstrap_env.py` 后上传到 ECS，再本地执行（见第 5.2 节）。

### 5.1 安装后立刻改成 ACR 镜像

```bash
cd ~/quantdinger
cp -a .env .env.bak.$(date +%Y%m%d%H%M%S)
nano .env
```

写入（按实际替换）：

```ini
# ---- 业务镜像：阿里云 ACR ----
BACKEND_IMAGE=<registry-id>-registry.cn-<region>.cr.aliyuncs.com/quantdinger/quantdinger-backend
FRONTEND_IMAGE=<registry-id>-registry.cn-<region>.cr.aliyuncs.com/quantdinger/quantdinger-frontend
MOBILE_IMAGE=<registry-id>-registry.cn-<region>.cr.aliyuncs.com/quantdinger/quantdinger-mobile
IMAGE_TAG=1.0.0

# ---- 基础镜像加速（Postgres / Redis）----
IMAGE_PREFIX=docker.m.daocloud.io/library/

# ---- 访问地址与端口（按冲突情况调整）----
FRONTEND_HOST=0.0.0.0
FRONTEND_PORT=18888
MOBILE_HOST=127.0.0.1
MOBILE_PORT=8889
BACKEND_PORT=127.0.0.1:5000
REDIS_BIND=127.0.0.1:16379
FRONTEND_URL=http://你的公网IP:18888

# 管理员仅首次初始化需要；已有数据卷后不要随意更换 POSTGRES_PASSWORD
```

然后重新拉取并启动：

```bash
docker compose -f docker-compose.yml pull
docker compose -f docker-compose.yml up -d --force-recreate
docker compose -f docker-compose.yml ps
curl -fsS http://127.0.0.1:5000/api/health
```

浏览器访问：`http://你的公网IP:18888`（或你设置的 `FRONTEND_PORT`）。安全组放行对应端口（可限制为办公 IP）。

### 5.2 手动安装（不用管道执行 install.sh）

```bash
mkdir -p ~/quantdinger/scripts
cd ~/quantdinger

# 从本 fork 获取文件（或 scp 上传）
curl -fsSL -o docker-compose.yml \
  https://raw.githubusercontent.com/jinweiyang126-code/QuantDinger/main/docker-compose.ghcr.yml
curl -fsSL -o .env.example \
  https://raw.githubusercontent.com/jinweiyang126-code/QuantDinger/main/.env.example
curl -fsSL -o scripts/bootstrap_env.py \
  https://raw.githubusercontent.com/jinweiyang126-code/QuantDinger/main/scripts/bootstrap_env.py

python3 scripts/bootstrap_env.py --compose-file docker-compose.yml
# 编辑 .env：ADMIN_*、FRONTEND_*、BACKEND_IMAGE / FRONTEND_IMAGE / MOBILE_IMAGE、IMAGE_TAG、IMAGE_PREFIX

docker login "${ACR_HOST}"
docker compose -f docker-compose.yml pull
docker compose -f docker-compose.yml up -d
```

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

本机：

```bash
# 构建并推送新 IMAGE_TAG，例如 1.0.1
docker push "${ACR_HOST}/${ACR_NS}/quantdinger-backend:1.0.1"
# frontend / mobile 同理
```

ECS：

```bash
cd ~/quantdinger
# 修改 .env 中 IMAGE_TAG=1.0.1
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
| `install.sh` 仍拉到官方配置 | 确认 curl 的是 `jinweiyang126-code/QuantDinger`；脚本内 `GITHUB_RAW` 已默认指向本 fork |
| `pull` 业务镜像 401/403 | ECS 上重新 `docker login` ACR；检查仓库权限 |
| 拉到的仍是 GHCR 镜像 | `.env` 未设置或未生效 `BACKEND_IMAGE` 等，改完后 `up -d --force-recreate` |
| 端口被占 | 改 `FRONTEND_PORT` / `REDIS_BIND`，安全组同步放行 |
| 内存不足 | 停掉同机其他容器（如 nautilus、new-api）；或升到 8C16G |
| GitHub raw 超时 | 本机下载安装文件后 scp 到 ECS，走第 5.2 节 |

## 10. 检查清单

- [ ] 本机已将 backend / frontend / mobile 推送到 ACR  
- [ ] ECS ≥ 4C8G，Docker / Compose v2 可用  
- [ ] ECS 已 `docker login` ACR  
- [ ] 使用 **本 fork** 的 `install.sh` 或手动文件完成安装  
- [ ] `.env` 中 `*_IMAGE` 指向 ACR，`IMAGE_TAG` 正确  
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
