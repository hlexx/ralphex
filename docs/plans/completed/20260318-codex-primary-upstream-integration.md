# Integrate upstream v0.24.1 and preserve codex-primary execution

## Overview

Update the fork to the current upstream core (`origin/master` at `v0.24.1`) and re-apply the local
Codex-first execution model as an explicit, optional layer. The default upstream Claude-first flow
must remain intact. Local extensions must keep Codex as a first-class primary agent for task
execution, planning, and Codex-specific review prompts.

## Context

- The local branch diverged substantially from upstream in the same files:
  `cmd/ralphex/main.go`, `pkg/processor/runner.go`, `pkg/config/config.go`, `README.md`
- Upstream added new core behavior (`session_timeout`, `review_patience`, `wait_on_limit`,
  configurable `codex_review.txt`) while the fork changed executor semantics
- A raw rebase would be high-risk: text conflicts are manageable, but semantic drift in executor
  selection and review flow is the real problem
- Product requirement for the fork: Codex must remain usable as the primary executor without turning
  that mode into the default behavior for all users

## Solution Overview

- Start from fresh upstream (`origin/master`) on a dedicated integration branch
- Reintroduce `--codex-primary`, `--codex-model`, and `--codex-thinking` as an explicit runtime
  layer on top of upstream
- Keep upstream defaults for normal execution, but make plan mode and Codex review phases use Codex
  with stronger defaults (`gpt-5.4`, `xhigh`)
- Preserve separate prompt surfaces for Claude-first review and Codex-primary review
- Merge the local Codex error-pattern fix into the newer upstream retry and timeout behavior

## Development Approach

- Testing approach: regular (code first, then package-level verification)
- Preserve upstream behavior as the baseline; fork-specific behavior must be opt-in
- Any shared contract changes must be documented in code, README, CLAUDE.md, and llms.txt

## Implementation Steps

### Task 1: Reintroduce Codex-primary CLI and runtime config

**Files:**
- Modify: `cmd/ralphex/main.go`
- Modify: `cmd/ralphex/main_test.go`
- Modify: `pkg/config/defaults/config`

- [x] add `--codex-primary`, `--codex-model`, and `--codex-thinking`
- [x] validate that `--codex-primary` requires writable Codex sandbox
- [x] make plan mode use Codex availability checks instead of Claude-only checks
- [x] add runtime config resolution helpers for Codex model/reasoning overrides
- [x] keep default upstream execution path unchanged when `--codex-primary` is not set
- [x] add CLI and config resolution tests

### Task 2: Rewire runner for optional Codex primary execution

**Files:**
- Modify: `pkg/processor/runner.go`
- Modify: `pkg/processor/prompts.go`
- Modify: `pkg/processor/prompts_test.go`

- [x] extend `processor.Config` with `UseCodexForPrimary`
- [x] split Codex executor creation into primary-task and review-specific executors
- [x] keep external review Codex path intact for upstream default mode
- [x] make first and second review prompt selection depend on `UseCodexForPrimary`
- [x] add tests for Codex-primary prompt selection

### Task 3: Restore Codex-primary prompt/config surface

**Files:**
- Modify: `pkg/config/config.go`
- Modify: `pkg/config/prompts.go`
- Modify: `pkg/config/config_test.go`
- Modify: `pkg/config/prompts_test.go`
- Add: `pkg/config/defaults/prompts/review_first_codex.txt`
- Add: `pkg/config/defaults/prompts/review_second_codex.txt`

- [x] add config wiring for `review_first_codex.txt` and `review_second_codex.txt`
- [x] load embedded defaults for both prompt files
- [x] preserve existing Claude review prompt files for default mode
- [x] add tests that verify the new prompt files are present and load correctly

### Task 4: Merge Codex execution fixes with upstream core

**Files:**
- Modify: `pkg/executor/codex.go`
- Modify: `pkg/executor/codex_test.go`

- [x] preserve the local false-positive fix for Codex rate-limit/error detection
- [x] make error-pattern matching consider stderr on failed runs only
- [x] keep upstream limit retry and timeout flow unchanged
- [x] add regression test for stderr-tail pattern matching

### Task 5: Update product and operator documentation

**Files:**
- Modify: `README.md`
- Modify: `CLAUDE.md`
- Modify: `llms.txt`

- [x] document `--codex-primary`, `--codex-model`, and `--codex-thinking`
- [x] document writable-sandbox requirement for Codex-primary mode
- [x] document phase-specific Codex defaults (`medium` for task execution, `xhigh` for planner and review)
- [x] document new prompt files used by Codex-primary mode

## Verification

- [x] `go test -run '^$' ./cmd/ralphex`
- [x] `go test -run '^$' ./pkg/config ./pkg/processor ./pkg/executor`
- [x] `go test ./pkg/config`
- [x] `go test ./pkg/processor`
- [x] `go test ./pkg/executor`
- [x] targeted `cmd/ralphex` tests for mode detection, config overrides, runner wiring, and plan mode
- [ ] `go test ./...` — not fully green in this environment; unrelated `pkg/progress` color test is environment-sensitive and full-suite package runs are slow/hanging outside the touched layers

## Post-Completion Notes

- The fork now tracks upstream core from a fresh integration branch instead of extending the old
  divergent branch
- `codex-primary` remains first-class but opt-in
- Future upstream pulls should continue to treat the Codex-primary behavior as a narrow integration
  layer, not as a replacement for the upstream default architecture
