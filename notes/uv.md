# uv 使用指南

> 面向已经使用过 `pip + venv` 的 Python 开发者。
>
> 本文重点介绍 uv 如何替代以前的 `pip + venv` 工作流，以及日常开发中最常用的命令。

---

## 1. uv 是什么？

`uv` 是一个用 Rust 编写的 Python 工具链，主要用于：

* Python 版本管理
* 虚拟环境管理
* Python 包安装
* 项目依赖管理
* 依赖版本锁定
* 运行 Python 程序
* 运行 Python CLI 工具

以前我们可能需要：

```text
python
├── venv
├── pip
└── requirements.txt
```

现在可以大量统一到：

```text
uv
├── Python
├── venv
├── package management
├── dependency lock
└── project management
```

---

# 2. pip + venv 和 uv 的对应关系

如果你以前习惯：

```bash
python -m venv .venv
pip install requests
pip freeze > requirements.txt
```

那么 uv 中主要对应：

```bash
uv venv
uv add requests
uv lock
```

常见对应关系：


| 以前                              | uv                         |
| --------------------------------- | -------------------------- |
| `python -m venv .venv`            | `uv venv`                  |
| `pip install requests`            | `uv add requests`          |
| `pip uninstall requests`          | `uv remove requests`       |
| `pip install -r requirements.txt` | `uv sync`                  |
| `pip freeze`                      | `uv pip freeze`            |
| `python main.py`                  | `uv run main.py`           |
| `pip install xxx`                 | `uv pip install xxx`       |
| `pip`                             | `uv pip`                   |
| `requirements.txt`                | `pyproject.toml`+`uv.lock` |

需要特别注意：

> `uv add` 和 `uv pip install` 不是一回事。

后面会详细解释。

---

# 3. 安装 uv

Windows 推荐：

```powershell
winget install astral-sh.uv
```

安装完成后：

```powershell
uv --version
```

例如：

```text
uv 0.x.x
```

检查：

```powershell
where uv
where uvx
```

官方 uv 常用的两个命令：

```text
uv
uvx
```

其中：

* `uv`：项目、依赖、Python、虚拟环境管理
* `uvx`：临时运行 Python CLI 工具

---

# 4. uv 的三个常见命令

## 4.1 uv

主要命令：

```bash
uv
```

例如：

```bash
uv init
uv add requests
uv remove requests
uv sync
uv run main.py
uv venv
uv python list
```

日常开发主要使用它。

---

## 4.2 uvx

`uvx` 用来临时运行 Python CLI 工具。

例如：

```bash
uvx ruff check .
```

相当于：

```bash
npx ruff
```

可以运行：

```bash
uvx black .
uvx ruff check .
uvx pytest
```

它适合开发工具，不需要把工具永久安装到当前项目。

---

# 5. 创建一个新的 uv 项目

推荐的新项目方式：

```bash
uv init myproject
cd myproject
```

目录大致：

```text
myproject/
├── .python-version
├── pyproject.toml
└── README.md
```

然后：

```bash
uv sync
```

uv 会自动创建：

```text
.venv/
```

最终：

```text
myproject/
├── .venv/
├── .python-version
├── pyproject.toml
└── README.md
```

---

# 6. pyproject.toml

使用 uv 后，项目最重要的文件之一就是：

```text
pyproject.toml
```

例如：

```toml
[project]
name = "myproject"
version = "0.1.0"
description = "My Python project"
requires-python = ">=3.12"
dependencies = [
    "requests>=2.32.0",
]
```

这里：

```toml
dependencies = [
    "requests>=2.32.0",
]
```

就是项目依赖。

以前我们可能把它写在：

```text
requirements.txt
```

现在推荐放到：

```text
pyproject.toml
```

---

# 7. 安装依赖：uv add

以前：

```bash
pip install requests
```

现在推荐：

```bash
uv add requests
```

例如：

```bash
uv add requests
uv add fastapi
uv add redis
uv add sqlalchemy
```

执行：

```bash
uv add requests
```

uv 会：

1. 修改 `pyproject.toml`
2. 解析依赖
3. 更新 `uv.lock`
4. 安装依赖到 `.venv`

---

# 8. 删除依赖

以前：

```bash
pip uninstall requests
```

现在：

```bash
uv remove requests
```

例如：

```bash
uv remove redis
```

uv 会同时更新：

```text
pyproject.toml
uv.lock
```

---

# 9. uv.lock 是什么？

这是 uv 非常重要的一个文件：

```text
uv.lock
```

例如：

```text
pyproject.toml
        ↓
     依赖声明
        ↓
    uv.lock
        ↓
精确依赖版本
```

假设：

```text
FastAPI
 ↓
Starlette
 ↓
anyio
 ↓
sniffio
```

`pyproject.toml` 主要描述：

```text
我要 FastAPI
```

而：

```text
uv.lock
```

会锁定整个依赖树的具体版本。

因此团队开发时：

```bash
git add pyproject.toml uv.lock
```

通常应该把：

```text
uv.lock
```

提交到 Git。

---

# 10. 安装项目全部依赖

当你从 Git 拉下来一个项目：

```bash
git clone xxx
cd myproject
```

不需要：

```bash
python -m venv .venv
pip install -r requirements.txt
```

直接：

```bash
uv sync
```

uv 会根据：

```text
pyproject.toml
uv.lock
```

创建：

```text
.venv
```

并安装依赖。

然后：

```bash
uv run python main.py
```

---

# 11. uv run

这是 uv 日常开发非常常用的命令。

以前：

```bash
.venv\Scripts\activate
python main.py
```

现在：

```bash
uv run python main.py
```

例如：

```bash
uv run python main.py
```

或者：

```bash
uv run pytest
```

或者：

```bash
uv run ruff check .
```

uv 会自动使用项目对应的虚拟环境。

因此通常不需要手动：

```bash
.venv\Scripts\activate
```

---

# 12. 是否还需要 activate？

不需要。

以前：

```bash
.venv\Scripts\activate
python main.py
```

uv：

```bash
uv run python main.py
```

两者效果类似。

当然，如果你习惯手动激活，也可以：

```powershell
.venv\Scripts\activate
```

然后：

```bash
python main.py
```

uv 并没有禁止这种用法。

只是：

> 使用 uv 后，通常没必要依赖 activate。

---

# 13. 查看虚拟环境

项目执行：

```bash
uv sync
```

后：

```text
.venv/
```

就是虚拟环境。

可以查看：

```bash
uv run python --version
```

查看 Python：

```bash
uv run where python
```

Windows 下可能看到：

```text
...\myproject\.venv\Scripts\python.exe
```

---

# 14. 手动创建虚拟环境

如果你只是想使用 uv 替代：

```bash
python -m venv .venv
```

可以：

```bash
uv venv
```

指定 Python：

```bash
uv venv --python 3.12
```

然后：

```bash
uv pip install requests
```

这种方式更接近以前的：

```bash
python -m venv .venv
pip install requests
```

---

# 15. uv pip

uv 提供了一个与 pip 类似的接口：

```bash
uv pip
```

例如：

```bash
uv pip install requests
```

```bash
uv pip uninstall requests
```

```bash
uv pip list
```

```bash
uv pip freeze
```

```bash
uv pip show requests
```

因此：

```text
uv pip
```

可以理解成：

> 一个兼容 pip 工作方式的高速包管理接口。

---

# 16. uv add 和 uv pip install 的区别

这是刚开始使用 uv 时最容易搞混的地方。

## uv add

```bash
uv add requests
```

用于：

> **项目依赖管理**

它会修改：

```text
pyproject.toml
uv.lock
```

例如：

```bash
uv add fastapi
```

项目以后会记录：

```toml
dependencies = [
    "fastapi>=..."
]
```

---

## uv pip install

```bash
uv pip install requests
```

更接近传统：

```bash
pip install requests
```

它主要是直接操作 Python 环境。

不会按照 `uv add` 那样把依赖作为项目依赖写入：

```text
pyproject.toml
```

---

## 简单记忆

新项目：

```bash
uv add xxx
```

临时操作环境：

```bash
uv pip install xxx
```

---

# 17. requirements.txt 怎么办？

如果你以前有：

```text
requirements.txt
```

例如：

```text
fastapi
redis
requests
sqlalchemy
```

可以：

```bash
uv init
uv add -r requirements.txt
```

之后 uv 会把依赖迁移到：

```text
pyproject.toml
```

然后生成：

```text
uv.lock
```

以后主要维护：

```text
pyproject.toml
uv.lock
```

而不是：

```text
requirements.txt
```

---

# 18. 从 requirements.txt 创建环境

如果你暂时不想迁移项目，也可以继续使用：

```bash
uv pip install -r requirements.txt
```

所以旧项目可以非常平滑地迁移：

```text
旧项目

requirements.txt
       ↓
uv pip install -r requirements.txt
```

不需要一次性重构整个项目。

---

# 19. Python 版本管理

uv 不只是包管理器，也可以管理 Python。

查看可用 Python：

```bash
uv python list
```

安装 Python 3.12：

```bash
uv python install 3.12
```

安装 Python 3.13：

```bash
uv python install 3.13
```

查看已经安装的版本：

```bash
uv python list
```

---

# 20. 指定项目 Python 版本

例如项目使用 Python 3.12：

```bash
uv python pin 3.12
```

项目目录会生成：

```text
.python-version
```

内容类似：

```text
3.12
```

以后进入项目：

```bash
uv sync
```

uv 可以根据项目要求使用对应的 Python。

---

# 21. 一个完整项目示例

假设我要创建：

```text
FastAPI + Redis
```

项目。

首先：

```bash
uv init myapi
cd myapi
```

指定 Python：

```bash
uv python pin 3.12
```

安装依赖：

```bash
uv add fastapi
uv add redis
uv add uvicorn
```

最终：

```text
myapi/
├── .venv/
├── .python-version
├── pyproject.toml
├── uv.lock
└── main.py
```

运行：

```bash
uv run uvicorn main:app --reload
```

整个过程中：

```text
不需要手动创建 venv
不需要 pip install
不需要 activate
不需要手动维护 requirements.txt
```

---

# 22. 开发工具

例如使用 Ruff。

以前可能：

```bash
pip install ruff
```

现在可以使用：

```bash
uvx ruff check .
```

如果希望 Ruff 成为项目开发依赖：

```bash
uv add --dev ruff
```

然后：

```bash
uv run ruff check .
```

---

# 23. dev dependency

生产运行依赖和开发依赖应该区分。

例如：

```bash
uv add fastapi
```

这是正常项目依赖。

而：

```bash
uv add --dev pytest
uv add --dev ruff
```

是开发依赖。

例如：

```text
运行项目：
fastapi
redis
sqlalchemy

开发项目：
pytest
ruff
```

可以理解成以前经常单独维护：

```text
requirements.txt
requirements-dev.txt
```

现在 uv 可以在：

```text
pyproject.toml
```

里统一管理。

---

# 24. 更新依赖

查看项目依赖：

```bash
uv tree
```

更新 lock：

```bash
uv lock --upgrade
```

然后：

```bash
uv sync
```

如果想更新某个依赖，可以：

```bash
uv add "requests>=2.32"
```

或者重新调整版本约束后：

```bash
uv lock
uv sync
```

---

# 25. 查看依赖树

非常实用：

```bash
uv tree
```

例如：

```text
myproject
├── fastapi
│   ├── starlette
│   └── pydantic
├── redis
└── requests
```

当你遇到：

```text
为什么安装了 A 之后又多出来 B？
```

可以用：

```bash
uv tree
```

查看依赖关系。

---

# 26. 清理/同步环境

如果环境和项目定义不一致：

```bash
uv sync
```

通常就可以重新同步。

如果需要更严格地让环境与 lock 文件保持一致：

```bash
uv sync --locked
```

这个在 CI/CD 中比较有用。

意思可以理解成：

> 不允许自动修改 lock 文件，必须严格按照现有 `uv.lock` 安装。

---

# 27. Git 应该提交什么？

推荐：

```text
.gitignore
```

加入：

```text
.venv/
```

不要提交：

```text
.venv/
```

应该提交：

```text
pyproject.toml
uv.lock
.python-version
```

例如：

```text
myproject/
├── .venv/             ← 不提交
├── .python-version    ← 提交
├── pyproject.toml     ← 提交
├── uv.lock            ← 提交
└── main.py            ← 提交
```

---

# 28. Docker 中使用 uv

以后做 Docker 项目时，uv 也很好用。

例如：

```dockerfile
FROM python:3.12

COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

WORKDIR /app

COPY pyproject.toml uv.lock ./

RUN uv sync --frozen

COPY . .

CMD ["uv", "run", "python", "main.py"]
```

这里：

```bash
uv sync --frozen
```

表示严格按照：

```text
uv.lock
```

安装。

---

# 29. 网络和 PyPI 镜像

uv 默认使用 PyPI：

```text
https://pypi.org/simple
```

如果网络可以正常访问 PyPI：

> 不需要配置镜像。

例如使用代理网络时，通常直接：

```bash
uv add requests
```

即可。

如果访问 PyPI 很慢，再考虑配置镜像。

不建议没有网络问题时为了“习惯”而配置镜像。

---

# 30. 常用命令速查

## 项目

```bash
uv init
uv init myproject
uv sync
uv lock
uv tree
```

## 依赖

```bash
uv add requests
uv add fastapi redis
uv add --dev pytest
uv remove requests
```

## 运行

```bash
uv run python main.py
uv run pytest
uv run ruff check .
```

## 虚拟环境

```bash
uv venv
uv venv --python 3.12
```

## Python

```bash
uv python list
uv python install 3.12
uv python pin 3.12
```

## pip 兼容模式

```bash
uv pip install requests
uv pip uninstall requests
uv pip list
uv pip freeze
```

## 临时工具

```bash
uvx ruff check .
uvx black .
uvx pytest
```

---

# 31. 最推荐的日常工作流

如果你已经熟悉：

```text
pip + venv
```

那么以后新项目可以直接按照下面这个流程。

## 创建项目

```bash
uv init myproject
cd myproject
```

## 指定 Python

```bash
uv python pin 3.12
```

## 添加依赖

```bash
uv add fastapi
uv add redis
uv add sqlalchemy
```

开发依赖：

```bash
uv add --dev pytest
uv add --dev ruff
```

## 编写代码

例如：

```text
main.py
```

## 运行

```bash
uv run python main.py
```

FastAPI：

```bash
uv run uvicorn main:app --reload
```

## 提交 Git

提交：

```text
pyproject.toml
uv.lock
.python-version
源代码
```

不提交：

```text
.venv/
```

## 其他开发者拉代码

```bash
git clone xxx
cd myproject
uv sync
uv run python main.py
```

---

# 32. 从 pip + venv 迁移的核心思维

以前：

```text
创建环境
    ↓
python -m venv .venv
    ↓
activate
    ↓
pip install
    ↓
pip freeze
    ↓
requirements.txt
```

现在：

```text
创建项目
    ↓
uv init
    ↓
uv add
    ↓
pyproject.toml
    ↓
uv.lock
    ↓
uv run
```

最重要的几个命令只需要记：

```bash
uv init
uv add
uv remove
uv sync
uv run
uvx
```

以及：

```bash
uv pip
```

用于兼容以前的 pip 工作方式。

---

# 33. 最后：什么时候用什么？

### 新建 Python 项目

推荐：

```bash
uv init
```

### 添加项目依赖

推荐：

```bash
uv add xxx
```

### 删除项目依赖

```bash
uv remove xxx
```

### 安装别人项目的依赖

```bash
uv sync
```

### 运行项目

```bash
uv run xxx
```

### 临时运行 Python 工具

```bash
uvx xxx
```

### 想按照以前 pip 的方式操作环境

```bash
uv pip xxx
```

---

# 34. 一句话总结

如果以前你的 Python 工作流是：

```text
pip + venv + requirements.txt
```

那么推荐逐渐迁移到：

```text
uv + pyproject.toml + uv.lock
```

最常用的日常命令就是：

```bash
uv init
uv add xxx
uv remove xxx
uv sync
uv run xxx
```

而：

```bash
uv pip
```

负责兼容以前的 pip 使用方式，

```bash
uvx
```

负责临时运行 Python CLI 工具。

对于新项目，优先使用 **`uv add` / `uv sync` / `uv run`**，不要把 `uv` 仅仅当成一个“更快的 pip”。
