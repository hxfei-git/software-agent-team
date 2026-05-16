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

把下面内容发给 Claude Code，替换项目名和需求。团队会先整理 `requirements.md` 并复述理解，等你确认后再自动开发。

```text
请使用 master_agent.md 中的软件项目多智能体交付系统，从头完成当前项目。

项目名：my-project
PROJECT_DIR=/home/ubuntu/1.project/my-project

项目需求：
{在这里写需求}

明确不做：
{可选：写不希望实现的功能、行为或交付物}

技术约束：
{可选：写技术栈、运行环境、数据格式、接口、兼容性等约束}

验收标准：
{在这里写验收标准}

请先不要开发。先把我的原始需求整理为 requirements.md，保留原始需求摘录，写出团队理解版需求，并为每条需求和验收标准分配 Rxx / Axx ID。
然后用中文向我展示简短确认摘要：项目目标、使用场景、功能范围、非目标范围、输入输出或接口边界、技术约束、交付物、验收标准、风险与假设。
等我明确回复“确认”后，再进入自动交付模式，启动架构、开发、测试、审查和交付流程。
```


## 运行中监控面板

建议另开一个 SSH 窗口，用下面脚本观察团队是否在工作。

```bash
while true; do
  clear
  echo "== files =="
  find . -maxdepth 3 -type f | sort
  echo
  echo "== team-log tail =="
  tail -30 team-log.md 2>/dev/null
  echo
  echo "== subagents =="
  find ~/.claude/projects/-home-ubuntu-1-project-todo-lite-2 -name "agent-*.meta.json" 2>/dev/null | sort | tail -10
  sleep 5
done
```
