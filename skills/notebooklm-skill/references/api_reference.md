# NotebookLM 技能 API 参考

所有 NotebookLM 技能模块的完整 API 文档。

## 重要：始终使用 run.py 包装器

**所有命令必须通过 `run.py` 包装器执行，以确保环境正确：**

```bash
# ✅ 正确：
python scripts/run.py [脚本名].py [参数]

# ❌ 错误：
python scripts/[脚本名].py [参数]  # 没有虚拟环境会失败！
```

## 核心脚本

### ask_question.py
通过自动化浏览器向 NotebookLM 提问。

```bash
# 基础用法
python scripts/run.py ask_question.py --question "你的问题"

# 指定笔记本
python scripts/run.py ask_question.py --question "..." --notebook-id notebook-id

# 直接使用链接
python scripts/run.py ask_question.py --question "..." --notebook-url "https://..."

# 显示浏览器（调试用）
python scripts/run.py ask_question.py --question "..." --show-browser
```

**参数：**
- `--question`（必填）：要提的问题
- `--notebook-id`：使用笔记本库中的笔记本
- `--notebook-url`：直接使用链接
- `--show-browser`：让浏览器可见

**返回值：** 回答文本，末尾附带追问提示

### notebook_manager.py
笔记本库的增删改查管理。

```bash
# 智能添加（先发现内容）
python scripts/run.py ask_question.py --question "这个笔记本包含什么内容？涵盖哪些主题？请简要概述" --notebook-url "[URL]"
# 然后用发现的信息添加
python scripts/run.py notebook_manager.py add \
  --url "https://notebooklm.google.com/notebook/..." \
  --name "名称" \
  --description "描述" \
  --topics "主题1,主题2"

# 直接添加（已知内容时）
python scripts/run.py notebook_manager.py add \
  --url "https://notebooklm.google.com/notebook/..." \
  --name "名称" \
  --description "包含的内容" \
  --topics "主题1,主题2"

# 列出笔记本
python scripts/run.py notebook_manager.py list

# 搜索笔记本
python scripts/run.py notebook_manager.py search --query "关键词"

# 激活笔记本
python scripts/run.py notebook_manager.py activate --id notebook-id

# 移除笔记本
python scripts/run.py notebook_manager.py remove --id notebook-id

# 查看统计信息
python scripts/run.py notebook_manager.py stats
```

**子命令：**
- `add`：添加笔记本（需要 --url、--name、--topics）
- `list`：显示所有笔记本
- `search`：按关键词查找笔记本
- `activate`：设置默认笔记本
- `remove`：从库中删除
- `stats`：显示库统计信息

### auth_manager.py
处理 Google 认证和浏览器状态。

```bash
# 设置（浏览器可见以便登录）
python scripts/run.py auth_manager.py setup

# 检查状态
python scripts/run.py auth_manager.py status

# 重新认证
python scripts/run.py auth_manager.py reauth

# 清除认证
python scripts/run.py auth_manager.py clear
```

**子命令：**
- `setup`：首次认证（浏览器必须可见）
- `status`：检查是否已认证
- `reauth`：清除并重新设置
- `clear`：移除所有认证数据

### cleanup_manager.py
带保留选项的数据清理。

```bash
# 预览清理内容
python scripts/run.py cleanup_manager.py

# 执行清理
python scripts/run.py cleanup_manager.py --confirm

# 保留笔记本库
python scripts/run.py cleanup_manager.py --confirm --preserve-library

# 强制执行（跳过确认）
python scripts/run.py cleanup_manager.py --confirm --force
```

**选项：**
- `--confirm`：实际执行清理
- `--preserve-library`：保留笔记本库
- `--force`：跳过确认提示

### run.py
自动处理环境设置的脚本包装器。

```bash
# 用法
python scripts/run.py [脚本名].py [参数]

# 示例
python scripts/run.py auth_manager.py status
python scripts/run.py ask_question.py --question "..."
```

**自动执行的操作：**
1. 如 `.venv` 不存在则创建
2. 安装依赖
3. 激活虚拟环境
4. 执行目标脚本

## Python API 用法

### 通过 subprocess 调用 run.py

```python
import subprocess
import json

# 始终使用 run.py 包装器
result = subprocess.run([
    "python", "scripts/run.py", "ask_question.py",
    "--question", "你的问题",
    "--notebook-id", "notebook-id"
], capture_output=True, text=True)

answer = result.stdout
```

### 直接导入（虚拟环境已存在时）

```python
# 仅在虚拟环境已创建并激活时有效
from notebook_manager import NotebookLibrary
from auth_manager import AuthManager

library = NotebookLibrary()
notebooks = library.list_notebooks()

auth = AuthManager()
is_auth = auth.is_authenticated()
```

## 数据存储

位置：`~/.claude/skills/notebooklm/data/`

```
data/
├── library.json       # 笔记本元数据
├── auth_info.json     # 认证状态
└── browser_state/     # 浏览器 Cookie
    └── state.json
```

**安全提示：** 已通过 `.gitignore` 保护，切勿提交。

## 环境变量

可选的 `.env` 文件配置：

```env
HEADLESS=false           # 浏览器是否可见
SHOW_BROWSER=false       # 默认是否显示
STEALTH_ENABLED=true     # 模拟真人行为
TYPING_WPM_MIN=160       # 打字速度范围
TYPING_WPM_MAX=240
DEFAULT_NOTEBOOK_ID=     # 默认笔记本
```

## 错误处理

常见模式：

```python
# 使用 run.py 可以避免大多数错误
result = subprocess.run([
    "python", "scripts/run.py", "ask_question.py",
    "--question", "问题"
], capture_output=True, text=True)

if result.returncode != 0:
    error = result.stderr
    if "rate limit" in error.lower():
        # 等待或切换账号
        pass
    elif "not authenticated" in error.lower():
        # 执行认证设置
        subprocess.run(["python", "scripts/run.py", "auth_manager.py", "setup"])
```

## 速率限制

免费 Google 账号：每天 50 次查询

解决方案：
1. 等待重置（太平洋时间午夜）
2. 使用 `reauth` 切换账号
3. 使用多个 Google 账号

## 进阶用法

### 并行查询

```python
import concurrent.futures
import subprocess

def query(question, notebook_id):
    result = subprocess.run([
        "python", "scripts/run.py", "ask_question.py",
        "--question", question,
        "--notebook-id", notebook_id
    ], capture_output=True, text=True)
    return result.stdout

# 同时执行多个查询
with concurrent.futures.ThreadPoolExecutor(max_workers=3) as executor:
    futures = [
        executor.submit(query, q, nb)
        for q, nb in zip(questions, notebooks)
    ]
    results = [f.result() for f in futures]
```

### 批量处理

```python
def batch_research(questions, notebook_id):
    results = []
    for question in questions:
        result = subprocess.run([
            "python", "scripts/run.py", "ask_question.py",
            "--question", question,
            "--notebook-id", notebook_id
        ], capture_output=True, text=True)
        results.append(result.stdout)
        time.sleep(2)  # 避免触发速率限制
    return results
```

## 模块类

### NotebookLibrary
- `add_notebook(url, name, topics)`
- `list_notebooks()`
- `search_notebooks(query)`
- `get_notebook(notebook_id)`
- `activate_notebook(notebook_id)`
- `remove_notebook(notebook_id)`

### AuthManager
- `is_authenticated()`
- `setup_auth(headless=False)`
- `get_auth_info()`
- `clear_auth()`
- `validate_auth()`

### BrowserSession（内部使用）
- 处理浏览器自动化
- 管理模拟真人行为
- 不建议直接调用

## 最佳实践

1. **始终使用 run.py** — 确保环境正确
2. **操作前先检查认证** — 避免无效操作
3. **处理速率限制** — 实现重试逻辑
4. **提供充足上下文** — 每次提问都是独立的
5. **定期清理会话** — 使用 cleanup_manager
