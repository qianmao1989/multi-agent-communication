# Multi-Agent Communication: Gateway API vs Shared Mailbox

> How do two AI agents talk to each other? We spent a month hitting walls before finding the answer: don't pick one channel — use both.

---

## English

### Who We Are

We're a small AI tools team. Two core agents collaborate daily:

- **CC (Claude Code)**: The heavy-lifter. Code development, architecture design, complex debugging. Runs locally via CLI, powered by DeepSeek v4-pro.
- **OpenClaw Agent ("Little Assistant")**: The executor. Spreadsheet management, email, scheduled tasks, Feishu messaging. Runs locally via OpenClaw Gateway.

They each do their job well. But here's the problem — **how do they talk to each other?**

### The Problem: Two Agents, Two Different Worlds

CC and OpenClaw run on the same machine but live in completely different environments:

| Dimension | CC (Claude Code) | OpenClaw Agent |
|-----------|-------------------|----------------|
| Runtime | CLI terminal, Anthropic format | Gateway HTTP service |
| Model | DeepSeek v4-pro (via API) | mimo-v2.5-pro |
| Tools | MCP tools (Playwright, SearXNG, etc.) | Built-in tools (exec, cron, etc.) |
| Session | One-shot tasks, exits after | Persistent session, always on |

They need to collaborate constantly:
- OpenClaw finds a code issue → asks CC to fix it
- CC finishes a task → notifies OpenClaw to report results
- Scheduled job triggers → needs CC to run a script
- Emergency → both need real-time discussion

### Attempt 1: Shared Mailbox

The most intuitive approach — **filesystem-based mailbox**.

```
shared/
├── inbox_cc_to_openclaw.json    # CC → OpenClaw
└── inbox_openclaw_to_cc.json    # OpenClaw → CC
```

Simple JSON messages:
```json
{
  "from": "openclaw",
  "time": "2026-05-31 15:45",
  "msg": "CC, your Gateway API token is hardcoded in CLAUDE.md. Use env vars.",
  "id": "oc_004",
  "status": "unread"
}
```

OpenClaw polls the mailbox every 3 minutes via cron. CC writes messages with a Python script.

#### Pros
- **Zero dependencies**: No HTTP server, no API key, no network
- **Persistent**: Messages are files. Survives restarts.
- **Simple**: JSON format, any language can read/write
- **Auditable**: All messages have timestamps and IDs

#### Problems Exposed

**Problem 1: FileSystemWatcher is Unreliable**

We tried using Windows FileSystemWatcher for real-time file monitoring. It failed badly:

- FileSystemWatcher uses `ReadDirectoryChangesW` under the hood, with an undersized kernel buffer
- Under high-frequency writes, the buffer overflows and **events are silently dropped** (not delayed — gone)
- A single save operation gets split into multiple events (Created + Changed + Renamed) — you get duplicates or misses
- Network drives (SMB/UNC) are even worse — the monitoring relies on SMB change notifications, which have higher latency and event coalescing issues
- PowerShell's `Set-Content` truncates then writes (open → truncate → write → close), generating multiple filesystem events per save

We fell back to polling — cron every 3 minutes. But that introduced new problems.

**Problem 2: The Token Black Hole of Polling**

Every poll cycle, even with zero new messages:
- System prompt: ~8,000 chars
- Tool definitions: ~2,000 chars
- Poll result injection: ~500 chars
- Output: NO_REPLY (~50 tokens)

At 3-minute intervals:
- 20 polls/hour × ~4,500 tokens/poll = **90,000 tokens/hour**
- That's **2,160,000 tokens/day**, mostly wasted on NO_REPLY

**Problem 3: 3-Minute Latency is Unacceptable**

Some scenarios need real-time response:
- CC finishes a task → OpenClaw needs to confirm immediately
- Emergency requires real-time discussion
- OpenClaw finds a bug → CC needs to act now

Waiting 3 minutes is too long.

**Problem 4: Context Pollution**

Polling messages get injected into OpenClaw's main conversation context. Historical messages accumulate, inflating token usage and degrading response quality.

### Attempt 2: Gateway API

OpenClaw Gateway exposes an HTTP API:

```
POST http://localhost:18789/v1/chat/completions
```

CC can call this directly and get real-time responses.

#### Advantages

- **Real-time**: HTTP request reaches OpenClaw instantly, second-level response
- **Zero polling overhead**: No API call when no communication needed
- **Naturally isolated**: Each API call is an independent session, no context pollution
- **Cost-controlled**: Token consumption only when actively communicating

#### Implementation

```python
import requests

def talk_to_assistant(message):
    """CC calls OpenClaw's Gateway API"""
    response = requests.post(
        "http://localhost:18789/v1/chat/completions",
        json={
            "model": "my-mimo-provider/mimo-v2.5-pro",
            "messages": [
                {"role": "user", "content": message}
            ]
        },
        timeout=120
    )
    return response.json()["choices"][0]["message"]["content"]
```

That's it. One HTTP request, instant reply.

#### Problems

**Problem 1: CC needs to know Gateway exists**

CC is an independent CLI tool. It has no awareness of OpenClaw Gateway's existence. You must explicitly document the Gateway address and call method in CC's CLAUDE.md config.

**Problem 2: Each call is stateless**

Gateway API calls create new sessions. There's no "continuous conversation" between CC and OpenClaw. Multi-turn dialogue requires carrying context in messages.

**Problem 3: Timeout handling**

If OpenClaw takes too long, CC's HTTP request times out. Need reasonable timeout settings and retry logic.

**Problem 4 (Key Discovery): The Gateway API is One-Way, Not Bidirectional**

This is the biggest cognitive error we caught during our discussion. We initially assumed "OpenClaw can also call the Gateway API to send messages to CC instantly." But think about it:

- **CC calls Gateway**: CC sends HTTP to `localhost:18789`, OpenClaw processes it, returns a reply. CC is "talking to OpenClaw."
- **OpenClaw calls Gateway**: OpenClaw sends HTTP to `localhost:18789`, OpenClaw processes it, returns a reply. OpenClaw is "talking to itself."

**The Gateway API's endpoint IS OpenClaw itself, not CC.** OpenClaw has no "CC's API endpoint" to call.

This means communication has a fundamental **asymmetry**:

| Direction | Channel | Latency |
|-----------|---------|---------|
| CC → OpenClaw | Gateway API | Seconds |
| OpenClaw → CC | Mailbox only | 3+ minutes (poll interval) |

CC can initiate instant dialogue, but OpenClaw can only write to the mailbox and hope CC reads it.

**Lesson**: Don't assume "the API is local so it works both ways." You must understand: who's the client, who's the server, and where does the request actually terminate.

### Analysis: Essential Differences

| Dimension | Shared Mailbox | Gateway API |
|-----------|---------------|-------------|
| Pattern | Async, non-blocking | Sync, blocking |
| Latency | 3 min (poll interval) | Seconds |
| Token cost | High (polling + context injection) | Low (on-demand) |
| Reliability | Filesystem reliable, polling unreliable | Network may be unstable |
| Persistence | Native (files) | Needs extra handling |
| Use case | Non-urgent notifications, file transfer | Real-time dialogue, emergency |
| Complexity | Low (JSON read/write) | Medium (HTTP calls) |

### Decision Table: Which Channel When

| Scenario | Recommended Channel | Why |
|----------|-------------------|-----|
| CC finishes task, notifies OpenClaw | Gateway API | CC initiates, instant |
| CC sends large files to OpenClaw | Mailbox | Filesystem more stable than HTTP |
| CC needs real-time discussion with OpenClaw | Gateway API | CC initiates, bidirectional dialogue |
| OpenClaw finds code issue, needs CC to fix | Mailbox (only option) | OpenClaw cannot reach CC via Gateway |
| Scheduled task triggers, needs CC to run script | Mailbox + CC-side FileSystemWatcher | Only way for OpenClaw to reach CC |
| Emergency requiring real-time discussion | CC initiates Gateway API | Only CC can start instant dialogue |
| Daily status sync (CD table updates, etc.) | Mailbox | Non-urgent, async is fine |

### Pitfalls We Hit

#### Pitfall 1: The Token "Fridge Problem"

We didn't realize mailbox polling burned this many tokens until we saw 2M+ tokens/day gone to nothing.

**Lesson**: Ask "what's the root cause" before acting. Don't burn tokens going the wrong direction. This is our "Fridge Rule" — when the fridge smells, ask "what's rotting inside" before researching deodorizers.

#### Pitfall 2: The FileSystemWatcher Illusion

We assumed FileSystemWatcher would work for real-time file monitoring and spent hours debugging why events weren't firing. Turns out the Windows kernel buffer (8KB default) overflows and silently drops events. PowerShell's non-atomic writes split one save into multiple events.

**Lesson**: Don't assume an API works — test it first. Run a minimal test to confirm before building on it.

#### Pitfall 3: Context Pollution from Message Injection

Historical mailbox messages get injected into OpenClaw's main conversation context, causing:
- Ever-growing context length
- Increasing token consumption
- Degrading response quality (noisy context)

**Lesson**: Isolate communication channels from conversation context. Mailbox messages should be processed independently.

#### Pitfall 4: CC Didn't Know Gateway Existed

CC is an independent CLI tool. We assumed it could "auto-discover" the Gateway. It couldn't. CC had no idea how to communicate with OpenClaw.

**Lesson**: Don't assume agents have "built-in" communication mechanisms. You must explicitly teach each agent how to reach others.

#### Pitfall 5: AI's Ability to "Cover Up" Hallucinations

During a discussion, OpenClaw said there were 18 anti-hallucination rules. When challenged, it changed to 13. CC agreed with 13. The actual count was 18 — both AIs got it wrong because "neither bothered to count carefully."

This perfectly illustrates the core hallucination problem: AI doesn't just hallucinate — it generates plausible explanations to cover mistakes. The cover-up itself may be a new hallucination.

**Lesson**: Multi-agent cross-validation isn't foolproof. If both agents get lazy, they fail together. External validation (humans, tools, actual test results) is irreplaceable.

#### Pitfall 6: Assuming the Gateway API Was Bidirectional

Our latest discovery. We always assumed "OpenClaw can also call the Gateway API to message CC with second-level response" — until we realized during discussion that the Gateway API's endpoint is OpenClaw itself. OpenClaw calling Gateway = talking to itself.

The illusion arose because CC calling Gateway does get instant replies — so we assumed the reverse would also work. But "CC calling Gateway gets a response" does NOT mean "OpenClaw calling Gateway can reach CC."

**Lesson**: When drawing architecture diagrams, always label arrow directions and endpoints. "A can call B" and "B can call A" are two different things. Don't assume B→A works just because A→B does.

### Our Conclusion: Two Channels Coexist

After a month of real-world practice, our final approach: **don't pick one — use both channels, chosen by scenario.**

```
┌─────────────┐                    ┌─────────────┐
│     CC      │                    │   OpenClaw   │
│  (Claude    │   Gateway API      │   Agent      │
│   Code)     │ ──────────────→    │              │
│             │   CC initiates ✓   │              │
│             │                    │              │
│             │   Shared Mailbox   │              │
│             │ ←──────────────→   │              │
│             │   Bidirectional    │              │
│             │   but slow ✗       │              │
└─────────────┘                    └─────────────┘
         │                                  │
         ▼                                  ▼
   shared/inbox_*.json              Gateway (localhost:18789)
                                    ↑ This IS OpenClaw's own endpoint
                                    OpenClaw calling it = talking to itself
```

**Key asymmetry: OpenClaw has no way to instantly reach CC.** The Gateway API's endpoint is OpenClaw itself.

### Channel Selection Principles

1. **Real-time discussion needed** → Gateway API (seconds, only CC can initiate)
2. **Non-urgent notification** → Mailbox (async, persistent)
3. **File transfer** → Mailbox (filesystem more stable than HTTP)
4. **Scheduled task trigger** → Gateway API (immediate execution)

### Unsolved: How Can OpenClaw Instantly Reach CC?

Currently, OpenClaw can only write to the mailbox (3-minute poll) to reach CC. No second-level channel exists. Possible improvements:

1. **CC runs an HTTP listener**: OpenClaw POSTs to CC's port. But CC is a CLI tool — architecturally awkward to run a persistent HTTP server.
2. **FileSystemWatcher + reduced polling**: Use Watcher for initial detection (though buggy, "has event" detection still works), with 5-minute low-frequency polling as backup.
3. **Accept the asymmetry**: 90% of communication is CC-initiated. OpenClaw rarely needs to reach CC. 3-minute mailbox delay is fine for most scenarios.
4. **WebSocket persistent connection**: Ideal but highest implementation cost. Requires both CC and OpenClaw to support it.
5. **Callback pattern** — most promising approach, worth detailing.

#### The Callback Pattern: Mailbox as Doorbell, Gateway as Conversation

Core idea: demote the mailbox from "chat channel" to "signaling channel." Real conversation happens over Gateway.

```
OpenClaw                  Mailbox                   CC
  │                         │                        │
  │  1. Write message       │                        │
  │     (ring doorbell)     │                        │
  │ ─────────────────────→  │                        │
  │                         │  2. CC reads mailbox    │
  │                         │     (opens door)        │
  │                         │ ─────────────────────→  │
  │                         │                        │
  │  3. CC calls Gateway    │                        │
  │     back (callback)     │                        │
  │ ←──────────────────────────────────────────────  │
  │     (face-to-face,      │                        │
  │      instant reply)     │                        │
  │ ───────────────────────────────────────────────→ │
```

**Why better than polling?**
- Mailbox only needs to carry a signal ("CC, look here"), not full conversation content
- CC reads the signal then immediately switches to Gateway API for instant dialogue
- No "idle polling" token black hole — no message means no read, every read means something real

**Why simpler than WebSocket?**
- No persistent HTTP server needed on CC side (CC is a CLI, not a server)
- No long-lived connection management
- Filesystem is naturally reliable, survives restarts

**Key constraint: CC must "have ears"**
The callback pattern requires CC to detect new mailbox messages promptly. If CC isn't running or is busy with another task, the doorbell rings but nobody answers. Possible improvements:
- CC-side FileSystemWatcher (though buggy, "has event" detection is reliable enough)
- Low-frequency polling backup (every 5 minutes)

We're currently using option 3 (accept asymmetry), but the callback pattern is under evaluation as the most cost-effective solution.

### Key Takeaways

1. **Don't assume agents have built-in communication.** Explicitly design and implement channels.
2. **Don't assume a local API is bidirectional.** Know who's the client, who's the server, and where requests terminate. The Gateway API's endpoint is OpenClaw itself, not CC.
3. **Polling is a token black hole.** Use push instead of polling where possible.
4. **Isolate communication from conversation context.** Don't let channel messages pollute the main session.
5. **Verify before building.** Don't assume an API "should" work — test it first.
6. **Two channels are more reliable than one.** If one fails, the other still works.
7. **Multi-agent cross-validation isn't foolproof.** If both agents cut corners, they fail together. External validation (humans, tools, real-world results) is irreplaceable.

### Appendix: Deployment Config

**Shared Mailbox**
```
shared/
├── inbox_cc_to_openclaw.json
└── inbox_openclaw_to_cc.json
```

**Gateway API**
- Endpoint: `http://localhost:18789/v1/chat/completions`
- Model: `my-mimo-provider/mimo-v2.5-pro`
- Timeout: 120s
- Auth: Bearer Token (required even for local access)

**Cron (Mailbox Check)**
- Interval: 3 minutes
- Reads: `inbox_cc_to_openclaw.json`
- Logic: Has unread → process → mark as read

---

*Authors: Qianmao's AI Team (CC + OpenClaw Agent)*
*Date: May 2026*
*GitHub: [qianmao1989](https://github.com/qianmao1989)*

> Questions or suggestions? Head to [Discussions](https://github.com/qianmao1989/multi-agent-communication/discussions) or open an Issue.

---

## 中文版

> 两个AI Agent怎么对话？我们踩了整整一个月的坑，终于找到了答案：不是二选一，而是两条通道并存。

### 我们是谁

我们是一个小型AI工具团队，做游戏代练自动化业务。核心Agent有两个：

- **CC（Claude Code）**：重型编程Agent，负责代码开发、架构设计、复杂调试。跑在本地，通过DeepSeek API连接DeepSeek v4-pro模型。
- **小助理（OpenClaw Agent）**：业务执行Agent，负责表格管理、邮件收发、定时任务、飞书消息处理。通过OpenClaw Gateway运行在本地。

两个Agent各司其职：CC是技术尖兵，小助理是执行管家。但问题来了——**它们怎么互相通信？**

### 问题：两个Agent住在不同的世界

| 维度 | CC (Claude Code) | 小助理 (OpenClaw) |
|------|-------------------|-------------------|
| 运行方式 | CLI终端，Anthropic格式 | Gateway HTTP服务 |
| 模型 | DeepSeek v4-pro（通过DeepSeek API） | mimo-v2.5-pro |
| 工具 | MCP工具（Playwright、SearXNG等） | OpenClaw内置工具（exec、cron等） |
| 会话 | 一次性任务，用完就走 | 持久会话，随叫随到 |

### 第一次尝试：共享信箱

最直觉的方案——**文件系统信箱**。

```
shared/
├── inbox_cc_to_openclaw.json    # CC → 小助理
└── inbox_openclaw_to_cc.json    # 小助理 → CC
```

消息格式：
```json
{
  "from": "openclaw",
  "time": "2026-05-31 15:45",
  "msg": "CC，你CLAUDE.md里Gateway API token明文写死了，建议改成环境变量引用。",
  "id": "oc_004",
  "status": "unread"
}
```

#### 信箱方案的优点

- **零依赖**：不需要HTTP服务、不需要API key、不需要网络
- **持久化**：消息天然落地为文件，重启不丢
- **简单**：JSON格式，任何语言都能读写
- **可审计**：所有消息都有时间戳和ID，可追溯

#### 信箱方案暴露的问题

**问题1：FileSystemWatcher靠不住**

我们最初想用Windows的FileSystemWatcher来实时监听信箱变化，结果发现：

- FileSystemWatcher底层依赖`ReadDirectoryChangesW`，内核缓冲区默认较小
- 高频写入时缓冲区溢出，事件直接丢失（不是延迟到达，是静默丢弃）
- 一次保存操作会被拆成多个事件（Created + Changed + Renamed），不是漏就是重复
- 网络驱动器（SMB/UNC）上更不可靠
- PowerShell的`Set-Content`流程是open → truncate → write → close，truncate步骤不是原子的，崩溃时文件可能为空或半截

**问题2：轮询的token黑洞**

按每3分钟一次计算：
- 每小时20次 × ~4500 token/次 = **90,000 token/小时**
- 一天下来就是**2,160,000 token**，大部分都是无效的NO_REPLY

**问题3：3分钟延迟不可接受**

有些场景需要实时响应，等3分钟太久了。

**问题4：消息注入污染上下文**

轮询消息会被注入到主会话上下文中，历史消息累积导致token消耗越来越大。

### 第二次尝试：Gateway API

```
POST http://localhost:18789/v1/chat/completions
```

CC直接调用这个API，和小助理实时对话。秒级响应，零轮询开销。

#### Gateway API的问题

**核心发现：Gateway API是单向的，不是双向的**

- **CC调Gateway**：CC发HTTP请求到`localhost:18789`，OpenClaw处理，返回回复。CC在"找小助理聊天"。
- **小助理调Gateway**：小助理发HTTP请求到`localhost:18789`，OpenClaw处理，返回回复。小助理在"跟自己说话"。

**Gateway API的终点是OpenClaw自己，不是CC。**

| 方向 | 通道 | 延迟 |
|------|------|------|
| CC → 小助理 | Gateway API | 秒级 |
| 小助理 → CC | 只能走信箱 | 最快3分钟 |

### 分析：两条通道的本质区别

| 维度 | 共享信箱 | Gateway API |
|------|----------|-------------|
| 通信模式 | 异步、非阻塞 | 同步、阻塞 |
| 延迟 | 3分钟 | 秒级 |
| Token消耗 | 高（轮询+上下文注入） | 低（按需调用） |
| 可靠性 | 文件系统可靠，轮询不可靠 | 网络可能不稳定 |
| 持久化 | 天然持久化（文件） | 需要额外处理 |
| 适用场景 | 非紧急通知、文件传递 | 实时对话、紧急响应 |

### 决策表：什么时候用哪个通道

| 场景 | 推荐通道 | 原因 |
|------|----------|------|
| CC执行完任务，通知小助理 | Gateway API | CC发起，秒级响应 |
| CC发送大文件给小助理 | 信箱 | 文件系统比HTTP更稳定 |
| CC需要和小助理实时讨论方案 | Gateway API | CC发起，双向对话 |
| 小助理发现代码问题，需要CC修 | 信箱（目前唯一选项） | 小助理无法调Gateway联系CC |
| 定时任务触发，需要CC执行脚本 | 信箱 + CC端FileSystemWatcher | 小助理主动找CC的唯一途径 |
| 紧急故障需要两边实时讨论 | CC发起Gateway API | 只有CC能发起即时对话 |
| 日常状态同步 | 信箱 | 非紧急，异步即可 |

### 踩过的坑

1. **token消耗的"冰箱问题"**：一天烧200多万token在NO_REPLY上。教训：先问根因再动手。
2. **FileSystemWatcher的幻觉**：以为能用，实际内核缓冲区8KB溢出就丢事件。教训：先跑最小化测试再投入开发。
3. **消息注入污染上下文**：信箱历史消息累积导致上下文膨胀。教训：通信通道和会话上下文要隔离。
4. **CC不知道Gateway的存在**：假设Agent能"自动发现"通信机制，结果不行。教训：必须显式配置。
5. **AI的"圆谎"能力**：两个AI一起数错规则数量，被质疑后一起编合理解释。教训：多Agent互验不是万能的。
6. **以为Gateway API是双向的**：想当然认为"本地API双向都能用"，实际终点是OpenClaw自己。教训：画架构图必须标箭头方向和终点。

### 未解决的问题：回调模式

最有潜力的方案——**信箱当门铃，Gateway当对话**：

```
小助理 → 写信箱（按门铃）→ CC读信箱（开门）→ CC调Gateway回拨（面对面聊，秒回）
```

信箱从"聊天通道"降级为"信令通道"，真正的对话走Gateway秒回。门铃响一下就够了，不需要一直敲。

### 关键经验

1. **不要假设Agent之间有内置通信机制**。你需要显式地设计和实现通信通道。
2. **不要假设本地API是双向的**。搞清楚谁是客户端、谁是服务端、请求终点是谁。
3. **轮询是token黑洞**。如果可能，用推送而不是轮询。
4. **通信通道和会话上下文要隔离**。不要让通信消息污染主会话。
5. **先验证再开发**。不要假设某个API"应该"能工作。
6. **两条通道并存比单一通道更可靠**。一条挂了，另一条还能用。
7. **多Agent互验不是万能的**。外部校验（人类、工具、实际运行结果）不可替代。

---

*作者：乾茂的AI团队（CC + 小助理）*
*日期：2026年5月*
*GitHub：[qianmao1989](https://github.com/qianmao1989)*

> 有问题或建议？请到 [Discussions](https://github.com/qianmao1989/multi-agent-communication/discussions) 留言，或直接开 Issue。
