# MiniCPM5-2B 本地 GPU 推理与远程 API 部署方案

## 1. 目标

在第二台机器上使用 AMD RX 6750 GRE 10GB GPU 部署 MiniCPM5-2B，并提供 OpenAI 兼容 API。需要支持三种可在线切换的运行模式：

| 模式 | 权重文件 | 上下文长度 | 目标 |
|---|---|---:|---|
| `quality` | F16 | 8192 | 质量优先 |
| `daily` | Q8_0 | 8192 | 日常默认 |
| `speed` | Q4_K_M | 16384 | 速度和显存占用优先 |

三种权重都属于同一个 MiniCPM5-2B 模型的不同精度版本，不是三个不同模型。

## 2. 目标硬件

- 内存：32GB
- GPU：AMD Radeon RX 6750 GRE，10GB 显存
- CPU：Intel Core i5-12600KF
- 推荐系统：Windows + Vulkan llama.cpp

Windows 下优先使用 Vulkan，不把部署建立在 CUDA 或纯 CPU 推理上。Linux + ROCm/HIP 也可尝试，但配置和兼容性验证成本更高。

## 3. 模型大小与显存判断

官方 GGUF 文件约为：

- `MiniCPM5-2B-F16.gguf`：5.04GB
- `MiniCPM5-2B-Q8_0.gguf`：2.68GB
- `MiniCPM5-2B-Q4_K_M.gguf`：1.56GB

模型最大上下文虽然是 131072，但 F16 权重加上 128K 的 BF16/F16 KV cache、临时张量和运行时开销，会超过 10GB 显存。因此不要把“F16 + 128K + 全部 GPU”作为目标。

建议：

- F16：先使用 8192 上下文
- Q8_0：先使用 8192，可测试 16384
- Q4_K_M：使用 16384；如要尝试更长上下文，先观察显存
- 不把 128K 作为默认配置；即使量化后能启动，长上下文的首 token 延迟和计算时间也会明显增加

## 4. 安装 llama.cpp

在 Windows PowerShell 中执行：

```powershell
winget install llama.cpp
```

确认命令可用：

```powershell
llama-server --help
llama-server --list-devices
```

启动时必须确认日志中出现 Vulkan 和 AMD GPU 设备。若没有 AMD/Vulkan 设备，不要继续上线，先排查 llama.cpp 构建、驱动和 Vulkan 环境。

## 5. 下载三份模型

先创建模型目录：

```powershell
New-Item -ItemType Directory -Force D:\models\minicpm5 | Out-Null
```

使用 Hugging Face CLI 下载（也可以使用 `llama-server -hf` 自动下载）：

```powershell
hf download openbmb/MiniCPM5-2B-GGUF MiniCPM5-2B-F16.gguf `
  --local-dir D:\models\minicpm5

hf download openbmb/MiniCPM5-2B-GGUF MiniCPM5-2B-Q8_0.gguf `
  --local-dir D:\models\minicpm5

hf download openbmb/MiniCPM5-2B-GGUF MiniCPM5-2B-Q4_K_M.gguf `
  --local-dir D:\models\minicpm5
```

如果系统只有旧版命令，可将 `hf download` 替换为 `huggingface-cli download`。

三份文件总计约 10GB，下载前预留至少 15GB 磁盘空间。

## 6. 先单独验证 GPU 推理

不要一开始就配置公网访问。先分别验证三个模型。

### 6.1 Q8 日常模式

```powershell
llama-server `
  -m D:\models\minicpm5\MiniCPM5-2B-Q8_0.gguf `
  --host 127.0.0.1 `
  --port 8080 `
  -c 8192 `
  -ngl 99 `
  --parallel 1 `
  --jinja `
  --api-key "CHANGE_THIS_TO_A_LONG_RANDOM_KEY"
```

### 6.2 F16 质量模式

```powershell
llama-server `
  -m D:\models\minicpm5\MiniCPM5-2B-F16.gguf `
  --host 127.0.0.1 `
  --port 8080 `
  -c 8192 `
  -ngl 99 `
  --parallel 1 `
  --jinja `
  --api-key "CHANGE_THIS_TO_A_LONG_RANDOM_KEY"
```

### 6.3 Q4 速度模式

```powershell
llama-server `
  -m D:\models\minicpm5\MiniCPM5-2B-Q4_K_M.gguf `
  --host 127.0.0.1 `
  --port 8080 `
  -c 16384 `
  -ngl 99 `
  --parallel 1 `
  --jinja `
  --api-key "CHANGE_THIS_TO_A_LONG_RANDOM_KEY"
```

测试 API：

```powershell
curl http://127.0.0.1:8080/v1/chat/completions `
  -H "Content-Type: application/json" `
  -H "Authorization: Bearer CHANGE_THIS_TO_A_LONG_RANDOM_KEY" `
  -d '{"messages":[{"role":"user","content":"请简短介绍你自己"}],"max_tokens":128,"stream":false}'
```

确认返回内容正常，并查看响应中的 `timings` 字段。

## 7. 三模式在线切换：llama-server Router

llama.cpp 支持 Router 模式：主进程根据请求 JSON 中的 `model` 字段动态加载和卸载模型。

创建 `D:\models\minicpm5\models.ini`：

```ini
[quality]
model = D:\models\minicpm5\MiniCPM5-2B-F16.gguf
c = 8192

[daily]
model = D:\models\minicpm5\MiniCPM5-2B-Q8_0.gguf
c = 8192

[speed]
model = D:\models\minicpm5\MiniCPM5-2B-Q4_K_M.gguf
c = 16384
```

启动 Router：

```powershell
llama-server `
  --models-preset D:\models\minicpm5\models.ini `
  --models-max 1 `
  --models-autoload `
  --host 127.0.0.1 `
  --port 8080 `
  --api-key "CHANGE_THIS_TO_A_LONG_RANDOM_KEY" `
  -ngl 99 `
  --parallel 1 `
  --jinja
```

`--models-max 1` 很重要：10GB 显存不适合同时驻留三个模型。切换模式时，旧模型会被卸载，新模型加载；首次切换会有加载延迟。

请求时使用对应的 `model` 名称：

```json
{
  "model": "daily",
  "messages": [
    {"role": "user", "content": "你好，请介绍一下自己"}
  ],
  "max_tokens": 512,
  "stream": true
}
```

把 `model` 改成 `quality` 或 `speed` 即可切换模式。

如果当前 llama.cpp 版本对 preset 的自定义名称行为不同，先通过 `GET /models` 查看实际模型 ID，再使用返回的 ID 作为请求中的 `model` 字段。

## 8. 远程访问方案

### 推荐方案：ECS 作为 HTTPS 网关

```text
远程客户端
    ↓ HTTPS + API Key
ECS（Caddy/Nginx，公网入口）
    ↓ WireGuard / Tailscale / Cloudflare Tunnel
本地第二台机器（127.0.0.1:8080）
    ↓ Vulkan
RX 6750 GRE
```

推荐让 llama-server 只监听 `127.0.0.1`，由 ECS 或隧道负责公网入口。这样不需要直接暴露家用网络端口，ECS 只转发请求，实际推理仍在本地 GPU 上进行。

### 备选方案：本机公网 IP

如果必须直连公网：

- 不要直接把 llama-server 管理端口暴露到公网
- 通过 Caddy/Nginx 终止 TLS
- 使用强 API Key
- 在 Windows 防火墙和路由器中只开放反向代理端口
- 尽量增加来源 IP 白名单和速率限制
- 禁止未授权访问 `/models`、`/props`、`/metrics` 等管理接口

## 9. 性能预期

在 RX 6750 GRE 10GB、Vulkan、单用户、`-ngl 99` 条件下，可先按以下范围估算：

| 模式 | 预计生成速度 |
|---|---:|
| F16 + 8192 | 15–30 tokens/s |
| Q8_0 + 8192 | 25–45 tokens/s |
| Q4_K_M + 16384 | 35–60 tokens/s |

实际速度必须以本机 `timings` 或 `llama-bench` 测试为准。思考模式会生成额外 token，用户看到的最终回答速度可能低于底层 decode 速度。

## 10. 上线前检查清单

- [ ] `llama-server --list-devices` 能看到 Vulkan AMD GPU
- [ ] 三个模型都能在本地启动
- [ ] 日志确认使用了 Vulkan，而不是纯 CPU
- [ ] F16 在 8192 上下文下不会显存溢出
- [ ] Router 能通过 `model` 字段切换三个模式
- [ ] `--models-max 1` 已启用
- [ ] API Key 已替换为随机强密钥
- [ ] llama-server 只监听 `127.0.0.1`
- [ ] ECS/隧道层已经配置 HTTPS
- [ ] 公网访问已配置限流和访问日志
- [ ] 未公开 Hugging Face token、API Key 或本地路径中的敏感信息

## 11. 参考资料

- [MiniCPM5 模型集合](https://huggingface.co/collections/openbmb/minicpm5)
- [MiniCPM5-2B-GGUF](https://huggingface.co/openbmb/MiniCPM5-2B-GGUF)
- [MiniCPM5-2B 配置文件](https://huggingface.co/openbmb/MiniCPM5-2B/raw/main/config.json)
- [MiniCPM 官方 llama.cpp 部署文档](https://github.com/OpenBMB/MiniCPM/blob/main/docs/deployment/llama_cpp.md)
- [llama.cpp llama-server 文档](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md)
- [llama.cpp Vulkan 构建文档](https://github.com/ggml-org/llama.cpp/blob/master/docs/build.md#vulkan)

