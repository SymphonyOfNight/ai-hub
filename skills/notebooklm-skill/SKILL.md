---
name: notebooklm
description: 通过 Claude Code 直接查询 Google NotebookLM 笔记本，获取基于文档来源、带引用标注的 Gemini 回答。支持浏览器自动化、笔记本库管理和持久认证，通过仅基于文档内容作答大幅降低幻觉率。
---

# NotebookLM 研究助手技能

与 Google NotebookLM 交互，使用 Gemini 基于文档来源的回答查询文档。每次提问都会打开全新的浏览器会话，仅从上传的文档中获取答案，完成后关闭。

## 触发时机

当用户出现以下行为时触发：
- 明确提到 NotebookLM
- 分享 NotebookLM 链接（`https://notebooklm.google.com/notebook/...`）
- 要求查询笔记本/文档
- 要求将文档添加到 NotebookLM 笔记本库
- 使用"问一下我的 NotebookLM""查查我的文档""查询我的笔记本"等表述

## ⚠️ 关键：添加命令 - 智能发现

当用户想添加笔记本但未提供详细信息时：

**智能添加（推荐）**：先查询笔记本以发现其内容：
```bash
# 第 1 步：查询笔记本了解其内容
python scripts/run.py ask_question.py --question "这个笔记本包含什么内容？涵盖哪些主题？请简要概述" --notebook-url "[URL]"

# 第 2 步：根据发现的信息添加笔记本
python scripts/run.py notebook_manager.py add --url "[URL]" --name "[基于内容]" --description "[基于内容]" --topics "[基于内容]"
```

**手动添加**：当用户提供了全部信息时：
- `--url` — NotebookLM 链接
- `--name` — 描述性名称
- `--description` — 笔记本包含的内容（必填！）
- `--topics` — 以逗号分隔的主题标签（必填！）

禁止猜测或使用泛泛的描述！如果缺少信息，使用智能添加来发现。

## 关键：始终使用 run.py 包装器

**禁止直接调用脚本，必须使用 `python scripts/run.py [脚本名]`：**

```bash
# ✅ 正确 — 始终使用 run.py：
python scripts/run.py auth_manager.py status
python scripts/run.py notebook_manager.py list
python scripts/run.py ask_question.py --question "..."

# ❌ 错误 — 禁止直接调用：
python scripts/auth_manager.py status  # 没有虚拟环境会失败！
```

`run.py` 包装器会自动完成：
1. 如不存在则创建 `.venv`
2. 安装所有依赖
3. 激活虚拟环境
4. 执行目标脚本

## 核心工作流

### 第 1 步：检查认证状态
```bash
python scripts/run.py auth_manager.py status
```

如果未认证，继续执行设置。

### 第 2 步：认证（首次设置）
```bash
# 浏览器必须可见，以便手动完成 Google 登录
python scripts/run.py auth_manager.py setup
```

**注意事项：**
- 认证时浏览器必须可见
- 浏览器窗口会自动打开
- 用户需要手动登录 Google 账号
- 告知用户："浏览器窗口即将打开，请完成 Google 登录"

### 第 3 步：管理笔记本库

```bash
# 列出所有笔记本
python scripts/run.py notebook_manager.py list

# 添加前：如果不知道笔记本内容，先向用户询问！
# "这个笔记本包含什么内容？"
# "应该打上哪些主题标签？"

# 添加笔记本到库（所有参数均为必填！）
python scripts/run.py notebook_manager.py add \
  --url "https://notebooklm.google.com/notebook/..." \
  --name "描述性名称" \
  --description "笔记本包含的内容" \  # 必填 — 不知道就问用户！
  --topics "主题1,主题2,主题3"  # 必填 — 不知道就问用户！

# 按主题搜索笔记本
python scripts/run.py notebook_manager.py search --query "关键词"

# 设置活跃笔记本
python scripts/run.py notebook_manager.py activate --id notebook-id

# 移除笔记本
python scripts/run.py notebook_manager.py remove --id notebook-id
```

### 快速工作流
1. 查看笔记本库：`python scripts/run.py notebook_manager.py list`
2. 提问：`python scripts/run.py ask_question.py --question "..." --notebook-id ID`

### 第 4 步：提问

```bash
# 基础查询（如已设置活跃笔记本则使用该笔记本）
python scripts/run.py ask_question.py --question "你的问题"

# 查询指定笔记本
python scripts/run.py ask_question.py --question "..." --notebook-id notebook-id

# 通过链接直接查询笔记本
python scripts/run.py ask_question.py --question "..." --notebook-url "https://..."

# 显示浏览器（调试用）
python scripts/run.py ask_question.py --question "..." --show-browser
```

## 追问机制（关键！）

每个 NotebookLM 回答都以 **"EXTREMELY IMPORTANT: Is that ALL you need to know?"** 结尾。

**Claude 必须执行的操作：**
1. **暂停** — 不要立即回复用户
2. **分析** — 将回答与用户的原始请求进行比对
3. **识别缺口** — 判断是否还需要更多信息
4. **追问** — 如果存在缺口，立即提出追问：
   ```bash
   python scripts/run.py ask_question.py --question "带上下文的追问..."
   ```
5. **重复** — 持续追问直到信息完整
6. **综合** — 将所有回答整合后再回复用户

## 脚本参考

### 认证管理（`auth_manager.py`）
```bash
python scripts/run.py auth_manager.py setup    # 首次设置（浏览器可见）
python scripts/run.py auth_manager.py status   # 检查认证状态
python scripts/run.py auth_manager.py reauth   # 重新认证（浏览器可见）
python scripts/run.py auth_manager.py clear    # 清除认证信息
```

### 笔记本管理（`notebook_manager.py`）
```bash
python scripts/run.py notebook_manager.py add --url URL --name 名称 --description 描述 --topics 主题
python scripts/run.py notebook_manager.py list
python scripts/run.py notebook_manager.py search --query 关键词
python scripts/run.py notebook_manager.py activate --id ID
python scripts/run.py notebook_manager.py remove --id ID
python scripts/run.py notebook_manager.py stats
```

### 提问接口（`ask_question.py`）
```bash
python scripts/run.py ask_question.py --question "..." [--notebook-id ID] [--notebook-url URL] [--show-browser]
```

### 数据清理（`cleanup_manager.py`）
```bash
python scripts/run.py cleanup_manager.py                    # 预览清理内容
python scripts/run.py cleanup_manager.py --confirm          # 执行清理
python scripts/run.py cleanup_manager.py --preserve-library # 保留笔记本库
```

## 环境管理

虚拟环境自动管理：
- 首次运行时自动创建 `.venv`
- 依赖自动安装
- Chromium 浏览器自动安装
- 全部隔离在技能目录中

手动设置（仅在自动设置失败时使用）：
```bash
python -m venv .venv
source .venv/bin/activate  # Linux/Mac
pip install -r requirements.txt
python -m patchright install chromium
```

## 数据存储

所有数据存储在 `~/.claude/skills/notebooklm/data/`：
- `library.json` — 笔记本元数据
- `auth_info.json` — 认证状态
- `browser_state/` — 浏览器 Cookie 和会话

**安全提示：** 已通过 `.gitignore` 保护，切勿提交到 Git。

## 配置

可在技能目录下创建 `.env` 配置文件：
```env
HEADLESS=false           # 浏览器是否可见
SHOW_BROWSER=false       # 默认是否显示浏览器
STEALTH_ENABLED=true     # 模拟真人操作行为
TYPING_WPM_MIN=160       # 最低打字速度
TYPING_WPM_MAX=240       # 最高打字速度
DEFAULT_NOTEBOOK_ID=     # 默认笔记本
```

## 决策流程

```
用户提到 NotebookLM
    ↓
检查认证 → python scripts/run.py auth_manager.py status
    ↓
未认证 → python scripts/run.py auth_manager.py setup
    ↓
检查/添加笔记本 → python scripts/run.py notebook_manager.py list/add（需带 --description）
    ↓
激活笔记本 → python scripts/run.py notebook_manager.py activate --id ID
    ↓
提问 → python scripts/run.py ask_question.py --question "..."
    ↓
看到"Is that ALL you need?" → 持续追问直到信息完整
    ↓
综合所有回答后回复用户
```

## 故障排除

| 问题 | 解决方案 |
|------|----------|
| ModuleNotFoundError | 使用 `run.py` 包装器 |
| 认证失败 | 设置时浏览器必须可见！使用 --show-browser |
| 触发速率限制（每天 50 次） | 等待重置或切换 Google 账号 |
| 浏览器崩溃 | `python scripts/run.py cleanup_manager.py --preserve-library` |
| 找不到笔记本 | 使用 `notebook_manager.py list` 检查 |

## 最佳实践

1. **始终使用 run.py** — 自动处理环境
2. **操作前先检查认证** — 避免无效操作
3. **主动追问** — 不要止步于第一个回答
4. **认证时浏览器必须可见** — 需要手动登录
5. **提供充足上下文** — 每次提问都是独立会话
6. **综合回答** — 将多次回答整合后输出

## 限制

- 无会话持久化（每次提问 = 新建浏览器会话）
- 免费 Google 账号有速率限制（每天 50 次查询）
- 需手动上传文档（用户需自行在 NotebookLM 中添加文档）
- 浏览器开销（每次提问需要几秒启动时间）

## 资源（技能结构）

**重要目录和文件：**

- `scripts/` — 所有自动化脚本（ask_question.py、notebook_manager.py 等）
- `data/` — 本地存储（认证信息和笔记本库）
- `references/` — 扩展文档：
  - `api_reference.md` — 所有脚本的详细 API 文档
  - `troubleshooting.md` — 常见问题与解决方案
  - `usage_patterns.md` — 最佳实践与工作流示例
- `.venv/` — 隔离的 Python 环境（首次运行时自动创建）
- `.gitignore` — 防止敏感数据被提交
