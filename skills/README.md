# skills

## 用途

`skills` 是整个仓库的真源核心，所有可分发的 skill 都从这里读取，再由 `bin/publish-skills/publish-skills` 生成到 `dist/`。

## 目录内容

- `.system/`：系统来源的通用技能，目前可见 `skill-creator`、`skill-installer` 等，也保留了未启用但已落盘的系统 skill。
- `superpowers/`：对开源项目superpowers做了中文化处理，通用流程型技能，如 `using-superpowers`、`systematic-debugging`、`verification-before-completion`。
- `iho-java/`：IHO Java 项目开发规范。
- `iho-db-pgsql/`：IHO PostgreSQL DDL 设计规范。
- `spec/`：简易版的需求、设计、任务三段式 spec 技能与模板。
- `wangjie-defaults/`：全局默认行为规范。

单个 skill 目录通常包含：

- `SKILL.md`：技能入口与执行规则
- `references/`：参考资料
- `agents/`：技能专用代理定义
- `scripts/`、`assets/`、`resources/`：可复用脚本、素材或模板

## 用法

1. 新增或修改 skill 时，先在这里维护真源文件。
2. 如需让 skill 对某个工具生效，再更新仓库根目录的 `registry.yaml`。
3. 完成后同步分发产物：

```bash
/Users/wangjie/ai-hub/bin/publish-skills/publish-skills
```

4. 最后用诊断命令确认 `dist/` 与安装目录状态：

```bash
/Users/wangjie/ai-hub/bin/doctor/doctor
```

## 注意事项

- 是否进入某个工具的分发结果，不由目录存在与否决定，而由 `registry.yaml` 的 `enabled` 配置决定。
- 不要直接改 `dist/skills`；那只是从这里复制出去的产物。
