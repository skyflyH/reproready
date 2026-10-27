# ReproReady 本地开发环境准备指南

版本：v1.0  
日期：2026-10-27  
适用平台：Windows 10/11 x64、PowerShell 7 或 Windows PowerShell 5.1  
依据：`ReproReady-技术设计文档.md` v0.2

## 1. 项目英文名称

- 产品全称：**ReproReady: Bug Reproduction Readiness Assistant**
- 简短名称：**ReproReady**
- GitHub 仓库名及项目根目录：**`reproready`**
- Python 包名：**`repro-ready`**（在 `pyproject.toml` 中使用）；Python 导入包名：**`repro_ready`**

建议仓库首页描述：`Evidence-backed bug reproduction readiness assistant powered by RAG.`

## 2. 本机环境检查结果

检查时间：2026-10-27。版本来自当前 PowerShell 会话；安装或升级后应重新打开终端复查。

| 工具 | 检查结果 | 项目是否需要 | 处理建议 |
|---|---|---|---|
| Windows x64 | 当前开发环境 | 是 | 满足本指南适用范围 |
| Python | `python` 指向 Anaconda Python 3.7.0；`py` 未发现可用 Python | 是，要求 Python 3.11 | 安装独立 Python 3.11 x64；不要用现有 3.7 环境运行项目 |
| uv | 未安装 | 推荐 | 安装后用于 Python 版本管理、依赖同步和运行命令 |
| Git | 2.50.1.windows.1 | 是 | 已可用，无需重装 |
| Node.js / npm | Node.js 22.18.0 / npm 11.17.0 | 否 | 前端为 Streamlit，不需要 Node 工具链 |
| Java | 21.0.12.1 | 否 | 项目不使用 JVM；无需为本项目调整 |
| Docker | 未安装 | 可选 | 仅在选择 Docker Compose 演示时安装；本地 Python 模式不需要 |
| Cursor / VS Code CLI | Cursor 命令 `code` 可用 | 推荐但非运行依赖 | 可继续使用现有编辑器 |
| SQLite | Python 标准库支持 | 是 | 无需单独安装数据库服务器 |
| Chroma | 尚未安装 | 是，作为 Python 项目依赖 | 由 `uv sync` 安装，不单独下载服务端 |

本机检查不等于项目已经可以运行：Python 版本不符合设计要求，uv 和项目依赖还未安装，LLM 与 Embedding 服务凭据也需要配置。

## 3. 必需技术栈与安装方式

### 3.1 安装 Python 3.11 x64

推荐使用 Windows Package Manager：

```powershell
winget install --id Python.Python.3.11 --exact
```

安装完成后关闭并重新打开 PowerShell，检查：

```powershell
py -3.11 --version
py -0p
```

预期首条命令显示 `Python 3.11.x`。若 `winget` 找不到包或安装失败，可从 Python 官方 Windows 下载页选择 Python 3.11.x 的 64-bit installer；安装向导中启用 Python Launcher，并允许安装程序配置 PATH。不要卸载现有 Anaconda；后续使用 `uv`/`py -3.11` 明确选择项目解释器。

### 3.2 安装 uv

推荐使用 winget：

```powershell
winget install --id astral-sh.uv --exact
```

重开终端后检查：

```powershell
uv --version
```

如果本机 winget 源暂不可用，可按 uv 官方 Windows 安装说明使用 PowerShell 安装脚本。安装脚本会从网络下载可执行文件；在企业网络中应遵循本机软件安装策略。

### 3.3 克隆仓库并准备 Python 依赖

项目代码创建并推送到远程仓库后，在希望放置代码的目录执行：

```powershell
git clone https://github.com/<你的 GitHub 用户名>/reproready.git
Set-Location .\reproready
uv python install 3.11
```

设计文档规划使用 `pyproject.toml` 和 `uv.lock`。仓库若已提交锁文件，执行：

```powershell
uv sync --locked
```

首次实现阶段如果尚未生成 `uv.lock`，先执行 `uv sync` 生成并检查锁文件，再将 `uv.lock` 提交到 Git；之后所有开发环境统一使用 `uv sync --locked`。依赖清单至少应覆盖设计文档中的 FastAPI、Uvicorn、Pydantic v2、Streamlit、HTTPX、LangChain Core/Text Splitters、所选模型 Provider 的 LangChain 集成、Chroma、SQLAlchemy 2、python-multipart、pydantic-settings、Jinja2、pytest 和 Ruff。具体 Provider 依赖需在选定模型服务后写入 `pyproject.toml`，不要安装所有厂商 SDK。

检查 Python 与项目依赖：

```powershell
uv run python --version
uv run python -c "import fastapi, streamlit, chromadb, sqlalchemy; print('core imports ok')"
```

若 Chroma 的底层依赖在当前 Windows/Python 组合安装失败，优先使用受支持的 Python 3.11 x64 和项目锁文件；Docker 是替代运行环境，不应通过随意升级单个底层包绕过锁文件。

### 3.4 配置模型服务

模型 API 不随 Python 包安装。项目需要一个支持结构化输出的聊天模型服务，以及与所选向量维度相匹配的 Embedding 服务。若同一供应商同时提供两者，可以共用服务商，但需分别填写模型名和必要凭据。

```powershell
Copy-Item .env.example .env
```

根据 `.env.example` 配置 `LLM_PROVIDER`、`LLM_API_KEY`、`LLM_BASE_URL`、`LLM_MODEL`、`EMBEDDING_API_KEY`、`EMBEDDING_BASE_URL` 和 `EMBEDDING_MODEL`。实际变量名以实现后的 `pydantic-settings` 配置为准。`.env` 必须加入 `.gitignore`，不得提交 API Key、真实日志或用户数据。模型服务可能接收脱敏后的问题和证据，演示前需使用合成数据并确认服务商的数据处理政策。

### 3.5 启动和检查（代码完成后）

在仓库根目录开两个 PowerShell 窗口。先从 `.env.example` 创建 `.env` 并确认本地配置。

窗口一启动 API：

```powershell
uv run uvicorn repro_ready.api.main:app --host 127.0.0.1 --port 8000
```

窗口二启动 Streamlit：

```powershell
uv run streamlit run src/repro_ready/ui/app.py
```

执行自动检查：

```powershell
uv run pytest
uv run ruff check .
uv run python scripts/evaluate.py
```

以上是技术设计文档预期命令，只有在相应源代码和脚本完成后才可执行；当前仅完成环境盘点和准备指南。

## 4. 可选：Docker Desktop

本地 Python 开发不需要 Docker。若要按设计文档运行 `docker compose up --build`，先安装 Docker Desktop：

```powershell
winget install --id Docker.DockerDesktop --exact
```

按提示启用/安装 WSL 2、重启 Windows，并启动 Docker Desktop。之后检查：

```powershell
docker --version
docker compose version
docker info
```

Docker Desktop 的 WSL 2 与系统虚拟化要求以 Docker 官方 Windows 安装文档为准。安装期间如提示开启 Windows 功能或重启，应完成系统提示后再验证。容器模式仍须提供 `.env` 中的模型服务凭据。

## 5. 非必需工具

- Node.js、npm：仅当未来单独开发 JavaScript 前端或配套工具时再安装；当前 Streamlit UI 不使用它们。
- Java：项目没有 Java/Spring 服务，不需要 JDK。
- PostgreSQL、MySQL、Redis：设计使用 SQLite、本地 Chroma 和进程内队列，首版不需要这些服务。
- 独立 Chroma Server：设计使用持久化本地模式，首版不需要启动 Chroma 服务。
- LangGraph、多 Agent 框架：不在首版技术范围内。

## 6. 建议安装顺序与完成判定

1. 保留现有 Git 和 Cursor；安装 Python 3.11 x64。
2. 安装 uv，并确认 `py -3.11 --version` 与 `uv --version` 正常。
3. 获取 `reproready` 源码，确认 `pyproject.toml` 和 `uv.lock` 已提交，再执行 `uv sync --locked`。
4. 复制 `.env.example` 为 `.env`，配置所选模型与 Embedding 服务；不要把密钥提交到仓库。
5. 运行导入检查、测试、Ruff 和评估脚本；代码尚未实现时，这些应用级检查暂不能通过。
6. 仅在需要容器演示时安装 Docker Desktop 并验证 Compose。

环境就绪的最低判定：Python 3.11 被项目虚拟环境选中；锁文件依赖同步成功；核心依赖导入成功；Git 可用；模型与 Embedding 配置通过应用内连通性检查。Docker 仅在容器演示目标下纳入判定。

## 7. 参考链接

- Python for Windows：<https://www.python.org/downloads/windows/>
- uv 安装与文档：<https://docs.astral.sh/uv/getting-started/installation/>
- winget 文档：<https://learn.microsoft.com/windows/package-manager/winget/>
- Docker Desktop for Windows：<https://docs.docker.com/desktop/setup/install/windows-install/>
- Git for Windows：<https://git-scm.com/download/win>
