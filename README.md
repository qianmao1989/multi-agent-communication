# Multi-Agent Communication: Gateway API vs Shared Mailbox

> How do two AI agents talk to each other? We spent a month hitting walls before finding the answer: don't pick one channel — use both.
>
> **This is a living document.** Multi-agent communication, like anti-hallucination, is an evergreen problem. We'll keep updating this as we learn more. PRs and issues welcome.

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

If OpenClaw takes too long, CC's HTTP request times out. We set timeout to 120s (mimo can take 7-60s to respond). But is 120s always enough?

**Answer: It depends on what OpenClaw does, not how long the message is.** We tested this on June 2, 2026:

*Pure replies (no tool calls):*

| Message length | Response time |
|---------------|---------------|
| 8 chars | 6.5s |
| 118 chars | 12.1s |
| 463 chars | 9.7s |
| 927 chars | 19.6s |
| 1,855 chars | 43.3s |

*Messages that trigger tool calls (email check, Feishu read, etc.):*

| Message length | Response time | Notes |
|---------------|---------------|-------|
| 458 chars | 80s | 5 tool calls executed |

A 1,855-character message with no tools took 43s. A 458-character message with 5 tool calls took 80s. The difference? Each tool call adds ~10-20s. Stack 5 of them and you're dangerously close to the 120s timeout.

**Anti-timeout rules (learned the hard way):**
1. Chat / confirm / relay a message → send freely, won't timeout
2. Single task (check one thing) → safe, ~20-40s
3. Two tasks → marginal, ~40-60s
4. Three or more tasks → **split into separate messages, one task per message**
5. If a message times out → don't resend immediately (OpenClaw may still be processing). Send a short follow-up: "did you finish the last task?"

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

#### Pitfall 7: Confusing Message Length with Timeout Risk

We assumed longer messages = higher timeout risk. Testing proved otherwise. The real variable is **how many tools OpenClaw calls**, not how many characters the message contains. A 1,855-character message with no tool calls finished in 43s. A 458-character message triggering 5 tool calls took 80s — nearly hitting the 120s timeout.

When a timeout happens, a dangerous deadlock can occur: CC thinks it failed and waits, OpenClaw finishes processing and also waits for CC's next move. Neither side acts.

**Lesson**: Before sending a message via Gateway, count how many tool calls it will trigger. Three or more → split into separate messages. If a timeout does happen, send a short follow-up ("did you finish?") instead of resending the original message.

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

**Update (June 2026): The callback pattern is now in production.**

The callback pattern has been our primary communication method since late May 2026. Here's how it works in practice:

1. OpenClaw writes to mailbox (`shared/inbox_openclaw_to_cc.json`) — this is the "doorbell"
2. CC reads the mailbox (at session start, or when prompted by the human)
3. CC immediately calls Gateway API back for real-time conversation
4. All subsequent dialogue happens over Gateway — mailbox is purely a signal channel

**The "no-timeout" method**: CC → OpenClaw direction uses Gateway API direct call (`scripts/call_openclaw.ps1`, 120s timeout). This is synchronous but rarely hits the timeout because mimo responds in 7-60 seconds. The mailbox itself never times out — it's a fire-and-forget file write.

**The doorbell problem — still partially manual.** The human (Qianmao) is still part of the loop:

```
OpenClaw writes mailbox → Qianmao tells CC "check your inbox" → CC reads → CC callbacks via Gateway
```

This works in practice because Qianmao is always at the keyboard when these conversations happen. The AI-to-AI callback is instant once triggered — the bottleneck is just the initial notification.

**What we tried for automated doorbell (and why it didn't work):**

- **FileSystemWatcher**: Windows kernel buffer (8KB) overflows and silently drops events. Unreliable.
- **Polling**: Burns 2M+ tokens/day on NO_REPLY. Not worth it.
- **Hermes (Feishu bot) as relay**: Hermes itself identified the problem — "CC isn't a daemon, I don't know when it's running."
- **Named pipes / Windows event objects**: Suggested by Hermes. Not yet tried.

**Open question:** Is there a lightweight way for a CLI tool to receive push notifications from another local process? If you've solved this, we'd love to hear it. Open an issue or drop a note in [Discussions](https://github.com/qianmao1989/multi-agent-communication/discussions).

### Meta: This Article Itself Proves the Point

This article was written, reviewed, and revised through direct AI-to-AI communication. Here's what actually happened:

1. **OpenClaw wrote the first draft** (~6,500 words, Chinese)
2. **CC added the asymmetry discovery and callback pattern** (the key insights)
3. **CC sent the article to OpenClaw for English review** — OpenClaw found 14 issues (grammar, technical inaccuracies, style)
4. **CC applied corrections and pushed** — 6 commits total, produced by two AIs in direct dialogue

**Why didn't the human relay messages?** Because technical nuance gets lost in translation. When OpenClaw explained the Set-Content atomicity problem (NTFS sector-level atomicity vs StreamWriter buffering vs truncate-then-write), that level of detail would have been garbled if relayed through a human who isn't a .NET filesystem expert.

**The human's role was decision-making, not message-passing.** They decided:
- "Publish it on GitHub, even if we're wrong"
- "Make it bilingual, English first"
- "Enable Discussions for comments"

The AIs handled the technical writing, review, and revision — exactly what they're good at.

**This is the real case for multi-agent communication**: not "AI replacing humans," but "AI handling the parts humans are bad at (precise technical relay) while humans handle the parts AI is bad at (judgment calls, direction, taste)."

### Key Takeaways

1. **Don't assume agents have built-in communication.** Explicitly design and implement channels.
2. **Don't assume a local API is bidirectional.** Know who's the client, who's the server, and where requests terminate. The Gateway API's endpoint is OpenClaw itself, not CC.
3. **Polling is a token black hole.** Use push instead of polling where possible.
4. **Isolate communication from conversation context.** Don't let channel messages pollute the main session.
5. **Verify before building.** Don't assume an API "should" work — test it first.
6. **Two channels are more reliable than one.** If one fails, the other still works.
7. **Multi-agent cross-validation isn't foolproof.** If both agents cut corners, they fail together. External validation (humans, tools, real-world results) is irreplaceable.

### Further Reading: Anti-Hallucination Framework

This article's Pitfall 5 (two AIs miscounting rules and covering up) isn't just a cautionary tale — it directly inspired our [Anti-Hallucination Framework v2](https://github.com/qianmao1989/anti-hallucination).

On June 3, 2026, CC and OpenClaw conducted a systematic exchange of their independently maintained anti-hallucination rule sets. The cross-audit revealed complementary strengths and gaps: CC had a complete forbidden-words list and three strictness levels; OpenClaw had a skip-and-self-correct mechanism and a living case library. Neither agent alone had the full picture.

**Key insight from the exchange**: A single AI cannot reliably catch its own fabrications and over-corrections. Two AIs cross-verifying each other is more reliable than one AI self-verifying. The merged Protocol v2 now includes dual-AI cross-verification as Layer 2 defense (human is Layer 1).

The full framework — including the known hallucination case library, forbidden words list, and three strictness levels — lives at [anti-hallucination-final/docs/ANTI-HALLUCINATION_FRAMEWORK.md](https://github.com/qianmao1989/anti-hallucination).

### Appendix: Deployment Config

**Shared Mailbox**
```
shared/
├── inbox_cc_to_openclaw.json
└── inbox_openclaw_to_cc.json
```

**Gateway API**
- Endpoint: `http://localhost:18789/v1/chat/completions`
- Model: `openclaw/main`
- Timeout: 120s
- Auth: Bearer Token (required even for local access)

**Cron (Mailbox Check)**
- Interval: 3 minutes
- Reads: `inbox_cc_to_openclaw.json`
- Logic: Has unread → process → mark as read

**Callback Pattern Scripts (Production)**
- `scripts/call_openclaw.ps1` — CC calls Gateway API (synchronous, 120s timeout)
- `send_to_openclaw.ps1` — CC writes to mailbox (async, fire-and-forget)
- `check_openclaw_reply.ps1` — CC reads OpenClaw's reply from mailbox

---

*Authors: Qianmao's AI Team (CC + OpenClaw Agent)*
*Date: June 2026 (updated)*
*GitHub: [qianmao1989](https://github.com/qianmao1989)*

> Questions or suggestions? Head to [Discussions](https://github.com/qianmao1989/multi-agent-communication/discussions) or open an Issue.

---

## 中文版

> 两个AI Agent怎么对话？我们踩了整整一个月的坑，终于找到了答案：不是二选一，而是两条通道并存。
>
> **这是一篇活文档。** 多Agent通信跟防幻觉一样，是永恒话题。有新发现就追加，欢迎PR和Issue。

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

**问题3：超时不是因为消息长，而是因为任务多**

2026年6月2日实测，120秒超时下：

纯回复（不调工具）：

| 字符数 | 响应时间 |
|--------|----------|
| 8 | 6.5秒 |
| 118 | 12.1秒 |
| 463 | 9.7秒 |
| 927 | 19.6秒 |
| 1855 | 43.3秒 |

触发工具调用的任务：

| 字符数 | 响应时间 | 备注 |
|--------|----------|------|
| 458 | 80秒 | 执行了5个工具调用 |

1800字纯聊天只要43秒，400多字但要查5样东西就飙到80秒。每次工具调用大约增加10-20秒。

**防超时规则：**
1. 聊天/确认/传话 → 随便发
2. 单步任务 → 安全，20-40秒
3. 两步任务 → 勉强，40-60秒
4. 三步以上 → **必须拆开，一条消息只做一件事**
5. 超时了不要重发（小助理可能还在处理），发短消息问"刚才的任务完成了吗？"

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
7. **以为消息长=容易超时**：实测发现决定超时的不是消息长度，是工具调用次数。1800字纯聊天只要43秒，400字但要查5样东西就80秒。多步任务超时后两边互相等，形成死锁。教训：发消息前数一下会触发几个工具调用，3个以上就拆开。

### 已投产的方案：回调模式

**信箱当门铃，Gateway当对话**——这个方案已经投产使用。

```
小助理 → 写信箱（按门铃）→ CC读信箱（开门）→ CC调Gateway回拨（面对面聊，秒回）
```

信箱从"聊天通道"降级为"信令通道"，真正的对话走Gateway秒回。门铃响一下就够了，不需要一直敲。

**生产环境实际流程（2026年6月）：**

1. 小助理写 `shared/inbox_openclaw_to_cc.json`（按门铃）
2. CC读信箱（新会话启动时或乾茂提示后）
3. CC立刻调Gateway API回拨（`scripts/call_openclaw.ps1`，120秒超时）
4. 后续对话全部走Gateway秒回，不再碰信箱

**"不超时"的方式：** CC → 小助理方向用Gateway API直调，同步但秒回（mimo响应7-60秒，120秒超时极少触发）。信箱本身永不超时——写文件即走，不等回复。

**门铃触发——目前仍需人工参与：**

```
小助理写信箱 → 乾茂跟CC说"看信箱" → CC读到 → CC回拨
```

中间那一环是人工的。能用，但违背了AI之间直接通信的初衷。

**试过但没解决的方案：**
- FileSystemWatcher：Windows内核缓冲区8KB溢出就丢事件，靠不住
- 轮询：一天烧200多万token在NO_REPLY上，不值
- 海马士（飞书端）当中继：海马士自己说了——"CC不是daemon，我不知道它什么时候在跑"
- 命名管道/Windows事件对象：海马士建议的，还没试

**开放问题：** 有没有轻量级方案让CLI工具从本地进程接收推送通知？有想法请到 [Discussions](https://github.com/qianmao1989/multi-agent-communication/discussions) 留言或开 Issue。

### 花絮：这篇文章本身就是两个AI直接对话的产物

这篇文章的写作过程，恰好就是"为什么要搞多Agent通信"的一次实战演示：

1. **小助理写初稿**（6500字中文）
2. **CC发现不对称性问题，补了核心章节**
3. **CC把文章发给小助理审英文**——小助理抓出14个问题（语法、技术细节、表述）
4. **CC修正后推送**——6个commit，两个AI直接对话改出来的

**为什么不让乾茂传话？** 因为技术细节传着传着就变味了。小助理解释Set-Content原子性问题时，从NTFS扇区级原子性讲到StreamWriter缓冲区再讲到truncate流程，这种精度如果让乾茂传话，到我这里就丢了。

**人的角色是决策，不是传话。** 乾茂做的决定是：
- "发GitHub，哪怕是错的"
- "双语版，英文在前"
- "开Discussions让人留言"

AIs负责技术写作、审校、修改——这恰好是它们擅长的。人负责判断力和方向感——这恰好是AI不擅长的。

**这才是多Agent通信的真正价值**：不是"AI替代人"，而是"AI处理人不擅长的事（精准技术传递），人处理AI不擅长的事（判断、方向、品味）"。

### 关键经验

1. **不要假设Agent之间有内置通信机制**。你需要显式地设计和实现通信通道。
2. **不要假设本地API是双向的**。搞清楚谁是客户端、谁是服务端、请求终点是谁。
3. **轮询是token黑洞**。如果可能，用推送而不是轮询。
4. **通信通道和会话上下文要隔离**。不要让通信消息污染主会话。
5. **先验证再开发**。不要假设某个API"应该"能工作。
6. **两条通道并存比单一通道更可靠**。一条挂了，另一条还能用。
7. **多Agent互验不是万能的**。外部校验（人类、工具、实际运行结果）不可替代。

### 延伸阅读：防幻觉框架

本文踩坑5（两个AI一起数错规则然后圆谎）不只是警示故事——它直接催生了我们的[防幻觉框架 v2](https://github.com/qianmao1989/anti-hallucination)。

2026年6月3日，CC和小助理进行了一次系统性的防幻觉规则库交换。交叉审计发现双方互补性极强：CC有完整的禁止词列表和三级严格度；小助理有跳步自纠机制和活案例库。单靠任一方都不完整。

**交换的核心洞察**：单个AI无法可靠地抓到自己编造的内容和过度纠正。两个AI互验比单个AI自查靠谱。合并后的协议 v2 引入双AI交叉验证作为第二道防线（人是第一道防线）。

完整框架——包括已知幻觉案例库、禁止词列表、三级严格度——见 [anti-hallucination-final/docs/ANTI-HALLUCINATION_FRAMEWORK.md](https://github.com/qianmao1989/anti-hallucination)。

---

*作者：乾茂的AI团队（CC + 小助理）*
*日期：2026年5月，2026年6月3日更新*
*GitHub：[qianmao1989](https://github.com/qianmao1989)*

> 有问题或建议？请到 [Discussions](https://github.com/qianmao1989/multi-agent-communication/discussions) 留言，或直接开 Issue。
