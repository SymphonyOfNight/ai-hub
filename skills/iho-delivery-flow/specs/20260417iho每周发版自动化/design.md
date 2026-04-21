# 设计:iho 每周发版流程自动化(iho-delivery-flow)

- **创建日期**:2026-04-17
- **作者**:wangjie
- **关联文档**:`./requirements.md`
- **实现路径**:`/Users/wangjie/ai-hub/skills/iho-delivery-flow/`
- **实现方案**:方案 D(轻量 Playwright 脚本 + 锚点清单)
- **交付记录**:本 skill 为 skill 开发,**豁免** `/doc/feat/feat_xxxx.md` 记录要求(详见 §12)

---

## 1. 架构总览

### 1.1 三层分工

| 层 | 内容 | Claude 运行期是否加载 |
|---|---|---|
| **SKILL.md(顶层 + 9 个子)** | 触发词、步骤清单、Bash 调用样板 | ✅ 每次激活都读 |
| **references/anchors.json** | 页面锚点清单(10-20 行/页) | ⚠️ 仅在新增/修改锚点时读 |
| **.runtime/scripts/*.mjs** | Playwright 操作代码 | ❌ 运行期完全不读 |
| **.runtime/lib/*.mjs** | 共享工具(browser、plan、state、anchors) | ❌ 同上 |
| **collect-archie/<日期_版本>/** | 运行期产出(state、plans、results、logs、下载文件) | ⚠️ 只读 state.json、results/*.json |

**核心设计意图**:Claude 只做**对话 + 计划 + 状态**,Playwright 脚本做**浏览器执行**,选择器和 DOM 永远不进对话上下文。token 消耗比纯 MCP 方案低 15 倍量级。

### 1.2 运行时数据流

```
用户 → 触发词(如 "发版-submit-sql") → Claude 激活子 skill
  │
  ├─ 前置阶段(Claude)
  │    1. Bash: state check-deps <step>
  │    2. Bash: node lib/plan.mjs --build --step <step>  (脚本读 feat+git,生成 plan JSON)
  │    3. Claude 读 plan JSON,打印摘要给用户
  │    4. 等用户确认 / 微调(改 JSON 文件或对话里改条目)
  │
  ├─ 执行阶段(脚本)
  │    5. Bash: node scripts/<step>.mjs --plan ... --out ... --state ... --config ...
  │       a. 启动 chromium + 加载 storageState
  │       b. 探测登录态,必要时 loginFlow
  │       c. 按 plan 循环执行条目,写 result
  │       d. 失败时落 trace;浏览器关闭;进程退出(exitCode 0/1/2)
  │
  └─ 后置阶段(Claude)
       6. 读 result JSON,打印人类可读摘要
       7. 等用户确认 / 手工标记 / 重跑 / 中止
       8. Bash: state set <step>.status ...
```

---

## 2. 目录结构

```
/Users/wangjie/ai-hub/skills/iho-delivery-flow/
├── SKILL.md                          顶层编排 skill
├── README.md                          使用说明(用户视角)
├── config.yaml                        全局配置(URL、锚点探针、路径)
├── specs/
│   └── 20260417iho每周发版自动化/
│       ├── requirements.md
│       ├── design.md                  本文档
│       └── tasks.md                   由 writing-plans 后续生成
│
├── .runtime/                          Node 项目(用户视角隐藏)
│   ├── package.json
│   ├── tsconfig.json                  仅做类型提示,不编译
│   ├── node_modules/                  (gitignore)
│   ├── lib/
│   │   ├── browser.mjs                launch + storageState + trace 封装
│   │   ├── anchors.mjs                锚点 JSON → Playwright locator 翻译
│   │   ├── state.mjs                  state.json 读写 + 原子替换
│   │   ├── plan.mjs                   plan/result JSON 读写 + builder 分发
│   │   ├── logger.mjs                 统一日志
│   │   └── plan-builders/             各 step 的计划生成器
│   │       ├── create-version.mjs
│   │       ├── submit-sql.mjs
│   │       └── ...
│   └── scripts/
│       ├── setup.sh                   首次安装脚本
│       ├── selfcheck.mjs              冷启动自检
│       ├── state-cli.mjs              state init/get/set/mark/reset/summary/list/check-deps
│       ├── run-step.mjs               子 skill 入口路由
│       ├── login-main.mjs             手动登录(保底)
│       ├── login-esb.mjs
│       ├── create-version.mjs
│       ├── create-patch.mjs
│       ├── download-esb.mjs
│       ├── submit-sql.mjs
│       ├── sync-params.mjs
│       ├── sync-resources.mjs
│       ├── download-resources.mjs
│       ├── collect-archie.mjs         纯本地,不开浏览器
│       ├── finalize-release.mjs
│       └── jenkins-trigger.mjs        复用 jenkins-flow 的 curl 逻辑
│
├── create-version/
│   ├── SKILL.md
│   └── references/
│       └── anchors.json
├── create-patch/         ...同结构
├── download-esb/         ...
├── submit-sql/           ...
├── sync-params/          ...
├── sync-resources/       ...
├── download-resources/   ...
├── finalize-release/     ...
│
├── collect-archie/
│   ├── SKILL.md                       汇总 skill
│   ├── references/
│   │   └── summary-template.md
│   └── 20260417_4.4.3/                运行期产出(gitignore)
│       ├── state.json
│       ├── summary.md
│       ├── plans/*.json
│       ├── results/*.json
│       ├── logs/*.log + *.trace.zip
│       ├── esb/
│       └── resources/
│
└── .gitignore
```

---

## 3. 顶层 SKILL.md 契约

### 3.1 Frontmatter

```yaml
---
name: iho-delivery-flow
description: |
  用户说 "发版-启动 <版本号>" 时启动每周发版流程(版本号 X.Y.Z 为大版本,X.Y.Z.N 为补丁,自动识别);
  说 "发版-<子步骤>" 触发对应子 skill;
  说 "发版-状态" / "发版-标记" / "发版-重置" / "发版-清单" / "发版-切换" / "发版-重登" 操作状态。
  启动后不会自动串联所有子 skill,而是初始化 state 并打印子步骤触发词清单,由用户按节奏触发。
---
```

### 3.2 触发词清单

| 触发词 | 作用 |
|---|---|
| `发版-启动 4.4.3` | 启动大版本流程,初始化 state(通过版本号段数识别:3 段=版本,4 段=补丁) |
| `发版-启动 4.4.3.1` | 启动补丁流程 |
| `发版-清单` | 打印当前发版的 9 步触发词 + 依赖图 |
| `发版-状态` | 打印 state.json 的人类可读摘要 |
| `发版-标记 <step> <状态>` | 手工覆盖某步状态,如 `发版-标记 submit-sql done` |
| `发版-重置 <step>` | 清空某步状态以便重跑 |
| `发版-重置 <step> --item <id>` | 清单步中单条目,精细重跑 |
| `发版-切换 <日期_版本>` | 切到另一个已存在的发版目录(多版本并存) |
| `发版-重登 main / 发版-重登 esb` | 强制重走登录流程 |
| `发版-编辑计划 <step>` | 打开 `plans/<step>.json` 让用户编辑(长计划微调用) |
| `发版-<子步骤>` | 触发对应子 skill,如 `发版-submit-sql` |

### 3.3 启动流程

```
1. 校验版本号格式(X.Y.Z 或 X.Y.Z.N)
2. 推断 mode:3 段 → version;4 段 → patch
3. artifactsDir = collect-archie/<yyyyMMdd>_<版本号>/
4. 存在则询问"续跑 / 重置 / 取消"(默认续跑)
5. 不存在则:
   a. 创建目录结构(plans/ results/ logs/ esb/ resources/)
   b. state init 生成初始 state.json
6. 打印依赖图和子步骤触发词清单
```

### 3.4 与子 skill 的关系

顶层 **不直接调用** 子 skill。子 skill 是**用户主动说触发词**激活。顶层只:
- 初始化 / 切换发版上下文
- 路由状态类命令(state-cli)
- 打印清单让用户知道能触发什么

**子 skill 触发词前缀统一 `发版-`**,避免与日常对话冲撞。

---

## 4. 数据契约

所有契约文件均为 JSON(包括 plans 和 results),方便机器解析和原子操作。

### 4.1 state.json 完整 schema

```json
{
  "version": "4.4.3",
  "mode": "version",
  "artifactsDir": "/Users/wangjie/ai-hub/skills/iho-delivery-flow/collect-archie/20260417_4.4.3",
  "startedAt":  "2026-04-17T10:00:00+08:00",
  "updatedAt":  "2026-04-17T11:30:00+08:00",

  "context": {
    "releaseId":    "REL-8832",
    "releaseUrl":   "https://main-test.iho.com/release/detail/8832",
    "imageVersion": "4.4.3",
    "feedDocPath":  "/Users/wangjie/workspace/project/iho-mrms/doc/feat/feat_v4.4.3.md",
    "parentVersion": null
  },

  "steps": {
    "create-version": {
      "status":         "done",
      "startedAt":      "2026-04-17T10:01:00+08:00",
      "finishedAt":     "2026-04-17T10:05:00+08:00",
      "manuallyMarked": false,
      "planPath":       "plans/create-version.json",
      "resultPath":     "results/create-version.json",
      "tracePath":      null,
      "output": {
        "releaseId":  "REL-8832",
        "releaseUrl": "https://main-test.iho.com/release/detail/8832"
      },
      "note": ""
    },
    "download-esb":        { "status": "pending", ... },
    "submit-sql":          { "status": "pending", ... },
    "sync-params":         { "status": "pending", ... },
    "sync-resources":      { "status": "pending", ... },
    "download-resources":  { "status": "pending", ... },
    "collect-archie":      { "status": "pending", ... },
    "finalize-release":    { "status": "pending", ... },
    "jenkins":             { "status": "pending", ... }
  }
}
```

**原子写入**:state-cli 写时先写 `state.json.tmp` 再 `rename()`。
**状态取值**:`pending | in_progress | done | failed | skipped`。
**手工标记**:`manuallyMarked: true` 代表用户手工设 done/skipped,未实际执行。

### 4.2 plans/<step>.json 通用结构

```json
{
  "step": "submit-sql",
  "version": "4.4.3",
  "mode": "version",
  "generatedAt": "2026-04-17T10:30:00+08:00",
  "generatedFrom": [
    "doc/feat/feat_v4.4.3.md#SQL",
    "git diff --name-only master...HEAD -- '*.sql'"
  ],
  "items": [
    {
      "id": 1,
      "order": 1,
      "enabled": true,
      "uniqueKey": "调整 patient_info 索引",
      "type": "sql",
      "source": "feat_v4.4.3.md#第 3 条",
      "title": "调整 patient_info 索引",
      "file": "/abs/path/to/xxx.sql",
      "content": null,
      "lines": 42,
      "mode": "paste",
      "description": "为 idx_patient_phone 添加 where 条件",
      "note": ""
    }
  ],
  "options": {
    "dryRun": false,
    "stopOnFirstFail": true
  }
}
```

**关键字段**:
- `uniqueKey`:用于跨次运行去重(submit-sql 用 title,sync-params 用 参数 key,sync-resources 用 资源 id,download 类用文件名)
- `enabled`:用户可设 false 跳过单条
- `mode`(仅 submit-sql):`paste`(< 100 行) 或 `upload`(≥ 100 行),首次生成时自动判断
- `content`(可选):脚本启动时若为 null 则自 `file` 读入

### 4.3 results/<step>.json 通用结构

```json
{
  "step": "submit-sql",
  "version": "4.4.3",
  "startedAt": "2026-04-17T10:45:00+08:00",
  "finishedAt": "2026-04-17T10:48:12+08:00",
  "overall": "partial",
  "summary": { "total": 5, "ok": 4, "failed": 1, "skipped": 0 },
  "items": [
    { "id": 1, "status": "ok", "output": { "submittedId": "SQL-9901" } },
    { "id": 2, "status": "skipped" },
    { "id": 3, "status": "failed",
      "error": { "type": "AnchorMiss", "anchor": "btn_save",
                 "url": "https://.../sql/new",
                 "screenshot": "logs/miss-btn_save-....png" },
      "tracePath": "logs/submit-sql-20260417T104700.trace.zip" }
  ],
  "exitCode": 1
}
```

### 4.4 state-cli.mjs 命令集

```
state init        --version 4.4.3                初始化 state.json
state get         --version 4.4.3 --path steps.submit-sql.status
state set         --version 4.4.3 --path steps.submit-sql.status --value done
state mark        --version 4.4.3 --step submit-sql --status done    (会同时设 manuallyMarked:true)
state reset       --version 4.4.3 --step submit-sql [--item <id>]
state summary     --version 4.4.3                人类可读摘要
state list                                       列所有 collect-archie/<日期_版本>/
state check-deps  --version 4.4.3 --step download-resources
                                                 退出码 0=通过,非 0=未满足依赖
```

---

## 5. 子 skill 通用三阶段模板

### 5.1 三阶段流程

```
前置(生成并确认计划)
  P1. state check-deps <step> → 未满足则拒绝执行
  P2. node lib/plan.mjs --step <step> --build → 生成 plans/<step>.json
  P3. Claude 读 plan JSON,打印摘要
  P4. 用户答复:
       "OK" → 进入执行
       "删第 N 条 / 改第 N 条 / 加一条" → Claude Edit 文件(diff 显示),回 P3
       "改好了" → Claude 重读,回 P3
       "全部重新生成" → plan.mjs --rebuild --keep-user-edits,回 P3

执行(调脚本)
  E1. state set <step>.status in_progress
  E2. node scripts/<step>.mjs
        --plan plans/<step>.json
        --out  results/<step>.json
        --state state.json
        --config config.yaml

后置(汇报并确认)
  A1. Claude 读 results/<step>.json,按统一格式打印
  A2. 用户答复:
       "确认"              → state mark <step> done
       "手工标记第 N 为完成" → 改 result + state,不跑
       "重跑第 N 条"       → plan 只留失败项,回 E1
       "全部重跑"           → plan 全部 enabled:true,回 E1
       "中止并查看 trace"   → 打开 npx playwright show-trace <路径>
       "中止"              → state set <step>.status failed
```

### 5.2 子 skill SKILL.md 标准模板

```markdown
---
name: 发版-<step-name>
description: 在"发版"流程中,用户说"发版-<step-name>"时,按三阶段模板执行
---

# 发版-<step-name>

## 前置依赖
- <前置步骤列表,留空则无>

## 计划原料
- <feat 文档哪一节>
- <是否需要 git diff,过滤什么路径>
- <是否需要读其他步骤的 state.output>

## 执行器
- 脚本: `.runtime/scripts/<step-name>.mjs`
- 锚点: `<step-name>/references/anchors.json`
- 登录态: <main | esb | none>

## 典型微调场景
- <列出本步典型微调场景>

## 回滚指引
- <如何手工撤销本步,文字版步骤>

## 执行规范
遵循通用三阶段模板(见 iho-delivery-flow/SKILL.md §三阶段)。
```

### 5.3 微调处理细则

- **短微调**(改 enabled / note / order / 单条内容):Claude 用 Edit 工具直接改 plan JSON,以 diff 形式展示
- **长微调**(批量改、自己编辑):用户触发 `发版-编辑计划 <step>`,Claude 打开 `plans/<step>.json` 文件;用户改完说"改好了",Claude 重读并摘要
- **微调输出**:打印**差异**,不打印完整计划,避免刷屏

### 5.4 重跑策略

- 默认 `enabled:true && status:!done` 的条目跑
- 脚本启动时按 `uniqueKey` 查 `state.steps.<step>.output` 已完成集合,重复的条目自动 `status:ok, reused:true`
- 用户可 `发版-重置 <step> --item <id>` 清单条进行强制重跑

---

## 6. 9 个子 skill 规格卡

### 6.1 `发版-create-version`(大版本 · main 系统)

- **前置依赖**:无
- **计划原料**:version(取自 state.version)
- **核心动作**:进入"发版管理"菜单 → 点"新建版本"→ 填版本号/版本名/发版日期 → 保存 → 抓取 releaseId + releaseUrl
- **产出到 context**:`releaseId`、`releaseUrl`
- **典型微调**:版本名前缀、发版日期
- **回滚指引**:登录主业务 → 发版管理 → 找到刚创建的版本 → 点"删除"
- **锚点需求**:进入菜单链接 / 新建版本按钮 / 版本号输入 / 版本名输入 / 发版日期输入 / 保存按钮 / 详情页 URL 正则

### 6.2 `发版-create-patch`(补丁 · main 系统)

- 骨架与 6.1 相同;按钮名 `新建补丁`;版本号 4 段
- **额外**:脚本从 `4.4.4.1` 自动解析 `4.4.4` 作为"所属大版本",在表单中选择下拉
- **锚点额外需求**:所属大版本下拉选择器
- **额外写入 state.context**:`parentVersion = "4.4.4"`(供后续步骤知道挂在哪个大版本上)

### 6.3 `发版-download-esb`(esb 系统)

- **前置依赖**:create-version / create-patch
- **计划原料**:feat 文档 `ESB` 章节(如有) + 版本号
- **核心动作**:进入 ESB 指定菜单 → 按版本号/日期过滤 → 勾选文件列表 → 批量下载 → saveAs 到 `artifacts/<版本>/esb/`
- **产出**:`files: [...]`(相对路径列表)
- **回滚指引**:本地删除 `esb/` 目录
- **锚点需求**:菜单入口 / 过滤输入 / 查询按钮 / 列表行 / 勾选框 / 批量下载按钮

### 6.4 `发版-submit-sql`(main 系统)

- **前置依赖**:create-version / create-patch
- **计划原料**:feat 文档 `SQL` 章节 + `git diff master...HEAD -- '*.sql'` 定位本次新增 SQL 文件
- **核心动作**:进入 SQL 菜单 → 每条 SQL:新增 → 按 `mode` 字段决定 paste(粘贴内容)或 upload(上传 .sql 文件)→ 填标题/描述/关联 releaseId → 保存 → 抓单号
- **产出**:`submittedIds: [...]`,items 级 ok/failed
- **uniqueKey**:title(标题唯一)
- **回滚指引**:SQL 菜单 → 按单号搜索 → 撤回/删除
- **锚点需求**:菜单 / 新增按钮 / 粘贴模式文本域 / 上传模式文件 input(若是同页面两种,两个锚点都要;若切换 tab 还要切换按钮) / 标题输入 / 描述输入 / releaseId 关联字段 / 保存按钮 / 详情页单号定位

### 6.5 `发版-sync-params`(main 系统)

- **前置依赖**:create-version / create-patch
- **计划原料**:feat 文档 `参数` 章节(固定章节名)
- **核心动作**:进入参数菜单 → 每条参数:新增/修改 → 填 key/value/说明/关联 releaseId → 保存
- **产出**:参数列表及状态
- **uniqueKey**:参数 key
- **回滚指引**:参数菜单 → 反向操作
- **锚点需求**:菜单 / 查询输入(去重检查) / 新增按钮 / 字段输入 / 保存按钮

### 6.6 `发版-sync-resources`(main 系统)

- **前置依赖**:create-version / create-patch
- **计划原料**:feat 文档 `资源` 章节(固定章节名)
- **核心动作**:进入资源菜单 → 每条:新增/修改 → 填属性 → 保存
- **产出**:资源列表及状态
- **uniqueKey**:资源 id
- **回滚/锚点**:同 6.5

### 6.7 `发版-download-resources`(main 系统)

- **前置依赖**:**sync-params AND sync-resources** 均 done(state check-deps 校验)
- **计划原料**:releaseId
- **核心动作**:进入资源下载菜单 → 按 releaseId 过滤 → 批量下载 → saveAs 到 `artifacts/<版本>/resources/`
- **产出**:`files: [...]`
- **回滚指引**:本地删除 `resources/` 目录
- **锚点需求**:菜单 / 过滤输入 / 查询按钮 / 全选 / 下载按钮

### 6.8 `发版-collect-archie`(本地,无浏览器)

- **前置依赖**:download-esb + submit-sql + download-resources 全 done
- **计划原料**:state.json + feat 文档 + results/*.json
- **核心动作**(纯 Node,不启动 Playwright):
  1. 检查 `esb/` 和 `resources/` 目录非空且文件齐
  2. 读 feat 文档 + 各 results 汇总生成 `summary.md`(含 SQL 单号列表、参数清单、资源清单、下载文件清单)
  3. 产出清单写到 state.steps.collect-archie.output
- **产出**:`summaryPath`、`fileManifest`
- **回滚指引**:删除本地文件

### 6.9 `发版-finalize-release`(main 系统)

- **前置依赖**:collect-archie done
- **计划原料**:state.context.releaseUrl + summary.md + 各子目录文件
- **核心动作**:导航到 releaseUrl → 编辑 → 按 uploadSlots 配置逐个 slot 上传对应文件(每个文件单独上传,**不打 zip**) → 填发布说明(粘贴 summary.md) → 保存 → 确认状态变为"已提交"
- **uploadSlots 结构**:
```json
{
  "uploadSlots": [
    { "name": "esb", "sourceDir": "esb/", "button": {...}, "fileInput": {...}, "accept": ["xml","zip"], "successText": "上传成功" },
    { "name": "resources", "sourceDir": "resources/", ... }
  ],
  "descriptionField": { "role": "textbox", "name": "发布说明" },
  "saveButton":       { "role": "button",  "name": "提交" }
}
```
- **产出**:`finalStatus: submitted`
- **回滚指引**:发版单 → 撤回 → 删除附件

### 6.10 `发版-jenkins`(Jenkins,复用 jenkins-flow 的 curl 参数)

- **前置依赖**:finalize-release done
- **触发方式**:两种
  - (a) `finalize-release` 后置确认成功后,顶层 skill 打印"finalize 已成功,是否立即触发 Jenkins 构建 master 4.4.3?(Y/n)",默认 Y 自动触发
  - (b) 用户主动触发 `发版-jenkins`
- **核心动作**:`node .runtime/scripts/jenkins-trigger.mjs --project iho-mrms --version <ver>`,内部等价于 jenkins-flow 的 master 构建 curl
- **凭据**:读 `$JENKINS_TOKEN` 环境变量
- **产出**:`httpStatus: 201`
- **失败处理**:非 201 写 state.steps.jenkins.status=failed + error,不重试

---

## 7. 登录态管理

### 7.1 文件位置

```
~/.claude/state/iho-delivery/
├── main.json           (storageState)
├── main.meta.json      (最后登录时间等元数据)
├── esb.json
└── esb.meta.json
```

文件权限 `chmod 600`,不在项目仓库内。

### 7.2 config.yaml 中的登录配置

```yaml
systems:
  main:
    baseUrl: https://main-test.iho.com
    storageState: ~/.claude/state/iho-delivery/main.json
    loginProbe:   { role: button, name: "退出登录" }
    loginPage:    /login
    loginAnchors:
      userInput: { role: textbox, name: "用户名" }
      passInput: { role: textbox, name: "密码" }
      submitBtn: { role: button,  name: "登录" }
    postLoginUrlNotContains: ["/login"]
  esb:
    baseUrl: https://esb-test.iho.com
    storageState: ~/.claude/state/iho-delivery/esb.json
    loginProbe:   { role: link,   name: "我的" }
    loginPage:    /login
    loginAnchors: { ... }
    postLoginUrlNotContains: ["/login"]

artifactsRoot: /Users/wangjie/ai-hub/skills/iho-delivery-flow/collect-archie
feedDocsRoot:  /Users/wangjie/workspace/project/iho-mrms/doc/feat
defaultBrowser: chromium
tracing: on-failure
```

### 7.3 启动时登录态流程

```
1. 读 config.systems.<sys>
2. launch browser + newContext({ storageState: sys.storageState })  (文件不存在则空 state)
3. page.goto(sys.baseUrl)
4. 探测 loginProbe 5s 内可见? → 已登录,进入业务
                             → 未登录,调 loginFlow
5. loginFlow:
   a. 读 process.env.IHO_<SYS>_USER / _PASS
   b. 缺失 → 抛错提示运行 login-<sys>.mjs 手动登,exit 2
   c. 存在 → page.goto(baseUrl + loginPage),填表,提交
   d. 等 URL 不再包含 /login(postLoginUrlNotContains 判断)
   e. 保存 storageState + 写 meta
6. 连续 2 次 loginFlow 失败 → exit 2,锁定
```

**有效期不看 mtime,只看 loginProbe**,每次启动都探测一次,不相信时间。

### 7.4 手动登录脚本(保底)

`scripts/login-main.mjs` / `login-esb.mjs`:
- `headless: false` 打开登录页
- 等待 loginProbe 可见(你手动登)
- 保存 storageState
- 关闭浏览器

用场景:首次、遇到验证码、storageState 被清。

### 7.5 环境变量

```bash
export IHO_MAIN_USER=... IHO_MAIN_PASS=...
export IHO_ESB_USER=...  IHO_ESB_PASS=...
export JENKINS_TOKEN=...
```

密码不落盘、不入仓、不写 config.yaml。

---

## 8. 错误处理、回滚指引、重跑幂等

### 8.1 exit code 约定

| exit | 含义 | Claude 动作 |
|---|---|---|
| 0 | 全部成功 | 后置汇报 ✅,状态设 done |
| 1 | 部分失败 | 后置汇报 ⚠️,按条目展示,让用户选重跑哪些 |
| 2 | 致命错误(登录坏/锚点完全丢/网络断) | 后置汇报 ❌,状态设 failed |
| 99 | 用户中断 | 状态设 in_progress,plan 保留供续跑 |

### 8.2 锚点不命中处理

每个锚点查找统一用 `findOrDie(page, anchor)`:
- `expect(locator).toBeVisible({ timeout: 8000 })`
- 失败:截图 `logs/miss-<anchor>-<ts>.png` + 抛 `AnchorMissError`
- result.json 记录 `anchor / url / screenshot`
- 后置汇报时展示截图路径,你看图判断按钮文字/结构是否变了 → 改 anchors.json(不动脚本)

### 8.3 重试策略(保守)

- **导航/等待类**:超时不重试,直接留 trace
- **动作类**(click/fill):**不重试**(避免重复提交)
- **下载失败**:自动重试 1 次(文件流中断场景)
- **登录失败**:最多 2 次,第 3 次 exit 2

### 8.4 回滚指引

- **形式**:纯文字,硬写在每个子 skill 的 SKILL.md 的 "## 回滚指引" 章节
- **触发**:后置汇报时用户说"我要回滚",Claude 原样打印该章节
- **不自动回滚任何网页操作**

### 8.5 重跑幂等核心机制

脚本启动时从 state.steps.<step>.output 读取已完成 `uniqueKey` 集合,plan 中已完成条目直接标 `status:ok, reused:true`,不重复执行。强制重跑走 `发版-重置 <step> --item <id>`。

### 8.6 后置汇报统一格式

```
步骤 submit-sql 执行完毕(2 分 12 秒)
  overall: partial (4 成功 / 1 失败 / 0 跳过)
  ✅ SQL-9901  调整 patient_info 索引
  ✅ SQL-9902  清理过期缓存
  ✅ SQL-9903  新增 phi_log 表
  ✅ SQL-9904  更新配置默认值
  ❌ 第 5 条  调整视图定义
       错误:AnchorMiss(btn_save 不可见)
       URL: https://.../sql/new
       截图:artifacts/.../logs/miss-btn_save-....png
       Trace:artifacts/.../logs/submit-sql-....trace.zip
       回滚指引:见 SKILL.md "## 回滚指引"

请选择:
  1. 确认当前结果(进入下一步)
  2. 手动标记第 5 条为完成(不实际执行)
  3. 重跑第 5 条
  4. 中止并查看 trace (会自动打开 show-trace)
  5. 中止并打印回滚指引
```

选项 4 由 Claude 直接 Bash 执行 `npx playwright show-trace <路径>`,自动弹浏览器。

### 8.7 失败后续跑

- 失败退出后 `plans/<step>.json` 保持不变
- 用户修复问题后再次触发子 skill
- 前置询问"沿用现有计划 / 重新生成":
  - 沿用 → 直接到 P3 再确认
  - 重新生成 → 备份旧 plan 到 `plans/<step>.<ts>.bak.json`,生成新 plan

---

## 9. 页面交付协议(用户 → Claude 提供页面信息的规范)

每个页面按以下格式交付,Claude 产出 `<step>/references/anchors.json` + `.runtime/scripts/<step>.mjs` + 首次 dry-run 截图。

### 9.1 交付两档

| 档位 | 你提供什么 | 产出准确率 |
|---|---|---|
| **基础档**:截图 + URL + 操作描述 | 页面 URL、每个动作的截图、一句话描述动作序列 | 80%(需 dry-run 一轮微调) |
| **完整档**:基础档 + codegen 片段 | 额外提供 `npx playwright codegen <url>` 的输出 | 95%(通常一跑即过) |

### 9.2 推荐分档

- **表单重的页面走完整档**:create-version、create-patch、submit-sql、finalize-release
- **纯点击/下载页走基础档**:download-esb、download-resources

### 9.3 交付模板

```markdown
## 页面:create-version

- **URL(列表)**:https://main-test.iho.com/release/list
- **URL(详情 URL 正则)**:/release/detail/(\d+)
- **目标**:创建一条发版记录,拿到详情页 URL 和 releaseId
- **登录系统**:main

### 操作序列
1. 进入列表页 → 点击右上角「新建版本」按钮 → 截图 1
2. 弹出表单(对话框) → 填版本号、版本名 → 截图 2
3. 点击「保存」→ 跳转到详情页 → 截图 3

### codegen 片段(可选)
\`\`\`js
await page.getByRole('button', { name: '新建版本' }).click();
await page.getByLabel('版本号').fill('4.4.3');
await page.getByLabel('版本名').fill('iho-mrms 4.4.3');
await page.getByRole('button', { name: '保存' }).click();
\`\`\`

### 特殊要求
- 版本号格式 X.Y.Z(大版本)/ X.Y.Z.N(补丁)
- 保存失败会弹红色 toast("版本号已存在"),需捕获并汇报
```

### 9.4 Claude 产出交付物

- `<step>/references/anchors.json`:锚点清单
- `.runtime/scripts/<step>.mjs`:完整脚本(猜测部分加 `// CONFIRM` 注释)
- 首次 dry-run 命令示例:`node .runtime/scripts/<step>.mjs --plan ... --dry-run`
- dry-run 产出:`results/<step>.dry.json` + 每步截图

---

## 10. 测试、验收、首次部署

### 10.1 首次部署步骤

```bash
# a. 安装
cd /Users/wangjie/ai-hub/skills/iho-delivery-flow/.runtime
pnpm init -y
pnpm add -D playwright @playwright/test js-yaml minimist
npx playwright install chromium

# b. 手填 config.yaml 的 baseUrl + 登录锚点 + loginProbe

# c. 配置环境变量 ~/.zshrc:
#    IHO_MAIN_USER/PASS, IHO_ESB_USER/PASS, JENKINS_TOKEN

# d. 首次登录
node .runtime/scripts/login-main.mjs
node .runtime/scripts/login-esb.mjs

# e. 在 ~/.claude/CLAUDE.md "Skills 索引" 章节注册(见 §10.2)

# f. selfcheck 验证
node .runtime/scripts/selfcheck.mjs
```

`setup.sh` 封装 a 和 d,其余手动。

### 10.2 CLAUDE.md 注册条目

```markdown
### iho-delivery-flow
- 触发条件:用户说 "发版-启动 <版本号>" / "发版-<子步骤>" / "发版-状态" 等发版相关指令时
- SKILL.md: `$SKILLS/iho-delivery-flow/SKILL.md`
- 参考文件: `$SKILLS/iho-delivery-flow/specs/20260417iho每周发版自动化/`

### iho-delivery-flow 子 skill(通过顶层 iho-delivery-flow 激活)
- `$SKILLS/iho-delivery-flow/create-version/SKILL.md`
- `$SKILLS/iho-delivery-flow/create-patch/SKILL.md`
- `$SKILLS/iho-delivery-flow/download-esb/SKILL.md`
- `$SKILLS/iho-delivery-flow/submit-sql/SKILL.md`
- `$SKILLS/iho-delivery-flow/sync-params/SKILL.md`
- `$SKILLS/iho-delivery-flow/sync-resources/SKILL.md`
- `$SKILLS/iho-delivery-flow/download-resources/SKILL.md`
- `$SKILLS/iho-delivery-flow/collect-archie/SKILL.md`
- `$SKILLS/iho-delivery-flow/finalize-release/SKILL.md`
```

### 10.3 测试策略

#### 单元级(`.runtime/test/`)
- `plan-builders/*`:mock feat 文档,校验抽取结果
- `anchors.mjs`:JSON → locator 翻译
- `state.mjs`:读写原子性(并发写 100 次)
- 命令:`pnpm test`

#### 烟雾(dry-run)
每个子 skill 脚本支持 `--dry-run`:
- plan 照读、浏览器启动
- 所有"点击/提交"改为 log + 截图
- 写 `results/<step>.dry.json`

每个页面实跑前先 `--dry-run` 看截图验证锚点。

#### 冷启动自检(`selfcheck.mjs`)
- Node ≥ 18
- Chromium 已装
- config.yaml 字段齐
- 两套 storageState 存在或提示
- artifactsRoot 父目录可写

### 10.4 分阶段上线

| 阶段 | 内容 | 验证 |
|---|---|---|
| I | 骨架 + 顶层 + state-cli + setup.sh | `发版-启动 4.4.3-test` 能建 state |
| II | create-version(dry-run) | 截图确认锚点 |
| III | create-version(实跑到测试环境) | 成功抓 releaseId |
| IV | 其余 8 个子 skill 迭代 | 每个 dry → 实跑 |
| V | finalize-release + jenkins-trigger | 端到端联调 |
| VI | 在测试环境完整跑 `发版-启动 9.9.9` | 9 步全过,AC 全中 |

每阶段独立评审(requesting-code-review)。

### 10.5 验收标准对齐(对应 requirements.md §9)

| AC | 实现节点 | 验证方式 |
|---|---|---|
| AC-1 `发版-启动` 初始化 state + 打印依赖图 | 阶段 I | selfcheck + 手动 |
| AC-2 三阶段循环 + 微调 diff | 阶段 II-IV 每步 | 手动演练 |
| AC-3 中途退出后 `发版-状态` | 阶段 I state-cli | 单元测试 |
| AC-4 `发版-标记` 含 manuallyMarked | 阶段 I | 单元测试 |
| AC-5 补丁模式 parentVersion 自动推断 | 阶段 IV(create-patch) | 手动演练 |
| AC-6 finalize uploadSlots 上传 | 阶段 V | 手动验收 |
| AC-7 Jenkins 自动触发 + 确认 | 阶段 V | 脚本检查 HTTP 201 |
| AC-8 登录态跨步骤复用 | 阶段 IV 二次运行 | 手动观察 |
| AC-9 无明文密码入仓 | 所有阶段 | grep 扫描 |
| AC-10 回滚指引为文字 | SKILL.md 审查 | 脚本扫 Markdown |
| AC-11 退出后无残留进程 | 所有阶段 | `ps | grep` 验证 |

### 10.6 git 管理

`iho-delivery-flow/.gitignore`:

```
.runtime/node_modules/
collect-archie/*/
!collect-archie/SKILL.md
!collect-archie/references/
```

---

## 11. 附录

### 11.1 config.yaml 完整示例

(见 §7.2 已列,增加默认字段):

```yaml
defaultBrowser: chromium      # 或 chrome(抗检测,但需本机装 Chrome)
tracing: on-failure           # on | off | on-failure
sqlPasteThreshold: 100        # SQL 行数阈值,< 即 paste,≥ 即 upload
```

### 11.2 anchors.json 通用格式

```json
{
  "baseUrl": "auto-from-config",
  "entry": "/release/list",
  "anchors": {
    "btn_new":  { "role": "button",  "name": "新建版本" },
    "fld_ver":  { "role": "textbox", "name": "版本号" },
    "btn_save": { "role": "button",  "name": "保存" }
  },
  "waits": {
    "afterSave": { "urlContains": "/release/detail/" }
  },
  "extract": {
    "releaseId":  { "from": "url", "regex": "/detail/(\\d+)" },
    "releaseUrl": { "from": "url" }
  }
}
```

支持的锚点定位器类型:`role` / `label` / `placeholder` / `text` / `testid` / `css`,优先级从高到低。

### 11.3 run-step.mjs 路由

```
node .runtime/scripts/run-step.mjs <step-name> [--dry-run] [--version <ver>]
  内部:
    1. 定位 artifactsDir(按 --version 或最新 updatedAt)
    2. 拼参数
    3. import(`./<step-name>.mjs`)
    4. 调用默认导出 async 函数,返回 exit code
```

### 11.4 子 skill 典型 SKILL.md 示例(以 submit-sql 为例)

```markdown
---
name: 发版-submit-sql
description: 在"发版"流程中,用户说"发版-submit-sql"时,按三阶段模板提交本次版本的 SQL
---

# 发版-submit-sql

## 前置依赖
- 发版-create-version 或 发版-create-patch 已完成

## 计划原料
- feat 文档 `SQL` 章节
- `git diff --name-only master...HEAD -- '*.sql'`
- state.context.releaseId

## 执行器
- 脚本: `.runtime/scripts/submit-sql.mjs`
- 锚点: `submit-sql/references/anchors.json`
- 登录态: main

## 典型微调场景
- 跳过已在生产执行的 SQL(enabled=false)
- 调整执行顺序(order)
- 合并多条(手动改 title + content)
- 修改 mode(paste ↔ upload)

## 回滚指引
1. 登录主业务系统
2. 进入"发版管理 > SQL 变更"菜单
3. 按单号(见 results/submit-sql.json 的 submittedIds)搜索
4. 点击每一条 → "撤回/删除"
5. 最后:`发版-重置 submit-sql`

## 执行规范
遵循通用三阶段模板(见 iho-delivery-flow/SKILL.md §三阶段)。
```

---

## 12. 交付记录豁免

本需求是 **skill 开发**,非 iho-mrms 业务需求:

- **不**生成 iho-mrms/doc/feat/feat_vX.Y.Z.md 记录
- **不**涉及 iho-mrms 的 SQL / 参数 / 接口 / 资源变更
- 本 skill 的 **skill 自身迭代**若未来涉及破坏性改动,会在 `specs/` 目录新建独立需求文档

skill 产出物位置:

- `/Users/wangjie/ai-hub/skills/iho-delivery-flow/`:skill 本体、锚点、scripts、references
- `/Users/wangjie/.claude/CLAUDE.md`:Skills 索引注册
- `/Users/wangjie/.claude/state/iho-delivery/`:运行期登录态(不入仓)
- `collect-archie/<日期_版本>/`:运行期产出(不入仓)

---

## 13. 关键决策摘要(对齐 brainstorming 讨论)

| # | 决策 | 理由 |
|---|---|---|
| D-1 | 方案 D(锚点清单 + 脚本),不用 MCP | 每次发版 token 消耗从 ~250K 降到 ~15K |
| D-2 | 9 个子 skill 独立触发,不自动串联 | 每步独立重跑、调试、微调 |
| D-3 | 顶层维护 state.json,子 skill 读写 | 续跑、中途查看、手工标记都靠它 |
| D-4 | 每步前置计划 + 后置确认,支持微调 | 发版风险高,人工把关每一步 |
| D-5 | 不自动回滚,仅文字指引 | 网页撤销路径不确定,风险高 |
| D-6 | 两套 storageState 独立管理 | 两个系统两套账号 |
| D-7 | 有效期判断只看 loginProbe,不看 mtime | 时间戳会骗人,实测最准 |
| D-8 | SQL < 100 行 paste,≥ 100 行 upload | 发版系统 UI 两种模式,按行数自动选 |
| D-9 | finalize 每文件单独上传,不打 zip | 系统有多 slot,每个 slot 有格式要求 |
| D-10 | Jenkins 自动触发(保留一次确认) | 发版单提交后触发镜像构建是固定流程 |
| D-11 | uniqueKey 用标题/key/id 而非 md5 | 计算轻,够用 |
| D-12 | 每个脚本独立进程,跑完即退 | 内存立即释放,不占用空闲资源 |
