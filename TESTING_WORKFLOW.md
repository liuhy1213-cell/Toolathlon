# Toolathlon 完整测试流程

本文档记录在119.13.79.178（Huawei Cloud EulerOS 2.0）上从零跑通 Toolathlon 评估的完整步骤，包括环境踩坑和修复。

> **环境快照（2026-07-23）**
> - OS: Huawei Cloud EulerOS 2.0 (x86_64)
> - Kernel: 5.10.0-182.0.0.95.r3353_273.hce2.x86_64
> - Docker: 26.1.4
> - kind: v0.20.0
> - kubectl: 已安装

---

## 1. 环境安装

### 1.1 克隆代码

```bash
git clone https://github.com/liuhy1213-cell/Toolathlon.git
cd Toolathlon
```

### 1.2 安装 uv

Toolathlon 使用 [uv](https://github.com/astral-sh/uv) 管理 Python 依赖。

```bash
# 安装 uv（默认装到 $HOME/.local/bin，可能需要加 PATH）
curl -LsSf https://astral.sh/uv/install.sh | sh

# 验证
which uv
uv --version
```

如果 `which uv` 找不到，把下面加到 `~/.bashrc` 并 source：

```bash
export PATH="$HOME/.local/bin:$PATH"
```

### 1.3 安装项目依赖

```bash
bash global_preparation/install_env_minimal.sh [true|false]
```

- 第一个参数传 `true`：脚本会尝试用 `sudo` 安装系统级依赖（如 inotify 配置等）。
- 传 `false` 或留空：只安装当前用户能装的依赖。

本机示例（有 root 权限）：

```bash
bash global_preparation/install_env_minimal.sh true
```

### 1.4 安装 Docker/Podman

Toolathlon 每个任务都在独立容器里跑，需要 Docker 或 Podman。本机用的是 Docker。

```bash
# 检查 Docker
docker version
docker info

# 如果没有安装，参考官方文档安装：
# https://docs.docker.com/engine/install/
```

然后在 `configs/global_configs.py` 里确认容器运行时选择：

```python
# 本机配置示例
podman_or_docker = "docker"   # 或 "podman"
```

### 1.5 拉取任务镜像

```bash
bash global_preparation/pull_toolathlon_image.sh
```

这会拉取默认任务镜像 `lockon0927/toolathlon-task-image:1016beta`。


---

## 2. 配置模型 API（Base URL 和 API Key）

Toolathlon 的 agent scaffold 依赖 OpenAI SDK 兼容的接口。通过环境变量配置：

```bash
export TOOLATHLON_OPENAI_API_KEY="your-api-key"
export TOOLATHLON_OPENAI_BASE_URL="https://your-openai-compatible-endpoint.com/v1"
```

### 2.1 常见 provider 示例

**OpenRouter：**

```bash
export TOOLATHON_OPENAI_API_KEY="sk-or-v1-xxx"
export TOOLATHLON_OPENAI_BASE_URL="https://openrouter.ai/api/v1"
```

**Anthropic（官方 OpenAI 兼容端点）：**

```bash
export TOOLATHLON_OPENAI_API_KEY="sk-ant-xxx"
export TOOLATHLON_OPENAI_BASE_URL="https://api.anthropic.com/v1/"
```

**本地 vLLM / SGLang：**

```bash
# 本地部署通常不需要 api key
export TOOLATHLON_OPENAI_API_KEY=""
export TOOLATHLON_OPENAI_BASE_URL="http://localhost:8000/v1"
```

**本机示例（unified provider）：**

```bash
export TOOLATHLON_OPENAI_API_KEY=""
export TOOLATHLON_OPENAI_BASE_URL="http://115.120.31.142:8000/v1"
```

### 2.2 验证接口可用

```bash
curl -s "$TOOLATHLON_OPENAI_BASE_URL/models" \
  -H "Authorization: Bearer $TOOLATHLON_OPENAI_API_KEY" | head -20
```

如果能返回模型列表，说明 base url 配置正确。

### 2.3 （可选）通过 global_configs.py 配置多个 provider

如果需要管理多个 LLM API，可以编辑 `configs/global_configs.py`，在里面填写各 provider 的 key 和 endpoint。详细说明见 `utils/api_model/model_provider.py`。

对于统一入口跑评估，环境变量方式最简单。

---

## 3. 必做：修复 Docker cgroup 驱动不一致

这是本机最大的坑。`/etc/docker/daemon.json` 默认配置了 `systemd` cgroup 驱动，但 Docker 进程实际跑的是 `cgroupfs`，导致：

- `kind create cluster` 控制面起不来
- `docker exec` 进 kind 节点报 `cgroup.procs: no such file or directory`
- `deploy_containers.sh` 的 readiness 探测卡死

### 3.1 检查当前状态

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

### 3.2 修复步骤

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

## 4. 必做：安装 `nc`

`deploy_containers.sh` 用 `nc` 探测 poste 的 IMAP/SMTP 端口。本机默认没有 `nc`， readiness 循环永远不过。

```bash
yum install -y nc
# 或
dnf install -y nc

# 验证
nc --version
```

---

## 5. 部署基础设施

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

## 6. Google 凭证申请与配置（完整模式需要）

`normal` 模式下，容器启动脚本会强制把 Google OAuth 凭证复制到 `~/.gmail-mcp/` 和 `~/.calendar-mcp/`。如果不需要 Gmail/Calendar MCP，直接用第 7 节的 `quickstart` 模式跳过即可。

### 6.1 准备工作

1. 注册一个**新的 Google/Gmail 账号**（建议专门用于 Toolathlon，避免影响主账号）。
2. 访问 [Google Cloud Console](https://console.cloud.google.com/)，登录该账号并接受服务条款。
3. （可选）安装 `gcloud` SDK：

```bash
curl -O https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-sdk-456.0.0-linux-x86_64.tar.gz
tar -xf google-cloud-sdk-*-x86_64.tar.gz
./google-cloud-sdk/install.sh
source ~/.bashrc
```

### 6.2 创建 OAuth 2.0 客户端

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

### 6.3 生成 `google_credentials.json`

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

### 6.4 验证凭证

```bash
ls -la /home/Toolathlon/configs/gcp-oauth.keys.json
ls -la /home/Toolathlon/configs/google_credentials.json

# 简单检查格式
python3 -c "import json; print(json.load(open('configs/google_credentials.json')).keys())"
```

应看到类似 `dict_keys(['client_id', 'client_secret', 'refresh_token', ...])`。

### 6.5 费用说明

- 创建 GCP 项目和 OAuth 客户端：**免费**
- Gmail API / Calendar API：有免费额度，评估用量通常足够
- **建议不绑定 billing account（信用卡）**，超出免费额度只会被限制，不会扣费

---

## 7. 运行安装检查（可选）

如果不需要 Gmail/Calendar MCP，用 `quickstart` 模式：

```bash
bash global_preparation/check_installation_containerized.sh lockon0927/toolathlon-task-image:1016beta quickstart
```

如果已经按第 6 节配置好 Google 凭证，跑完整模式：

```bash
bash global_preparation/check_installation_containerized.sh
```

---

## 8. 运行模型评估

### 8.1 快速示例：单任务

```bash
bash scripts/run_single_containerized.sh finalpool/find-alita-paper quickstart ./dumps_quick_start/ glm-5 unified 100
```

参数说明：

| 位置 | 参数 | 示例 |
|------|------|------|
| 1 | task_dir | `finalpool/find-alita-paper` |
| 2 | runmode | `quickstart` 或 `normal` |
| 3 | dump_path | `./dumps_quick_start/` |
| 4 | modelname | `glm-5` |
| 5 | provider | `unified` |
| 6 | maxstep | `100` |

### 8.2 并行评估

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

如果已经配置好 Google 凭证，去掉第 8 个参数（默认 `normal`）：

```bash
bash scripts/run_parallel.sh glm-5 ./dumps/glm-5/ unified 10
```

### 8.3 查看结果

```bash
# 总体统计
cat ./dumps/glm-5/eval_stats.json

# 执行报告
python3 - <<'PY'
import json
d = json.load(open('./dumps/glm-5/execution_report_finalpool_glm-5_full.json'))
print(json.dumps(d['summary'], indent=2, ensure_ascii=False))
PY

# 单任务日志
ls ./dumps/glm-5/finalpool/<task-name>/
```

---

## 9. 常见问题和修复

### 9.1 `kind create cluster` 控制面超时

**现象**：

```text
ERROR: failed to create cluster: failed to init node with kubeadm:
... timed out waiting for the condition
```

**原因**：Docker cgroup 驱动和 kind 节点内部 cgroup 驱动不匹配。

**修复**：见本文档第 3 节，统一 Docker cgroup 驱动。

### 9.2 `docker exec` 进 kind 节点报 cgroup 错误

**现象**：

```text
OCI runtime exec failed: error adding pid ... to cgroups:
failed to write ...: openat2 /sys/fs/cgroup/blkio/docker/.../cgroup.procs: no such file or directory
```

**原因**：Docker 实际运行驱动和 daemon.json 配置不一致。

**修复**：见本文档第 3 节。

### 9.3 `deploy_containers.sh` readiness 一直不过

**现象**：

```text
Waiting up to 1800s for all services to be ready...
  …waiting for services to be ready (1700s left)
```

**可能原因**：

1. 没有 `nc` → 安装 `nc`
2. Docker cgroup 驱动不一致 → 修复 Docker
3. 某个服务真没起来 → 看对应组件日志 `/tmp/toolathlon-deploy-XXXXX/<component>.log`

### 9.4 `run_parallel.sh` 总任务数为 0 / 分数为 0

**现象 1**：

```text
cp: cannot stat './configs/gcp-oauth.keys.json': No such file or directory
cp: cannot stat './configs/google_credentials.json': No such file or directory
Process ended with code: 1
```

**原因**：runmode 是 `normal`，缺少 Google 凭证文件，容器启动脚本 `set -e` 直接退出。

**修复**：在第 8 个参数传 `quickstart`：

```bash
bash scripts/run_parallel.sh glm-5 ./dumps/glm-5/ unified 10 "" "" "" quickstart
```

**现象 2**：

```text
not_executed: 108
passed: 0
failed: 0
```

但 run.log 里没有 `cp` 报错。

**原因**：可能是模型 API 没返回、任务预处理失败、或容器内环境异常。看单个任务的 `run.log` 和 `container.log` 定位。

### 9.5 poste 容器 unhealthy

**可能原因**：服务初始化未完成或被中断。

**修复**：

```bash
bash deployment/poste/scripts/setup.sh stop true
bash deployment/poste/scripts/setup.sh start true
```

### 9.6 模型 API 返回错误

**检查**：

```bash
curl -s "$TOOLATHLON_OPENAI_BASE_URL/models" \
  -H "Authorization: Bearer $TOOLATHLON_OPENAI_API_KEY"
```

如果返回 401/403/404，说明 base url 或 api key 配置有误。

---

## 10. 一键清理（如需重来）

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

## 11. 关键文件变更清单

本机修复过程中新增/修改的文件：

| 文件 | 说明 |
|------|------|
| `deployment/k8s/kind-config.yaml` | kind 集群配置，显式指定 cgroupfs 驱动，兼容多种 Docker 环境 |
| `deployment/k8s/scripts/setup.sh` | 创建集群时传入 `kind-config.yaml` |
| `.gitignore` | 允许跟踪 `deployment/k8s/kind-config.yaml` 和 `setup.sh` |
| `run_parallel.py` | 修复 `runmode` 参数未正确传给 `run_single_containerized.sh` 的问题 |
| `scripts/run_parallel.sh` | 修复 containerized runner 未传递 `--runmode` 的问题 |

---

## 12. 验证清单

部署完成后逐项确认：

- [ ] `uv --version` 有输出
- [ ] `docker info` 显示 `CgroupDriver: systemd`（或与你选择的方案一致）
- [ ] `nc --version` 有输出
- [ ] `bash global_preparation/deploy_containers.sh` 输出 `Deploy attempt 1 succeeded`
- [ ] `kubectl --kubeconfig=deployment/k8s/configs/cluster-inst-alpha1-config.yaml get nodes` 显示 Ready
- [ ] `docker exec cluster-inst-alpha1-control-plane kubectl --kubeconfig=/etc/kubernetes/admin.conf get nodes` 能正常执行
- [ ] `bash scripts/run_parallel.sh glm-5 ./dumps/glm-5/ unified 10 "" "" "" quickstart` stdout 显示 `Run mode: quickstart`
- [ ] 单个任务的 `run.log` 显示 `Runmode: quickstart`
- [ ] 运行一段时间后 `eval_stats.json` 里 `total_tasks` > 0

全部打勾后，环境即可正常跑评估。

全部打勾后，环境即可正常跑评估。
