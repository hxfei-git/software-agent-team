---
name: codex-software-worker
description: Use inside Claude Code subagents when they need to delegate software engineering work through openai/codex-plugin-cc.
---

# Codex Software Worker Skill

本 skill 规定软件团队子 Agent 如何通过 `openai/codex-plugin-cc` 把工作委托给 Codex。

该插件的核心接口是 Claude Code slash commands：

- `/codex:setup`：检查 Codex CLI 是否已安装并登录。
- `/codex:rescue`：把调查、实现、修复、继续工作委托给 `codex:codex-rescue` 子 Agent。
- `/codex:review`：对当前 git 工作区或分支做只读代码审查。
- `/codex:adversarial-review`：做可指定关注点的挑战式只读审查。
- `/codex:status`：查看当前仓库中运行中和最近的 Codex job。
- `/codex:result`：读取已完成 job 的完整结果。
- `/codex:cancel`：取消运行中的后台 job。

---

## 前置检查

在第一次使用 Codex 前，主 Agent 或当前子 Agent 应确认插件可用：

```text
/codex:setup
```

如果 setup 提示 Codex 未登录，按提示执行：

```text
!codex login
```

如果用户已在本机配置 Codex CLI、自定义 API key、base URL 或 `.codex/config.toml`，插件会复用同一套 Codex CLI 配置。

---

## 何时使用哪个命令

### 开发 / 修复：使用 `/codex:rescue`

用于：

- 实现一个开发任务
- 调查 bug
- 尝试修复测试失败
- 继续上一次 Codex 工作
- 让 Codex 做一轮更快或更便宜的实现尝试

常用语法：

```text
/codex:rescue --background {任务说明}
/codex:rescue --wait {任务说明}
/codex:rescue --resume {继续上一轮工作的说明}
/codex:rescue --fresh {新任务说明}
/codex:rescue --model gpt-5.4-mini --effort medium {任务说明}
/codex:rescue --model spark {任务说明}
```

规则：

- 长任务默认用 `--background`。
- 小而明确的修复可以用 `--wait`。
- 继续同一 Codex 线程用 `--resume`。
- 强制开新线程用 `--fresh`。
- 不主动指定 `--model` 或 `--effort`，除非用户或主 Agent 明确要求。
- 用户说 `spark` 时，插件会映射到 `gpt-5.3-codex-spark`。

### 普通代码审查：使用 `/codex:review`

用于只读审查当前未提交改动或相对基准分支的变更。

```text
/codex:review --background
/codex:review --wait
/codex:review --base main --background
```

规则：

- 该命令只读，不修复代码。
- 不支持自定义关注点；需要关注点时使用 `/codex:adversarial-review`。

### 挑战式审查：使用 `/codex:adversarial-review`

用于质疑实现方向、架构选择、风险假设和失败模式。

```text
/codex:adversarial-review --background look for race conditions and data loss risks
/codex:adversarial-review --base main --background challenge the caching and retry design
/codex:adversarial-review --wait question whether this abstraction is necessary
```

规则：

- 该命令只读，不修复代码。
- 可以在 flags 后追加 focus text。
- 适合交付前、高风险模块、架构争议、可靠性/安全性风险。

### 后台任务管理

```text
/codex:status
/codex:status {job-id}
/codex:result
/codex:result {job-id}
/codex:cancel
/codex:cancel {job-id}
```

规则：

- `--background` 启动的任务完成前，用 `/codex:status` 查看。
- 完成后用 `/codex:result` 获取完整结果。
- 如需中止，用 `/codex:cancel`。

---

## 子 Agent 使用约束

### sw-developer

开发工程师可以使用：

```text
/codex:rescue --background {开发或修复任务}
```

或对小任务使用：

```text
/codex:rescue --wait {开发或修复任务}
```

开发工程师可以让 Codex 产生或修改代码，但必须：

1. 核对 Codex 输出是否符合 `requirements.md`、`architecture.md`、`dev-plan.md`。
2. 检查是否越权改动无关文件。
3. 运行必要自测。
4. 只向主 Agent 返回修改文件、命令结果和摘要。

### sw-tester

测试工程师优先自己运行测试。需要 Codex 辅助分析失败时使用只读语义：

```text
/codex:rescue --wait investigate this test failure without editing files: {失败摘要和报告路径}
```

测试工程师不能要求 Codex 修改业务代码。需要审查当前工作区时，可使用：

```text
/codex:review --background
```

### sw-code-reviewer

代码审查工程师使用只读命令：

```text
/codex:review --background
/codex:adversarial-review --background {关注点}
```

不能使用 `/codex:rescue` 来修复代码。若发现问题，写入审查报告，让主 Agent resume `sw-developer` 修复。

### sw-architect

架构师可以用 `/codex:rescue` 做代码库调查或方案辅助，但任务必须是分析型：

```text
/codex:rescue --wait analyze this repository and suggest module boundaries without editing files
```

架构师不应让 Codex 直接改业务代码。

### sw-delivery-reviewer

交付审查工程师优先读项目文件和报告。必要时使用：

```text
/codex:adversarial-review --background check whether the delivered changes satisfy the stated acceptance criteria
```

交付审查阶段不直接修代码。

---

## 标准委托提示词

把任务交给 Codex 时，使用以下结构：

```markdown
项目目录：{PROJECT_DIR}
角色：{开发/测试/审查/架构/交付}
任务：{任务ID} {任务标题}

目标：
{一句话说明要 Codex 完成什么}

必读文件：
- {PROJECT_DIR}/requirements.md
- {PROJECT_DIR}/architecture.md
- {PROJECT_DIR}/dev-plan.md
- {PROJECT_DIR}/testing-strategy.md
- {相关源码或测试路径}

允许修改：
- {仅开发工程师填写允许修改范围；只读角色写“无”}

禁止修改：
- {无关目录、生成物、锁文件等}

必须遵守：
- 现有代码风格
- requirements.md 的验收标准
- architecture.md 的模块边界
- testing-strategy.md 的验证要求

期望输出：
- 修改文件列表或审查发现
- 必跑命令
- 风险提示
- 后续建议
```

---

## 开发任务推荐模板

```text
/codex:rescue --background 项目目录：{PROJECT_DIR}
任务：{任务ID} {任务标题}

请实现该任务。必须阅读 requirements.md、architecture.md、dev-plan.md、testing-strategy.md，以及相关源码。

允许修改：
- {允许修改范围}

禁止修改：
- {禁止修改范围}

完成后请返回：
1. 修改文件列表
2. 自测命令和结果
3. 风险或未完成项
```

---

## 修复任务推荐模板

```text
/codex:rescue --resume 项目目录：{PROJECT_DIR}
任务：{任务ID} {任务标题}

请根据以下失败报告修复问题：
- {test-report.md}
- {review-report.md}

要求：
1. 一次性修复所有 FAIL 项。
2. 不修改无关文件。
3. 运行必要自测。
4. 返回修改文件列表和测试结果。
```

---

## 结果处理

Codex 返回后，当前子 Agent 必须：

1. 如果是后台任务，先用 `/codex:status` 确认完成，再用 `/codex:result` 获取结果。
2. 检查 Codex 是否修改或建议修改了越权文件。
3. 检查结果是否满足当前任务验收标准。
4. 运行或记录必要验证命令。
5. 将测试/审查/交付结论写入对应报告。
6. 只向主 Agent 返回精简摘要，不粘贴 Codex 完整输出。

---

## 禁止事项

- 测试、审查、交付角色不得让 Codex 修改业务代码。
- 不要把 `/codex:review` 当成可修复命令。
- 不要在无明确需求时启用长期后台任务。
- 不要把整个项目无边界地交给 Codex。
- 不要把 Codex 长输出复制到主 Agent 上下文。
