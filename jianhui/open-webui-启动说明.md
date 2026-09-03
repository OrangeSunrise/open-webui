# open-webui-启动说明

本项目在windows11上运行，安装依赖可能需要vpn，请提前开启！

## 创建自己的git分支

根据dev-jianhui分支创建自己的分支

## 环境要求

```bash
python3.11 # 推荐
nodejs 21 # 推荐
```

## ollama

- 安装ollama
- 运行qwen3-4b模型（`ollama run qwen3:4b`）
- 启动后端需要安装好本地模型并运行ollama！

## 根据模板生成env文件

```bash
cd ~/projects/open-webui

cp .env.example .env
```

## 后端启动

```bash
cd C:\Users\cjh\Documents\Project\open-webui

cd backend

# 创建虚拟环境，推荐uv，镜像源已经在项目配置完成
uv venv --python 3.11

# 激活虚拟环境
.venv\Scripts\activate

# 检查
python --version

# 安装依赖
uv pip install -r requirements.txt

# 只在第一次运行执行，以后运行不需要执行

$env:CORS_ALLOW_ORIGIN = "http://localhost:5173;http://localhost:8080" 

# 生成 64 位随机 key，写入 backend/.env
# （env.py 会自动加载 BASE_DIR/.env，一劳永逸，以后新窗口不用再设）
$key = -join ((48..57)+(65..90)+(97..122) | Get-Random -Count 64 | ForEach-Object { [char]$_ })
if (Test-Path .env) { Add-Content .env "`nWEBUI_SECRET_KEY=$key" } else { Set-Content .env "WEBUI_SECRET_KEY=$key" }

# 启动后端
uvicorn open_webui.main:app --port 8080 --host 0.0.0.0 --ws-per-message-deflate true --reload

# 验证后端是否成功启动
curl http://localhost:8080/health
# 应返回 {"status":true} 之类

```

## 前端启动

```bash
cd C:\Users\cjh\Documents\Project\open-webui

# 确保nodejs版本>=19，<=21，推荐21
node -v

$env:npm_config_registry = "https://registry.npmmirror.com"
# CYPRESS_DOWNLOAD下载需要开vpn
npm install
npm run dev

```
