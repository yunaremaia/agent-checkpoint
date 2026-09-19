# Agent Checkpoint

> Crash recovery that preserves exact AI agent state — decisions, reasoning log, accumulated context — with deterministic resume. No re-sampling after crash.

## The Problem

Long-running AI agents crash. Servers restart. Networks partition. Current recovery strategies:

- **Restart from scratch**: Wastes work, duplicates side effects, changes context.
- **Resume with memory loss**: Agent "forgets" decisions it made before the crash.
- **Human-in-the-loop**: Expensive, slow, breaks autonomous workflows.

The real problem: **decision drift after crash**. Even if you resume, the agent re-reasons from partial state and makes *different* decisions — because accumulated context and past decisions aren't deterministically restored.

## The Solution

`agent-checkpoint` provides crash recovery with **exact state preservation**:

```
Before crash:        After crash (current tools):        After crash (this tool):
─────────────        ────────────────────────────        ──────────────────────────
Step 1 ✅            Start over                           Skip Step 1 (cached)
Step 2 ✅            Re-execute Step 1 (wastes work)    Skip Step 2 (cached)
Step 3 💥 CRASH      Re-execute Step 2 (wastes work)    Resume from Step 3
Step 4 (pending)     Guess Step 3 context                Exact state restored
Step 5 (pending)     Different decisions (drift!)        Same decisions preserved
```

## Key Principle

**Replay-resume, not restart.** Completed steps are skipped — their outputs are loaded verbatim. Pending steps resume with the exact accumulated state (decisions, reasoning log, tool call history) they would have had if no crash occurred.

## Features

- **State serialization** — After every step, full agent state is checkpointed: progress, decisions, reasoning log, tool history.
- **Deterministic resume** — On restart, completed steps are skipped; their results loaded verbatim.
- **Decision preservation** — No re-sampling. Past decisions are restored exactly.
- **Crash-proof journaling** — Atomic checkpoint writes survive mid-step crashes.
- **Framework-agnostic** — Works with any agent (Claude Code, Codex, OpenCode, Hermes, Aider, custom).
- **Rollback support** — `undo.sh` generation from checkpoints.

## Install

```bash
pip install agent-checkpoint
```

## Quick Start

```python
from agent_checkpoint import Checkpointer

checkpoint = Checkpointer(".agent-checkpoints/session-001")

# Before each step
checkpoint.before_step(step_id="step-3", task="Deploy to production")

try:
    result = run_deployment()
    checkpoint.after_step(result=result, decisions=["chose canary deployment"])
except Crash:
    # On resume, completed steps are skipped
    checkpoint.resume()  # Loads step-1, step-2 results verbatim
    # Continue from step-3
```

Or as middleware:

```bash
# Wrap any agent invocation
agent-checkpoint -- claude "Fix the auth bug and deploy"
```

## What Gets Checkpointed

| Component | Preserved | Why |
|-----------|-----------|-----|
| Step outputs | ✅ | Avoid re-execution |
| Agent decisions | ✅ | Prevent drift |
| Reasoning log | ✅ | Maintain context |
| Tool call history | ✅ | Enable replay |
| File system changes | ✅ | Restore workspace |
| Environment state | ✅ | Reproduce conditions |
| Accumulated context | ✅ | Preserve learning |

## Comparison

| Tool | State Pres. | Deterministic Resume | Drift-Free | Framework-Agn. |
|------|-------------|---------------------|------------|----------------|
| **agent-checkpoint** | Full | ✅ | ✅ | ✅ |
| agent-undo (yunaremaia) | Op-level | ❌ (no resume) | ❌ | ✅ |
| kompensa | Step-level | Partial | ❌ | JS only |
| agent-checkpoint-resume | Basic | ❌ | ❌ | ❌ |

## Integration with `agent-undo`

`agent-undo` records operations and generates rollback scripts. `agent-checkpoint` preserves *decision state* for resume. They complement:

- `agent-undo`: What happened? How to undo it?
- `agent-checkpoint`: Where were we? How to continue?

## Roadmap

- [ ] Distributed checkpointing for multi-agent systems
- [ ] Automatic crash detection and resume
- [ ] `agent-undo` integration for rollback+resume combo
- [ ] CLI wrapper for any agent invocation

## License

MIT
