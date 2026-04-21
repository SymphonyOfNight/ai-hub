# NotebookLM 技能使用模式

高效使用 NotebookLM 技能的进阶模式。

## 关键：始终使用 run.py

**所有命令必须通过 run.py 包装器执行：**
```bash
# ✅ 正确：
python scripts/run.py auth_manager.py status
python scripts/run.py ask_question.py --question "..."

# ❌ 错误：
python scripts/auth_manager.py status  # 会失败！
```

## 模式 1：首次设置

```bash
# 1. 检查认证（使用 run.py！）
python scripts/run.py auth_manager.py status

# 2. 如果未认证，执行设置（浏览器必须可见！）
python scripts/run.py auth_manager.py setup
# 告知用户："请在浏览器窗口中登录 Google 账号"

# 3. 添加第一个笔记本 — 先向用户确认信息！
# 问："这个笔记本包含什么内容？"
# 问："应该打上哪些主题标签？"
python scripts/run.py notebook_manager.py add \
  --url "https://notebooklm.google.com/notebook/..." \
  --name "用户提供的名称" \
  --description "用户提供的描述" \  # 禁止猜测！
  --topics "用户,提供,的标签"  # 禁止猜测！
```

**关键注意事项：**
- 虚拟环境由 run.py 自动创建
- 认证时浏览器必须可见
- 必须通过查询发现内容或向用户询问笔记本元数据

## 模式 2：添加笔记本（智能发现！）

**当用户分享 NotebookLM 链接时：**

**方案 A：智能发现（推荐）**
```bash
# 1. 查询笔记本以发现其内容
python scripts/run.py ask_question.py \
  --question "这个笔记本包含什么内容？涵盖哪些主题？请简要概述" \
  --notebook-url "[URL]"

# 2. 根据发现的信息添加
python scripts/run.py notebook_manager.py add \
  --url "[URL]" \
  --name "[基于内容]" \
  --description "[基于发现]" \
  --topics "[提取的主题]"
```

**方案 B：询问用户（备选）**
```bash
# 如果发现失败，询问用户：
"这个笔记本包含什么内容？"
"涵盖哪些主题？"

# 然后用用户提供的信息添加：
python scripts/run.py notebook_manager.py add \
  --url "[URL]" \
  --name "[用户的回答]" \
  --description "[用户的描述]" \
  --topics "[用户的标签]"
```

**禁止：**
- 猜测笔记本的内容
- 使用泛泛的描述
- 跳过内容发现步骤

## 模式 3：日常研究工作流

```bash
# 查看笔记本库
python scripts/run.py notebook_manager.py list

# 用详细问题进行研究
python scripts/run.py ask_question.py \
  --question "带有完整上下文的详细问题" \
  --notebook-id notebook-id

# 看到"Is that ALL you need to know?"时进行追问
python scripts/run.py ask_question.py \
  --question "带上一轮回答上下文的追问"
```

## 模式 4：追问（关键！）

当 NotebookLM 回答以 "EXTREMELY IMPORTANT: Is that ALL you need to know?" 结尾时：

```python
# 1. 暂停 — 先不回复用户
# 2. 分析 — 回答是否完整？
# 3. 如有缺口，立即追问：
python scripts/run.py ask_question.py \
  --question "基于上一轮回答的具体追问"

# 4. 重复直到信息完整
# 5. 最后综合所有回答再回复用户
```

## 模式 5：多笔记本交叉研究

```python
# 查询不同笔记本以作对比
python scripts/run.py notebook_manager.py activate --id notebook-1
python scripts/run.py ask_question.py --question "问题"

python scripts/run.py notebook_manager.py activate --id notebook-2
python scripts/run.py ask_question.py --question "同一问题"

# 对比并综合回答
```

## 模式 6：错误恢复

```bash
# 认证失败时
python scripts/run.py auth_manager.py status
python scripts/run.py auth_manager.py reauth  # 浏览器可见！

# 浏览器崩溃时
python scripts/run.py cleanup_manager.py --preserve-library
python scripts/run.py auth_manager.py setup  # 浏览器可见！

# 触发速率限制时
# 等待或切换账号
python scripts/run.py auth_manager.py reauth  # 用其他账号登录
```

## 模式 7：批量处理

```bash
#!/bin/bash
NOTEBOOK_ID="notebook-id"
QUESTIONS=(
    "第一个详细问题"
    "第二个详细问题"
    "第三个详细问题"
)

for question in "${QUESTIONS[@]}"; do
    echo "正在提问：$question"
    python scripts/run.py ask_question.py \
        --question "$question" \
        --notebook-id "$NOTEBOOK_ID"
    sleep 2  # 避免触发速率限制
done
```

## 模式 8：自动化研究脚本

```python
#!/usr/bin/env python
import subprocess

def research_topic(topic, notebook_id):
    question = f"""
    请详细说明 {topic}：
    1. 核心概念
    2. 实现细节
    3. 最佳实践
    4. 常见陷阱
    5. 示例
    """

    result = subprocess.run([
        "python", "scripts/run.py", "ask_question.py",
        "--question", question,
        "--notebook-id", notebook_id
    ], capture_output=True, text=True)

    return result.stdout
```

## 模式 9：笔记本分类管理

```python
# 按领域分类 — 元数据必须准确
# 必须向用户确认描述信息！

# 后端笔记本
add_notebook("后端 API", "完整的 API 文档", "api,rest,后端")
add_notebook("数据库", "表结构和查询语句", "数据库,sql,后端")

# 前端笔记本
add_notebook("React 文档", "React 框架文档", "react,前端")
add_notebook("CSS 框架", "样式文档", "css,样式,前端")

# 按领域搜索
python scripts/run.py notebook_manager.py search --query "后端"
python scripts/run.py notebook_manager.py search --query "前端"
```

## 模式 10：与开发流程集成

```python
# 开发过程中查询文档
def check_api_usage(api_endpoint):
    result = subprocess.run([
        "python", "scripts/run.py", "ask_question.py",
        "--question", f"{api_endpoint} 的参数和返回格式",
        "--notebook-id", "api-docs"
    ], capture_output=True, text=True)

    # 判断是否需要追问
    if "Is that ALL you need" in result.stdout:
        follow_up = subprocess.run([
            "python", "scripts/run.py", "ask_question.py",
            "--question", f"给出 {api_endpoint} 的代码示例",
            "--notebook-id", "api-docs"
        ], capture_output=True, text=True)

    return combine_answers(result.stdout, follow_up.stdout)
```

## 最佳实践

### 1. 问题表述
- 问题要具体、完整
- 每次提问都包含充足上下文
- 要求结构化的回答
- 需要时请求示例

### 2. 笔记本管理
- **必须向用户确认元数据**
- 使用描述性名称
- 添加全面的主题标签
- 保持链接为最新

### 3. 性能优化
- 将相关问题批量处理
- 不同笔记本可并行查询
- 关注速率限制（每天 50 次）
- 必要时切换账号

### 4. 错误处理
- 始终使用 run.py 避免虚拟环境问题
- 操作前检查认证
- 实现重试逻辑
- 准备备用笔记本

### 5. 安全
- 使用专用 Google 账号
- 切勿提交 data/ 目录
- 定期刷新认证
- 记录所有访问

## Claude 常用工作流

### 工作流 1：用户发送 NotebookLM 链接

```python
# 1. 检测消息中的链接
if "notebooklm.google.com" in user_message:
    url = extract_url(user_message)

    # 2. 检查是否已在库中
    notebooks = run("notebook_manager.py list")

    if url not in notebooks:
        # 3. 向用户确认元数据（关键！）
        name = ask_user("这个笔记本叫什么名字？")
        description = ask_user("这个笔记本包含什么内容？")
        topics = ask_user("涵盖哪些主题？")

        # 4. 用用户提供的信息添加
        run(f"notebook_manager.py add --url {url} --name '{name}' --description '{description}' --topics '{topics}'")

    # 5. 使用笔记本
    answer = run(f"ask_question.py --question '{user_question}'")
```

### 工作流 2：研究任务

```python
# 1. 理解任务
task = "实现功能 X"

# 2. 准备全面的问题
questions = [
    "X 的完整实现指南",
    "X 的错误处理方案",
    "X 的性能优化建议"
]

# 3. 带追问的查询
for q in questions:
    answer = run(f"ask_question.py --question '{q}'")

    # 判断是否需要追问
    if "Is that ALL you need" in answer:
        follow_up = run(f"ask_question.py --question '关于 {q} 的具体细节'")

# 4. 综合信息后开始实现
```

## 实用技巧

1. **始终使用 run.py** — 避免所有虚拟环境问题
2. **确认元数据** — 禁止猜测笔记本内容
3. **问题要详细** — 包含所有上下文
4. **自动追问** — 看到提示就继续追问
5. **关注速率限制** — 每天 50 次查询
6. **批量操作** — 将相关问题归组
7. **保存重要回答** — 本地留存
8. **版本管理笔记本** — 跟踪变更
9. **定期测试认证** — 在重要任务前确认
10. **做好记录** — 维护笔记本清单

## 快速参考

```bash
# 始终使用 run.py！
python scripts/run.py [脚本].py [参数]

# 常用操作
run.py auth_manager.py status          # 检查认证
run.py auth_manager.py setup           # 登录（浏览器可见！）
run.py notebook_manager.py list        # 列出笔记本
run.py notebook_manager.py add ...     # 添加（先确认元数据！）
run.py ask_question.py --question ...  # 提问
run.py cleanup_manager.py ...          # 清理
```

**核心原则：** 拿不准的时候，用 run.py，向用户确认笔记本信息！
