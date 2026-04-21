# NotebookLM 技能故障排除指南

## 快速排查表

| 错误 | 解决方案 |
|------|----------|
| ModuleNotFoundError | 使用 `python scripts/run.py [脚本].py` |
| 认证失败 | 设置时浏览器必须可见 |
| 浏览器崩溃 | `python scripts/run.py cleanup_manager.py --preserve-library` |
| 触发速率限制 | 等待 1 小时或切换账号 |
| 找不到笔记本 | `python scripts/run.py notebook_manager.py list` |
| 脚本无法运行 | 始终使用 run.py 包装器 |

## 关键：始终使用 run.py

大多数问题都可以通过使用 run.py 包装器解决：

```bash
# ✅ 正确 — 始终这样做：
python scripts/run.py auth_manager.py status
python scripts/run.py ask_question.py --question "..."

# ❌ 错误 — 绝对不要这样：
python scripts/auth_manager.py status  # 会报 ModuleNotFoundError！
```

## 常见问题与解决方案

### 认证相关

#### 未认证错误
```
Error: Not authenticated. Please run auth setup first.
```

**解决方案：**
```bash
# 检查状态
python scripts/run.py auth_manager.py status

# 设置认证（浏览器必须可见！）
python scripts/run.py auth_manager.py setup
# 用户需要手动登录 Google 账号

# 如果设置失败，尝试重新认证
python scripts/run.py auth_manager.py reauth
```

#### 认证频繁过期
**解决方案：**
```bash
# 清除旧的认证信息
python scripts/run.py cleanup_manager.py --preserve-library

# 重新设置认证
python scripts/run.py auth_manager.py setup --timeout 15

# 使用持久化浏览器配置
export PERSIST_AUTH=true
```

#### Google 阻止自动化登录
**解决方案：**
1. 使用专用 Google 账号进行自动化操作
2. 如果可用，启用"不够安全的应用的访问权限"
3. 始终使用可见浏览器：
```bash
python scripts/run.py auth_manager.py setup
# 浏览器必须可见 — 用户手动登录
# 没有 headless 参数 — 调试时使用 --show-browser
```

### 浏览器相关

#### 浏览器崩溃或卡住
```
TimeoutError: Waiting for selector failed
```

**解决方案：**
```bash
# 结束卡住的进程
pkill -f chromium
pkill -f chrome

# 清理浏览器状态
python scripts/run.py cleanup_manager.py --confirm --preserve-library

# 重新认证
python scripts/run.py auth_manager.py reauth
```

#### 找不到浏览器
**解决方案：**
```bash
# 通过 run.py 安装 Chromium（自动）
python scripts/run.py auth_manager.py status
# run.py 会自动安装 Chromium

# 如需手动安装
cd ~/.claude/skills/notebooklm
source .venv/bin/activate
python -m patchright install chromium
```

### 速率限制

#### 超出速率限制（每天 50 次查询）
**解决方案：**

**方案一：等待**
```bash
# 查看限制重置时间（通常是太平洋时间午夜）
date -d "tomorrow 00:00 PST"
```

**方案二：切换账号**
```bash
# 清除当前认证
python scripts/run.py auth_manager.py clear

# 使用其他账号登录
python scripts/run.py auth_manager.py setup
```

**方案三：轮换账号**
```python
# 使用多个账号
accounts = ["account1", "account2"]
for account in accounts:
    # 触发限制时切换账号
    subprocess.run(["python", "scripts/run.py", "auth_manager.py", "reauth"])
```

### 笔记本访问问题

#### 找不到笔记本
**解决方案：**
```bash
# 列出所有笔记本
python scripts/run.py notebook_manager.py list

# 搜索笔记本
python scripts/run.py notebook_manager.py search --query "关键词"

# 如果不在库中则添加
python scripts/run.py notebook_manager.py add \
  --url "https://notebooklm.google.com/..." \
  --name "名称" \
  --topics "主题"
```

#### 拒绝访问
**解决方案：**
1. 确认笔记本是否仍处于公开共享状态
2. 使用更新后的链接重新添加笔记本
3. 确认使用了正确的 Google 账号

#### 使用了错误的笔记本
**解决方案：**
```bash
# 检查活跃笔记本
python scripts/run.py notebook_manager.py list | grep "active"

# 激活正确的笔记本
python scripts/run.py notebook_manager.py activate --id correct-id
```

### 虚拟环境问题

#### ModuleNotFoundError
```
ModuleNotFoundError: No module named 'patchright'
```

**解决方案：**
```bash
# 始终使用 run.py — 它会自动处理虚拟环境！
python scripts/run.py [任意脚本].py

# run.py 会自动：
# 1. 创建 .venv（如不存在）
# 2. 安装依赖
# 3. 运行脚本
```

#### Python 版本不对
**解决方案：**
```bash
# 检查 Python 版本（需要 3.8+）
python --version

# 如版本不对，指定正确的 Python
python3.8 scripts/run.py auth_manager.py status
```

### 网络问题

#### 连接超时
**解决方案：**
```bash
# 增加超时时间
export TIMEOUT_SECONDS=60

# 检查网络连通性
ping notebooklm.google.com

# 如需使用代理
export HTTP_PROXY=http://proxy:port
export HTTPS_PROXY=http://proxy:port
```

### 数据问题

#### 笔记本库损坏
```
JSON decode error when listing notebooks
```

**解决方案：**
```bash
# 备份当前库
cp ~/.claude/skills/notebooklm/data/library.json library.backup.json

# 重置库
rm ~/.claude/skills/notebooklm/data/library.json

# 重新添加笔记本
python scripts/run.py notebook_manager.py add --url ... --name ...
```

#### 磁盘空间不足
**解决方案：**
```bash
# 查看磁盘使用情况
df -h ~/.claude/skills/notebooklm/data/

# 执行清理
python scripts/run.py cleanup_manager.py --confirm --preserve-library
```

## 调试技巧

### 启用详细日志
```bash
export DEBUG=true
export LOG_LEVEL=DEBUG
python scripts/run.py ask_question.py --question "测试" --show-browser
```

### 逐个测试组件
```bash
# 测试认证
python scripts/run.py auth_manager.py status

# 测试笔记本访问
python scripts/run.py notebook_manager.py list

# 测试浏览器启动
python scripts/run.py ask_question.py --question "测试" --show-browser
```

### 出错时保存截图
在脚本中添加调试代码：
```python
try:
    # 你的代码
except Exception as e:
    page.screenshot(path=f"error_{timestamp}.png")
    raise e
```

## 恢复流程

### 完全重置
```bash
#!/bin/bash
# 结束进程
pkill -f chromium

# 备份笔记本库（如存在）
if [ -f ~/.claude/skills/notebooklm/data/library.json ]; then
    cp ~/.claude/skills/notebooklm/data/library.json ~/library.backup.json
fi

# 清除所有数据
cd ~/.claude/skills/notebooklm
python scripts/run.py cleanup_manager.py --confirm --force

# 删除虚拟环境
rm -rf .venv

# 重新安装（run.py 会自动处理）
python scripts/run.py auth_manager.py setup

# 恢复笔记本库（如有备份）
if [ -f ~/library.backup.json ]; then
    mkdir -p ~/.claude/skills/notebooklm/data/
    cp ~/library.backup.json ~/.claude/skills/notebooklm/data/library.json
fi
```

### 部分恢复（保留数据）
```bash
# 保留认证和库，仅修复运行环境
cd ~/.claude/skills/notebooklm
rm -rf .venv

# run.py 会自动重建虚拟环境
python scripts/run.py auth_manager.py status
```

## 错误信息速查

### 认证错误
| 错误 | 原因 | 解决方案 |
|------|------|----------|
| Not authenticated | 无有效认证 | `run.py auth_manager.py setup` |
| Authentication expired | 会话过期 | `run.py auth_manager.py reauth` |
| Invalid credentials | 账号不对 | 检查 Google 账号 |
| 2FA required | 安全验证 | 在可见浏览器中完成验证 |

### 浏览器错误
| 错误 | 原因 | 解决方案 |
|------|------|----------|
| Browser not found | Chromium 未安装 | 使用 run.py（自动安装） |
| Connection refused | 浏览器已崩溃 | 结束进程后重启 |
| Timeout waiting | 页面加载慢 | 增加超时时间 |
| Context closed | 浏览器被终止 | 检查日志排查崩溃原因 |

### 笔记本错误
| 错误 | 原因 | 解决方案 |
|------|------|----------|
| Notebook not found | ID 无效 | `run.py notebook_manager.py list` |
| Access denied | 未共享 | 在 NotebookLM 中重新共享 |
| Invalid URL | 格式不对 | 使用完整的 NotebookLM 链接 |
| No active notebook | 未选择 | `run.py notebook_manager.py activate` |

## 预防建议

1. **始终使用 run.py** — 可预防 90% 的问题
2. **定期维护** — 每周清理浏览器状态
3. **关注查询次数** — 跟踪每日用量以免触发限制
4. **备份笔记本库** — 定期导出笔记本列表
5. **使用专用账号** — 单独的 Google 账号用于自动化

## 获取帮助

### 收集诊断信息
```bash
# 系统信息
python --version
cd ~/.claude/skills/notebooklm
ls -la

# 技能状态
python scripts/run.py auth_manager.py status
python scripts/run.py notebook_manager.py list | head -5

# 检查数据目录
ls -la ~/.claude/skills/notebooklm/data/
```

### 常见问答

**问：为什么在 Claude 网页版不能用？**
答：网页版没有本地网络访问权限，需要使用本地 Claude Code。

**问：可以使用多个 Google 账号吗？**
答：可以，使用 `run.py auth_manager.py reauth` 切换账号。

**问：如何提高速率限制？**
答：使用多个账号或升级到 Google Workspace。

**问：对我的 Google 账号安全吗？**
答：建议使用专用账号进行自动化操作，该技能仅访问 NotebookLM。
