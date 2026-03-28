# 共享状态协议

## 目标

定义 AI Council 的共享状态对象，确保主持人与各参会 agent 在多轮讨论中始终对齐，并支持中断恢复、跨工具传递和最少 3 人 MVP 运行。

MVP 固定最低阵容：

- 皮皮（主持人）
- Kimi
- GPT Codex

可选扩展：

- MiniMax
- Gemini

## 设计原则

- 机器可读：优先 JSON，必要时可镜像为 YAML。
- 轮次驱动：每轮结束后必须更新一次状态对象。
- 主持人单写：只有皮皮有权写入正式状态，其他 agent 只读。
- 最小完备：既能支持 3 人会议，也能扩展到更多 agent。
- 可恢复：任意中断后，主持人可直接基于状态对象重启会议。
- 协议兼容：字段命名尽量贴近 ACP/A2A 常见的 `session`、`agents`、`artifacts`、`next_action` 思路。

## 状态文件位置建议

- 临时运行态：`./.ai-council/state.json`
- 归档快照：`./archives/YYYYMMDD_council_主题.state.json`

如果没有文件系统落点，也至少要在主持人的上下文中保留一份最新状态对象。

## 输出文件与目录约定

为避免会议过程只存在于 PTY 或 session 历史中，MVP 统一约定以下目录结构：

```text
<meeting-dir>/
├── .ai-council/
│   ├── state.json
│   ├── meeting-prompt.md
│   ├── orchestration-log.md
│   └── raw/
│       └── round-<N>-<agent-id>.raw.log
├── rounds/
│   ├── round-<N>-<agent-id>.md
│   └── round-<N>-summary.md
└── decisions/
    ├── final-resolution.md
    └── executive-summary.md
```

命名规则：

- agent 清洗后正文：`rounds/round-<N>-<agent-id>.md`
- 主持人轮次总结：`rounds/round-<N>-summary.md`
- agent 原始日志：`.ai-council/raw/round-<N>-<agent-id>.raw.log`
- 完整决议：`decisions/final-resolution.md`
- 决议摘要：`decisions/executive-summary.md`

`<agent-id>` 统一使用状态对象中的 `agents[].id`，例如：

- `pipi`
- `kimi`
- `gpt-codex`
- `minimax`
- `gemini`

落盘原则：

- 原始日志先落 `.raw.log`
- 清洗后的可读正文再落 `rounds/`
- 主持人必须在更新 `state.json` 前确认本轮文件已写入
- 不得以“终端里看过了”替代文件落盘

## 生命周期

### 1. 会前初始化

主持人创建状态对象，填入：

- 会议主题
- 边界
- 会议模式
- 参会名单
- 初始待答问题
- 资料包摘要

### 2. 每轮前广播

主持人在发起新一轮前，向所有 agent 广播状态摘要：

- 当前轮次
- 上轮共识
- 上轮分歧
- 本轮必须回答的问题
- 当前争议度

### 3. 每轮后更新

每轮结束后，主持人必须更新：

- `round.current`
- 各 agent 最新立场
- 本轮新增共识
- 本轮新增分歧
- 未决问题
- 下一步动作
- 是否继续会议
- 本轮落盘产物路径

### 4. 收敛或终止

当会议收敛、降级终止或人工中断时，写入最终状态：

- `meeting.status`
- `decision.summary`
- `decision.unresolved`
- `next_step`

## 状态对象字段

推荐 JSON 结构如下：

```json
{
  "protocol_version": "ai-council.state.v1",
  "compatible_with": ["ACP-like session envelope", "A2A-like participant handoff"],
  "meeting": {
    "id": "council-20260328-topic-slug",
    "topic": "会议主题",
    "type": "project|general",
    "mode": "light|standard|deep",
    "status": "preparing|in_progress|converged|degraded|stopped",
    "created_at": "2026-03-28T19:00:00+08:00",
    "updated_at": "2026-03-28T19:20:00+08:00",
    "host": "皮皮",
    "min_agents": 3,
    "boundary": {
      "in_scope": ["纳入讨论项1"],
      "out_of_scope": ["排除讨论项1"],
      "focus": ["风险", "实现路径"]
    }
  },
  "agents": [
    {
      "id": "pipi",
      "name": "皮皮",
      "role": "host",
      "status": "active",
      "required": true
    },
    {
      "id": "kimi",
      "name": "Kimi",
      "role": "member",
      "status": "active",
      "required": true
    },
    {
      "id": "gpt-codex",
      "name": "GPT Codex",
      "role": "member",
      "status": "active",
      "required": true
    }
  ],
  "materials": {
    "summary": "结构化材料包摘要",
    "key_files": ["path/to/file"],
    "evidence": ["文档A", "报错B"]
  },
  "artifacts": {
    "meeting_prompt": ".ai-council/meeting-prompt.md",
    "state_file": ".ai-council/state.json",
    "round_outputs": [
      {
        "round": 1,
        "agent_id": "kimi",
        "raw_log": ".ai-council/raw/round-1-kimi.raw.log",
        "clean_text": "rounds/round-1-kimi.md",
        "status": "written"
      },
      {
        "round": 1,
        "agent_id": "gpt-codex",
        "raw_log": ".ai-council/raw/round-1-gpt-codex.raw.log",
        "clean_text": "rounds/round-1-gpt-codex.md",
        "status": "written"
      }
    ],
    "round_summaries": [
      {
        "round": 1,
        "path": "rounds/round-1-summary.md"
      }
    ],
    "decisions": {
      "full": "decisions/final-resolution.md",
      "summary": "decisions/executive-summary.md"
    }
  },
  "round": {
    "current": 2,
    "max_rounds": 3,
    "history": [
      {
        "round": 1,
        "goal": "独立陈述立场",
        "new_consensus": ["共识1"],
        "new_disagreements": ["分歧1"],
        "must_answer_next": ["问题1"]
      }
    ]
  },
  "positions": {
    "pipi": {
      "stance": "当前主持判断",
      "confidence": "medium",
      "changed": false,
      "last_round": 2
    },
    "kimi": {
      "stance": "Kimi 当前立场",
      "confidence": "medium",
      "changed": true,
      "last_round": 2
    },
    "gpt-codex": {
      "stance": "Codex 当前立场",
      "confidence": "high",
      "changed": false,
      "last_round": 2
    }
  },
  "consensus": [
    {
      "id": "c1",
      "content": "已达成共识",
      "strength": "strong",
      "formed_in_round": 2
    }
  ],
  "disagreements": [
    {
      "id": "d1",
      "content": "尚未解决的分歧",
      "severity": "high",
      "owners": ["kimi", "gpt-codex"],
      "formed_in_round": 2,
      "status": "open"
    }
  ],
  "open_questions": [
    {
      "id": "q1",
      "question": "下轮必须回答的问题",
      "priority": "high",
      "owner": "all"
    }
  ],
  "controversy": {
    "score": 0.67,
    "level": "medium",
    "new_substantive_disagreement": true,
    "note": "第2轮出现新的关键实现分歧"
  },
  "next_step": {
    "action": "start_next_round",
    "reason": "仍有高优先级分歧未收敛",
    "target_round": 3
  },
  "decision": {
    "summary": "",
    "unresolved": []
  }
}
```

## 关键字段解释

- `agents`：参会 agent 列表。所有文档统一使用 `agents` 字段名。
- `mode`：会议模式，取值为 `light`、`standard`、`deep`。
- `min_agents`：允许继续开会的最少人数，MVP 固定为 `3`。
- `positions`：保存每个 agent 当前立场，而不是只保存原始发言。
- `artifacts`：保存本次会议的材料包、轮次输出、总结和决议文件路径，保证可恢复与可归档。
- `disagreements`：必须显式记录分歧归属，防止“看起来都同意”。
- `controversy.score`：争议度，范围建议为 `0` 到 `1`。
- `new_substantive_disagreement`：本轮是否真的产生了新的实质性分歧。
- `next_step.action`：建议值为 `start_next_round`、`request_evidence`、`degrade_and_continue`、`finalize`、`stop`.

## 争议度计算建议

主持人无需复杂数学模型，MVP 使用规则打分即可：

- 存在高重要性分歧：`+0.4`
- 两名及以上 agent 明确相互反驳：`+0.2`
- 关键问题无证据支撑：`+0.2`
- 仅措辞不同、结论一致：`+0`
- 本轮新增实质性分歧：额外 `+0.2`

区间建议：

- `0.00 - 0.19`：低
- `0.20 - 0.59`：中
- `0.60 - 1.00`：高

## 中断恢复协议

恢复时，主持人必须先读取最新状态对象，并按以下顺序恢复：

1. 读取 `meeting`、`agents`、`materials`。
2. 确认 `round.current` 与 `next_step`。
3. 检查 `artifacts.round_outputs` 与 `artifacts.round_summaries`，确认最近一轮文件是否完整。
4. 把 `consensus`、`disagreements`、`open_questions` 压缩成恢复摘要。
5. 向所有仍在线 agent 广播恢复摘要。
6. 仅从 `next_step.target_round` 继续，不重做已完成轮次。

恢复提示建议包含：

- 当前会开到第几轮
- 目前共识是什么
- 仍有哪些分歧
- 本轮只回答哪些问题

## 与 ACP / A2A 的兼容策略

不强绑定某个外部协议，但保持以下映射关系：

- `meeting.id` 对应会话或任务 ID
- `agents` 对应 agent registry
- `materials` 对应共享 artifacts
- `artifacts` 对应会议落盘产物与文件索引
- `round.history` 对应 message thread / turn log
- `next_step` 对应 handoff instruction

这样做的意义是：

- 将来可以直接封装成 ACP/A2A 的消息载荷
- 当前不依赖外部协议实现，先满足 skill 内部自洽

## 主持人强制动作

每轮结束后，皮皮必须执行以下动作：

1. 更新状态对象。
2. 判断争议度是否上升、下降或伪下降。
3. 检查是否出现“没有新增分歧但也没有新增证据”的假收敛。
4. 决定下一轮目标，或直接结束会议。

如果没有更新状态对象，不得进入下一轮。
