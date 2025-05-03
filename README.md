# 🚀 Qwen3 模型本地部署 + Gradio WebUI 项目文档
## 🔧 环境准备（Ubuntu 系统）
```bash
sudo apt update && sudo apt install -y git build-essential cmake python3 python3-venv python3-pip libcurl4-openssl-dev
```
## 📦 获取 llama.cpp 并编译
```bash
mkdir -p ~/qwen-chatbot && cd ~/qwen-chatbot
git clone https://github.com/ggerganov/llama.cpp.git
cd llama.cpp
rm -rf build
mkdir build && cd build
cmake .. -DGGML_CUDA=on
cmake --build . -j4
```

## 🧠 下载模型（GGUF 格式）

> 使用 Hugging Face 的 Python 接口下载模型（需要登录一次）

1. 安装 huggingface_hub

```bash
pip install huggingface_hub
```

2. 登录 Hugging Face 账户（只需一次）
```bash
huggingface-cli login
```
系统会提示输入 Access Token，复制你的 Hugging Face Token 粘贴进去（访问：`https://huggingface.co/settings/tokens` 创建）。

3. 下载模型脚本
创建下载脚本：

```bash
cd ~/qwen-chatbot/models
nano download.py
```

粘贴以下内容保存（保存：Ctrl+O -> Enter -> Ctrl+X）：

```python
from huggingface_hub import hf_hub_download

hf_hub_download(
    repo_id="Qwen/Qwen3-1.5B-Instruct-GGUF",
    filename="qwen3_1.5b_instruct_q4_K_M.gguf",
    local_dir=".",
    local_dir_use_symlinks=False
)
```

运行：
```bash
python3 download.py
```
模型将保存到 models/qwen3_1.5b_instruct_q4_K_M.gguf
并重命名：

```bash
mv qwen3_1.5b_instruct_q4_K_M.gguf qwen3.gguf
```

## ✅ 模型快速运行测试
```bash
cd ~/qwen-chatbot
./llama.cpp/build/bin/llama-run --ngl 35 -n 200 ./models/qwen3.gguf
```
然后输入"你好，请你自我介绍一下。"

## 🌐 Gradio 网页界面部署
1. 创建 Python 虚拟环境并安装 gradio
```bash
cd ~/qwen-chatbot
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install gradio
```

2. 创建 gradio_app.py 脚本
```bash
nano gradio_app.py
```

粘贴以下内容并保存：

```python
import gradio as gr
import subprocess

MODEL_PATH = "./models/qwen3.gguf"
LLAMA_RUN = "./llama.cpp/build/bin/llama-run"

def chat_with_qwen(prompt, max_tokens=200):
    try:
        result = subprocess.run(
            [LLAMA_RUN, "--ngl", "35", "--color", "-n", str(max_tokens), MODEL_PATH, prompt],
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
            universal_newlines=True
        )
        output = result.stdout
        lines = output.splitlines()
        reply = "\n".join(line for line in lines if not line.strip().startswith(">"))
        return reply.strip()
    except Exception as e:
        return f"发生错误：{e}"

with gr.Blocks(title="Qwen3 Chatbot") as demo:
    gr.Markdown("## 🧠 本地 Qwen3 Chatbot\\n无需联网，即刻体验通义千问")
    with gr.Row():
        with gr.Column():
            prompt_box = gr.Textbox(label="请输入内容", lines=6)
            submit_btn = gr.Button("Submit")
            clear_btn = gr.Button("Clear")
        with gr.Column():
            output_box = gr.Textbox(label="output", lines=20)

    def clear_input():
        return "", ""

    submit_btn.click(fn=chat_with_qwen, inputs=[prompt_box], outputs=[output_box])
    clear_btn.click(fn=clear_input, outputs=[prompt_box, output_box])

demo.launch(server_name="0.0.0.0", server_port=7860)
```

3. 启动 WebUI
```bash
cd ~/qwen-chatbot
source venv/bin/activate
python gradio_app.py
```

4. 打开浏览器访问：
```http://localhost:7860```

## 💡 常见问题
1. 如果运行失败显示 invalid argument，请检查是否使用了过时参数，例如 `--ngl` 应该配合 `llama-run` 而非 `llama-cli`。

2. 若默认使用核显而非 NVIDIA GPU，请确认是否正确执行了 `cmake .. -DGGML_CUDA=on` 并重编译。

3. 如果无法访问 HuggingFace 或 GitHub，建议科学上网或提前离线下载模型文件。
