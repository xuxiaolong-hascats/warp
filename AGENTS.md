# AGENTS.md

This file guides contributors and AI agents working anywhere in this repository.

Warp is a large Rust workspace for the open-source Warp client: an agentic development environment built around a terminal UI. Prefer repository facts and existing patterns over assumptions.

## Scope and Intent

- This file is for both human contributors and AI coding agents.
- When the guidance below conflicts with a more specific `AGENTS.md` in a subdirectory, the deeper file wins for that subtree.
- Treat this file as operational guidance, not a substitute for product or architectural docs. Read `README.md`, `WARP.md`, and nearby module docs before making changes.

## Repository Shape

- `app/` contains the main Warp application and entrypoints.
- `crates/` contains the shared Rust libraries that make up most of the system.
- `crates/warpui/` and `crates/warpui_core/` contain Warp's custom UI framework.
- `crates/integration/` contains the integration test framework.
- `script/` contains the standard setup, run, and presubmit commands.
- `resources/` and `assets/` contain bundled resources and static assets.

## Working Style

- Make the smallest change that fully solves the task.
- Match existing patterns in the touched area before introducing a new one.
- Do not refactor unrelated code, rename unrelated symbols, or reformat broad areas "while you are here".
- If you notice a real adjacent problem that should not be changed as part of the task, mention it separately instead of folding it into the diff.
- Prefer explicit, verifiable outcomes over speculative cleanup.

## Before You Edit

- Read the relevant top-level docs first: `README.md` and `WARP.md`.
- Read the target module and the immediate surrounding code before changing anything.
- Identify the verification path up front. For bug fixes, prefer reproducing the issue first. For behavior changes, identify the smallest test or manual check that proves the result.
- For larger work, state a brief step plan and the check for each step.

## Build, Run, and Verify

Use the existing repo scripts and commands instead of inventing new workflows.

- Initial setup: `./script/bootstrap`
- Run locally: `./script/run`
- Full presubmit: `./script/presubmit`
- Format Rust: `cargo fmt`
- Lint Rust: `cargo clippy --workspace --all-targets --all-features --tests -- -D warnings`
- Run workspace tests: `cargo nextest run --no-fail-fast --workspace --exclude command-signatures-v2`

Prefer the smallest relevant verification that proves your change during development, then use broader checks when preparing work for review.

## Architecture Notes That Matter In Practice

### WarpUI

- Warp uses a custom UI framework, not a web frontend.
- Views and models are owned by the global `App` and referenced via handles.
- Access to handle-backed entities is context-bound. Follow existing `AppContext` / `ViewContext` / `ModelContext` usage patterns instead of improvising ownership changes.
- If you are editing UI code, preserve the established WarpUI structure unless the task explicitly requires changing it.

### Terminal Model Locking

- Be careful with `model.lock()` on `TerminalModel`.
- Do not add nested or redundant locking on the same terminal model in the same call chain.
- Prefer passing an already-locked reference down rather than reacquiring locks.
- Keep lock scope tight and avoid calling into code that may lock again.

### Feature Flags

- Prefer runtime checks with `FeatureFlag::YourFlag.is_enabled()` over new `#[cfg(...)]` gates unless compilation truly requires a compile-time guard.
- Keep flags product-level, not scattered per call site.
- Gate new UI entry points behind the same flag as the underlying behavior.

## Code Style Expectations

- Follow the local style in the files you touch.
- Avoid unnecessary type annotations, especially in obvious closures.
- Prefer imports over long path qualifiers, except where local cfg-gated code makes that awkward.
- If a function takes an app/view/model context parameter, name it `ctx` and keep it last unless the function takes a closure, in which case the closure stays last.
- Remove truly unused parameters rather than prefixing them with `_` when you are already changing that code.
- Prefer inline format args such as `format!("{value}")`.
- Do not remove or rewrite existing comments unless the underlying behavior changed.
- When editing `match` statements, prefer exhaustive handling over wildcard arms where practical.

## Testing Expectations

- Every non-trivial code change should have a verification path.
- Add or update tests when behavior changes and the codebase already has a natural place for that coverage.
- Follow the repo's Rust test layout conventions:
  - Prefer separate test files like `foo_tests.rs` or `mod_test.rs`
  - Include them at the end of the corresponding module with `#[cfg(test)]`
- Use integration tests in `crates/integration/` for cross-feature or end-to-end behavior, not for narrow unit logic.

## Expectations For AI Agents

- Do not guess product intent from a file name alone. Read enough surrounding code to understand the feature boundary.
- Do not silently choose between multiple plausible interpretations of a task when that choice changes behavior materially. Call out the ambiguity.
- Do not introduce abstractions for hypothetical reuse. Single-use code should stay simple.
- Do not make unrelated "cleanup" edits in files you touch.
- If a task requires broad changes across modules, explain the slice you intend to change before editing.
- If you encounter unexpected user changes or a dirty worktree, work with them. Do not revert unrelated modifications.
- Before finalizing, summarize what changed, how it was verified, and any checks you could not run.

## Pull Request Readiness

Before opening or updating a PR:

- Run `cargo fmt`
- Run the relevant clippy/test commands for the changed area
- Run broader checks when the scope warrants it, ideally via `./script/presubmit`
- Use `.github/pull_request_template.md`
- Add a changelog entry only when appropriate for the user-facing impact

## Good Defaults

- Prefer surgical diffs.
- Prefer existing patterns.
- Prefer direct verification.
- Prefer clarity over cleverness.

## Personal Fork Workflow

For this fork, use the following maintenance model unless a future owner explicitly changes it:

- `origin` should continue to point at the upstream Warp repository: `warpdotdev/warp`
- `personal` should point at the personal fork used for custom builds
- Keep an upstream-tracking branch close to upstream so official changes can be synced cleanly
- Keep personal features and packaging changes on a long-lived personal branch instead of mixing them directly into the upstream-tracking branch

Recommended branch roles:

- upstream-tracking branch: mirrors the official Warp branch as closely as possible
- personal feature/release branch: carries local features, packaging tweaks, and self-use changes

Recommended update flow:

1. Sync the upstream-tracking branch from `origin`
2. Merge or rebase the upstream-tracking branch into the personal feature/release branch
3. Build and validate from the personal branch
4. Push personal changes only to the `personal` remote

If an agent changes remotes, branch names, or release flow for this fork, it should explain why before doing so.
