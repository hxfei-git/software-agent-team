# 软件项目多智能体交付系统

这是一套用于 Claude Code 的软件项目交付团队 harness。主 Agent 面向用户做项目经理和编排者，内部通过多个 Claude Code 子 Agent 分工完成架构、开发、测试、代码审查和交付审查。开发子任务可通过 `openai/codex-plugin-cc` 调用 Codex 处理。

## 团队角色

| 角色 | Agent | 职责 |
|---|---|---|
| 项目经理 / 编排者 | 主 Agent | 需求确认、任务调度、日志、状态管理、交付汇报 |
| 架构师 | `sw-architect` | 架构设计、任务拆分、测试策略 |
| 开发工程师 | `sw-developer` | 代码实现、修复测试反馈、调用 Codex |
| 测试工程师 | `sw-tester` | 运行测试、写测试报告，只读代码 |
| 代码审查工程师 | `sw-code-reviewer` | 质量、安全、边界条件审查，只读代码 |
| 交付审查工程师 | `sw-delivery-reviewer` | 验收映射、交付报告、风险汇总 |

## 目录结构

```text
software-agent-team/
  master_agent.md 或 主智能体提示词.md
  .claude/
    agents/
      sw-architect.md
      sw-developer.md
      sw-tester.md
      sw-code-reviewer.md
      sw-delivery-reviewer.md
    skills/
      codex-software-worker/SKILL.md
  agents/
  skills/
```

`.claude/` 是给 Claude Code 实际加载的目录；顶层 `agents/` 和 `skills/` 作为源文件和备份。

## 前置要求

服务器上需要：

- Claude Code
- Codex CLI
- `openai/codex-plugin-cc`
- Python/Node 等项目所需运行时

在 Claude Code 中安装并检查 Codex 插件：

```text
/plugin marketplace add openai/codex-plugin-cc
/plugin install codex@openai-codex
/reload-plugins
/codex:setup
```

如果提示 Codex CLI 未登录，在服务器 shell 中运行：

```bash
codex login
codex whoami
```

查看 Codex 默认模型：

```bash
cat ~/.codex/config.toml
```

## 在新项目中使用

假设团队 harness 位于：

```bash
/home/ubuntu/1.project/software-agent-team
```

创建一个新项目：

```bash
mkdir -p /home/ubuntu/1.project/my-project
cd /home/ubuntu/1.project/my-project
cp -r /home/ubuntu/1.project/software-agent-team/.claude .
cp /home/ubuntu/1.project/software-agent-team/master_agent.md .
```

启动 Claude Code。建议在可信 VM 或隔离环境中使用免批准模式，避免长任务中途卡住：

```bash
claude --permission-mode bypassPermissions
```

如果当前版本不支持，可使用：

```bash
claude --dangerously-skip-permissions
```

进入 Claude Code 后检查：

```text
/reload-plugins
/codex:setup
/agents
```

`/agents` 应能看到：

```text
sw-architect
sw-developer
sw-tester
sw-code-reviewer
sw-delivery-reviewer
```

## 推荐启动提示词

把下面内容发给 Claude Code，替换项目名和需求：

```text
请使用 master_agent.md 中的软件项目多智能体交付系统，从头完成当前项目。

项目名：my-project
PROJECT_DIR=/home/ubuntu/1.project/my-project

需求已确认，进入自动交付模式。PROJECT_DIR 内不需要再问我批准，也不需要中途等待我确认。
默认全程使用中文和我沟通；所有团队文档、日志、测试报告和交付报告也使用中文。代码、命令、文件名和错误信息保持原文。

强制要求：
1. 主 Agent 只做 PM、调度、日志和状态管理，不得直接写业务代码。
2. 必须使用 sw-architect 生成 architecture.md、dev-plan.md、testing-strategy.md。
3. 必须使用 sw-developer 完成开发任务。
4. sw-developer 必须通过 codex-software-worker skill 调用 /codex:rescue 委托 Codex 处理子任务。
5. 必须使用 sw-tester 写测试报告。
6. 必须使用 sw-code-reviewer 写审查报告。
7. 必须使用 sw-delivery-reviewer 生成 delivery-report.md。
8. 每次启动或完成子 Agent，都必须在 team-log.md 记录：任务ID、Agent名称、Agent ID、报告路径、Codex 调用证据。
9. 没有 Agent ID 和 Codex 调用证据的开发任务，不得标记完成。
10. 如果 Codex 或某个 Agent 无法调用，必须记录为阻塞项，但继续完成可执行部分，最终写入 delivery-report.md。

项目需求：
{在这里写需求}

验收标准：
{在这里写验收标准}
```

## 自动交付模式

需求确认后，主 Agent 进入自动交付模式：

- 在 `PROJECT_DIR` 内创建、修改、移动项目文件。
- 运行测试、构建、lint、typecheck、格式化等命令。
- 失败后自动 resume 开发 Agent 修复，最多 3 轮。
- 第 3 轮仍失败时标记 `⚠️ 低质量通过`，继续后续任务，并在交付报告中说明风险。
- 遇到账号、密钥、外部系统权限等无法自动解决的问题时，标记 `❌ 阻塞`，继续可执行任务。

禁止：

- 修改或删除 `PROJECT_DIR` 之外的用户文件。
- 把密码、token、密钥写入仓库或日志。
- 伪造测试通过、外部服务调用成功或交付证据。

## 运行中监控面板

建议另开一个 SSH 窗口，用下面脚本观察团队是否在工作。

将 `PROJECT_DIR` 和 `PROJECT_SLUG` 改成你的项目。路径 `/home/ubuntu/1.project/todo-lite-2` 对应的 Claude 项目 slug 通常是 `-home-ubuntu-1-project-todo-lite-2`。

```bash
cd /home/ubuntu/1.project/todo-lite-2

while true; do
  clear

  echo "== files =="
  find . -maxdepth 3 -type f | sort

  echo
  echo "== team-log tail =="
  tail -30 team-log.md 2>/dev/null

  echo
  echo "== dev-plan =="
  sed -n '1,120p' dev-plan.md 2>/dev/null

  echo
  echo "== test reports =="
  find test-reports -maxdepth 2 -type f -print 2>/dev/null | sort

  echo
  echo "== subagents =="
  find ~/.claude/projects/-home-ubuntu-1-project-todo-lite-2 \
    -name "agent-*.meta.json" 2>/dev/null | sort | tail -10

  echo
  echo "== subagent mentions: codex / sw-* =="
  grep -R "sw-architect\|sw-developer\|sw-tester\|sw-code-reviewer\|sw-delivery-reviewer\|/codex:rescue\|/codex:review\|/codex:adversarial-review\|codex" \
    ~/.claude/projects/-home-ubuntu-1-project-todo-lite-2 2>/dev/null | tail -20

  echo
  echo "== codex processes =="
  ps -ef | grep -E "codex|node.*codex" | grep -v grep || true

  echo
  echo "== codex recent files =="
  find ~/.codex -maxdepth 4 -type f -printf '%T@ %TY-%Tm-%Td %TH:%TM:%TS %p\n' 2>/dev/null \
    | sort -nr | head -15 \
    | cut -d' ' -f2-

  echo
  echo "== codex log tail =="
  tail -30 ~/.codex/log/codex-tui.log 2>/dev/null

  sleep 5
done
```

说明：

- `/codex:status` 是 Claude Code slash command，不能直接在普通 shell 中执行。
- shell 监控只能观察 Codex 进程、日志、最近文件和 subagent jsonl 中是否出现 `/codex:rescue`。
- 真正的 Codex job 状态需要在 Claude Code 里输入：

```text
/codex:status
/codex:result
```

## 如何确认团队 Agent 是否工作

### 1. 查看当前运行中的 Agent

在 Claude Code 中：

```text
/agents
```

如果显示：

```text
No subagents are currently running.
```

只代表当前没有正在运行的子 Agent，不代表之前没跑过。

### 2. 查看历史子 Agent 文件

```bash
find ~/.claude/projects/-home-ubuntu-1-project-todo-lite-2 \
  -name "agent-*.meta.json" -o -name "agent-*.jsonl" 2>/dev/null | sort
```

查看最近输出：

```bash
for f in ~/.claude/projects/-home-ubuntu-1-project-todo-lite-2/*/subagents/agent-*.jsonl; do
  echo "==== $f"
  tail -80 "$f"
done
```

搜索团队角色和 Codex 调用：

```bash
grep -R "sw-architect\|sw-developer\|sw-tester\|sw-code-reviewer\|sw-delivery-reviewer\|/codex:rescue\|/codex:review\|/codex:adversarial-review" \
  ~/.claude/projects/-home-ubuntu-1-project-todo-lite-2 2>/dev/null | tail -120
```

### 3. 查看项目证据

```bash
cd /home/ubuntu/1.project/todo-lite-2
tail -80 team-log.md
cat dev-plan.md
find test-reports -maxdepth 2 -type f -print 2>/dev/null
cat delivery-report.md 2>/dev/null
```

可信的完成证据应包括：

- `team-log.md` 记录每个阶段。
- `dev-plan.md` 填入 `DEV_ID`、`TEST_ID`、`REVIEW_ID`。
- `test-reports/` 下有测试和审查报告。
- `delivery-report.md` 有最终判定和验收映射。
- Claude subagent jsonl 中能搜到 `sw-*` 和 `/codex:*` 记录。

## Codex 使用方式

`codex-software-worker` skill 基于 `openai/codex-plugin-cc`：

- 开发/修复：`/codex:rescue`
- 普通审查：`/codex:review`
- 挑战式审查：`/codex:adversarial-review`
- 后台状态：`/codex:status`
- 结果读取：`/codex:result`

如果没有显式 `--model`，Codex 使用 `~/.codex/config.toml` 中的默认模型。例如：

```toml
model = "gpt-5.5"
model_reasoning_effort = "high"
```

## 常见问题

### 为什么一直问 Bash 是否批准？

当前 Claude Code 不是免批准模式。建议在可信 VM 中重启：

```bash
claude --permission-mode bypassPermissions
```

或：

```bash
claude --dangerously-skip-permissions
```

### 我按 Esc 中断了怎么办？

不要从头开始，先恢复状态：

```text
我刚才中断了自动执行。请不要从头开始，也不要主 Agent 直接写业务代码。
请读取 team-log.md、dev-plan.md、test-reports/ 和最近 subagent 结果，判断当前停在哪个任务，然后继续自动交付。
```

### 怎么默认中文？

在启动提示中加入：

```text
默认全程使用中文和我沟通；所有团队文档、日志、测试报告和交付报告也使用中文。代码、命令、文件名和错误信息保持原文。
```

### `/codex:status` 能在 shell 里跑吗？

不能。它是 Claude Code slash command，只能在 Claude Code 中输入。普通 shell 只能通过进程、日志、`.codex` 文件和 subagent jsonl 间接观察。

## 交付产物清单

每个项目完成后应至少包含：

```text
requirements.md
architecture.md
dev-plan.md
testing-strategy.md
lessons-learned.md
team-log.md
test-reports/
delivery-report.md
README.md
```

其中 `delivery-report.md` 是最终验收入口，`team-log.md` 是过程追踪入口。
