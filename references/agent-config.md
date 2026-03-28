# Agent Config

## 角色分工

### 皮皮

- 身份：主持人
- 职责：
  - 确认需求和边界
  - 准备会议资料
  - 主持多轮讨论
  - 质疑缺少依据的观点
  - 每轮做总结
  - 判断是否收敛
  - 输出决议与摘要
  - 归档同步
  - 向用户发送摘要和链接
- 多轮机制：使用当前 session 持续维护上下文

### Kimi-K2.5

- 适用：快速整理资料、形成结构化分析、参与多轮观点攻防
- 多轮机制：`kimi -p "..." -S <session-id>`
- 注意事项：
  - 复用同一个 `session-id` 维护会议上下文
  - `kimi -C` 会续接当前目录下的上一次 session
  - `kimi -S <id>` 会显式指定 session，会议场景推荐固定使用 `-S <meeting-id>` 保证多轮连续
  - 明确要求其输出观点、逻辑、支持材料三段式内容
  - 在第 2 轮起要求其针对其他 agent 逐项回应

示例：

```bash
kimi -p "你正在参加 AI Council 第 2 轮讨论。请先回应其他 agent 的关键观点，再给出你的更新判断。输出必须包含：1. 观点 2. 思考逻辑 3. 支持性文件或依据" -S <session-id>
```

### GPT（Codex）

- 适用：项目制会议、编码实现、架构判断、代码审阅、执行规划
- 模型选择：
  - 项目制会议：优先 GPT Codex
  - 一般议题会议：优先 GPT Codex
- 多轮机制：PTY + send-keys 持续交互
- 注意事项：
  - 保持同一个交互会话，避免丢失前几轮上下文
  - 明确要求引用相关文件、代码路径、报错信息等证据

控制要点：

- 在项目制会议中，要求 Codex 明确指出：
  - 涉及的文件
  - 可能的实现路径
  - 风险与验证方式
- 在一般议题会议中，要求 Codex 明确指出：
  - 判断依据
  - 备选方案
  - 取舍理由

### MiniMax-M2.5

- 适用：一般议题会议、发散思考、补充不同表达与判断视角
- 多轮机制：API `chatcompletion`，通过 `messages` 数组维护上下文
- 注意事项：
  - 每轮调用时都附上之前的关键上下文
  - 不要只保留上一轮，应至少保留会议目标、材料包摘要、前轮共识与分歧
  - 适合承担补充视角和分歧检查角色

维护建议：

```json
[
  {"role":"system","content":"你正在参加 AI Council，请按观点/逻辑/支持材料格式发言。"},
  {"role":"user","content":"会议目标与材料包摘要"},
  {"role":"assistant","content":"第 1 轮发言"},
  {"role":"user","content":"第 2 轮任务：回应其他 agent 并更新判断"}
]
```

### Gemini

- 适用：按项目情况补充外部视角或不同风格的归纳能力
- 是否参会：可选
- 使用原则：
  - 仅在需要额外观点、多模型交叉验证或复杂开放式问题时加入
  - 若加入，仍必须遵守每轮三段式输出要求

## 会议类型到阵容映射

### 项目制会议

- 默认阵容：皮皮 + Kimi + GPT Codex
- 可选补充：MiniMax-M2.5、Gemini
- 重点关注：
  - 技术可行性
  - 文件与代码证据
  - 实现路径
  - 风险与验证

### 一般议题会议

- 默认阵容：皮皮 + Kimi + GPT Codex
- 标准扩展：MiniMax-M2.5
- 可选补充：Gemini
- 重点关注：
  - 观点多样性
  - 判断依据
  - 取舍逻辑
  - 分歧收敛

## 发言协议

要求所有 agent 统一遵守以下格式：

```md
## 观点

## 思考逻辑

## 支持材料
```

第 1 轮额外要求：

- 先独立输出，不先阅读其他 agent 的发言摘要。
- 明确写出“当前我最不确定的点”。
- 如果已有倾向结论，也要写出至少一个反例或反对理由。

从第 2 轮开始，额外要求：

```md
## 对其他观点的回应
- 确认哪些观点
- 反驳哪些观点
- 补充哪些遗漏
```

从第 2 轮开始，强制补充：

```md
## 仍保留的分歧 / 异议 / 未决项
- 我仍不同意什么
- 我认为还缺什么证据
- 哪个结论暂时只能条件成立
```

## 伪共识防护

### 机制一：先独立输出，再交换

- 第 1 轮只向各 agent 发送同一份材料包和问题，不发送其他人答案。
- 主持人收齐或达到超时阈值后，再汇总进入第 2 轮。
- 若某 agent 第 1 轮缺席，后续只能读取状态摘要补位，不能倒逼其他人重做独立轮。

### 机制二：每轮强制保留异议

- 主持人不得接受只有“我同意前面观点”的发言。
- 每位 agent 每轮都要显式写出：
  - 仍保留的异议
  - 证据不足点
  - 结论成立的前提

### 机制三：检查是否出现实质性新增分歧

主持人每轮结束后都要判断：

- 本轮是否新增了之前没有被说清的分歧
- 分歧是否只是措辞变化
- 是否只是快速附和，没有新增证据或反证

如果没有新增分歧，也没有新增证据，不得轻易判定为收敛。

### 机制四：状态对象显式记录争议度

主持人必须在共享状态中维护：

- `controversy.score`
- `controversy.level`
- `controversy.new_substantive_disagreement`

用于区分：

- 真收敛：分歧减少且证据增加
- 假收敛：分歧表述减少，但证据没有增加

## 主持人检查清单

- 该 agent 是否明确回答了本轮问题。
- 该 agent 是否提供了完整逻辑链。
- 该 agent 是否给出支持性文件、证据或依据。
- 该 agent 是否回应了其他 agent 的关键观点。
- 该发言是否形成了新增共识或新增分歧。
- 该 agent 是否显式写出了仍保留的异议或未决项。
- 本轮是否出现了新增实质性分歧。
- 如果大家看起来趋同，是否有新增证据支撑这种趋同。

## 异常处理

- 某 agent 输出过短：要求补充观点依据和逻辑。
- 某 agent 未提供支持材料：暂停进入下一轮，要求补证。
- 某 agent 偏题：重申会议边界并要求重答。
- 某 agent 与前文明显矛盾：要求显式说明修正原因。
- 某轮信息过长：由主持人先压缩为“共识 / 分歧 / 待答问题”摘要，再继续下一轮。
- 某 agent 超时超过 5 分钟：重试一次，仍失败则按 `references/degradation.md` 降级处理。
- 某 agent 调用失败或 session 失效：先尝试恢复原会话，失败后以最新共享状态重拉；仍失败则降级。
- 多人会议只部分返回：活跃人数不少于 3 时可继续，但必须在状态对象与轮次总结中记录缺席情况。

## 各 Agent CLI 实战踩坑与解决方案

### Kimi CLI

调用命令模板：

```bash
kimi -p "你正在参加 AI Council 第 N 轮讨论。请先回应其他 agent 的关键观点，再给出你的更新判断。输出必须包含：1. 观点 2. 思考逻辑 3. 支持性文件或依据" -S <session-id> --final-message-only
```

踩坑：

1. `--final-message` 不存在，正确参数是 `--final-message-only`
2. 文件不能作为位置参数传给 `--print` 模式，只能放在工作目录中让 Kimi 自己读取
3. `session` 机制用 `-S <id>` 实现多轮共享上下文；`-C` 是续接当前目录下的上一次 session，不适合多会议并行时做稳定标识
4. 带模型参数 `-m kimi-coding/k2p5` 时可能报 `LLM not set` 错误（已遇到一次）

解决方案：

- 会议场景默认使用 `-S <meeting-id>` 指定会议 session ID，每轮用同一个 id
- 仅在临时续接当前目录最近一次会话时使用 `-C`
- 使用 `--final-message-only` 去掉思考过程，减少 token 消耗
- 需要传入文件时，先 `cp` 到工作目录

### Codex CLI（GPT）

调用命令模板：

```bash
# PTY 模式启动（推荐用于多轮会议）
/Applications/Codex.app/Contents/Resources/codex \
  -m gpt-5.4 \
  --dangerously-bypass-approvals-and-sandbox \
  --search

# 多轮通过 PTY send-keys 逐条发送，每条消息都是完整的一轮发言请求
# 注意：不能加 | clean-ansi.sh 管道（会导致 "stdout is not a terminal" 报错）
# ANSI 清理由皮皮在日志解析时处理，不影响 Codex 运行
```

多轮交互示例：

```bash
# 1. 启动 Codex PTY 会话
exec(pty: true, background: true, command: "/Applications/Codex.app/Contents/Resources/codex -m gpt-5.4 --dangerously-bypass-approvals-and-sandbox --search", workdir: "/tmp/ai-council-<meeting-id>")

# 2. 等待首次提示符/信任确认，发送 Return
process(action: send-keys, keys: ["Return"])

# 3. 发送第 1 轮问题
process(action: send-keys, data: "你正在参加 AI Council 第 1 轮讨论...")

# 4. 等待输出完成，发送第 2 轮
process(action: send-keys, data: "第 2 轮：请回应以下其他 agent 的观点...")

# 5. 结束时 Ctrl+C
process(action: send-keys, keys: ["C-c"])
```

性能策略：
- **联网搜索密集型议题**：exec timeout 设为 600s+（Codex 搜索较慢）
- **非搜索型议题**：exec timeout 300s
- Codex 响应较慢，通常 2-5 分钟才有完整输出

踩坑：

1. 必须在 git repo 中运行，否则拒绝执行。临时目录需要 `mktemp + git init`
2. **默认使用 `--dangerously-bypass-approvals-and-sandbox`**，不在 sandbox 只读环境工作，不要怕犯错
3. 需要 PTY（不能纯 exec），多轮通过 PTY send-keys
4. 首次进入新目录弹出信任确认，需额外 Return
5. PTY 输出含大量 ANSI 转义码，需要在采集后单独清理；不要直接把 Codex 启动命令接到 `| ~/.openclaw/scripts/clean-ansi.sh`
6. `send-keys` 只能发文本，不能传文件 / 图片
7. Codex 会自动搜索网页验证事实，web search 日志会混入结果
8. 多模态全被限制：CLI 纯文本 stdin / stdout，无法传图 / 音频 / 视频
9. 无 `--print` 模式做非交互多轮，exec 是一次性
10. **`--dangerously-bypass-approvals-and-sandbox` 与 `--full-auto` 互斥**，同时使用会报错 `the argument '--full-auto' cannot be used with '--dangerously-bypass-approvals-and-sandbox'`
11. **`--reasoning-effort` 参数在顶层 CLI 不存在**，已实测报错 `unexpected argument '--reasoning-effort'`，不要使用
12. **ClashX 代理**（`http://127.0.0.1:7890`）只是当前 Mac mini 环境的本地配置，通过 `~/.zshenv` 继承；这不是通用推荐，部署到其他 Mac/Linux 环境时应先确认代理需求，不要默认写死
13. exec 默认 timeout 不够（OpenClaw 默认可能 60s），复杂任务需要 300s+；联网搜索密集型议题需要 600s+
14. 响应较慢，可能 2-5 分钟才有完整输出
15. **PTY 管道模式限制**：`codex | clean-ansi.sh` 会让 Codex 检测到非终端而报错 `stdout is not a terminal`，ANSI 清理只能在皮皮侧处理，不能在 agent 调用时用管道
16. **`-y` 不是 Codex CLI 参数**，实测会报错 `unexpected argument '-y'`，不要出现在任何命令模板里

解决方案：

- 工作目录：每次会议用 `mktemp -d && git init` 创建临时工作区
- 写文件：需要可写工作区时使用 `--dangerously-bypass-approvals-and-sandbox`
- 授权策略：会议期间默认使用 `--dangerously-bypass-approvals-and-sandbox`，不要同时加 `--full-auto`
- 长对话：PTY 模式 + send-keys 逐条发送，不是 exec 一次性
- 输出清理：不要在 Codex 启动命令后直接接 `| ~/.openclaw/scripts/clean-ansi.sh`。正确做法是先采集 PTY 原始输出，再由主持人在采集层、日志解析层或归档前统一清理
- 代理：如果当前就是那台配置了 ClashX 的 Mac mini，可确认 `~/.zshenv` 中有 `http_proxy` / `https_proxy`；其他环境不要默认依赖这个配置
- 超时：exec timeout 设为 300+
- 信任确认：首次进新目录会多一个 Return

### Gemini CLI

调用命令模板：

```bash
gemini -y 2>&1 | ~/.openclaw/scripts/clean-ansi.sh
```

踩坑：

1. 没有 `-S session ID` 机制，只能用 `--resume latest` 或 `--resume <index>`
2. sandbox 限制：只能读取 workspace 目录和 gemini tmp 目录内的文件，外部文件需先复制进来
3. `-y` 必须加，否则会卡在确认提示（PTY 下才能交互确认）
4. 图片理解：将图片放在 workspace 目录下，在 prompt 中引用路径即可

解决方案：

- 长对话用 `--resume latest` 续接
- 外部文件先 `cp` 到工作目录
- 始终加 `-y` 参数

## 会议结束资源清理（新增强制步骤）

在会议流程最后一步（发布决议之后），必须执行以下清理：

1. 检查并杀掉所有本次会议启动的后台进程
2. 清理临时工作目录
3. 清理 Kimi 一次性 session（如使用非持久 session-id）
4. 归档共享状态文件（`state.json`）到会议记录目录
5. 确认内存释放

具体命令模板：

```bash
# 1. 杀掉本次会议启动的后台进程（记录在会议状态对象 pids 字段中）
for pid in $MEETING_PIDS; do kill $pid 2>/dev/null; done

# 2. 清理临时工作目录（归档后清理）
rm -rf /tmp/ai-council-<meeting-id>

# 3. Kimi session 处理
#    - 持久 session（如 -S ai-council-opc-20260328）：保留，供后续续接或复盘
#    - 一次性 session（未指定 -S）：无需额外清理，Kimi 自动管理

# 4. 归档状态文件（会议正式结束后归档，不要在会议中途清理）
ARCHIVE_DIR=~/.openclaw/workspace/memory/council-archives/<meeting-id>/
mkdir -p "$ARCHIVE_DIR"
cp /tmp/ai-council-<meeting-id>/state.json "$ARCHIVE_DIR/state.json"
# 如果有 agent 输出文件，也一并归档
cp /tmp/ai-council-<meeting-id>/*.md "$ARCHIVE_DIR/" 2>/dev/null

# 5. 验证：确认无残留进程
ps aux | grep -E 'kimi|codex|gemini' | grep -v grep || echo "无残留进程"
```

⚠️ 清理顺序：先归档再删除，不要反过来。
```
