# 运行命令

- 项目：Dia（nari-labs/dia）1.6B 文本对话 TTS
- 生成时间：2026-09-08
- 运行方式：直接运行（`uv`）
- 硬件评估：**满足** — 官方 float16/bfloat16 约 4.4GB VRAM、float32 约 7.9GB；本机 RTX 4080 16GB，启动前约 1.3GB 占用、剩余约 15GB。推荐精度 float16。

## 环境准备

本机已有：`pwsh` 7、`uv`、ffmpeg（`E:\Programs\ffmpeg-master-latest-win64-gpl\bin`）、NVIDIA 驱动。

网络探测结论（拉取前）：

| 源 | 结果 | 实际使用 |
|----|------|----------|
| pypi.org | 可用但慢（~940ms） | 清华镜像 |
| download.pytorch.org | 超时 | 阿里云 PyTorch cu126 |
| huggingface.co | 超时 | `HF_ENDPOINT=https://hf-mirror.com` |
| github.com Release（DAC `weights.pth`） | 直连超时 | 从 `https://hf-mirror.com/descript/descript-audio-codec/resolve/main/weights.pth` 下到 `%USERPROFILE%\.cache\descript\dac\weights_44khz_8kbps_0.0.1.pth` |

在仓库根目录执行：

```powershell
pwsh -NoProfile -Command @'
$env:UV_INDEX_URL = 'https://pypi.tuna.tsinghua.edu.cn/simple'
$env:UV_INDEX_PYTORCH_CU126_URL = 'https://mirrors.aliyun.com/pytorch-wheels/cu126'
$env:HF_ENDPOINT = 'https://hf-mirror.com'
uv sync
'@
```

首次启动会从 Hugging Face（经镜像）下载 `nari-labs/Dia-1.6B-0626`；Descript Audio Codec（约 293MB）若 GitHub Release 不可达，按上表用 hf-mirror 预置到本机 cache。

## 启动

- 推荐：`pwsh -NoProfile -File .\start.ps1`
- 说明：单服务 `webui`，无参直接启动（跳过菜单）。Docker：`-Mode docker`。
- 等价手动命令：

```powershell
$env:Path = "E:\Programs\ffmpeg-master-latest-win64-gpl\bin;$env:Path"
$env:HF_ENDPOINT = 'https://hf-mirror.com'
$env:HF_HUB_DISABLE_SYMLINKS_WARNING = '1'
$env:PYTHONUNBUFFERED = '1'
$env:GRADIO_SERVER_NAME = '127.0.0.1'
$env:GRADIO_SERVER_PORT = '7860'   # 占用时由 start.ps1 顺延
.\.venv\Scripts\python.exe app.py
```

DAC 预下载（GitHub 不通时）：

```powershell
$dir = "$env:USERPROFILE\.cache\descript\dac"
New-Item -ItemType Directory -Force -Path $dir | Out-Null
curl.exe -L -o "$dir\weights_44khz_8kbps_0.0.1.pth" `
  "https://hf-mirror.com/descript/descript-audio-codec/resolve/main/weights.pth"
```

可选：

- CLI：`uv run cli.py --help`
- 示例推理：`uv run example/simple.py`（输出 `simple.mp3`）
- Docker（备选，非默认）：`docker build . -f docker/Dockerfile.gpu -t dia-gpu` 后 `docker run --rm --gpus all -p 7860:7860 dia-gpu`

## 验证

1. 浏览器打开 `http://127.0.0.1:<实际端口>`（默认起点 7860），Gradio UI 可访问。
2. 输入带 `[S1]` / `[S2]` 的英文对话文本，生成音频无 CUDA 报错。
3. 或跑：`uv run example/simple.py`，退出码 0 且生成 `simple.mp3`。

## 备注

- 默认精度：CUDA 上 `float16`（`app.py`）。
- 端口：`start.ps1` 从 7860 起探测，占用则 +1；本次实际：**7860**。
- 镜像源见上表；勿把 `origin` 永久改成镜像。
- `nvcc` 可不要求：本项目用预编译 `torch==2.6.0+cu126` 轮子。
- 本文件与 `start.ps1` 为本机运行材料，默认不提交。
- 2026-09-08 已跑通：Gradio `http://127.0.0.1:7860` 返回 HTTP 200；加载后显存约 7.3GB / 16GB。
