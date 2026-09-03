# Node 22 安装与排查记录（fnm 多版本方案）

> 时间：2026-09-02
> 背景：Open WebUI 前端用 `.npmrc` 的 `engine-strict=true` + package.json `engines`（`>=18.13.0 <=22.x.x`）强制 node 版本，机器原有 node v24.19.0 会被 `npm install` 直接拒绝（EBADENGINE）。需要 node 22 与 24 共存。
> 结果：采用 fnm，node v22.23.2 安装成功、24 保留；过程中踩了三个坑，全部记录在案。

---

## 一、方案选型

| 方案 | 结论 | 理由 |
|---|---|---|
| **fnm（采用）** | ✅ | 与现有 node 24 共存、支持镜像下载、可按项目自动切换 |
| zip + 系统 PATH | 备选 | 可行，但属全局固定切换；且必须把条目放进**系统变量**（Windows 的有效 PATH = 系统变量在前 + 用户变量在后，放用户变量抢不过 `C:\Program Files\nodejs`）；切换靠手动挪 PATH |
| nvm-windows | 排除 | 要求先卸载现有独立安装的 node，与"保留 24"冲突 |
| Docker（nodejs.org 官方指引） | 排除 | 那是"临时体验 node"的通用指引，不适合本项目前端开发：`--rm` 容器依赖不持久、挂载层破坏 Vite 热更新（文件事件跨不过去）、容器内 node_modules 与宿主机平台二进制冲突、cypress 不支持 alpine（npm install 即报错） |

## 二、安装步骤（最终成功版本）

```powershell
# 1. 安装 fnm
winget install Schniz.fnm          # → fnm 1.39.0
# 关闭并重开终端

# 2. PowerShell Profile 集成（一次性）
#    注意：pwsh7 与 Windows PowerShell 5.1 的 Profile 是两个不同的文件
Add-Content $PROFILE 'fnm env --use-on-cd --shell powershell | Out-String | Invoke-Expression'
# 重开终端（或在当前窗口 . $PROFILE）

# 3. 临时走淘宝镜像安装 node 22（窗口级环境变量，关窗失效）
$env:FNM_NODE_DIST_MIRROR = "https://npmmirror.com/mirrors/node/"
fnm install 22        # → v22.23.2，34MB 约 3 秒（镜像生效）
fnm default 22        # 以后新终端默认 22
fnm use 22            # 在已执行过 fnm env 的窗口里才有意义
node -v               # v22.23.2 ✅
```

## 三、踩坑与排查实录 ⭐

### 坑 1：`fnm use 22` 报"找不到环境变量"

```text
error: We can't find the necessary environment variables to replace the Node version.
You should setup your shell profile to evaluate `fnm env` ...
```

- **原因**：`fnm use` 依赖 `fnm env` 在**当前 shell** 里生成的一组环境变量（`FNM_MULTISHELL_PATH` 等）。当时只在旧窗口里执行了 `Add-Content`（写文件），从未执行过那行命令——**写进去 ≠ 运行了**。
- **修复**：重开窗口（Profile 自动执行），或在当前窗口手动跑一次：
  `fnm env --use-on-cd --shell powershell | Out-String | Invoke-Expression`

### 坑 2：新窗口仍不生效 → 命令被"粘"进了一行注释

诊断四连（在报错的新窗口里）：

```powershell
$PSVersionTable.PSVersion                                    # 7.6.5（pwsh7）
$PROFILE                                                     # Documents\PowerShell\...（pwsh7 专用）
Test-Path $PROFILE                                           # True
Get-ExecutionPolicy                                          # RemoteSigned（没问题）
Select-String -Path $PROFILE -Pattern 'fnm'                  # ← 关键一步
```

Select-String 输出暴露真相：

```text
...:24: # (& uvx --generate-shell-completion powershell) | Out-String | Invoke-Expressionfnm env --use-on-cd ...
```

- 原 Profile 最后一行**结尾没有换行符**，`Add-Content` 追加时直接粘在同一行，出现 `Invoke-Expressionfnm` 连体字
- 那行原本以 `#` 开头（注释掉的 uvx 补全配置），拼接后**整行连同我们的命令都成了注释**——Profile 加载不报错，但什么都没执行
- **修复**：`notepad $PROFILE` 手动拆成两行：原注释行保持原样；fnm 命令另起一行、顶格、不带 `#`

### 坑 3：拆好后报 `fnm` is not recognized

- **原因**：在 fnm 安装**之前**就开着的 VS Code / Windows Terminal 里开"新标签页"，继承的是**宿主启动那一刻**的 PATH，不含 winget 新装的 fnm 路径（`%LOCALAPPDATA%\Microsoft\WinGet\Links`）。**新标签 ≠ 新进程环境**。
- **修复**：VS Code 整个退出重开（不是关标签页）。
- **防御写法**（建议）：Profile 里的 fnm 行包一层判断，旧窗口安静降级不刷红字：

```powershell
if (Get-Command fnm -ErrorAction SilentlyContinue) {
    fnm env --use-on-cd --shell powershell | Out-String | Invoke-Expression
}
```

## 四、镜像变量速查（全部窗口级，关窗即失效，勿写入仓库 .npmrc）

| 变量 | 作用 | 值 |
|---|---|---|
| FNM_NODE_DIST_MIRROR | 下载 node 本体 | `https://npmmirror.com/mirrors/node/` |
| npm_config_registry | npm 依赖包 | `https://registry.npmmirror.com` |
| CYPRESS_DOWNLOAD_MIRROR | cypress 二进制（~200MB） | `https://npmmirror.com/mirrors/cypress/` |

注：旧地址 `registry.npm.taobao.org` 已废弃；根 pyproject 的 tuna 镜像只管 Python 包，管不到 npm/cypress。

## 五、最终状态

```text
fnm list
* v22.23.2 default
* system          ← 原 node v24.19.0，保留未动
```

- 新窗口默认 node v22.23.2 / npm 10.x；VS Code 集成终端走同一个 Profile，行为一致
- 想用回 24：`fnm install 24 ; fnm use 24`，或注释 Profile 里的 fnm 行
- 想按项目自动切换：仓库根建 `.node-version` 文件（内容 `22`），配合 `--use-on-cd` 生效

## 六、经验小结

1. `fnm use` 依赖 `fnm env` 在当前 shell 生成的变量——"写进 Profile"和"执行过 Profile"是两回事
2. `Add-Content` 追加前先确认目标文件末尾有没有换行符，否则粘行；改完 Profile 用 `Select-String` 验证内容
3. PowerShell 5.1 与 7 的 Profile 是两个独立文件，两边都要用就都写
4. 环境变量更新后必须**完全重启**宿主程序（VS Code / Windows Terminal），新标签页继承的是宿主的旧环境
5. 官方"通用指引"（如 nodejs.org 的 Docker 用法）未必适配具体项目的工作流，选型要看自己的场景
6. `engine-strict` + `engines` 是硬门槛：装依赖前先核对 node 版本
