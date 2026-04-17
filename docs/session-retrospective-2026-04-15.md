# Session Retrospective — 2026-04-15

## Purpose

This document captures the key pitfalls, wrong turns, and successful iterations from the long local customization session on the `oh-my-openagent` repo.

The goal is to preserve the practical lessons so future changes do not repeat the same mistakes.

## Highest-Impact Pitfalls

### 1. Changing repo code is not enough if OpenCode is not loading this repo

The biggest source of confusion was assuming that editing the local repository immediately changed real OpenCode runtime behavior.

That assumption was wrong multiple times.

What actually mattered was how OpenCode loaded the plugin:

- package-name plugin loading used cached/runtime-installed artifacts
- local repo edits only affected real runtime after OpenCode was configured to load the repo directly
- the durable fix was using the supported local plugin entry:

```json
"plugin": [
  "file:///C:/Users/wykk/Desktop/project/oh-my-openagent"
]
```

Key lesson: always verify the runtime load path before trusting local source changes.

### 2. Build output matters more than source edits for actual runtime

Another repeated pitfall was treating `src/` edits as if OpenCode would consume them directly.

In practice, the effective runtime artifact is the built plugin entry, centered on:

```txt
dist/index.js
```

Key lesson: local plugin development in this repo usually means:

1. edit source
2. run `bun run build`
3. verify runtime is loading this repo via `file:///...`

### 3. Hephaestus-first was blocked by more than one layer

Making Hephaestus the preferred implementation agent was not a single-switch change.

The session uncovered several separate blockers:

- Hephaestus was skipped from agent collection
- Hephaestus had the wrong mode for subagent use
- Sisyphus prompts still biased execution toward self-work or category delegation
- runtime agent lookup broke because display/runtime names did not align
- zero-width sorting prefixes made runtime matching fragile

Key lesson: delegation behavior lives across prompt policy, agent registration, runtime name mapping, and task execution.

### 4. Several early root-cause theories were wrong

Two important examples:

- The first explanation for the `sisyphus-junior` model issue was incorrect and had to be corrected after Oracle review.
- The first fix for Claude Opus “thinking strength” was also wrong because it changed thinking budget instead of passing `variant -> effort`.

Key lesson: if behavior depends on runtime params or provider translation, inspect the actual parameter path instead of patching a nearby-looking config field.

### 5. Runtime logs can look scary while actually indicating recovery

The repeated log line:

```txt
[tool-pair-validator] Repaired missing tool_result blocks
```

looked like a major error, but turned out to be a recovery hook repairing malformed or compacted tool message history.

Key lesson: distinguish between:

- fatal runtime errors
- automatic recovery hooks
- noisy but expected transform-time repairs

## Key Iterations That Mattered

### Iteration 1: `sisyphus-junior` model override debugging

The session started from the question of why configuring:

```json
"agents": {
  "sisyphus-junior": {
    "model": "nous/mimo-v2-pro"
  }
}
```

did not appear to take effect.

The important final conclusion was that category-level model settings could override the per-agent expectation in actual delegated flows, especially when using category-based task routing.

### Iteration 2: restoring Hephaestus as a real implementation agent

This happened in multiple stages:

1. un-skipping Hephaestus and `sisyphus-junior`
2. changing Hephaestus mode so it could be used as a subagent
3. adding missing metadata so it appeared in delegation guidance
4. rewriting Sisyphus prompt guidance so implementation tasks preferred Hephaestus
5. fixing runtime agent-name alignment so the delegation actually worked in a live session

The critical proof came from runtime logs showing:

- Sisyphus delegated to `Hephaestus - Deep Agent`
- task poll loop started with `Hephaestus - Deep Agent`
- child session actually ran as `Hephaestus - Deep Agent`

### Iteration 3: correcting Claude Opus effort handling

The wrong path was changing thinking budget.

The correct path was:

- keep default thinking budget behavior alone
- inject the configured agent `variant`
- let the Anthropic effort hook convert that into runtime `effort`

This led to the right mental model:

- for this Opus flow, the important control path is `variant -> effort`
- not “change some default thinking token budget and hope it behaves similarly”

### Iteration 4: logging and runtime observability

Logging work also evolved in stages:

1. inspect final output options
2. add focused Sisyphus delegation logging
3. reduce logs when they became too noisy
4. move logs from temp assumptions to the repo-specific desired path
5. finally realize log path debugging was blocked by plugin loading mode, not logger code itself

This was one of the most important lessons of the session.

### Iteration 5: durable fix for missing project log file

At first, the missing log problem looked like a logger bug.

It was not.

The durable fix came from changing OpenCode plugin loading to the supported local file URL path. Once real runtime was proven to load the local repo plugin, the project log file started appearing under:

```txt
C:\Users\wykk\Desktop\project\oh-my-openagent\log\oh-my-opencode.log
```

### Iteration 6: upstream sync while preserving customizations

The repo was updated from official upstream by:

1. stashing tracked local modifications
2. fast-forwarding local `dev` to latest `origin/dev`
3. restoring the stashed local changes
4. validating typecheck/build
5. proving the Hephaestus-first customization still worked after sync

This established the baseline update workflow that later became the new `sync-upstream` skill.

## Stable Conclusions Reached in This Session

### Local runtime workflow

For this repo, the stable local-dev workflow is:

1. point OpenCode plugin config at `file:///...` for this repo
2. edit source
3. run `bun run build`
4. verify behavior using real runtime commands and logs

### Hephaestus-first implementation behavior

The session reached a working state where Sisyphus can now delegate implementation work to Hephaestus in real runtime, instead of only doing so in prompt text.

### Logging interpretation

Not every alarming log line is a bug.

Specifically:

- `tool-pair-validator` repair logs are recovery noise
- `Agent not found` logs were real blockers
- provider/model fallback logs were important for understanding unexpected model switches

### Upstream update workflow

The safe repeatable workflow for this customized checkout is now documented in the project skill:

```txt
.opencode/skills/sync-upstream/SKILL.md
```

## Recommended Guardrails for Future Sessions

### Always verify runtime before trusting local edits

Before debugging behavior, confirm:

- which plugin path OpenCode is loading
- whether the local repo is loaded directly or through cached package artifacts
- whether `dist/index.js` reflects the source changes you think are active

### Prefer runtime evidence over theory

When possible, trust:

- real `opencode run ... --print-logs`
- live session logs
- explicit provider/model/agent snapshots

over indirect reasoning from source alone.

### Treat Oracle corrections seriously

This session repeatedly benefited from Oracle rejecting early explanations that sounded plausible but were wrong.

The most useful pattern was:

1. form a theory
2. verify with runtime evidence
3. ask Oracle to challenge the theory
4. correct the explanation if Oracle finds a gap

## Final State at End of Session

By the end of the session:

- official upstream code had been brought into the local repo
- local custom changes were preserved
- Hephaestus-first runtime delegation was working
- local plugin loading used the supported `file:///...` path
- project log generation was working under the repo `log/` directory
- a reusable `sync-upstream` skill was added for future updates

## Short Version

The biggest mistake was repeatedly debugging the repo as if the repo itself were the runtime. The biggest win was switching to runtime-first verification: proving which plugin was loaded, which model/provider/agent was chosen, and whether the final behavior actually happened in a real OpenCode session.
