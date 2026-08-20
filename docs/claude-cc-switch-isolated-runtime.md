# 使用 cc-switch 配置运行 Claude Code

本指南适用于需要在非交互式、隔离环境中调用 `claude` 的任务，例如 BMad Eval Runner、自动化回归测试或一次性批处理。

## 问题：为什么 Claude 会显示“Not logged in”

cc-switch 将当前 Claude provider 的认证、基础端点和模型映射写入 `~/.claude/settings.json` 的 `env` 对象。

许多评测或自动化运行器会刻意创建干净的 `HOME` 和环境，以避免宿主机配置、记忆和凭据污染结果。这样做是正确的隔离策略，但也意味着 `claude` 不会自动读取宿主机的 `~/.claude/settings.json`，进而报错：

```text
Not logged in · Please run /login
```

不要通过把真实 `HOME` 直接交给隔离任务来修复此问题。这会破坏隔离性，并可能让评测读写用户的 Claude 状态。

## 安全模式

保留隔离运行器自己的干净 `HOME`，但从 cc-switch 配置中读取必要环境变量，并仅将它们转发给 Claude 子进程。

当前 cc-switch 的 Claude 配置通常包含：

- `ANTHROPIC_AUTH_TOKEN`：认证令牌；
- `ANTHROPIC_BASE_URL`：当前 provider 的 API 端点；
- `ANTHROPIC_DEFAULT_OPUS_MODEL`、`ANTHROPIC_DEFAULT_SONNET_MODEL`、`ANTHROPIC_DEFAULT_HAIKU_MODEL`：模型路由；
- 同名的 `*_NAME` 变量及可选的 `CLAUDE_CODE_ATTRIBUTION_HEADER`。

变量名可以安全检查；绝不要打印变量值、把它们写入适配器文件，或提交到 Git。

## 适配器配置

为评测创建一个不含密钥的 Claude Code 适配器，例如 `evals/adapter-cc-switch.json`：

```json
{
  "name": "claude-code-cc-switch",
  "invocation": [
    "claude",
    "-p",
    "{prompt}",
    "--output-format",
    "stream-json",
    "--verbose",
    "--dangerously-skip-permissions"
  ],
  "auth_env": "ANTHROPIC_AUTH_TOKEN",
  "transcript": { "format": "stdout-jsonl" },
  "skill_dir": ".claude/skills",
  "load_signal": { "skill_tool": "Skill", "read_tool": "Read" },
  "env_passthrough": [
    "ANTHROPIC_BASE_URL",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL_NAME",
    "ANTHROPIC_DEFAULT_OPUS_MODEL",
    "ANTHROPIC_DEFAULT_OPUS_MODEL_NAME",
    "ANTHROPIC_DEFAULT_SONNET_MODEL",
    "ANTHROPIC_DEFAULT_SONNET_MODEL_NAME",
    "CLAUDE_CODE_ATTRIBUTION_HEADER"
  ]
}
```

`auth_env` 指定运行器应转发的认证变量；`env_passthrough` 指定其余路由变量。若 cc-switch 增加新的必要变量，将变量名加入 `env_passthrough`，不要把值填入 JSON。

`--dangerously-skip-permissions` 只适用于由运行器创建的、受控的评测工作目录和可信夹具。不要将它用于不可信仓库或生产目录。

## 启动隔离任务

下例在当前进程中读取 cc-switch 环境配置，将值放入子进程环境，并立即启动评测器。脚本不输出配置值，也不会修改 `~/.claude/settings.json`。

```bash
python3 -c '
import json, os
from pathlib import Path

settings = json.loads(Path("~/.claude/settings.json").expanduser().read_text())
env = os.environ.copy()
env.update({key: str(value) for key, value in (settings.get("env") or {}).items()})

os.execvpe(
    "uv",
    [
        "uv", "run", "<runner-script>",
        "--adapter", "evals/adapter-cc-switch.json",
        "<other-runner-arguments>"
    ],
    env,
)
'
```

以 BMad Eval Runner 为例，将 `<runner-script>` 替换为 `.../bmad-eval-runner/scripts/run_evals.py`，并传入 `--cases`、`--skill-path`、`--output-dir` 和 `--mode` 等正常参数。

评测器随后仍会：

1. 为每个 case 建立干净的 `HOME`；
2. 分别运行带技能与裸模型的配置；
3. 仅转发适配器允许的 cc-switch 环境变量；
4. 将转录、计时和结果写入评测输出目录。

## 故障排查

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| `Not logged in · Please run /login` | 隔离进程未获得 cc-switch 认证变量，或适配器仍在期待 `ANTHROPIC_API_KEY`。 | 使用 `ANTHROPIC_AUTH_TOKEN` 作为 `auth_env`，并通过上述引导过程加载 `settings.json` 的 `env`。 |
| 已认证但模型或端点不对 | `ANTHROPIC_BASE_URL` 或默认模型变量未转发。 | 将所需变量名加入 `env_passthrough`。 |
| 评测读取了宿主机的历史或状态 | 运行器没有使用干净 `HOME`，或手动覆盖了它。 | 恢复隔离运行器的默认 `HOME` 策略；只转发必要环境变量。 |
| 日志中出现令牌或端点值 | 启动脚本、调试命令或异常处理打印了完整环境。 | 立即停止输出该信息；仅检查键名；轮换可能泄露的凭据。 |

## 操作准则

- 将 cc-switch 的配置视为凭据来源；读取它是为了启动受控子进程，而不是为了记录或展示其内容。
- 适配器 JSON 可以保存在项目中，因为它只有变量名；任何包含变量值的文件都不应提交。
- 若只需验证配置，先运行一个最小的非交互式 Claude 请求，再启动成本更高的评测。
- 评测完成后，从输出目录读取结果；不要把 cc-switch 的环境值写入转录、报告或构建日志。
