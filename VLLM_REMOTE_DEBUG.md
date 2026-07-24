# 使用 VSCode + debugpy 远程调试 vLLM 容器

本文档介绍如何在远程服务器上的 Docker 容器里运行 vLLM，并用本地 VSCode attach 到容器内进行源码级调试。

---

## 整体思路

```text
本地 VSCode ──attach──▶ 远程服务器:5678 ──▶ 容器内 debugpy ──▶ vLLM 源码
```

---

## 1. 准备带源码的 vLLM 容器

### 1.1 在远程服务器上 clone vLLM 源码

```bash
ssh root@115.120.31.142
mkdir -p /opt/vllm-source
cd /opt/vllm-source
git clone https://github.com/vllm-project/vllm.git
```

### 1.2 启动容器并挂载源码

```bash
docker run -d \
  --name vllm-debug \
  --gpus all \
  -v /opt/vllm-source/vllm:/vllm \
  -v /path/to/your/model:/model \
  -p 8000:8000 \
  -p 5678:5678 \
  --ipc=host \
  vllm/vllm-openai:latest \
  sleep infinity
```

参数说明：

| 参数 | 含义 |
|------|------|
| `--gpus all` | 使用 GPU |
| `-v /opt/vllm-source/vllm:/vllm` | 把宿主机源码挂到容器 `/vllm` |
| `-v /path/to/your/model:/model` | 挂模型权重 |
| `-p 8000:8000` | OpenAI API 端口 |
| `-p 5678:5678` | debugpy 监听端口 |
| `--ipc=host` | vLLM 多卡需要 |
| `sleep infinity` | 先让容器保持运行，稍后手动启动服务 |

### 1.3 进入容器安装依赖

```bash
docker exec -it vllm-debug bash

# 如果容器里已有 vllm pip 包，先卸载
pip uninstall vllm -y

# 用可编辑模式安装源码
cd /vllm
pip install -e .

# 安装 debugpy
pip install debugpy
```

---

## 2. 用 debugpy 启动 vLLM

在容器里执行：

```bash
cd /vllm
python -m debugpy --listen 0.0.0.0:5678 --wait-for-client \
  -m vllm.entrypoints.openai.api_server \
  --model /model \
  --port 8000
```

参数说明：

- `--listen 0.0.0.0:5678`：debugpy 监听所有网卡的 5678 端口
- `--wait-for-client`：启动后暂停，等 VSCode attach 上来再运行
- `-m vllm.entrypoints.openai.api_server`：vLLM 的 OpenAI 兼容 server

执行后终端会停在：

```text
debugpy 1.8.x
Waiting for debugger to attach...
```

这时候服务还没开始跑，方便你从最开始断点。

---

## 3. 本地 VSCode 配置

### 3.1 安装扩展

- `Python`
- `debugpy`

### 3.2 配置 launch.json

按 `Ctrl+Shift+P` → `Debug: Open launch.json` → 选择 `Python`，然后改成：

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Attach to Remote vLLM",
            "type": "debugpy",
            "request": "attach",
            "connect": {
                "host": "115.120.31.142",
                "port": 5678
            },
            "pathMappings": [
                {
                    "localRoot": "${workspaceFolder}",
                    "remoteRoot": "/vllm"
                }
            ],
            "justMyCode": false
        }
    ]
}
```

关键字段：

| 字段 | 说明 |
|------|------|
| `host` | 远程服务器 IP |
| `port` | 5678 |
| `localRoot` | 本地 vLLM 源码路径 |
| `remoteRoot` | 容器内源码路径 `/vllm` |
| `justMyCode: false` | 能进入 vLLM 库内部代码 |

### 3.3 打开本地源码

在 VSCode 里打开你本地 clone 的 vLLM 源码：

```bash
git clone https://github.com/vllm-project/vllm.git ~/vllm-source
code ~/vllm-source
```

确保本地源码版本和容器里的一致，否则行号对不上。

---

## 4. 开始调试

### 4.1 在本地源码里打断点

例如打开：

```text
vllm/entrypoints/openai/api_server.py
```

在 `create_chat_completion` 或 `create_completion` 函数里打断点。

### 4.2 启动 attach

按 `F5` 或点击左侧 Debug 面板里的 `Attach to Remote vLLM`。

如果一切正常，容器里的终端会继续输出：

```text
Attached to debugger
...
INFO:     Started server process [xxx]
INFO:     Waiting for application startup.
```

### 4.3 发请求触发断点

在另一个终端发请求：

```bash
curl http://115.120.31.142:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "your-model-name",
    "messages": [{"role": "user", "content": "hello"}]
  }'
```

VSCode 里就会停在断点处，你可以：

- 单步执行（F10 / F11）
- 查看变量
- 查看调用栈
- 在 Debug Console 里执行表达式

---

## 5. 不断点直接启动的方式

如果你不想每次从开头等，可以去掉 `--wait-for-client`：

```bash
python -m debugpy --listen 0.0.0.0:5678 \
  -m vllm.entrypoints.openai.api_server \
  --model /model \
  --port 8000
```

这样 vLLM 正常启动，VSCode 随时按 F5 attach 上去。

---

## 6. 常见问题

### 6.1 attach 不上

检查：

```bash
# 在远程服务器上
ss -tlnp | grep 5678
# 或
netstat -tlnp | grep 5678
```

应该看到容器进程在监听 `0.0.0.0:5678`。

如果只有 `127.0.0.1:5678`，说明 bind 地址不对，要用 `--listen 0.0.0.0:5678`。

### 6.2 断点不生效 / 显示灰色

原因通常是本地源码和容器内版本不一致，或 `pathMappings` 配错了。

解决：

1. 确保本地和容器用同一个 commit：

```bash
# 本地
cd ~/vllm-source
git log --oneline -1

# 容器内
cd /vllm
git log --oneline -1
```

2. 检查 `pathMappings`：
   - 本地 `~/vllm-source` 对应远程 `/vllm`
   - 如果本地打开的是 `~/vllm-source/vllm/...`，`localRoot` 就用 `~/vllm-source`

### 6.3 容器里 `pip install -e .` 很慢

可以先在容器里只安装必要依赖：

```bash
pip install -e . --no-build-isolation
```

或者 build 一个已经装好 debugpy 和 editable vllm 的镜像。

### 6.4 想调试 vLLM 内部某个具体模块

比如想看 scheduler：

```bash
python -m debugpy --listen 0.0.0.0:5678 --wait-for-client \
  -m vllm.entrypoints.openai.api_server \
  --model /model \
  --port 8000
```

然后在本地打开：

```text
vllm/core/scheduler.py
```

打断点即可。

---

## 7. 快速启动脚本

把下面这段保存为远程服务器上的 `start-vllm-debug.sh`：

```bash
#!/bin/bash
docker rm -f vllm-debug 2>/dev/null

docker run -d \
  --name vllm-debug \
  --gpus all \
  -v /opt/vllm-source/vllm:/vllm \
  -v /path/to/your/model:/model \
  -p 8000:8000 \
  -p 5678:5678 \
  --ipc=host \
  vllm/vllm-openai:latest \
  bash -c "
    pip uninstall vllm -y && \
    cd /vllm && pip install -e . && \
    pip install debugpy && \
    python -m debugpy --listen 0.0.0.0:5678 --wait-for-client \
      -m vllm.entrypoints.openai.api_server \
      --model /model \
      --port 8000
  "
```

运行后等它停在 `Waiting for debugger to attach...`，然后本地 VSCode 按 F5。

---

## 参考

- [debugpy 官方文档](https://github.com/microsoft/debugpy)
- [vLLM OpenAI API Server](https://docs.vllm.ai/en/latest/serving/openai_compatible_server.html)
