# 需求:iho 每周发版流程自动化(iho-delivery-flow)

- **创建日期**:2026-04-17
- **作者**:wangjie
- **范围**:`/Users/wangjie/ai-hub/skills/iho-delivery-flow/` 及其子目录
- **交付形态**:一组通过 `Skill` 工具调用的 Markdown skill(1 个顶层编排 + 9 个子步骤 + references)

---

## 1. 背景

每周发版涉及跨两个 Web 系统的多步人工操作(ESB 系统 + 主业务系统),目前由人手完成。操作具有以下特征:

- 步骤多(9 步 + 1 次 Jenkins 构建)且跨系统
- 产出物多(多套文件需下载到本地并最终汇总上传)
- 内部存在并行依赖(部分步骤可并行,部分有硬顺序)
- 存在两种模式:**大版本发版**(create-version)与**小补丁发版**(create-patch),骨架一致、入口按钮与版本号规则不同

希望通过 Playwright 驱动浏览器 + 一组按步骤拆分的 skill,将整条流程半自动化;每个子步骤可独立调用、按用户节奏推进,并在每次执行前后接受用户确认与微调。

## 2. 目标与非目标

### 2.1 目标

- 把发版流程拆成可**独立触发、独立调试、独立重跑**的子步骤
- 每个子步骤**前置展示计划,后置展示结果**,支持微调
- 顶层维护**共享状态**(版本号、发版单 ID、各步产出物路径、完成标记),供各子步骤读取并写回
- 支持**两种发版模式**(大版本 / 补丁)
- 登录态统一管理,跨步骤复用,尽量减少重登
- 产出物归档到统一的本地目录 `artifacts/<版本号>/`,最终由 `finalize-release` 上传

### 2.2 非目标

- **不做**自动回滚(回滚只打印指引文字,由人工执行)
- **不做**发版审批或生产部署决策(由人工确认)
- **不替代** `jenkins-flow`、`create-feat`、`git-flow` 这些已有 skill,而是**复用**它们
- **不做**CI 集成、不做定时触发
- **不覆盖** Jenkins 构建结果轮询(触发后即返回,不等构建完成的状态回调)
- **不保存**账号密码到仓库,登录凭证走环境变量或一次性交互输入

## 3. 涉及的外部系统

| # | 系统 | 说明 | 账号体系 |
|---|---|---|---|
| S1 | ESB 系统 | 下载接口/服务包相关文件 | 独立账号 |
| S2 | 主业务系统 | 发版管理 + SQL + 参数 + 资源 多菜单集合 | 独立账号,覆盖 [1][4][5][6][7][9] 六步 |

注:具体 URL、菜单路径、账号存放方式在 `design.md` 中固化。

## 4. 流程与步骤(锁定)

### 4.1 依赖图

```
[1] create-version / create-patch   建发版单
        ↓
  ┌─── [3] download-esb       (独立,ESB 系统)
  ├─── [4] submit-sql         (独立,主业务系统)
  ├─── [5] sync-params ─┐     (主业务系统)
  └─── [6] sync-resources ─┴─→ [7] download-resources  (主业务系统,硬依赖 [5][6] 完成)
        ↓ (所有上述完成后)
[8] collect-archie            本地汇总
        ↓
[9] finalize-release          回发版页面上传 + 填描述 + 保存
        ↓
[2] jenkins-flow 构建 master  (复用现有 skill,流程终点)
```

### 4.2 并行与依赖规则

- [3] / [4] / {[5]→[7], [6]→[7]} 是**三条并行支**,[1] 完成后都可启动
- [5]、[6] 之间无先后,但 [7] 必须在 [5] **和** [6] 都完成后启动
- [8] 是汇总闸口,依赖 [3]、[4]、[7] 全部完成
- [9] 依赖 [8];[2] 依赖 [9]
- **注**:本 skill 不强制并发执行;"可并行"仅表示用户可以按任意顺序手动触发,顶层 state 跟踪完成情况

### 4.3 步骤清单

| # | Skill 目录 | 所属系统 | 输入 | 核心动作 | 产出 |
|---|---|---|---|---|---|
| 1 | `create-version` / `create-patch` | 主业务系统 | 版本号 | 点不同按钮 + 填表单 + 保存 | 发版单 ID、发版单 URL |
| 2 | (复用) `jenkins-flow` | Jenkins | 版本号 | 构建 master 并打镜像 | HTTP 201 触发成功 |
| 3 | `download-esb` | ESB 系统 | 版本号 | 进入指定菜单下载文件 | `artifacts/<版本>/esb/` |
| 4 | `submit-sql` | 主业务系统 | feat 文档 `SQL` 章节 + 本次新增 `.sql` 文件 | 进入 SQL 菜单逐条提交;< 100 行粘贴,≥ 100 行文件上传 | 提交单号列表 |
| 5 | `sync-params` | 主业务系统 | feat 文档 `参数` 章节(固定章节名) | 进入参数菜单同步 | 同步记录 |
| 6 | `sync-resources` | 主业务系统 | feat 文档 `资源` 章节(固定章节名) | 进入资源菜单同步 | 同步记录 |
| 7 | `download-resources` | 主业务系统 | 版本号 | 进入资源下载菜单下载 | `artifacts/<版本>/resources/` |
| 8 | `collect-archie` | 本地(无浏览器) | 前面所有产出 | 整理目录 + 生成 `summary.md`(本次改动描述),**不打 zip** | `artifacts/<版本>/` 完整目录 + 文件清单 |
| 9 | `finalize-release` | 主业务系统 | `artifacts/<版本>/` + 发版单 URL | 回发版页面按 uploadSlots 配置**每个文件单独上传**(esb 文件到 esb slot、resources 到 resources slot,格式要求不同) + 填发布说明 + 保存 | 发版单状态 = 已提交 |

## 5. 功能需求

### 5.1 顶层编排(`iho-delivery-flow`)

- **FR-1**:顶层 skill **不自动串联**所有步骤,仅提供路由 + 状态管理
- **FR-2**:启动入口触发词:`发版-启动 <版本号>`,通过版本号段数自动识别模式(3 段 `X.Y.Z` = 大版本,4 段 `X.Y.Z.N` = 补丁)
- **FR-3**:启动后提示用户:"已记录版本号 X,模式 Y,请按顺序手动触发各子 skill",并打印 9 个子步骤触发词清单和依赖图
- **FR-4**:维护 **state.json**(路径见 §6.1),任何子 skill 执行完毕必须写回状态
- **FR-5**:提供辅助触发词 `发版-状态`,打印当前 state.json 的人类可读摘要
- **FR-6**:提供辅助触发词 `发版-标记 <step> <状态>`,支持手工覆盖某步状态(例:`发版-标记 submit-sql done`),设置时自动标 `manuallyMarked:true` 以便审计
- **FR-7**:提供辅助触发词 `发版-重置 <step> [--item <id>]`,清空某步或单条目状态以便重跑
- **FR-8**:提供辅助触发词 `发版-切换 <日期_版本>`(多版本并存时切换上下文)、`发版-清单`(重新打印依赖图)、`发版-重登 main/esb`(强制重登)、`发版-编辑计划 <step>`(长计划微调)
- **FR-9**:所有子 skill 触发词以 `发版-` 为前缀,避免与日常对话冲撞

### 5.2 子 skill 通用契约

每个子 skill 必须满足:

- **FR-10 前置计划**:执行前先从 state + feat 文档 + git 等原料拼出"计划",展示给用户并等待确认
- **FR-11 计划介质**:
  - 短计划(≤ 10 行展示项):对话内直接展示摘要
  - 长计划(> 10 行展示项):写到 `artifacts/<版本>/plans/<step>.json`,用户可直接编辑文件或通过 `发版-编辑计划 <step>` 打开
  - 微调应用后**只打印 diff**,不重复完整计划,避免刷屏
- **FR-12 微调**:用户可在对话里用自然语言要求修改计划(改某一条、删某一条、加某一条),skill 须应用修改后再次展示
- **FR-13 执行**:确认后才通过 Playwright 执行
- **FR-14 后置汇报**:执行完打印结果(成功数/失败数/关键产出路径),等待用户确认
- **FR-15 状态覆盖**:用户可在后置阶段说"手工标记已完成",skill 将对应条目或整步标记为成功,不实际重跑
- **FR-16 重跑幂等**:对已完成的条目默认跳过(除非用户明确要求重做)
- **FR-17 回滚**:不自动回滚,仅在出错时打印"回滚指引"(系统页面路径 + 操作步骤文字)
- **FR-18 写回 state**:执行完必须更新 state.json 的对应步骤字段

### 5.3 两种模式差异

- **FR-20**:`create-version` 与 `create-patch` 是两个独立 skill,分别对应主业务系统上不同的"新建版本 / 新建补丁"按钮
- **FR-21**:两者填写字段、版本号校验规则不同,具体字段和规则在 design.md / references 中固化
- **FR-22**:其余步骤([3]-[9])两种模式共用,差异仅通过 state 中的 `mode` 字段体现(如填描述模板时选择不同前缀)

### 5.4 登录态管理

- **FR-30**:分别维护两套 Playwright `storageState`:
  - `~/.claude/state/iho-delivery/esb.json` (ESB 系统)
  - `~/.claude/state/iho-delivery/main.json` (主业务系统)
- **FR-31**:子 skill 启动时检测登录态是否有效 —— **只依赖 loginProbe 探测**(启动后探测一次"已登录标志元素"),不看文件 mtime
- **FR-32**:未登录时优先用环境变量 `IHO_<SYS>_USER/PASS` 自动登录;失败才退化为有头浏览器手动登录;成功后保存 storageState 续跑
- **FR-33**:连续 2 次 loginFlow 失败时,子 skill exit 2 并提示手动运行 `login-<sys>.mjs`
- **FR-34**:登录页锚点(用户名/密码/提交按钮)统一放在 `config.yaml` 的 `systems.<sys>.loginAnchors` 中,与业务锚点分离

### 5.5 产出目录

- **FR-40**:所有产出集中在 **`/Users/wangjie/ai-hub/skills/iho-delivery-flow/collect-archie/<yyyyMMdd>_<版本号>/`**,其中 `yyyyMMdd` 取启动顶层 skill 当天的日期。例:`collect-archie/20260417_4.4.3/`。文中其他位置出现的 `artifacts/<版本>/` 即指这个目录。
- **FR-41**:子目录规范:
  - `esb/`:ESB 下载文件
  - `resources/`:资源下载文件
  - `plans/<step>.json`:各步计划文件(JSON 格式)
  - `results/<step>.json`:各步执行结果
  - `logs/<step>-<timestamp>.log` 和 `logs/<step>-<timestamp>.trace.zip`:Playwright 执行日志 + 失败时的 trace
  - `state.json`:全局状态
  - `summary.md`:本次发版的汇总描述(最终上传的"发布说明"文本)
- **FR-42**:`collect-archie/` 下除了 SKILL.md / references/ 以外的所有 `<yyyyMMdd>_*` 子目录需加入 `.gitignore`(或父级 `.gitignore`),避免把运行期产出提交到仓库

## 6. 数据契约

### 6.1 state.json 最小字段(JSON 格式)

```json
{
  "version": "4.4.3",
  "mode": "version",
  "artifactsDir": "/Users/wangjie/ai-hub/skills/iho-delivery-flow/collect-archie/20260417_4.4.3",
  "startedAt": "2026-04-17T10:00:00+08:00",
  "updatedAt": "2026-04-17T10:00:00+08:00",
  "context": {
    "releaseId": "REL-8832",
    "releaseUrl": "https://.../detail/8832",
    "parentVersion": null,
    "feedDocPath": "/abs/doc/feat/feat_v4.4.3.md"
  },
  "steps": {
    "create-version": {
      "status": "done",
      "finishedAt": "...",
      "manuallyMarked": false,
      "output": {}
    },
    "download-esb": { "status": "pending" },
    "submit-sql":   { "status": "pending" },
    "...": "其余步骤同构"
  }
}
```

**字段说明**:
- 顶层 `version / mode / artifactsDir / startedAt / updatedAt`:本次发版身份
- `context.*`:跨步骤共享的关键产出(releaseId/Url 等),任一步骤写,后续步骤读
- `context.parentVersion`:仅补丁模式填写,自动从版本号推断(如 `4.4.4.1` → `4.4.4`)
- `steps.<step>.status`:`pending | in_progress | done | failed | skipped`
- `steps.<step>.manuallyMarked`:true 表示由用户手工设置,未实际执行

- **DR-1**:state.json 必须人工可读,所有 skill 写入前后必须格式化
- **DR-2**:同一个版本号只有一份 state.json,重复触发顶层 skill 提示已存在并询问"续跑 / 重置 / 查看"
- **DR-3**:时间戳统一使用带时区的 ISO 8601

### 6.2 plans/<step>.json 通用结构(JSON 格式)

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

- **DR-10**:用户修改 `enabled: false` 可跳过某条
- **DR-11**:增加条目需遵循相同字段,`uniqueKey` 必填(用于跨次运行去重)
- **DR-12**:submit-sql 的 `mode` 字段由脚本首次生成时自动判断(行数 `< 100` → `paste`;`>= 100` → `upload`),用户可手工覆盖

## 7. 非功能需求

- **NFR-1 语言**:所有 SKILL.md、references、提示语中文
- **NFR-2 风格**:对齐现有 `jenkins-flow` / `git-flow` / `create-feat` 风格:顶层短 SKILL.md + 触发词 + 步骤清单 + 命令;复杂模板放 `references/`
- **NFR-3 时间**:单次子 skill 人机对话交互 < 3 分钟不卡顿(不含等待页面加载/下载耗时)
- **NFR-4 稳健性**:单个选择器失败时打印当前页面 URL + 无障碍快照摘要,提示用户判断(不自动重试无限次)
- **NFR-5 可追溯**:每次 Playwright 执行都落 trace(失败必留,成功可选),路径 `logs/<step>-<ts>.trace.zip`
- **NFR-6 无秘密**:不写入密码;storageState 文件不进仓库
- **NFR-7 跨机**:产出目录、state 路径基于当前工作目录或环境变量,不硬编码个人路径

## 8. 约束

- **CN-1**:对接的两个系统的 URL / 选择器由 wangjie 按"页面交付协议"(见 design.md §9)分批次提供(截图+URL+操作描述,可选附 `npx playwright codegen` 片段),Claude 据此生成锚点 JSON + 脚本
- **CN-2**:账号密码不存仓库;统一走环境变量 `IHO_MAIN_USER/PASS`、`IHO_ESB_USER/PASS`、`JENKINS_TOKEN`;不加密文件回退
- **CN-3**:选择器优先级 `role > label > placeholder > text > testid > css`;禁止 XPath
- **CN-4**:禁用 `page.waitForTimeout(固定毫秒)`,改用 `expect(...).toBe*` 或 `waitForURL`
- **CN-5**:Jenkins 构建在 `finalize-release` 成功后 **由顶层 skill 自动触发**(保留一次确认提示,默认 Y),内部复用 `jenkins-flow` 的 curl 参数,凭据读 `$JENKINS_TOKEN`。`jenkins-flow` 的触发词(`构建 master <版本号>`)保持不变,作为用户手动跑的后备通道
- **CN-6**:feat 文档的固定章节名约定:`SQL`、`参数`、`资源`(可含可选 `ESB`),plan-builders 按此抽取。章节名变更需同步更新 plan-builders
- **CN-7**:每次脚本运行是独立一次性子进程(node + chromium),运行结束内存立即释放,不保留 daemon
- **CN-8**:`--max-old-space-size=512` 硬限 Node 堆;Chromium 启动参数最小化(关 GPU/扩展/后台网络等),单次峰值内存 ≤ 500MB

## 9. 验收标准

- **AC-1**:执行 `发版-启动 4.4.3`,顶层 skill 创建 state.json(mode=version)并打印子步骤清单和依赖图
- **AC-2**:依次触发 9 个子步骤(使用测试环境或 dry-run 模式),每步都出现"前置确认→执行→后置确认"循环,且支持中途说"删掉第 2 条"并被采纳,计划 diff 正确展示
- **AC-3**:任意子步骤中途退出,再次执行 `发版-状态` 能准确打印各步状态(含 manuallyMarked 标记)
- **AC-4**:`发版-标记 submit-sql done` 能在不实际执行的情况下更新 state,且 `manuallyMarked:true`
- **AC-5**:`发版-启动 4.4.3.1` 能识别为 patch 模式,`context.parentVersion` 自动置为 `4.4.3`
- **AC-6**:`finalize-release` 执行后,发版页面按 uploadSlots 配置各 slot 完成上传、发布说明填入、保存成功
- **AC-7**:`finalize-release` 成功后,顶层 skill 在一次确认后自动触发 Jenkins,接收 HTTP 201 并写回 state.steps.jenkins.status=done
- **AC-8**:登录态在同一发版过程中跨子步骤复用,不重复弹登录框(除非 loginProbe 探测失败)
- **AC-9**:整个流程不依赖任何明文密码存盘,不会把 storageState 提交到 git
- **AC-10**:失败场景中出现的"回滚指引"是可读的文字步骤,不是自动操作
- **AC-11**:每次脚本运行退出后,系统进程表无残留的 node / chromium 进程(内存立即释放)

## 10. 交付记录豁免

本需求是 **skill 开发**,非 `iho-mrms` 业务需求,故:

- **不**生成 `iho-mrms/doc/feat/feat_vX.Y.Z.md` 记录
- **不**涉及 iho-mrms 的 SQL/参数/接口/资源变更

本 skill **产出物**置于:

- `/Users/wangjie/ai-hub/skills/iho-delivery-flow/`(skill 本体 + references)
- `/Users/wangjie/.claude/CLAUDE.md` 的 "Skills 索引" 章节(注册条目)
- `/Users/wangjie/.claude/state/iho-delivery/`(运行期登录态,不入仓)
- `artifacts/<版本号>/`(运行期产出,不入仓)

## 11. 词汇表

- **发版单**:主业务系统里一条版本记录,有唯一 ID 与 URL
- **feat 文档**:`iho-mrms/doc/feat/feat_vX.Y.Z.md`,由 `create-feat` 产出,记录本次版本的所有变更,是 [4][5][6] 的主要计划原料
- **state**:本次发版的全局状态文件 `artifacts/<版本>/state.json`
- **storageState**:Playwright 保存登录态的 JSON 文件
