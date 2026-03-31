# Reverse-Engineering Claude Code: What Anthropic's Decompiled Source Reveals About the Future of AI Agents

**Author:** Security Research  
**Date:** March 2026  
**Source:** Decompiled `@anthropic-ai/claude-code` npm package (v1.0.33)  
**Codebase Stats:** 1,884 files · 512,000+ lines · TypeScript/TSX  

---

## Abstract

Claude Code — Anthropic's AI-powered command-line coding assistant — ships as an obfuscated Bun-compiled binary via npm. However, a full decompilation reveals the original TypeScript source, complete with inline source maps, developer comments, internal Slack links, and unreleased feature flags.

This article documents a comprehensive static analysis of the decompiled source. Our findings reveal that Claude Code is not merely a coding assistant — it is the foundation for a **fully autonomous AI agent platform** with persistent memory, scheduled tasks, multi-agent orchestration, web browsing, voice input, remote control, and even a hidden collectible pet system.

We document **23 unreleased features**, the complete model registry (including Opus 4.6 and Sonnet 4.6), pricing structures, internal codenames, and the engineering patterns Anthropic uses to separate internal-only functionality from public releases.

> **Disclaimer:** All data was extracted from the publicly distributed npm package. No servers were accessed, no authentication was bypassed, and no terms of service were violated. This analysis is provided for educational and security research purposes.

---

## Table of Contents

1. [How the Decompilation Works](#1-how-the-decompilation-works)
2. [Architecture Overview](#2-architecture-overview)
3. [The Feature Flag System: `bun:bundle`](#3-the-feature-flag-system-bunbundle)
4. [Anti-Debugging & Security Mechanisms](#4-anti-debugging--security-mechanisms)
5. [The Complete Model Registry](#5-the-complete-model-registry)
6. [Internal Codenames Decoded](#6-internal-codenames-decoded)
7. [23 Unreleased Features](#7-23-unreleased-features)
8. [The YOLO Classifier: How Auto-Mode Actually Works](#8-the-yolo-classifier-how-auto-mode-actually-works)
9. [DreamTask: Autonomous Memory Consolidation](#9-dreamtask-autonomous-memory-consolidation)
10. [The Buddy System: A Hidden Gacha Pet Game](#10-the-buddy-system-a-hidden-gacha-pet-game)
11. [Context Window Optimization Tricks](#11-context-window-optimization-tricks)
12. [The Excluded-Strings Build Pipeline](#12-the-excluded-strings-build-pipeline)
13. [Leaked Internal References](#13-leaked-internal-references)
14. [Security Implications](#14-security-implications)
15. [Conclusion](#15-conclusion)

---

## 1. How the Decompilation Works

Claude Code is distributed via npm as `@anthropic-ai/claude-code`. The package is compiled using **Bun's single-file bundler**, which merges all TypeScript source into a single JavaScript output. However, critically, the compiled output retains **inline source maps** as base64-encoded blobs:

```
//# sourceMappingURL=data:application/json;charset=utf-8;base64,eyJ2ZXJz...
```

These source maps contain:
- Original TypeScript file names and paths
- Complete original source code (pre-compilation)
- Line/column mappings between compiled and original code

By extracting and decoding these source maps, combined with analyzing the compiled bundle's structure (which preserves module boundaries, comments, and variable names), the full original source tree can be reconstructed.

**Key file:** The main entry point is `src/main.tsx` (4,684 lines, 803KB), which bootstraps the entire application including feature flag initialization, anti-debugging checks, and the React-based TUI rendering pipeline.

---

## 2. Architecture Overview

The codebase follows a React-based terminal UI pattern using a fork of [Ink](https://github.com/vadimdemedes/ink) (the React renderer for CLIs). The architecture has six major layers:

```
┌──────────────────────────────────────────┐
│                main.tsx                   │  Entry point, bootstrap, feature gates
├──────────────────────────────────────────┤
│            screens/REPL.tsx              │  Main interactive loop (5,000+ lines)
├──────────────────────────────────────────┤
│    components/  │  hooks/  │  context/   │  React UI layer
├──────────────────────────────────────────┤
│              tools/                      │  30+ tool implementations
├──────────────────────────────────────────┤
│   services/    │  utils/   │  cli/       │  Business logic, API, shell
├──────────────────────────────────────────┤
│           state/AppStateStore.ts          │  Centralized state management
└──────────────────────────────────────────┘
```

### Key Files by Function

| File | Lines | Purpose |
|------|-------|---------|
| `src/main.tsx` | 4,684 | Entry point, anti-debug, bootstrap |
| `src/screens/REPL.tsx` | 5,100+ | Main interactive REPL loop |
| `src/cli/print.ts` | 3,900+ | Message rendering, tool output |
| `src/QueryEngine.ts` | 1,200+ | API query orchestration |
| `src/utils/model/model.ts` | 619 | Model selection & resolution |
| `src/utils/permissions/yoloClassifier.ts` | 1,200+ | Auto-mode safety classifier |
| `src/buddy/CompanionSprite.tsx` | 45KB | Animated ASCII pet sprites |
| `src/tools/ScheduleCronTool/prompt.ts` | 136 | Cron job scheduling system |
| `src/tasks/DreamTask/DreamTask.ts` | 158 | AI memory consolidation |
| `src/utils/fastMode.ts` | 533 | "Penguin Mode" (Fast Mode) |
| `src/coordinator/coordinatorMode.ts` | 84 | Multi-agent orchestration |
| `src/utils/context.ts` | 222 | Context window management |

---

## 3. The Feature Flag System: `bun:bundle`

Claude Code uses a **compile-time dead code elimination** system via Bun's bundler. Feature flags are not runtime checks — they are resolved at build time and unused code paths are completely removed from the output binary.

**File:** `src/main.tsx`, lines throughout  
**Import:** `import { feature } from 'bun:bundle'`

```typescript
// At build time, feature('KAIROS') resolves to true or false
// If false, the entire block is tree-shaken out of the bundle
if (feature('KAIROS')) {
  const kairosModule = require('./assistant/kairos.js')
  // ... entire Kairos feature code
}
```

### Build-Time Constant: `USER_TYPE`

The most critical build variable is `USER_TYPE`, which determines internal vs. external builds:

```typescript
// In external builds (npm), this is replaced with the literal string "external"
// In internal builds, it's replaced with "ant"
process.env.USER_TYPE === 'ant'
```

In the decompiled source, we see this replacement has already occurred:

```typescript
// Line 266 of main.tsx — the string "external" was inserted at build time
if ("external" !== 'ant' && isBeingDebugged()) {
  process.exit(1);
}
```

### All Feature Flags Found

We identified **27 unique feature flags** in the codebase:

| Feature Flag | Category | Description |
|-------------|----------|-------------|
| `KAIROS` | Agent | Persistent assistant/daemon mode |
| `KAIROS_BRIEF` | Agent | Brief message display mode |
| `KAIROS_CHANNELS` | Agent | Channel-based communication |
| `KAIROS_GITHUB_WEBHOOKS` | Agent | GitHub webhook subscriptions |
| `PROACTIVE` | Agent | Autonomous agent loop |
| `COORDINATOR_MODE` | Agent | Multi-agent orchestration |
| `AGENT_TRIGGERS` | Agent | Cron/scheduled task system |
| `AGENT_MEMORY_SNAPSHOT` | Agent | Persistent agent memory |
| `WEB_BROWSER_TOOL` | Tool | Browser automation |
| `MONITOR_TOOL` | Tool | Background monitoring |
| `VOICE_MODE` | Input | Speech-to-text input |
| `BRIDGE_MODE` | Remote | Mobile/web remote control |
| `CCR_MIRROR` | Remote | Cloud session mirroring |
| `SSH_REMOTE` | Remote | SSH-tunneled tool execution |
| `DIRECT_CONNECT` | Remote | Direct WebSocket connection |
| `TRANSCRIPT_CLASSIFIER` | Safety | Auto-mode safety classifier |
| `BASH_CLASSIFIER` | Safety | Bash command classifier |
| `BUDDY` | UI | Collectible pet companion |
| `TORCH` | Internal | Mystery internal command |
| `ULTRAPLAN` | Planning | Advanced multi-step planning |
| `FORK_SUBAGENT` | Agent | Agent forking/branching |
| `BG_SESSIONS` | Agent | Background session management |
| `UDS_INBOX` | IPC | Unix Domain Socket messaging |
| `LODESTONE` | Protocol | Deep link URL protocol |
| `MESSAGE_ACTIONS` | UI | Rich message interactions |
| `HISTORY_PICKER` | UI | Enhanced history browser |
| `CHICAGO_MCP` | Protocol | Enhanced MCP integration |
| `UPLOAD_USER_SETTINGS` | Cloud | Cloud settings sync |
| `TERMINAL_PANEL` | UI | Built-in terminal panel (tmux) |

---

## 4. Anti-Debugging & Security Mechanisms

### Debug Detection (`main.tsx`, lines 232-271)

Claude Code actively detects and blocks debugging attempts for non-Anthropic users:

```typescript
function isBeingDebugged() {
  const isBun = isRunningWithBun();

  // Check for inspect flags in process arguments
  const hasInspectArg = process.execArgv.some(arg => {
    if (isBun) {
      return /--inspect(-brk)?/.test(arg);
    } else {
      return /--inspect(-brk)?|--debug(-brk)?/.test(arg);
    }
  });

  // Check NODE_OPTIONS for inspect flags
  const hasInspectEnv = process.env.NODE_OPTIONS && 
    /--inspect(-brk)?|--debug(-brk)?/.test(process.env.NODE_OPTIONS);

  // Check for active Node.js inspector
  try {
    const inspector = (global as any).require('inspector');
    const hasInspectorUrl = !!inspector.url();
    return hasInspectorUrl || hasInspectArg || hasInspectEnv;
  } catch {
    return hasInspectArg || hasInspectEnv;
  }
}

// The kill switch — only for external users
if ("external" !== 'ant' && isBeingDebugged()) {
  process.exit(1);  // Immediate termination, no cleanup
}
```

The check is deliberately placed **before** the graceful shutdown handler is initialized, ensuring a hard exit with no recovery path.

### Git HEAD Tamper Detection (`utils/git/gitFilesystem.ts`)

The git integration validates that `.git/HEAD` references haven't been tampered with, rejecting path traversal and argument injection:

```typescript
// Reject path traversal and argument injection from a tampered HEAD.
// Reject path traversal in a tampered symref chain.
```

### Model Codename Masking (`utils/model/antModels.ts`)

Internal model codenames are actively masked from external users:

```typescript
function maskModelCodename(baseName: string): string {
  const [codename = '', ...rest] = baseName.split('-')
  const masked = codename.slice(0, 3) + '*'.repeat(Math.max(0, codename.length - 3))
  return [masked, ...rest].join('-')
}
```

---

## 5. The Complete Model Registry

### Model Configuration (`utils/model/configs.ts`, `model.ts`, `modelCost.ts`)

The source reveals the full model registry with provider-specific IDs:

| Model | First-Party ID | Bedrock ID | Vertex ID |
|-------|---------------|-----------|----------|
| Sonnet 4 | `claude-sonnet-4-20250514` | `us.anthropic.claude-sonnet-4-20250514-v1:0` | `claude-sonnet-4@20250514` |
| Sonnet 4.5 | `claude-sonnet-4-5-20250929` | Present | Present |
| **Sonnet 4.6** | `claude-sonnet-4-6` | Present | Present |
| Opus 4.5 | `claude-opus-4-5-20251101` | Present | Present |
| **Opus 4.6** | `claude-opus-4-6` | Present | Present |
| Haiku 4.5 | `claude-haiku-4-5-20251001` | Present | Present |

**Key observation:** Sonnet 4.6 and Opus 4.6 have **no date suffix** in their model IDs — a first for Anthropic. This suggests they are "evergreen" models that receive silent updates.

### Pricing Table (`utils/modelCost.ts`)

```typescript
const STANDARD_COSTS = {
  // Sonnet tier (3/15)
  'claude-sonnet': { input: 3, output: 15, cacheWrite: 3.75, cacheRead: 0.3 },
  
  // Opus 4.5+ tier (5/25) — 3x CHEAPER than Opus 4/4.1!
  'claude-opus-4-5': { input: 5, output: 25, cacheWrite: 6.25, cacheRead: 0.5 },
  'claude-opus-4-6': { input: 5, output: 25, cacheWrite: 6.25, cacheRead: 0.5 },
  
  // Fast Mode (Penguin Mode) — 6x premium
  'claude-opus-4-6-fast': { input: 30, output: 150, cacheWrite: 37.5, cacheRead: 3.0 },
}
```

### Output Token Limits (`utils/context.ts`, lines 149-210)

| Model | Default Output | Maximum Output |
|-------|---------------|---------------|
| Opus 4.6 | **64,000** | **128,000** |
| Sonnet 4.6 | 32,000 | **128,000** |
| Opus 4.5 / Sonnet 4.x | 32,000 | 64,000 |
| Opus 4.0 / 4.1 | 32,000 | 32,000 |

### Model Migration System (`migrations/`)

The codebase contains explicit migration scripts that reveal the model evolution timeline:

- **`migrateFennecToOpus.ts`** — Reveals "Fennec" was the codename for Opus 4.6
- **`migrateLegacyOpusToCurrent.ts`** — Silently remaps Opus 4.0/4.1 → Opus 4.6
- **`migrateSonnet45ToSonnet46.ts`** — Upgrades first-party users to Sonnet 4.6

```typescript
// migrateFennecToOpus.ts
// fennec-latest → opus
// fennec-latest[1m] → opus[1m]
// fennec-fast-latest → opus[1m] + fast mode
```

---

## 6. Internal Codenames Decoded

| Codename | Real Name | Evidence |
|----------|-----------|---------|
| **Fennec** 🦊 | Opus 4.6 | `migrateFennecToOpus.ts` |
| **Penguin** 🐧 | Fast Mode | `claude_code_penguin_mode` API endpoint |
| **Tengu** 👺 | Analytics System | All events prefixed `tengu_*` |
| **Coral Reef** 🪸 | 1M Context A/B Test | `coral_reef_sonnet` feature gate |
| **Kairos** ⏰ | Persistent Assistant | `feature('KAIROS')` flag |
| **Torch** 🔥 | Mystery Command | `feature('TORCH')` — purpose unknown |

---

## 7. 23 Unreleased Features

### Tier 1 — Autonomous Agent Infrastructure

#### KAIROS: Persistent AI Daemon
**Files:** `src/assistant/`, `src/commands/assistant.ts`

Kairos transforms Claude Code from a session-based tool into a **persistent background daemon**. Key capabilities:
- Background process that survives terminal closure
- GitHub webhook subscriptions for repository events
- Channel-based messaging (multiple conversation threads)
- Session history browsing and continuation
- Special "Brief" display mode for chat-style interaction

#### Proactive Mode
**Files:** `src/proactive/`, `src/commands/proactive.ts`

An autonomous loop where Claude **keeps working without user prompts**:
```typescript
// Proactive mode: schedule a tick to keep the model looping autonomously
if (proactiveModule?.isProactiveActive() && !proactiveModule.isProactivePaused()) {
  scheduleProactiveTick!()
}
```
Includes a `SleepTool` that lets Claude pause between autonomous ticks.

#### Agent Triggers / Cron Scheduling
**Files:** `src/tools/ScheduleCronTool/` (5 files, ~27KB)

A complete cron job system with three tools:
- **CronCreate** — Schedule recurring or one-shot tasks
- **CronDelete** — Cancel scheduled jobs  
- **CronList** — View scheduled jobs

```typescript
// Durable mode persists to .claude/scheduled_tasks.json
// Session-only mode dies with the process
// Auto-expiry: recurring tasks auto-expire after N days
// Smart jitter: avoids :00/:30 stampedes across fleet
```

### Tier 2 — Advanced Capabilities

#### Web Browser Tool
**Files:** Referenced in `main.tsx` and `REPL.tsx`

Full browser automation using Bun's built-in `WebView` API. The tool is conditionally loaded:
```typescript
const WebBrowserPanelModule = feature('WEB_BROWSER_TOOL') 
  ? require('../tools/WebBrowserTool/WebBrowserPanel.js') 
  : null
```

#### Coordinator Mode
**Files:** `src/coordinator/coordinatorMode.ts` (84 lines)

Multi-agent orchestration where one Claude instance acts as a leader, delegating tasks to worker agents:
```typescript
export function isCoordinatorMode(): boolean {
  if (feature('COORDINATOR_MODE')) {
    return isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)
  }
  return false
}
```

#### Bridge Mode (Remote Control)
**Files:** Referenced throughout `hooks/useReplBridge.tsx`

Control Claude Code from mobile or web devices:
- WebSocket real-time communication
- JWT device trust management
- QR code pairing flow
- Message bridging between local and remote instances

### Tier 3 — All Other Hidden Features

| Feature | Flag | Purpose |
|---------|------|---------|
| Voice Mode | `VOICE_MODE` | Push-to-talk speech input |
| SSH Remote | `SSH_REMOTE` | Local REPL, remote tools via SSH |
| Direct Connect | `DIRECT_CONNECT` | Direct WebSocket to Claude server |
| Monitor Tool | `MONITOR_TOOL` | Persistent background monitoring |
| Transcript Classifier | `TRANSCRIPT_CLASSIFIER` | AI-based auto-approval |
| Fork Subagent | `FORK_SUBAGENT` | Branch a running agent |
| Background Sessions | `BG_SESSIONS` | Concurrent session management |
| Agent Memory Snapshot | `AGENT_MEMORY_SNAPSHOT` | Persistent agent memory |
| UDS Inbox | `UDS_INBOX` | Unix Domain Socket IPC |
| Lodestone | `LODESTONE` | Deep link URL protocol |
| Ultraplan | `ULTRAPLAN` | Advanced multi-step planning |
| Chicago MCP | `CHICAGO_MCP` | Enhanced MCP integration |
| Terminal Panel | `TERMINAL_PANEL` | Built-in tmux panel |
| Upload Settings | `UPLOAD_USER_SETTINGS` | Cloud settings sync |

---

## 8. The YOLO Classifier: How Auto-Mode Actually Works

**File:** `src/utils/permissions/yoloClassifier.ts` (1,200+ lines)

What users know as "auto mode" is internally called the **YOLO Classifier**. This is a separate AI call that evaluates whether a tool action is safe to auto-approve.

### Architecture:

```
User Request → Claude generates tool_use → YOLO Classifier evaluates
  ├─ if SAFE → Auto-approve, execute silently
  ├─ if UNSAFE → Show permission dialog to user
  └─ if DENIED → Block and notify user
```

### Key implementation details:

1. **Separate model call** — The classifier makes its own API call to Claude with a specialized system prompt
2. **Two classifier modes:**
   - `auto_mode` — Full transcript classifier for general tool calls
   - `bash_allow` — Specialized classifier for shell commands
3. **Speculative execution** — The classifier starts evaluating bash commands **before** the permission dialog appears, so if it approves in <2 seconds, the dialog never shows
4. **Denial tracking** — After too many denials, falls back to always prompting

```typescript
// From yoloClassifier.ts
export const YOLO_CLASSIFIER_TOOL_NAME = 'classify_result'

export async function classifyYoloAction(
  tool, input, toolUseContext, assistantMessage, toolUseID, context
): Promise<YoloClassifierResult> {
  const systemPrompt = await buildYoloSystemPrompt(context)
  // ... makes a separate Claude API call to classify the action
}
```

---

## 9. DreamTask: Autonomous Memory Consolidation

**File:** `src/tasks/DreamTask/DreamTask.ts` (158 lines)  
**Dependency:** `src/services/autoDream/consolidationLock.js`

This is perhaps the most philosophically interesting feature. Claude Code implements **autonomous memory consolidation** — a background process that reviews past conversations and updates its memory while idle.

```typescript
// "Background task entry for auto-dream (memory consolidation subagent)"
// "the dream prompt has a 4-stage structure: orient/gather/consolidate/prune"

export type DreamPhase = 'starting' | 'updating'

export type DreamTaskState = TaskStateBase & {
  type: 'dream'
  phase: DreamPhase
  sessionsReviewing: number
  filesTouched: string[]
  turns: DreamTurn[]
  abortController?: AbortController
  priorMtime: number
}
```

### The Four Stages:

1. **Orient** — Determine which past sessions to review
2. **Gather** — Read session data and extract key information
3. **Consolidate** — Merge new learnings into memory files (CLAUDE.md, etc.)
4. **Prune** — Remove outdated or redundant information

### Concurrency Control:

A file-based lock (`consolidationLock.ts`) prevents multiple dream agents from running simultaneously. If a dream is killed, the lock's mtime is rolled back so the next session can retry:

```typescript
async kill(taskId, setAppState) {
  // Rewind the lock mtime so the next session can retry
  if (priorMtime !== undefined) {
    await rollbackConsolidationLock(priorMtime)
  }
}
```

---

## 10. The Buddy System: A Hidden Gacha Pet Game

**Files:** `src/buddy/` (6 files, ~75KB total)

The most unexpected finding. Claude Code contains a **complete collectible pet system** with gacha mechanics, RPG stats, animated ASCII art, and rarity tiers.

### Species (18 total):
Duck, Goose, Blob, Cat, Dragon, Octopus, Owl, Penguin, Turtle, Snail, Ghost, Axolotl, Capybara, Cactus, Robot, Rabbit, Mushroom, Chonk

### Rarity Distribution:
```typescript
export const RARITY_WEIGHTS = {
  common: 60,      // 60%
  uncommon: 25,     // 25%
  rare: 10,         // 10%
  epic: 4,          //  4%
  legendary: 1,     //  1%
}
```

### Stats System:
Each companion has five RPG-like stats: **DEBUGGING**, **PATIENCE**, **CHAOS**, **WISDOM**, **SNARK** — rolled deterministically from a seeded PRNG (Mulberry32) using the user's account UUID.

### Cosmetics:
- **Eyes:** `·`, `✦`, `×`, `◉`, `@`, `°`
- **Hats:** crown, tophat, propeller, halo, wizard, beanie, tiny duck
- **Shiny:** 1% chance of shiny variant

### Anti-Leak Engineering:

Because one species name (`penguin`) collides with the codename for Fast Mode, **all species names are encoded as hex character codes** to avoid triggering the `excluded-strings.txt` build scanner:

```typescript
// Instead of: export const penguin = 'penguin'
export const penguin = c(0x70, 0x65, 0x6e, 0x67, 0x75, 0x69, 0x6e) as 'penguin'

// The comment explaining this:
// "One species name collides with a model-codename canary in excluded-strings.txt."
```

### Soul Generation:

The companion's personality is AI-generated — Claude creates a unique name and personality description for your pet, stored in config as `CompanionSoul`:

```typescript
export type CompanionSoul = {
  name: string
  personality: string
}
```

---

## 11. Context Window Optimization Tricks

### The 8K Default Cap (`utils/context.ts`, lines 18-25)

Anthropic's most impactful optimization. Despite models supporting 32K-128K output tokens, Claude Code defaults to **only 8,000 tokens**:

```typescript
// Capped default for slot-reservation optimization. BQ p99 output = 4,911
// tokens, so 32k/64k defaults over-reserve 8-16× slot capacity. With the cap
// enabled, <1% of requests hit the limit; those get one clean retry at 64k
export const CAPPED_DEFAULT_MAX_TOKENS = 8_000
export const ESCALATED_MAX_TOKENS = 64_000
```

If a response hits the 8K cap, the system transparently retries with 64K. Since p99 output is only ~5K tokens, fewer than 1% of requests need the retry — saving 8-16× GPU slot capacity fleet-wide.

### 1M Context Window Gating

The 1M context window feature has multiple control layers:

```typescript
// Admin override (C4E HIPAA compliance)
export function is1mContextDisabled(): boolean {
  return isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_1M_CONTEXT)
}

// Model support check
export function modelSupports1M(model: string): boolean {
  const canonical = getCanonicalName(model)
  return canonical.includes('claude-sonnet-4') || canonical.includes('opus-4-6')
}

// Secret A/B test (Coral Reef)
export function getSonnet1mExpTreatmentEnabled(model: string): boolean {
  return getGlobalConfig().clientDataCache?.['coral_reef_sonnet'] === 'true'
}
```

---

## 12. The Excluded-Strings Build Pipeline

**Referenced in:** `scripts/excluded-strings.txt` (file not in npm package)

Anthropic maintains a **build-time blacklist file** that scans the compiled output for strings that must never appear in external builds. If any blacklisted string is found, the build fails.

This system prevents:
- Internal model codenames from leaking
- Internal feature names from appearing
- Internal UUIDs from being exposed
- Employee-specific configuration keys from shipping

Evidence from the source:

```typescript
// utils/model/antModels.ts, line 33:
// @[MODEL LAUNCH]: Add the codename to scripts/excluded-strings.txt 
// to prevent it from leaking to external builds.

// utils/model/model.ts, line 4:
// scripts/excluded-strings.txt to avoid leaking them. Wrap any codename string

// buddy/types.ts, line 10:
// One species name collides with a model-codename canary in excluded-strings.txt.

// screens/REPL.tsx, line 112:
// eliminated from external builds (one UUID is on excluded-strings).
```

---

## 13. Leaked Internal References

### Anthropic Slack Channels

The decompiled source contains direct links to internal Anthropic Slack discussions:

| File | Slack Channel | Timestamp |
|------|--------------|-----------|
| `screens/REPL.tsx:304` | `C07VBSHV7EV` | `p1773545449871739` |
| `services/mcp/config.ts:1500` | `C093UA0KLD7` | `p1764975463670109` |
| `ink/termio/osc.ts:123` | `C07VBSHV7EV` | `p1773177228548119` |
| `utils/sessionState.ts:126` | `C093BJBD1CP` | `p1774152406752229` |
| `utils/messages.ts:2604` | `C06FE2FP0Q2` | `p1747586370117479` |
| `utils/auth.ts:251` | `C08428WSLKV` | `p1747331773214779` |

### Internal GitHub Issue References

The code references internal GitHub issues by number (e.g., `#23985`, `#31759`, `#37596`, `#24498`, `#18453`) which correlate to the `anthropics/claude-code` private repository.

### Internal-Only Commands (30+)

```typescript
// commands.ts, lines 225-254
export const INTERNAL_ONLY_COMMANDS = [
  backfillSessions, breakCache, bughunter, commit, commitPushPr,
  ctx_viz, goodClaude, issue, initVerifiers, forceSnip, mockLimits,
  bridgeKick, version, ultraplan, subscribePr, resetLimits,
  resetLimitsNonInteractive, onboarding, share, summary, teleport,
  antTrace, perfIssue, env, oauthRefresh, debugToolCall,
  agentsPlatform, autofixPr,
].filter(Boolean)
```

---

## 14. Security Implications

### 1. Source Code Exposure
The inclusion of inline source maps in the npm package effectively makes Claude Code **open-source by accident**. Any engineer with basic tooling can reconstruct the complete TypeScript source.

### 2. Feature Flag Bypass Potential
While `bun:bundle` feature flags are compile-time constants (not runtime toggleable), environment variables like `CLAUDE_CODE_COORDINATOR_MODE` and `CLAUDE_CODE_PROACTIVE` are checked at runtime and could potentially be set by users.

### 3. Internal Reference Leakage
Slack channel IDs, GitHub issue numbers, and internal API endpoints provide organizational intelligence that could be used for social engineering.

### 4. Anti-Debug Bypass
The debug detection in `main.tsx` can be trivially bypassed by patching the compiled JavaScript before execution, since the check is a simple `process.exit(1)` call.

### 5. Model Codename Leakage
Despite the `excluded-strings.txt` pipeline, codenames like "Fennec", "Penguin", "Coral Reef", and "Tengu" are all discoverable through migration scripts, API endpoints, and developer comments.

---

## 15. Conclusion

Claude Code's decompiled source reveals that Anthropic is building far more than a coding assistant. The feature flags, architectural patterns, and infrastructure code point to a **fully autonomous AI agent platform** with:

- **Persistent execution** (KAIROS daemon mode)
- **Self-directed work** (Proactive mode, scheduled tasks)
- **Self-improving memory** (DreamTask consolidation)
- **Multi-agent coordination** (Coordinator mode, Fork Subagent)
- **Multi-modal input** (Voice, browser, SSH remote)
- **Remote orchestration** (Bridge, Direct Connect, CCR Mirror)

The engineering quality is high — the codebase shows careful attention to security boundaries, build-time dead code elimination, and gradual feature rollout via GrowthBook. However, the accidental inclusion of source maps and incomplete sanitization of internal references provides an unusually detailed window into Anthropic's product roadmap and engineering culture.

The hidden Buddy pet system, while seemingly frivolous, reveals an important insight: Anthropic's engineering team views Claude Code as a **long-term relationship product**, not a disposable tool. You don't build gacha systems and AI-generated pet personalities for something you expect users to fire up once and forget.

---

## Appendix: File Index

| Path | Size | Description |
|------|------|-------------|
| `src/main.tsx` | 804KB | Entry point, anti-debug, bootstrap |
| `src/screens/REPL.tsx` | ~250KB | Interactive REPL with all feature gates |
| `src/cli/print.ts` | ~180KB | Message rendering pipeline |
| `src/utils/model/model.ts` | 20KB | Model selection logic |
| `src/utils/model/configs.ts` | 4KB | Model ID mappings |
| `src/utils/model/modelOptions.ts` | 18KB | TUI model picker |
| `src/utils/model/deprecation.ts` | 3KB | Deprecation schedule |
| `src/utils/model/aliases.ts` | 1KB | Model family aliases |
| `src/utils/model/antModels.ts` | 2KB | Internal-only models |
| `src/utils/modelCost.ts` | 8KB | Pricing 표 |
| `src/utils/fastMode.ts` | 18KB | Penguin Mode logic |
| `src/utils/context.ts` | 7KB | Context window management |
| `src/utils/permissions/yoloClassifier.ts` | ~50KB | YOLO mode classifier |
| `src/buddy/CompanionSprite.tsx` | 46KB | ASCII pet animations |
| `src/buddy/types.ts` | 4KB | Pet type definitions |
| `src/buddy/companion.ts` | 4KB | Pet generation logic |
| `src/buddy/sprites.ts` | 10KB | Sprite frame data |
| `src/tasks/DreamTask/DreamTask.ts` | 5KB | Memory dream system |
| `src/tools/ScheduleCronTool/prompt.ts` | 8KB | Cron scheduling prompts |
| `src/coordinator/coordinatorMode.ts` | 3KB | Multi-agent orchestration |
| `src/migrations/migrateFennecToOpus.ts` | — | Fennec → Opus 4.6 |
| `src/migrations/migrateLegacyOpusToCurrent.ts` | 2KB | Opus 4.0/4.1 → 4.6 |
| `src/commands.ts` | 25KB | Command registry w/ internals |

---

*This analysis was performed on the publicly available npm package `@anthropic-ai/claude-code` version 1.0.33. All information was obtained through static analysis of the distributed binary and its embedded source maps. No servers were accessed, no authentication was bypassed, and no terms of service were violated.*
