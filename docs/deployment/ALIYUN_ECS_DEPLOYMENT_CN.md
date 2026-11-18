# QuantDinger 阿里云 ECS 部署指南（Fork + ACR）

本文面向在阿里云 ECS 上自托管 **本仓库 fork**（`jinweiyang126-code/QuantDinger`）的场景：**业务 Docker 镜像走阿里云 ACR**；安装用的 `install.sh` / Compose / `.env` 模板用本机打包上传（或 OSS），**不要指望从 ACR 里 curl 安装脚本**。

通用 Nginx / HTTPS 细节仍可参考 [云服务器部署指南](CLOUD_DEPLOYMENT_CN.md)。安装方式总览见 [安装与部署总览](../getting-started/INSTALLATION_CN.md)。

> 明确启用实盘后可能提交真实订单。请先从研究、回测和 `signal` 模式开始。本项目不提供投资建议。

## 先分清两件事

| 东西 | 放哪里 | 说明 |
| --- | --- | --- |
| **业务镜像**（backend / frontend / mobile） | **阿里云 ACR** | `docker push` / `docker pull` |
| **安装文件**（`install.sh`、`docker-compose.ghcr.yml`、`.env.example`、`bootstrap_env.py`） | **本机 → scp/OSS → ECS** | 普通文件，不是容器镜像；ACR **不能**当 raw 脚本源来用 |

推荐主路径**不依赖** ECS 访问 GitHub：开发机打好安装包上传到 ECS，容器只从 ACR 拉取。

## 部署架构

```text
开发机（本地仓库 jinweiyang126-code/QuantDinger）
  ├─ docker build + push  →  阿里云 ACR（业务镜像）
  └─ 打包 install 文件     →  scp / 阿里云 OSS  →  ECS
                                              ↓
阿里云 ECS
  ├─ 解压安装包，配置 .env（*_IMAGE 指向 ACR）
  ├─ docker login ACR
  └─ docker compose pull && up -d   ← 镜像只从 ACR 拉
```

| 组件 | 来源 |
| --- | --- |
| `quantdinger-backend` | 本机构建 → **ACR** |
| `quantdinger-frontend` | 本机构建 → **ACR**（源码在独立 Vue 仓库） |
| `quantdinger-mobile` | 本机构建 → **ACR**（源码在独立 Mobile 仓库） |
| Postgres / Redis | Docker Hub 或国内镜像加速（`IMAGE_PREFIX`） |
| Kafka | Compose 默认镜像（可按需改 `KAFKA_IMAGE`） |
| `install.sh` / compose / `.env` | 本机上传，**不是** ACR |

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

## 5. 在 ECS 上安装（推荐：本机打包上传，镜像走 ACR）

### 5.1 开发机打包安装文件

在本地 `QuantDinger` 仓库目录执行：

```bash
cd /path/to/QuantDinger
mkdir -p /tmp/quantdinger-install/scripts
cp install.sh docker-compose.ghcr.yml .env.example /tmp/quantdinger-install/
cp scripts/bootstrap_env.py /tmp/quantdinger-install/scripts/
# Windows PowerShell 可用 Copy-Item 等价操作后打 zip
cd /tmp
tar czf quantdinger-install.tgz quantdinger-install
```

上传到 ECS（任选其一）：

```bash
# scp
scp quantdinger-install.tgz root@你的ECS公网IP:~/

# 或上传到阿里云 OSS 后，在 ECS：
# wget https://你的bucket.oss-cn-xxx.aliyuncs.com/quantdinger-install.tgz -O ~/quantdinger-install.tgz
```

### 5.2 ECS 解压并准备目录

```bash
cd ~
tar xzf quantdinger-install.tgz
mkdir -p ~/quantdinger/scripts
cp ~/quantdinger-install/docker-compose.ghcr.yml ~/quantdinger/docker-compose.yml
cp ~/quantdinger-install/.env.example ~/quantdinger/.env.example
cp ~/quantdinger-install/scripts/bootstrap_env.py ~/quantdinger/scripts/
cp ~/quantdinger-install/install.sh ~/quantdinger/
cd ~/quantdinger
```

生成 `.env`（交互式可用本地 `install.sh`；或直接 bootstrap）：

```bash
# 方式 A：本地执行已上传的 install.sh（它会再尝试联网下载文件；
# 若 ECS 访问不了 GitHub，请改用方式 B，不要依赖 curl raw.githubusercontent.com）
# bash ./install.sh ~/quantdinger

# 方式 B（推荐，纯离线配置文件）：
python3 scripts/bootstrap_env.py --compose-file docker-compose.yml
```

### 5.3 配置 `.env`：业务镜像全部指向 ACR

```bash
cp -a .env .env.bak.$(date +%Y%m%d%H%M%S)
nano .env
```

```ini
# ---- 业务镜像：只从阿里云 ACR 拉取（这才是 ACR 的用途）----
BACKEND_IMAGE=<registry-id>-registry.cn-<region>.cr.aliyuncs.com/quantdinger/quantdinger-backend
FRONTEND_IMAGE=<registry-id>-registry.cn-<region>.cr.aliyuncs.com/quantdinger/quantdinger-frontend
MOBILE_IMAGE=<registry-id>-registry.cn-<region>.cr.aliyuncs.com/quantdinger/quantdinger-mobile
IMAGE_TAG=1.0.0

# ---- 基础镜像加速（Postgres / Redis，不是业务镜像）----
IMAGE_PREFIX=docker.m.daocloud.io/library/

# ---- 访问地址与端口（按冲突情况调整）----
FRONTEND_HOST=0.0.0.0
FRONTEND_PORT=18888
MOBILE_HOST=127.0.0.1
MOBILE_PORT=8889
BACKEND_PORT=127.0.0.1:5000
REDIS_BIND=127.0.0.1:16379
FRONTEND_URL=http://你的公网IP:18888

ADMIN_USER=your_admin_user
ADMIN_PASSWORD=your_strong_password
# 管理员仅首次初始化需要；已有数据卷后不要随意更换 POSTGRES_PASSWORD
```

### 5.4 从 ACR 拉镜像并启动

```bash
docker login "${ACR_HOST}"
docker compose -f docker-compose.yml pull
docker compose -f docker-compose.yml up -d
docker compose -f docker-compose.yml ps
curl -fsS http://127.0.0.1:5000/api/health
```

浏览器访问：`http://你的公网IP:18888`（或你设置的 `FRONTEND_PORT`）。安全组放行对应端口（可限制为办公 IP）。

### 5.5 可选：仍用 GitHub raw（不推荐作主路径）

仅当 ECS 能稳定访问 GitHub、且你接受安装文件不走 ACR 时：

```bash
curl -fsSL https://raw.githubusercontent.com/jinweiyang126-code/QuantDinger/main/install.sh \
  | bash -s -- ~/quantdinger
```

装完后仍须按 **5.3** 把 `*_IMAGE` 改成 ACR，再 `pull` / `up -d`。  
**ACR 不会提供这个 curl 地址**；GitHub 只提供安装脚本文本，和镜像仓库是两回事。

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
| 以为要用 ACR 的 URL 跑 `curl install.sh` | ACR 只存镜像；安装文件用 **5.1/5.2 本机上传** |
| `pull` 业务镜像 401/403 | ECS 上重新 `docker login` ACR；检查仓库权限 |
| 拉到的仍是 GHCR 镜像 | `.env` 未设置或未生效 `BACKEND_IMAGE` 等，改完后 `up -d --force-recreate` |
| 端口被占 | 改 `FRONTEND_PORT` / `REDIS_BIND`，安全组同步放行 |
| 内存不足 | 停掉同机其他容器（如 nautilus、new-api）；或升到 8C16G |
| ECS 访问不了 GitHub | 不要用 raw curl；按第 5 节 scp/OSS 上传安装包 |

## 10. 检查清单

- [ ] 本机已将 backend / frontend / mobile 推送到 **ACR**  
- [ ] 安装包（compose / `.env.example` / `bootstrap_env.py` 等）已 **scp/OSS** 到 ECS  
- [ ] ECS ≥ 4C8G，Docker / Compose v2 可用  
- [ ] ECS 已 `docker login` ACR  
- [ ] `.env` 中 `*_IMAGE` 全部指向 ACR，`IMAGE_TAG` 正确  
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
