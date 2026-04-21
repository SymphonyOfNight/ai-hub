# 任务进度记录

> 记录时间：2026-04-17
> 用途：供新会话继续当前任务使用

## 任务总览

构建 **iho-delivery-flow** skill（基于 Playwright + 轻量 Node 脚本 + 锚点 JSON 方案 D），实现 IHO 每周发版自动化流程：
- 顶层 skill + 9 个子 skill（菜单式独立触发）
- 支持两种模式：**大版本（create-version）** 与 **补丁（create-patch）**
- 子步骤执行前后均需与用户确认，支持微调
- 一次性 Node 进程，内存即用即释放
- 最终打包 master（Jenkins 自动触发，位于流程最末）

目标目录：`/Users/wangjie/ai-hub/skills/iho-delivery-flow/`

## 当前阶段

按 `brainstorming` skill 流程推进，**7 步中已完成第 8 步「文档自审」**，当前停在 **「用户审阅关口」**，等待用户审阅 `requirements.md` 与 `design.md` 后决定是否进入 `writing-plans` 生成 `tasks.md`。

## 流程进度（brainstorming 10 步清单）

- [x] 1. 探索项目上下文
- [x] 2. 提供视觉辅助选项（本任务不需要）
- [x] 3. 提出澄清问题（已与用户完成多轮澄清）
- [x] 4. 确定需求目录并写需求文档
- [x] 5. 提出 2-3 个方案（最终确定 方案 D：轻量脚本 + 锚点 JSON）
- [x] 6. 展示设计（分段确认）
- [x] 7. 写设计文档
- [x] 8. 文档自审（占位符、一致性、范围、歧义）
- [ ] 9. **明确交付记录要求**（需求完成后补 `/doc/feat/feat_xxxx.md`） —— 已在 design.md §12 约定「feat 交付豁免」，待用户审阅确认
- [ ] 10. **请用户审阅文档** ← **当前停在这里**
- [ ] 11. 切换到 `writing-plans` skill 生成 `tasks.md`

## 已产出文档

| 文件 | 状态 |
|---|---|
| `specs/20260417iho每周发版自动化/requirements.md` | ✅ 已完成（12 节，FR 1-34，CN 1-8，AC 1-11） |
| `specs/20260417iho每周发版自动化/design.md` | ✅ 已完成（13 节，含 D-1 ~ D-12 决策摘要） |

## 关键决策回顾（12 项）

- D-1 架构：3 层（skill 层 / 脚本层 / 锚点层），采用方案 D
- D-2 编排：菜单式独立触发（B 方案），state.json 共享上下文
- D-3 模式判别：版本号点分段数（3 段=大版本，4 段=补丁，补丁自动推算 `parentVersion`）
- D-4 数据格式：统一 JSON（state.json / plans/*.json / results/*.json）
- D-5 依赖校验：由 `state check-deps` 统一管理
- D-6 触发词前缀：`发版-`（如 `发版-启动 4.4.3`、`发版-打包`）
- D-7 流程顺序：4 → 5/6/7（并行）→ 8 → 9 → Jenkins 打包（最末）
- D-8 SQL 处理：< 100 行贴到系统；≥ 100 行上传 `.sql` 文件
- D-9 finalize：每个文件单独上传（上传按钮位置不同，有格式要求，不打包 zip）
- D-10 Jenkins：自动触发（一次确认）
- D-11 登录态：仅凭 `loginProbe` 判定；账号来自环境变量 `IHO_MAIN_USER/PASS` 等
- D-12 进程模型：一次性 Node 进程，`--max-old-space-size=512`，即用即释放

## 9 个子 skill 清单

| # | 子 skill | 触发词 | 对应系统 |
|---|---|---|---|
| 1 | pre-check | `发版-预检` | 本地 |
| 2 | create-version | `发版-新建版本 <版本号>` | ESB |
| 3 | create-patch | `发版-新建补丁 <版本号>` | ESB |
| 4 | download-deliverables | `发版-下载交付物` | 主业务系统 |
| 5 | submit-sql | `发版-提交 SQL` | ESB |
| 6 | submit-params | `发版-提交参数` | ESB |
| 7 | download-resources | `发版-下载资源` | 主业务系统 |
| 8 | finalize-release | `发版-汇总` | ESB |
| 9 | package-master | `发版-打包` | Jenkins（API） |

顶层触发词：`发版-启动 <版本号>`（自动判别版本/补丁模式）

## 数据目录约定

- 运行时：`/Users/wangjie/ai-hub/skills/iho-delivery-flow/.runtime/`
  - `state.json`、`plans/*.json`、`results/*.json`、`storageState/*.json`、`config.yaml`、`anchors/*.json`
- 交付物归档：`/Users/wangjie/ai-hub/skills/iho-delivery-flow/collect-archie/<yyyyMMdd>_<版本号>/`

## 待用户处理事项

1. 审阅 `requirements.md`、`design.md` 两份文档
2. 如需修改：指出章节和期望的修改方向
3. 审阅通过后：回复「继续」或「进入 tasks」，触发 `writing-plans` skill

## 新会话快速恢复指引

在新会话中：

1. 先读本文件（`progress.md`）了解进度
2. 读 `requirements.md` 与 `design.md` 掌握当前约定
3. 根据用户反馈：
   - 若要求修改 → 回到第 4/7 步改文档后再跑一次自审
   - 若审阅通过 → 调用 `writing-plans` skill 生成 `tasks.md`
4. **不要**直接进入实现类 skill，必须先出 `tasks.md`

## Memory 提示

本项目根目录为 `/Users/wangjie/workspace/project/iho-mrms`，但本次任务的产物全部落在 `/Users/wangjie/ai-hub/skills/iho-delivery-flow/` 下（skill 本身的构建）。注意两处路径分离。
