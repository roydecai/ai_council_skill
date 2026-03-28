# 第 2 轮审查反馈

## 必须修复（P0）

### 1. --full-auto 与 --dangerously-bypass-approvals-and-sandbox 互斥
**实测结果**：同时使用报错 `the argument '--full-auto' cannot be used with '--dangerously-bypass-approvals-and-sandbox'`
**当前文档**：agent-config.md 踩坑 #10 写的是"可并用"
**需要修正为**：两个参数互斥，不能同时使用。会议期间推荐使用 `--dangerously-bypass-approvals-and-sandbox`（因为需要写文件），放弃 `--full-auto`。

### 2. -y 不是 Codex CLI 参数
**实测结果**：报错 `unexpected argument '-y'`
**当前文档**：agent-config.md 命令模板中包含 `-y`
**需要修正为**：移除所有 `-y` 参数

### 3. 不能用管道接 clean-ansi.sh
**实测结果**：管道导致 `stdout is not a terminal` 报错
**需要修正为**：ANSI 清理在日志解析时处理，不在 agent 调用时管道接入

## 建议改进（P1）

### 4. workflow.md 缺少具体 shell 命令示例
在初始化步骤中补充实际可用的命令，例如：
```bash
MEETING_DIR=$(mktemp -d /tmp/ai-council-XXXXXX) && cd $MEETING_DIR && git init
mkdir -p .ai-council/raw rounds decisions
```

### 5. state-protocol.md status 枚举值未定义
需要明确列出：
- `agents[].status`: `active`, `timeout`, `degraded`, `absent`, `offline`
- `artifacts.round_outputs[].status`: `pending`, `writing`, `written`, `failed`, `timeout`

### 6. agent-config.md Kimi -C 续接说明不足
补充：`kimi -C` 续接当前目录的上一次 session，`kimi -S <id>` 指定 session。会议场景推荐用 `-S <meeting-id>` 保证多轮连续。

### 7. agent-config.md ClashX 代理是环境特定配置
标注为"仅限当前 Mac mini 环境"，不作为通用推荐。
