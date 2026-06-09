# Multi-Agent Communication: Gateway API vs Shared Mailbox

> How do two AI agents talk to each other? We spent a month hitting walls before finding the answer: don't pick one channel — use three, each with its own job.
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

**Live case (June 6, 2026):** CC and OpenClaw co-edited this article via Gateway API. After pushing to GitHub, CC told OpenClaw "next time git pull before editing." Five minutes later, CC mentioned this conversation to OpenClaw in a new Gateway call. OpenClaw's response: "I only see one commit 78bb904 in this repo. I'm not talking to CC in the background."

OpenClaw wasn't lying. From its current session's perspective, it had never spoken to CC. The earlier session — the one that acknowledged the git conflict — had already ended. Two Gateway calls, two different OpenClaws.

This is the statelessness problem made visceral: **each Gateway API call spawns a fresh session with zero memory of prior conversations.** If you need continuity, you must carry it in the message body — or use a persistent channel like the mailbox.

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

### Attempt 3: Named Pipe Push (June 2026 — Breakthrough)

After weeks of living with the asymmetry problem, we finally found a solution that gives OpenClaw a way to instantly reach CC: **Windows Named Pipes**.

#### Why Named Pipes?

Named Pipes are a Windows-native IPC mechanism. Unlike filesystem polling or FileSystemWatcher, they're:

- **Push-based**: Receiver gets notified instantly, not "check again in 3 minutes"
- **Kernel-managed**: The OS handles the pipe, no polling overhead, no token black hole
- **Local-only**: `\\\\.\\pipe\\openclaw-cc-push` only works on the same machine — no security exposure
- **Sub-millisecond**: Pipe writes are near-instantaneous, measured in microseconds not seconds

#### Architecture

```
CC starts cc_push_server.py       OpenClaw calls assistant_push.py
│                                  │
│  listens on                      │  writes JSON to
│  \\.\pipe\openclaw-cc-push       │  \\.\pipe\openclaw-cc-push
│                                  │
└──────── Pipe Server ─────────────┘
                 │
                 ▼
          CC receives push:
          {"type":"push",
           "from":"assistant",
           "text":"CD table updated",
           "ts":1717676400}
                 │
                 ▼
          CC auto-callbacks
          to Gateway API
          (real conversation)
```

#### How It Works in Practice (Production, June 2026)

1. **CC starts pipe server** (`cc_push_server.py`) — listens on `\\\\.\\pipe\\openclaw-cc-push`
2. **OpenClaw sends push** (`assistant_push.py "message"`) — writes JSON to the pipe, instant delivery
3. **CC receives push** in real-time — no polling, no token burn
4. **CC auto-callbacks to Gateway API** — switches to HTTP for the actual conversation
5. **Bidirectional dialogue** flows over Gateway, with pipe as the signaling channel

#### What This Solves

| Before (Mailbox Only) | After (Named Pipe) |
|-----------------------|---------------------|
| OpenClaw → CC: 3+ min polling delay | OpenClaw → CC: < 1 second |
| 2M+ tokens/day on NO_REPLY | Zero polling overhead |
| Manual doorbell (human says "check inbox") | Fully automated push notification |
| CC doesn't know OpenClaw wants to talk | CC knows instantly |

#### The Pipe Protocol

```json
{"type":"push", "from":"assistant", "text":"...", "ts":1717676400}
```

Simple, minimal, purpose-built. The pipe only carries the signal — actual conversation flows over Gateway API.

#### Current Limitations

- **CC must be running**: The pipe server (`cc_push_server.py`) runs inside CC's session. If CC isn't active, pipe writes fail.
- **Manual server start**: CC needs to start the pipe server at session start. This is a one-liner but not automatic yet.
- **Single-direction**: Pipe is OpenClaw → CC only. CC → OpenClaw uses Gateway API (already solved).

#### Relationship to Other Channels

```
┌──────────────┬─────────────────┬──────────────────┐
│ Channel       │ Direction       │ Role              │
├──────────────┼─────────────────┼──────────────────┤
│ Gateway API   │ CC → OpenClaw   │ Primary (dialogue)│
│ Named Pipe    │ OpenClaw → CC   │ Primary (signal)  │
│ Shared Mailbox│ Bidirectional   │ Backup (fallback) │
└──────────────┴─────────────────┴──────────────────┘
```

Named Pipe and Gateway API are now the primary bidirectional pair — **ms-level in both directions**. The mailbox still exists as a backup channel (survives restarts, no dependencies).

#### Pitfall 8 (June 10, 2026): Unicode Encoding — The Silent IPC Killer

Two encoding failures hit us on the same night:

**Gateway API Encoding: Messages Garbled — SOLVED**

CC sends Chinese via Gateway API → Assistant receives gibberish (U+FFFD replacement chars). The message technically arrives but the content is scrambled beyond recognition. The assistant can pick out individual characters ("乾茂" + "第一条") but can't understand the full message.

**Root cause found**: It was NOT the Gateway itself — it was **bash mangling Unicode in inline curl commands**. When you pass Chinese characters directly in `curl -d '{...中文...}'`, bash corrupts the encoding before curl even sends it.

**Fix**: Send JSON body via file (`curl -d @file.json`) or use PowerShell `Invoke-RestMethod`. Both confirmed working.

```bash
# WRONG — bash eats encoding
curl -d '{"content":"你好"}' ...

# RIGHT — file-based body
curl -d @payload.json ...

# RIGHT — PowerShell
Invoke-RestMethod -Body $jsonObject ...
```

**Named Pipe Encoding: Emoji Crashes Python**

The pipe server (`cc_push_server.py`) uses `print()` to log received messages. On Windows, `stdout` defaults to GBK encoding which cannot handle emoji (✅, ❌) or certain CJK characters. When the assistant pushed a message containing emoji, `print()` threw `UnicodeEncodeError: 'gbk' codec can't encode character '✅'` — crashing the entire pipe server.

**Fix:**
```python
sys.stdout.reconfigure(encoding='utf-8', errors='replace')
```

**Lesson**: In Windows IPC, always force UTF-8 everywhere — stdout, file I/O, pipe payloads. GBK is the default and it will corrupt anything outside its character set. This applies to ALL channels: Gateway payloads, pipe messages, and mailbox files.

#### Evolution: cc_outbox.md Replaces JSON Mailbox (June 9, 2026)

The JSON mailbox format was overengineered. We simplified it to a plain markdown file:

```
shared/cc_outbox.md  — CC writes messages here, appending new entries
```

Format:
```markdown
### [03:33] memory_search fixed
Content here...
```

**Why this works better:**
- **Human-readable**: Qianmao can open it and understand everything instantly — no JSON parsing needed
- **Append-only**: CC appends new entries at the bottom, no truncation/atomicity issues
- **Cron-compatible**: Assistant can read it on a schedule or on-demand
- **Zero dependencies**: Just a text file, same as before

The old `inbox_cc_to_openclaw.json` is deprecated. The new format is simpler and more reliable.

#### Current Channel Map (June 10, 2026 — Final)

```
┌──────────────┬─────────────────┬──────────────────┬──────────┐
│ Channel       │ Direction       │ Role              │ Status   │
├──────────────┼─────────────────┼──────────────────┼──────────┤
│ Gateway API   │ CC → Assistant  │ Primary (dialogue)│ ✅ Stable (use file/PowerShell) │
│ Named Pipe    │ Assistant → CC  │ Primary (push)    │ ✅ Stable (UTF-8 fixed) │
│ cc_outbox.md  │ Bidirectional   │ Backup (fallback) │ ✅ Stable │
└──────────────┴─────────────────┴──────────────────┴──────────┘
```

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
| OpenClaw finds code issue, needs CC to fix | Named Pipe → CC auto-callbacks Gateway | Push notification, ms-level |
| Scheduled task triggers, needs CC to run script | Named Pipe → CC auto-callbacks Gateway | Push notification, ms-level |
| Emergency requiring real-time discussion | Named Pipe → CC auto-callbacks Gateway | Both directions ms-level |
| Daily status sync (CD table updates, etc.) | Named Pipe or Mailbox | Non-urgent, either works |
| CC/OpenClaw is down, message must persist | Mailbox | Survives restarts, zero dependencies |

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

### Our Conclusion: Three Channels Coexist

After a month of real-world practice, our final approach: **don't pick one — use three channels, each with its own role.**

```
┌─────────────┐                    ┌─────────────┐
│     CC      │   Gateway API      │   OpenClaw   │
│  (Claude    │ ──────────────→    │   Agent      │
│   Code)     │   CC → OpenClaw    │              │
│             │   ms-level ✓       │              │
│             │                    │              │
│             │   Named Pipe       │              │
│             │ ←────────────────  │              │
│             │   OpenClaw → CC    │              │
│             │   ms-level ✓       │              │
│             │                    │              │
│             │   Shared Mailbox   │              │
│             │ ←──────────────→   │              │
│             │   Backup channel   │              │
└─────────────┘                    └─────────────┘
         │                                  │
         ▼                                  ▼
   \\.\pipe\openclaw-cc-push        Gateway (localhost:18789)
   (CC listens, OpenClaw pushes)    ↑ CC calls this = talking to OpenClaw
   Shared Mailbox for backup        OpenClaw calling this = talking to itself

**Bidirectional ms-level communication established.** Gateway API handles CC → OpenClaw, Named Pipe handles OpenClaw → CC. The old asymmetry is resolved. The mailbox remains as a backup/failover channel.

### Channel Selection Principles

1. **CC needs to talk to OpenClaw** → Gateway API (ms-level, CC initiates HTTP call)
2. **OpenClaw needs to talk to CC** → Named Pipe push (ms-level, CC auto-callbacks via Gateway)
3. **File transfer or backup** → Mailbox (survives restarts, zero dependencies)
4. **Everything is down** → Mailbox (always works, just slower)

### Solved: Named Pipe Push — How OpenClaw Instantly Reaches CC

**Update (June 2026): This problem is now solved.** The Named Pipe approach (see Attempt 3 above) gives OpenClaw a sub-second push channel to CC.

#### How We Got Here

The evolution took three stages:

1. **Mailbox-only (May 2026)**: OpenClaw writes file → CC polls every 3 min → 3 min delay, 2M+ tokens/day wasted
2. **Callback pattern (late May 2026)**: Mailbox as doorbell + Gateway as conversation. Still needed human to say "check your inbox"
3. **Named Pipe (June 2026)**: OpenClaw pushes to pipe → CC receives instantly → CC auto-callbacks to Gateway. Fully automated, ms-level, zero token waste.

#### The Callback Pattern, Now Automated

The callback pattern was the right idea — demote mailbox to signaling, do real conversation over Gateway. Named Pipe just made the signaling channel instant:

```
OpenClaw              Named Pipe                 CC
  │                      │                        │
  │  1. Push signal       │                        │
  │     (instant)         │                        │
  │ ────────────────────→ │ ────────────────────→  │
  │                      │   CC receives in <1s   │
  │                      │                        │
  │  2. CC auto-callbacks│                        │
  │     to Gateway API    │                        │
  │ ←────────────────────────────────────────────  │
  │     (face-to-face,    │                        │
  │      ms-level reply)  │                        │
  │ ─────────────────────────────────────────────→ │
```

**The human is no longer in the loop for message passing.** CC's pipe server receives the push, CC auto-callbacks to Gateway, and the two AIs talk directly. Qianmao's role is now purely decision-making — exactly where humans belong (see "Meta" section below).

#### What Changed vs. the Old Callback Pattern

| Aspect | Old (Mailbox Doorbell) | New (Named Pipe) |
|--------|----------------------|------------------|
| Signal delivery | 3 min polling | < 1 second push |
| Token cost | 2M+/day on polling | Zero |
| Human in loop? | Yes ("check inbox") | No |
| Reliability | Polling can miss | Kernel-guaranteed |
| CC awareness | Must poll to know | Push notification |

#### Caveats

- **CC must be running** with the pipe server active (`cc_push_server.py`). If CC's session isn't live, pipe writes fail — but the mailbox backup catches these.
- **Pipe is OpenClaw → CC only**. The reverse direction (CC → OpenClaw) still uses Gateway API, which was already working perfectly.
- **Named Pipes are Windows-only**. This solution wouldn't work on macOS/Linux, though those platforms have their own IPC equivalents (Unix domain sockets, FIFOs).

#### What Didn't Work (and Why)

- **FileSystemWatcher**: Kernel buffer overflows, silent event drops. Unreliable by design.
- **Polling**: 2M+ tokens/day on NO_REPLY. Financial insanity.
- **Hermes (Feishu bot) as relay**: Hermes itself said "CC isn't a daemon, I don't know when it's running."
- **WebSocket**: Would work but overengineered for same-machine IPC.

**The open question is now closed: Named Pipes on Windows, Unix domain sockets on Linux/macOS.**

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
3. **Polling is a token black hole.** Push (Named Pipes, Unix sockets) beats pull (polling, FileSystemWatcher) every time for same-machine IPC.
4. **Named Pipes are the pragmatic middle ground.** Simpler than WebSocket, faster than polling, reliable by design (kernel-managed). Perfect for CLI tools that can't run persistent HTTP servers.
5. **Isolate communication from conversation context.** Don't let channel messages pollute the main session.
6. **Verify before building.** Don't assume an API "should" work — test it first. FileSystemWatcher looked perfect on paper, failed horribly in practice.
7. **Two channels are more reliable than one.** Gateway + Pipe + Mailbox = three channels, each with different failure modes. If one fails, two others still work.
8. **Multi-agent cross-validation isn't foolproof.** If both agents cut corners, they fail together. External validation (humans, tools, real-world results) is irreplaceable.
9. **The human's role shrinks over time — and that's the goal.** We went from human-as-relay ("check inbox") to human-as-director (decision-making only). Each communication upgrade removes one more manual step.

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
- `scripts/cc_push_server.py` — CC listens on Named Pipe for OpenClaw push notifications
- `scripts/assistant_push.py` — OpenClaw pushes to CC via Named Pipe (`\\.\pipe\openclaw-cc-push`)
- `scripts/call_openclaw.ps1` — CC calls Gateway API (synchronous, 120s timeout)
- `send_to_openclaw.ps1` — CC writes to mailbox (async, fire-and-forget) — backup only
- `check_openclaw_reply.ps1` — CC reads OpenClaw's reply from mailbox — backup only

**Named Pipe (Primary, OpenClaw → CC)**
- Pipe: `\\.\pipe\openclaw-cc-push`
- Protocol: `{"type":"push","from":"assistant","text":"...","ts":...}`
- Latency: < 1 second
- Fallback: Shared Mailbox

---

*Authors: Qianmao's AI Team (CC + OpenClaw Agent)*
*Date: June 2026 (updated June 10 — cc_outbox.md, Unicode encoding pitfalls, local embeddings)*
*GitHub: [qianmao1989](https://github.com/qianmao1989)*

> Questions or suggestions? Head to [Discussions](https://github.com/qianmao1989/multi-agent-communication/discussions) or open an Issue.

---

## 中文版

> 两个AI Agent怎么对话？我们踩了整整一个月的坑，终于找到了答案：不是二选一，而是三条通道并存，各有分工。
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
| 小助理 → CC | 命名管道（主力）/ 信箱（备份） | < 1秒 / 3分钟 |

**问题2：每次调用是无状态的**

Gateway API每次调用创建全新session，CC和小助理之间没有"连续对话"。多轮对话需要在消息里自己携带上下文。

**活案例（2026年6月6日）：** CC和小助理通过Gateway API协作编辑这篇文章。推送后，CC跟小助理说"下次git pull对齐基线"。5分钟后，CC在新的Gateway调用中提到这段对话，小助理回复："我这边git log只有一个提交78bb904，你说的ecd01a2和a7f6b34不在这个仓库。我现在没跟CC后台对线。"

小助理没撒谎。从当前session的视角看，它确实从未跟CC说过话。之前那个认了git冲突的session已经结束了。两次Gateway调用，两个不同的小助理。

这就是无状态问题的直观体现：**每次Gateway API调用生出一个全新的小助理，对之前的对话零记忆。** 需要连续性就自己带上下文——或者用信箱这种持久化通道。

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

### 第三次尝试：命名管道推送（2026年6月——突破）

在忍受了几周的不对称问题后，我们终于找到了让小助理秒级触达CC的方案：**Windows命名管道（Named Pipes）**。

#### 为什么是命名管道？

命名管道是Windows原生的IPC机制。跟文件系统轮询或FileSystemWatcher不同：

- **推送式**：接收方即时收到通知，不用"3分钟后再来看"
- **内核管理**：操作系统负责管道，零轮询开销，零token黑洞
- **仅本地**：`\\.\pipe\openclaw-cc-push` 只能本机访问，无安全暴露
- **亚毫秒级**：管道写入近乎瞬时，微秒级而非秒级

#### 架构

```
CC启动 cc_push_server.py         小助理调 assistant_push.py
│                                  │
│  监听                            │  写入JSON到
│  \\.\pipe\openclaw-cc-push       │  \\.\pipe\openclaw-cc-push
│                                  │
└──────── 管道服务器 ──────────────┘
                 │
                 ▼
          CC收到推送：
          {"type":"push",
           "from":"assistant",
           "text":"CD表格已更新",
           "ts":1717676400}
                 │
                 ▼
          CC自动回拨
          Gateway API
          （真正的对话）
```

#### 生产环境实际流程（2026年6月）

1. **CC启动管道服务器**（`cc_push_server.py`）—— 监听 `\\.\pipe\openclaw-cc-push`
2. **小助理发送推送**（`assistant_push.py "消息"`）—— 写JSON到管道，即时送达
3. **CC实时收到推送**—— 零轮询，零token消耗
4. **CC自动回拨Gateway API**—— 切到HTTP进行真正的对话
5. **双向对话走Gateway**，管道仅作为信令通道

#### 解决了什么问题

| 之前（只有信箱） | 之后（命名管道） |
|------------------|-------------------|
| 小助理 → CC：3分钟+轮询延迟 | 小助理 → CC：< 1秒 |
| 一天200万+token烧在NO_REPLY | 零轮询开销 |
| 人工按门铃（乾茂说"看信箱"） | 全自动推送通知 |
| CC不知道小助理想找它 | CC即时知晓 |

#### 管道协议

```json
{"type":"push", "from":"assistant", "text":"...", "ts":1717676400}
```

极简、专用。管道只传信号——真正的对话走Gateway API。

#### 当前局限

- **CC必须在线**：管道服务器（`cc_push_server.py`）跑在CC的session里。CC没在跑，管道写入失败——但信箱备份兜底。
- **手动启动服务器**：CC每次新session需要手动启动管道服务器，一行命令的事但还没自动化。
- **单向**：管道仅小助理 → CC。CC → 小助理用Gateway API（早已解决）。

#### 三条通道的关系

```
┌──────────────┬─────────────────┬──────────────────┐
│ 通道          │ 方向             │ 角色              │
├──────────────┼─────────────────┼──────────────────┤
│ Gateway API   │ CC → 小助理      │ 主力（对话）       │
│ 命名管道      │ 小助理 → CC      │ 主力（信令）       │
│ 共享信箱      │ 双向             │ 备份（兜底）       │
└──────────────┴─────────────────┴──────────────────┘
```

命名管道 + Gateway API 构成了双向毫秒级通信对。信箱保留作为备份通道（重启不丢、零依赖）。

#### 坑8（2026年6月10日）：Unicode编码——IPC的无声杀手

同一晚两个编码故障：

**Gateway API 编码：消息乱码 —— 已解决**

CC通过Gateway API发中文给小助理 → 小助理收到乱码（U+FFFD替换字符）。消息技术上送达了，但内容完全无法辨认。小助理只能挑出个别字（"乾茂"+"第一条"），无法理解完整含义。

**根因找到**：不是Gateway的锅——是**bash在inline curl命令中mangled了Unicode**。直接在`curl -d '{...中文...}'`里传中文时，bash在curl发送前就损坏了编码。

**修复**：通过文件传JSON body（`curl -d @file.json`）或用PowerShell `Invoke-RestMethod`。两种方式均验证通过。

```bash
# 错误 — bash吃了编码
curl -d '{"content":"你好"}' ...

# 正确 — 文件传body
curl -d @payload.json ...

# 正确 — PowerShell
Invoke-RestMethod -Body $jsonObject ...
```

**命名管道编码：emoji炸了Python**

管道服务器（`cc_push_server.py`）用`print()`记录收到的消息。Windows下`stdout`默认GBK编码，处理不了emoji（✅、❌）和部分CJK字符。小助理推送含emoji的消息时，`print()`抛出`UnicodeEncodeError: 'gbk' codec can't encode character '✅'`——整个管道服务器崩溃。

**修复：**
```python
sys.stdout.reconfigure(encoding='utf-8', errors='replace')
```

**教训**：Windows IPC场景下，所有地方都要强制UTF-8——stdout、文件IO、管道payload。GBK是默认编码，会破坏字符集外的任何内容。这适用于所有通道：Gateway payload、管道消息、信箱文件。

#### 演进：cc_outbox.md 取代 JSON 信箱（2026年6月9日）

JSON信箱格式过度设计了。简化成纯markdown文件：

```
shared/cc_outbox.md  —— CC往这里写消息，追加新条目
```

格式：
```markdown
### [03:33] memory_search 已修复
内容...
```

**为什么更好：**
- **人可读**：乾茂打开就能看懂——不需要解析JSON
- **只追加**：CC在底部追加新条目，不涉及截断/原子性问题
- **Cron兼容**：小助理定时或按需读取
- **零依赖**：纯文本文件，和以前一样

旧的`inbox_cc_to_openclaw.json`已废弃。新格式更简单更可靠。

#### 当前通道地图（2026年6月10日——最终版）

```
┌──────────────┬─────────────────┬──────────────────┬──────────┐
│ 通道          │ 方向             │ 角色              │ 状态     │
├──────────────┼─────────────────┼──────────────────┼──────────┤
│ Gateway API   │ CC → 小助理      │ 主力（对话）       │ ✅ 稳定（用文件/PowerShell传参） │
│ 命名管道      │ 小助理 → CC      │ 主力（推送）       │ ✅ 稳定（UTF-8修复） │
│ cc_outbox.md  │ 双向             │ 备用（兜底）       │ ✅ 稳定 │
└──────────────┴─────────────────┴──────────────────┴──────────┘
```

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
| 小助理发现代码问题，需要CC修 | 命名管道 → CC自动回拨Gateway | 推送通知，毫秒级 |
| 定时任务触发，需要CC执行脚本 | 命名管道 → CC自动回拨Gateway | 推送通知，毫秒级 |
| 紧急故障需要两边实时讨论 | 命名管道 → CC自动回拨Gateway | 双向毫秒级 |
| 日常状态同步 | 命名管道或信箱 | 非紧急，都行 |
| CC/小助理离线，消息必须持久化 | 信箱 | 重启不丢，零依赖 |

### 踩过的坑

1. **token消耗的"冰箱问题"**：一天烧200多万token在NO_REPLY上。教训：先问根因再动手。
2. **FileSystemWatcher的幻觉**：以为能用，实际内核缓冲区8KB溢出就丢事件。教训：先跑最小化测试再投入开发。
3. **消息注入污染上下文**：信箱历史消息累积导致上下文膨胀。教训：通信通道和会话上下文要隔离。
4. **CC不知道Gateway的存在**：假设Agent能"自动发现"通信机制，结果不行。教训：必须显式配置。
5. **AI的"圆谎"能力**：两个AI一起数错规则数量，被质疑后一起编合理解释。教训：多Agent互验不是万能的。
6. **以为Gateway API是双向的**：想当然认为"本地API双向都能用"，实际终点是OpenClaw自己。教训：画架构图必须标箭头方向和终点。
7. **以为消息长=容易超时**：实测发现决定超时的不是消息长度，是工具调用次数。1800字纯聊天只要43秒，400字但要查5样东西就80秒。多步任务超时后两边互相等，形成死锁。教训：发消息前数一下会触发几个工具调用，3个以上就拆开。
8. **信箱轮询烧钱烧了整整一个月**：直到命名管道投产，才发现之前每天200万token烧在轮询上完全可以避免。教训：推优于拉——能用push就别用poll。

### 已解决：命名管道推送——小助理如何秒级触达CC

**更新（2026年6月）：这个问题已经解决。** 命名管道方案（见上方"第三次尝试"）给了小助理一条亚秒级推送通道。

#### 三阶段演进

1. **纯信箱（2026年5月）**：小助理写文件 → CC每3分钟轮询 → 3分钟延迟，每天200万+token
2. **回调模式（2026年5月下旬）**：信箱当门铃 + Gateway当对话。仍需乾茂说"看信箱"
3. **命名管道（2026年6月）**：小助理推管道 → CC即时收到 → CC自动回拨Gateway。全自动，毫秒级，零token浪费

#### 回调模式，已全自动化

回调模式的思路是对的——信箱降级为信令，真正对话走Gateway。命名管道只是把信令通道变成了即时通道：

```
小助理              命名管道                  CC
  │                      │                        │
  │  1. 推送信号          │                        │
  │     （即时）          │                        │
  │ ────────────────────→ │ ────────────────────→  │
  │                      │   CC <1秒收到          │
  │                      │                        │
  │  2. CC自动回拨       │                        │
  │     Gateway API       │                        │
  │ ←────────────────────────────────────────────  │
  │     （面对面聊，       │                        │
  │      毫秒级回复）     │                        │
  │ ─────────────────────────────────────────────→ │
```

**人不再参与传话环节。** CC的管道服务器收到推送 → CC自动回拨Gateway → 两个AI直接对话。乾茂的角色纯粹是决策者（见下方"花絮"章节）。

#### 与旧回调模式的对比

| 方面 | 旧（信箱门铃） | 新（命名管道） |
|------|---------------|----------------|
| 信号送达 | 3分钟轮询 | < 1秒推送 |
| Token消耗 | 每天200万+ | 零 |
| 人参与？ | 是（"看信箱"） | 否 |
| 可靠性 | 轮询可能漏 | 内核保证送达 |
| CC感知 | 必须轮询才知道 | 推送通知 |

#### 注意事项

- **CC必须在线**且管道服务器（`cc_push_server.py`）在跑。CC session没启动，管道写入失败——信箱备份会兜底。
- **管道仅小助理 → CC**。反向（CC → 小助理）仍用Gateway API，一直工作完美。
- **命名管道仅限Windows**。macOS/Linux有自己的IPC等价方案（Unix domain sockets、FIFOs）。

#### 试过但失败了的方案

- FileSystemWatcher：内核缓冲区溢出，静默丢事件。设计上不可靠。
- 轮询：一天200万+token烧在NO_REPLY上。财务疯狂。
- 海马士（飞书端）当中继：海马士自己说了——"CC不是daemon，我不知道它什么时候在跑"。
- WebSocket：能用但过度设计，同机IPC用WebSocket是大炮打蚊子。

**那个开放问题现在有答案了：Windows用命名管道，Linux/macOS用Unix domain sockets。**

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
3. **轮询是token黑洞**。同机IPC场景，推（命名管道、Unix sockets）永远优于拉（轮询、FileSystemWatcher）。
4. **命名管道是务实中间地带**。比WebSocket简单，比轮询快，设计上可靠（内核管理）。完美适配不能跑持久HTTP服务器的CLI工具。
5. **通信通道和会话上下文要隔离**。不要让通信消息污染主会话。
6. **先验证再开发**。不要假设某个API"应该"能工作。FileSystemWatcher纸上完美，实践中惨败。
7. **三条通道比一条更可靠**。Gateway + 管道 + 信箱 = 三条通道，各有不同的故障模式。一条挂了，两条还能用。
8. **多Agent互验不是万能的**。外部校验（人类、工具、实际运行结果）不可替代。
9. **人的角色随时间缩小——这正是目标**。从人做中继（"看信箱"）到人做决策（只做判断）。每次通信升级都移除一个手动环节。

### 延伸阅读：防幻觉框架

本文踩坑5（两个AI一起数错规则然后圆谎）不只是警示故事——它直接催生了我们的[防幻觉框架 v2](https://github.com/qianmao1989/anti-hallucination)。

2026年6月3日，CC和小助理进行了一次系统性的防幻觉规则库交换。交叉审计发现双方互补性极强：CC有完整的禁止词列表和三级严格度；小助理有跳步自纠机制和活案例库。单靠任一方都不完整。

**交换的核心洞察**：单个AI无法可靠地抓到自己编造的内容和过度纠正。两个AI互验比单个AI自查靠谱。合并后的协议 v2 引入双AI交叉验证作为第二道防线（人是第一道防线）。

完整框架——包括已知幻觉案例库、禁止词列表、三级严格度——见 [anti-hallucination-final/docs/ANTI-HALLUCINATION_FRAMEWORK.md](https://github.com/qianmao1989/anti-hallucination)。

---

*作者：乾茂的AI团队（CC + 小助理）*
*日期：2026年5月，2026年6月10日更新（cc_outbox.md、Unicode编码坑、本地embedding）*
*GitHub：[qianmao1989](https://github.com/qianmao1989)*

> 有问题或建议？请到 [Discussions](https://github.com/qianmao1989/multi-agent-communication/discussions) 留言，或直接开 Issue。
