# Toolathlon 完整测试流程（以本机环境为例）

本文档记录在本机（Huawei Cloud EulerOS 2.0）上从零跑通 Toolathlon 评估的完整步骤，包括环境踩坑和修复。其他人拿到类似环境可以直接照着跑。

> **环境快照（2026-07-23）**
> - OS: Huawei Cloud EulerOS 2.0 (x86_64)
> - Kernel: 5.10.0-182.0.0.95.r3353_273.hce2.x86_64
> - Docker: 26.1.4
> - kind: v0.20.0
> - kubectl: 已安装
> - Node image: `kindest/node:v1.27.3`
> - Task image: `lockon0927/toolathlon-task-image:1016beta`

---

## 1. 前置准备

### 1.1 克隆代码

```bash
git clone https://github.com/liuhy1213-cell/Toolathlon.git
cd Toolathlon
```

### 1.2 检查工具

```bash
# Docker
docker version
docker info --format 'CgroupDriver: {{.CgroupDriver}}, CgroupVersion: {{.CgroupVersion}}'

# kind
kind version

# kubectl
kubectl version --client

# Python/uv（项目用 uv 管理依赖）
uv --version
```

---

## 2. 必做：修复 Docker cgroup 驱动不一致

这是本机最大的坑。`/etc/docker/daemon.json` 默认配置了 `systemd` cgroup 驱动，但 Docker 进程实际跑的是 `cgroupfs`，导致：

- `kind create cluster` 控制面起不来
- `docker exec` 进 kind 节点报 `cgroup.procs: no such file or directory`
- `deploy_containers.sh` 的 readiness 探测卡死

### 2.1 检查当前状态

```bash
cat /etc/docker/daemon.json
docker info --format 'CgroupDriver: {{.CgroupDriver}}'
```

如果输出是下面这样，说明需要修复：

```text
# daemon.json
{"exec-opts": ["native.cgroupdriver=systemd"], ...}

# 但实际运行驱动
CgroupDriver: cgroupfs
```

### 2.2 修复步骤

Docker 在本机不是 systemd 托管，直接杀进程重启：

```bash
# 1. 先清理已有容器（否则杀 dockerd 会强制中断）
kind delete cluster --name cluster-inst-alpha1 2>/dev/null || true
docker rm -f poste-inst-alpha woo-wp-inst-alpha woo-db-inst-alpha canvas-docker-inst-alpha 2>/dev/null || true
docker network rm woo-net-inst-alpha 2>/dev/null || true

# 2. 找到 dockerd PID 并杀掉
ps aux | grep dockerd | grep -v grep
kill -TERM <dockerd_pid>
sleep 5

# 3. 用 daemon.json 中的 systemd 驱动重新启动 dockerd
nohup dockerd --default-cgroupns-mode=host > /tmp/dockerd-systemd.log 2>&1 &
sleep 5

# 4. 确认驱动已切换
docker info --format 'CgroupDriver: {{.CgroupDriver}}'
# 应输出：CgroupDriver: systemd

# 5. 验证 docker exec 正常
docker run -d --name exec-test alpine sleep 60
docker exec exec-test echo "exec-works"
docker rm -f exec-test
```

> 如果 systemd 驱动启动失败（比如报错 `failed to write ... cgroup.procs`），说明宿主机不支持 systemd cgroup。此时改用方案 B：把 `/etc/docker/daemon.json` 里的 `native.cgroupdriver=systemd` 改成 `native.cgroupdriver=cgroupfs`，再重启 dockerd。

---

## 3. 必做：安装 `nc`

`deploy_containers.sh` 用 `nc` 探测 poste 的 IMAP/SMTP 端口。本机默认没有 `nc`， readiness 循环永远不过。

```bash
yum install -y nc
# 或
dnf install -y nc

# 验证
nc --version
```

---

## 4. 部署基础设施

```bash
cd /home/Toolathlon
bash global_preparation/deploy_containers.sh
```

正常输出结尾：

```text
Deploy attempt 1 succeeded.
Exit time: Thu Jul 23 16:20:19 CST 2026
Total time: 236 seconds
```

验证：

```bash
kubectl --kubeconfig=deployment/k8s/configs/cluster-inst-alpha1-config.yaml get nodes
docker ps --format 'table {{.Names}}\t{{.Status}}'
```

应看到：

- `cluster-inst-alpha1-control-plane` Ready
- `poste-inst-alpha` healthy
- `canvas-docker-inst-alpha` Up
- `woo-wp-inst-alpha` / `woo-db-inst-alpha` Up

---

## 5. Google 凭证申请与配置（完整模式需要）

`normal` 模式下，容器启动脚本会强制把 Google OAuth 凭证复制到 `~/.gmail-mcp/` 和 `~/.calendar-mcp/`。如果不需要 Gmail/Calendar MCP，直接用第 6 节的 `quickstart` 模式跳过即可。

### 5.1 准备工作

1. 注册一个**新的 Google/Gmail 账号**（建议专门用于 Toolathlon，避免影响主账号）。
2. 访问 [Google Cloud Console](https://console.cloud.google.com/)，登录该账号并接受服务条款。
3. （可选）安装 `gcloud` SDK：

```bash
curl -O https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-sdk-456.0.0-linux-x86_64.tar.gz
tar -xf google-cloud-sdk-*-x86_64.tar.gz
./google-cloud-sdk/install.sh
source ~/.bashrc
```

### 5.2 创建 OAuth 2.0 客户端

1. 进入 [Google Cloud Console](https://console.cloud.google.com/)，创建一个新项目（Project）。
2. 左侧菜单 → **APIs & Services** → **OAuth consent screen**。
3. 选择 **External**，填写应用名称、用户支持邮箱、开发者联系邮箱，保存。
4. 点击 **Publish App**（发布应用）。
5. 左侧菜单 → **Credentials** → **Create Credentials** → **OAuth client ID**。
6. Application type 选择 **Web application**。
7. 名称随便填，例如 `toolathlon`。
8. Authorized redirect URIs 添加：
   ```text
   http://localhost:3000/oauth2callback
   ```
9. 点击 **Create**，然后 **Download JSON**。
10. 把下载的 JSON 文件重命名为 `gcp-oauth.keys.json`，放到 `configs/` 目录：

```bash
mv /path/to/downloaded-client-secret.json /home/Toolathlon/configs/gcp-oauth.keys.json
```

### 5.3 生成 `google_credentials.json`

有了 `gcp-oauth.keys.json` 后，运行自动化脚本完成授权并生成包含 `refresh_token` 的完整凭证：

```bash
cd /home/Toolathlon
bash global_preparation/automated_google_setup.sh
```

脚本会：
- 创建/选择 GCP 项目
- 启用 Gmail/Calendar API
- 引导你完成 OAuth 授权（会给出 URL，用浏览器打开授权）
- 最终生成 `configs/google_credentials.json`

如果自动化脚本卡住，也可以分步做：

```bash
# 1. 手动启用 API
# 在 Cloud Console 里搜索并启用：Gmail API、Google Calendar API

# 2. 生成 credentials.json
uv run python global_preparation/create_google_credentials.py
# 或
uv run python global_preparation/simple_google_auth.py
```

### 5.4 验证凭证

```bash
ls -la /home/Toolathlon/configs/gcp-oauth.keys.json
ls -la /home/Toolathlon/configs/google_credentials.json

# 简单检查格式
python3 -c "import json; print(json.load(open('configs/google_credentials.json')).keys())"
```

应看到类似 `dict_keys(['client_id', 'client_secret', 'refresh_token', ...])`。

### 5.5 费用说明

- 创建 GCP 项目和 OAuth 客户端：**免费**
- Gmail API / Calendar API：有免费额度，评估用量通常足够
- **建议不绑定 billing account（信用卡）**，超出免费额度只会被限制，不会扣费

---

## 6. 运行安装检查（可选）

如果不需要 Gmail/Calendar MCP，用 `quickstart` 模式：

```bash
bash global_preparation/check_installation_containerized.sh lockon0927/toolathlon-task-image:1016beta quickstart
```

如果已经按第 5 节配置好 Google 凭证，跑完整模式：

```bash
bash global_preparation/check_installation_containerized.sh
```

---

## 7. 运行模型评估

### 6.1 设置模型 API（unified provider 示例）

```bash
export TOOLATHLON_OPENAI_BASE_URL="http://<你的模型服务地址>/v1"
export TOOLATHLON_OPENAI_API_KEY="<你的 api key>"
```

本机示例：

```bash
export TOOLATHLON_OPENAI_BASE_URL="http://115.120.31.142:8000/v1"
export TOOLATHLON_OPENAI_API_KEY=""
```

### 6.2 运行并行评估

不需要 Gmail/Calendar 时用 quickstart 模式：

```bash
bash scripts/run_parallel.sh glm-5 ./dumps/glm-5/ unified 10 "" "" "" quickstart
```

参数说明：

| 位置 | 含义 | 示例 |
|------|------|------|
| 1 | 模型名 | `glm-5` |
| 2 | dump 目录 | `./dumps/glm-5/` |
| 3 | provider | `unified` |
| 4 | 并行 worker 数 | `10` |
| 5 | task image | 空 = 默认 `lockon0927/toolathlon-task-image:1016beta` |
| 6 | config file | 空 = 自动生成 |
| 7 | runner | 空 = 默认 `containerized` |
| 8 | runmode | `quickstart` 跳过 Google 凭证 |

### 6.3 查看结果

```bash
# 总体统计
cat ./dumps/glm-5/eval_stats.json

# 执行报告
python3 - <<'PY'
import json
d = json.load(open('./dumps/glm-5/execution_report_finalpool_glm-5_full.json'))
print(json.dumps(d, indent=2, ensure_ascii=False))
PY

# 单任务日志
ls ./dumps/glm-5/finalpool/<task-name>/
```

---

## 7. 常见问题和修复

### 7.1 `kind create cluster` 控制面超时

**现象**：

```text
ERROR: failed to create cluster: failed to init node with kubeadm:
... timed out waiting for the condition
```

**原因**：Docker cgroup 驱动和 kind 节点内部 cgroup 驱动不匹配。

**修复**：见本文档第 2 节，统一 Docker cgroup 驱动。

### 7.2 `docker exec` 进 kind 节点报 cgroup 错误

**现象**：

```text
OCI runtime exec failed: error adding pid ... to cgroups:
failed to write ...: openat2 /sys/fs/cgroup/blkio/docker/.../cgroup.procs: no such file or directory
```

**原因**：Docker 实际运行驱动和 daemon.json 配置不一致。

**修复**：见本文档第 2 节。

### 7.3 `deploy_containers.sh` readiness 一直不过

**现象**：

```text
Waiting up to 1800s for all services to be ready...
  …waiting for services to be ready (1700s left)
```

**可能原因**：

1. 没有 `nc` → 安装 `nc`
2. Docker cgroup 驱动不一致 → 修复 Docker
3. 某个服务真没起来 → 看对应组件日志 `/tmp/toolathlon-deploy-XXXXX/<component>.log`

### 7.4 `run_parallel.sh` 总任务数为 0

**现象**：

```text
cp: cannot stat './configs/gcp-oauth.keys.json': No such file or directory
cp: cannot stat './configs/google_credentials.json': No such file or directory
Process ended with code: 1
```

**原因**：正常模式下缺少 Google 凭证文件，容器启动脚本 `set -e` 直接退出。

**修复**：在 `run_parallel.sh` 第 8 个参数传 `quickstart`：

```bash
bash scripts/run_parallel.sh glm-5 ./dumps/glm-5/ unified 10 "" "" "" quickstart
```

### 7.5 poste 容器 unhealthy

**可能原因**：服务初始化未完成或被中断。

**修复**：

```bash
bash deployment/poste/scripts/setup.sh stop true
bash deployment/poste/scripts/setup.sh start true
```

---

## 8. 一键清理（如需重来）

```bash
cd /home/Toolathlon

# 停止并删除所有组件
bash deployment/k8s/scripts/setup.sh stop
bash deployment/canvas/scripts/setup.sh stop
bash deployment/poste/scripts/setup.sh stop true
bash deployment/woocommerce/scripts/setup.sh stop

# 强制清理残留容器/网络
kind delete cluster --name cluster-inst-alpha1 2>/dev/null || true
docker rm -f poste-inst-alpha woo-wp-inst-alpha woo-db-inst-alpha canvas-docker-inst-alpha 2>/dev/null || true
docker network rm woo-net-inst-alpha 2>/dev/null || true

# 清理临时部署目录
rm -rf /tmp/toolathlon-deploy-*
rm -f /tmp/deploy_containers_monitor*.log
```

---

## 9. 关键文件变更清单

本机修复过程中新增/修改的文件：

| 文件 | 说明 |
|------|------|
| `deployment/k8s/kind-config.yaml` | kind 集群配置，显式指定 cgroupfs 驱动，兼容多种 Docker 环境 |
| `deployment/k8s/scripts/setup.sh` | 创建集群时传入 `kind-config.yaml` |
| `.gitignore` | 允许跟踪 `deployment/k8s/kind-config.yaml` 和 `setup.sh` |

---

## 10. 验证清单

部署完成后逐项确认：

- [ ] `docker info` 显示 `CgroupDriver: systemd`（或与你选择的方案一致）
- [ ] `nc --version` 有输出
- [ ] `bash global_preparation/deploy_containers.sh` 输出 `Deploy attempt 1 succeeded`
- [ ] `kubectl --kubeconfig=deployment/k8s/configs/cluster-inst-alpha1-config.yaml get nodes` 显示 Ready
- [ ] `docker exec cluster-inst-alpha1-control-plane kubectl --kubeconfig=/etc/kubernetes/admin.conf get nodes` 能正常执行
- [ ] `bash scripts/run_parallel.sh ... quickstart` 不再报 `cp: cannot stat './configs/gcp-oauth.keys.json'`

全部打勾后，环境即可正常跑评估。
