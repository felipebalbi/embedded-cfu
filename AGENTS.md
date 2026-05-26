# AGENTS.md — embedded-cfu

Operational guide for AI coding agents (GitHub Copilot, Claude, etc.) working
in the `openDevicePartnership/embedded-cfu` repository. This file is the
authoritative source for repository conventions, commands, and review
expectations. Human contributors are welcome to read it too, but its primary
audience is automated assistants that need a compact, accurate description of
how to operate inside the crate without trial-and-error discovery.

If anything in this file becomes stale, update it as part of the same change
that invalidates it — agents must keep this document trustworthy.

---

## 1. Repository at a glance

- **Name:** `embedded-cfu`
- **Crate name:** `embedded-cfu-protocol`
- **Purpose:** A `no_std`, async-friendly Rust implementation of the Windows
  Component Firmware Update (CFU) protocol. It provides the wire-format
  structs (commands, responses, headers), plus traits a host or client
  firmware can implement to drive CFU transactions over an arbitrary bus.
- **Spec reference:**
  <https://learn.microsoft.com/en-us/windows-hardware/drivers/cfu/cfu-specification>
- **License:** MIT (see `LICENSE`)
- **Default branch:** `main`
- **Edition:** 2021
- **MSRV:** **1.79** (pinned in CI; see `.github/workflows/check.yml`)
- **Workspace?** No. This is a **single-crate** repository — `Cargo.toml` at
  the repo root *is* the crate manifest. Do not introduce a virtual workspace
  unless the maintainers explicitly ask for it.

### 1.1 Crate layout

```
embedded-cfu/
├── Cargo.toml                  # crate manifest (single crate)
├── deny.toml                   # cargo-deny config (licenses, advisories, bans)
├── rustfmt.toml                # nightly-only fmt settings (see §6)
├── README.md
├── CONTRIBUTING.md             # ODP-wide contribution rules
├── CODE_OF_CONDUCT.md
├── CODEOWNERS
├── SECURITY.md
├── LICENSE
├── AGENTS.md                   # ← this file
├── .github/
│   ├── copilot-instructions.md # commit-message + AI attribution rules
│   └── workflows/
│       ├── check.yml           # fmt, doc, hack, deny, test, msrv
│       └── nostd.yml           # cross-build for thumbv8m.main-none-eabihf
└── src/
    ├── lib.rs                  # crate root: #![no_std], CfuImage trait
    ├── client.rs               # CfuReceiveContent (device/receiver side)
    ├── host.rs                 # CfuHostStates, CfuUpdateContent (sender side)
    ├── components.rs           # CfuComponentInfo/Storage/Traits
    ├── writer.rs               # CfuWriterAsync / CfuWriterSync + CfuWriterError
    ├── protocol_definitions.rs # wire structs, constants, conversions
    └── fmt.rs                  # log/defmt logging macro shims
```

### 1.2 Module responsibilities

| Module | Purpose | Key types |
|---|---|---|
| `lib.rs` | Crate entry, `CfuImage` trait, `read_from_exact` helper, `DataChunk` alias, re-export of `CfuProtocolError`. Declares `#![no_std]`. | `CfuImage`, `DataChunk` |
| `protocol_definitions` | All CFU wire structs and constants. LSB-first byte layouts. `TryFrom<&[u8]>`/`From<…> for [u8; N]` conversions. | `FwVersion`, `GetFwVersionResponse`, `FwUpdateOfferCommand`, `FwUpdateContentCommand`, `FwUpdateOfferResponse`, `FwUpdateContentResponse`, `ComponentId`, `OfferStatus`, `OfferRejectReason`, `CfuProtocolError`, `CfuUpdateContentResponseStatus`, `DEFAULT_DATA_LENGTH`, `MAX_CMPT_COUNT`, `MAX_SUBCMPT_COUNT`, `FW_UPDATE_FLAG_FIRST_BLOCK`, `FW_UPDATE_FLAG_LAST_BLOCK` |
| `components` | Traits a CFU-updatable component must implement: identity, offer validation, storage prep/write/finalize, post-update. | `CfuComponentInfo`, `CfuComponentStorage`, `CfuComponentTraits` |
| `client` | Receiver-side trait — process incoming CFU commands, prepare components. Generic over user-defined `T` (args), `C` (cmd), `E: Default` (error). | `CfuReceiveContent` |
| `host` | Sender-side traits — drive the offer/content state machine and chunk a `CfuImage` into properly sized writes. | `CfuHostStates`, `CfuUpdateContent` |
| `writer` | Bus-agnostic R/W traits used both to talk between Host↔Client and to write component storage. | `CfuWriterAsync`, `CfuWriterSync`, `CfuWriterError` |
| `fmt` | Provides `trace!`/`debug!`/`info!`/`warn!`/`error!` macros that route to either `log` or `defmt` depending on the active feature, or no-op when neither is enabled. | (macros only) |

### 1.3 Cargo features

```toml
[features]
default = []
defmt = ["dep:defmt"]
log    = ["dep:log"]
```

- `log` and `defmt` are **mutually exclusive**. Building with both enabled is
  rejected at compile time by a `compile_error!` in `src/fmt.rs`.
- Default build is **no-logging**; the `fmt::*` macros expand to no-ops.
- Lint table in `Cargo.toml` (do not weaken without strong justification):
  - `unsafe_code = "forbid"` (root-level — `unsafe` is not allowed anywhere)
  - `clippy::{correctness, expect_used, indexing_slicing, panic,
    panic_in_result_fn, perf, suspicious, style, todo, unimplemented,
    unreachable, unwrap_used}` all `deny`.

---

## 2. Golden rules for agents

These are non-negotiable. Violating any of them will get a PR rejected.

1. **No `unsafe`.** `unsafe_code = "forbid"` is set at the crate root. Do not
   add `#[allow(unsafe_code)]` escape hatches.
2. **No panicking constructs.** `unwrap`, `expect`, `panic!`, `todo!`,
   `unimplemented!`, `unreachable!`, raw indexing (`slice[i]`), and
   `panic_in_result_fn` are all `deny`'d by clippy. Use `?`, `ok_or`,
   `get(..)`, pattern matching, or explicit error variants. If you genuinely
   need indexing, prefer `.get(i).ok_or(CfuProtocolError::…)?` or
   `slice.first()/last()/split_first()` etc.
3. **Stay `no_std`.** `#![no_std]` is set in `src/lib.rs`. Do **not** add
   `std::` imports, `alloc`, `String`, `Vec`, `HashMap`, threads, files,
   sockets, timers, `Instant`, `Mutex`, or anything from `std`. Use `core::`
   and (where async is needed) `core::future::Future`.
4. **Keep the public API additive.** This is a protocol crate consumed by
   downstream firmware. Renaming or removing public items is a breaking
   change. If you must do it, call it out explicitly in the PR description.
5. **Preserve wire compatibility.** The structs in `protocol_definitions.rs`
   match a published Microsoft spec. Do not reorder fields, change widths,
   change endianness, or alter `From`/`TryFrom` byte layouts without a
   spec-level justification.
6. **`log` and `defmt` are mutually exclusive.** When adding code that logs,
   use the macros from `crate::fmt` (`trace!`, `debug!`, `info!`, `warn!`,
   `error!`) — never `log::*` or `defmt::*` directly. Never enable both
   features together (CI's `cargo hack` invocation already enforces this via
   `--mutually-exclusive-features=log,defmt`).
7. **MSRV is 1.79.** Do not use language or stdlib features stabilized after
   that. Common pitfalls: `let … else` is fine (1.65), `async fn` in traits
   is fine (1.75), but anything stabilized in ≥1.80 is off-limits unless
   MSRV is bumped in `.github/workflows/check.yml` and called out in the PR.
8. **`rustfmt.toml` requires nightly.** It sets `imports_granularity =
   "Module"` and `group_imports = "StdExternalCrate"`, both nightly-only.
   Stable `cargo fmt --check` will print warnings and pass; CI uses **nightly
   rustfmt**. Always run `cargo +nightly fmt` before committing if you have
   nightly installed; otherwise rely on CI to flag formatting drift.
9. **Commit-message rules are enforced.** See §7. Every AI-assisted commit
   needs an `Assisted-by:` trailer. Never add `Signed-off-by:` from an agent.
10. **Don't broaden dependencies casually.** `deny.toml` and `cargo-deny`
    police licenses, advisories, and bans. New deps must clear `cargo deny
    check --all-features`. Prefer doing the work with what's already in
    `Cargo.toml` (`embedded-io-async`, optionally `log` or `defmt`).

---

## 3. Local development workflow

### 3.1 Toolchain

- Stable Rust ≥ 1.79 (matches MSRV).
- Nightly Rust for `cargo fmt` and `cargo doc` (matches CI). Install with
  `rustup toolchain install nightly` and `rustup component add --toolchain
  nightly rustfmt`.
- For the `no-std` cross-build: `rustup target add thumbv8m.main-none-eabihf`.
- Optional tooling used by CI: `cargo-hack`, `cargo-deny`. Install with
  `cargo install cargo-hack cargo-deny` if you want to reproduce CI locally.

### 3.2 Bread-and-butter commands

Run these from the repo root. The first three are the minimum smoke test an
agent should run before declaring "done":

```sh
# Format (use nightly if available; stable will warn but pass)
cargo +nightly fmt --check        # or: cargo fmt --check

# Build each feature configuration (log/defmt are mutually exclusive)
cargo build                       # no features
cargo build -F log
cargo build -F defmt

# Clippy with warnings as errors, per feature configuration
cargo clippy --no-default-features -- -Dwarnings
cargo clippy -F log               -- -Dwarnings
cargo clippy -F defmt             -- -Dwarnings

# Tests (currently no unit tests, but keep the command green)
cargo test --no-default-features
cargo test -F log
cargo test -F defmt

# Docs (nightly, with the docsrs cfg as CI does)
RUSTDOCFLAGS="--cfg docsrs" cargo +nightly doc --no-deps --all-features

# No-std cross-build (matches nostd.yml)
cargo check --target thumbv8m.main-none-eabihf --no-default-features
```

### 3.3 Reproducing CI exactly

CI uses `cargo hack` to walk the feature powerset while honoring the
log/defmt exclusion. To do the same locally:

```sh
cargo hack --feature-powerset --mutually-exclusive-features=log,defmt check
cargo hack --feature-powerset --mutually-exclusive-features=log,defmt clippy -- -Dwarnings
cargo hack --feature-powerset --mutually-exclusive-features=log,defmt test
cargo deny --manifest-path ./Cargo.toml check --all-features
```

If `cargo hack` is unavailable, the explicit triplet in §3.2 (no-features /
`-F log` / `-F defmt`) is the practical equivalent for this crate.

### 3.4 PowerShell notes (Windows agents)

When running these from PowerShell, set `RUSTDOCFLAGS` like so:

```powershell
$env:RUSTDOCFLAGS = "--cfg docsrs"
cargo +nightly doc --no-deps --all-features
Remove-Item Env:RUSTDOCFLAGS
```

Do not chain commands with `&&` in PowerShell 5.1; use `;` or check
`$LASTEXITCODE` between steps.

---

## 4. CI overview

Two workflows live under `.github/workflows/`:

### 4.1 `check.yml` — main quality gate

| Job | Toolchain | What it runs |
|---|---|---|
| `fmt` | nightly | `cargo fmt --check` |
| `doc` | nightly | `RUSTDOCFLAGS=--cfg docsrs cargo doc --no-deps --all-features` |
| `hack` | stable | `cargo hack --feature-powerset --mutually-exclusive-features=log,defmt check` then `… clippy -- -Dwarnings` |
| `deny` | stable | `cargo-deny check --all-features` via `EmbarkStudios/cargo-deny-action@v2` |
| `test` | stable | `cargo hack --feature-powerset --mutually-exclusive-features=log,defmt test` |
| `msrv` | 1.79 | `cargo check -F log` and `cargo check -F defmt` |

Triggers: `push` to `main` and all `pull_request` events. Concurrency is
grouped by `head_ref`, so pushing twice to a PR cancels the in-flight run.

### 4.2 `nostd.yml` — embedded sanity

Runs `cargo check --target thumbv8m.main-none-eabihf --no-default-features`
on stable. Confirms the crate still compiles for a real Cortex-M target.

### 4.3 What agents should do before pushing

At minimum, locally run:

1. `cargo +nightly fmt --check` (or stable equivalent + accept CI may flag).
2. `cargo build` and `cargo clippy -- -Dwarnings` for **each** of: no
   features, `-F log`, `-F defmt`.
3. `cargo test` (or the three-feature variant) — even if tests are empty
   today, this surfaces dependency/feature-resolution issues.
4. `cargo doc --no-deps --all-features` to catch broken intra-doc links.
5. If a new dependency or license condition is introduced: `cargo deny check
   --all-features`.

---

## 5. Coding conventions

### 5.1 Style

- Follow `rustfmt.toml`: 120-column max, `group_imports = "StdExternalCrate"`,
  `imports_granularity = "Module"`. These are nightly-only knobs but produce
  predictable output that stable rustfmt won't fight.
- Import order: `std`/`core` → external crates → `crate::` → `self::` /
  `super::`, separated by blank lines (nightly fmt handles this automatically
  via `group_imports`).
- Use `core::` for everything (no `std::`). For futures, use
  `core::future::Future` (the crate uses `impl Future<Output = …>` return
  positions rather than `async fn` in most traits — match the surrounding
  style).
- Doc comments on every public item. The crate's published rustdoc is the
  primary reference for downstream firmware authors.

### 5.2 Error handling

- The crate-wide error enum is `protocol_definitions::CfuProtocolError`,
  re-exported at the crate root with `pub use CfuProtocolError::*;`. Prefer
  adding a variant there over inventing a new error type.
- Bus/IO errors live in `writer::CfuWriterError`. Storage and component
  errors should map into one of these two enums where possible.
- Public APIs return `Result<…, CfuProtocolError>` or
  `Result<…, CfuWriterError>` — keep that pattern.
- All error enums derive `Debug, Clone, Copy, PartialEq, Eq, Ord, PartialOrd,
  Hash` and gate a `defmt::Format` derive behind `#[cfg_attr(feature =
  "defmt", derive(defmt::Format))]`. New enums should do the same.

### 5.3 Logging

Use the macros from `crate::fmt`. They are zero-cost when neither feature is
on:

```rust
use crate::{trace, debug, info, warn, error};

trace!("offer accepted for component {=u8}", component_id);
```

The `=u8` and similar formatting specifiers are `defmt` syntax; they are
parsed but ignored by the `log` shim. When in doubt, match the style already
used in `host.rs` and `client.rs`.

### 5.4 Async

- Trait methods that perform I/O return `impl Future<Output = Result<…>>`
  rather than using `async fn` in trait position. This keeps generated
  futures `Send`-agnostic and avoids forcing implementers to deal with
  return-position-impl-trait quirks on older toolchains.
- Do not require `Send`/`Sync` bounds on returned futures unless the trait
  already does — single-threaded firmware is a primary target.

### 5.5 Wire structs (`protocol_definitions.rs`)

- Layouts are documented as "LSB first" in the existing doc comments —
  preserve that ordering when round-tripping to/from `u32`/`[u8; N]`.
- Provide explicit `From`/`TryFrom` impls for new wire structs; do not rely
  on `repr(C)` reinterpretation. The crate forbids `unsafe`, so any
  byte-level decoding must be done with safe arithmetic and slice indexing
  via `get(..)`.
- Constants like `DEFAULT_DATA_LENGTH`, `MAX_CMPT_COUNT`,
  `FW_UPDATE_FLAG_*` come from the CFU spec — don't redefine them locally.

---

## 6. Testing

- The crate currently ships **no unit tests** (the `test` CI job exists to
  catch resolution/feature issues and to make adding tests trivial later).
- When you add new logic, prefer adding `#[cfg(test)] mod tests { … }` at
  the bottom of the relevant module. Keep tests `no_std`-compatible where
  possible (`cargo test` on the host enables `std`, but the production
  build must remain `no_std`).
- Doc-tests on public items are welcome; gate any that need `std` with
  ```text
  /// ```no_run
  ```
  or `ignore` if they can't compile in the crate's `no_std` context.
- Run `cargo test` for each of: no features, `-F log`, `-F defmt`.

---

## 7. Commit & PR conventions

Source of truth: `CONTRIBUTING.md` (ODP-wide) and
`.github/copilot-instructions.md` (this repo's commit/AI rules).

### 7.1 Commit messages

- Subject: ≤ 50 chars, capitalized, imperative mood ("Add foo", not
  "Added foo" / "Adds foo").
- Blank line, then a body wrapped at 72 chars explaining **what** and
  **why**, not how.
- One logical change per commit. Each commit must build cleanly without
  warnings (CI does not squash; history is preserved).

### 7.2 AI attribution (mandatory)

Every AI-assisted commit must include an `Assisted-by:` trailer:

```
Assisted-by: AGENT_NAME:MODEL_VERSION [TOOL1] [TOOL2]
```

- `AGENT_NAME` — e.g. `GitHub Copilot`, `Claude Code`.
- `MODEL_VERSION` — the **actual** model the agent is running on. Verify it
  at runtime; do **not** copy a value from a previous session or hard-code
  it. Examples: `claude-opus-4.7`, `claude-sonnet-4.5`, `gpt-5.3-codex`.
- Optional tool tags in brackets for non-trivial analyzers used (e.g.
  `clang-tidy`, `coccinelle`). Common dev tools (git, cargo, rustc, editor)
  should **not** be listed.
- Agents must **never** add `Signed-off-by:` — only humans can certify the
  Developer Certificate of Origin.

Example:

```
Add CfuComponent helper for dual-bank flush

The dual-bank storage layout requires a post-write swap step that
wasn't covered by the existing CfuComponentStorage trait. This adds
…

Assisted-by: GitHub Copilot:claude-opus-4.7
```

### 7.3 Pull requests

- Open as a **draft PR first** (CONTRIBUTING.md). Wait for `check.yml` and
  `nostd.yml` to go green before marking ready for review.
- Keep the `.github/` folder intact on your branch so CI runs.
- Squash-merging is **disabled** — keep a clean, bisectable history.
  Squash fixup commits into their parent before pushing for review (use
  `git rebase -i` / `git commit --fixup` + `git rebase -i --autosquash`).
- If your change is a regression report, run `git bisect` first and cite the
  first offending commit in the PR description.

---

## 8. Branching, forks, and pushes (for agents)

When an agent operates on this repo through a personal fork (the typical
Copilot setup), follow this protocol:

1. **Fork once** with `gh repo fork openDevicePartnership/embedded-cfu
   --clone=false --remote=false` (idempotent if the fork already exists).
2. **Clone the fork**, then add upstream:
   ```sh
   git remote add upstream https://github.com/openDevicePartnership/embedded-cfu.git
   git fetch upstream
   git checkout -B <topic-branch> upstream/main
   ```
3. **Set `core.autocrlf=false`** on the local clone before touching files
   (Windows agents especially). The repo is LF-only.
4. **Commit as the human author**, per-invocation, never globally:
   ```sh
   git -c user.name="Felipe Balbi" \
       -c user.email="felipe.balbi@microsoft.com" \
       commit -m "Subject" -m "Body" \
       -m "Assisted-by: GitHub Copilot:<model>"
   ```
5. **Push to the fork only.** Never push to `upstream`. Never force-push
   without explicit human approval — squash/rewrite is incompatible with the
   "clean, additive history" rule above.
6. **Do not open the PR autonomously** unless the task explicitly says so.
   Stop after `git push` and report the branch URL.

---

## 9. Common pitfalls (and how to avoid them)

| Symptom | Cause | Fix |
|---|---|---|
| `cargo build --all-features` fails with `features 'log' and 'defmt' are mutually exclusive` | The `compile_error!` in `src/fmt.rs` fires when both features are on. | Build them separately: `cargo build`, `cargo build -F log`, `cargo build -F defmt`. CI uses `cargo hack` with `--mutually-exclusive-features=log,defmt`. |
| `cargo doc --all-features` *works* despite the rule above | `src/fmt.rs` gates the `compile_error!` with `not(doc)` so both feature implementations document side-by-side. | Don't "fix" this — it's intentional for rustdoc. |
| `cargo fmt --check` prints "unstable features only available in nightly" | `rustfmt.toml` uses nightly knobs. | Run with `cargo +nightly fmt`. CI uses nightly. |
| clippy fails on `slice[i]` | `clippy::indexing_slicing = "deny"`. | Use `slice.get(i).ok_or(…)?`, `first()`, `split_at`, iterators, or pattern matching. |
| clippy fails on `.unwrap()` / `.expect()` | `unwrap_used` and `expect_used` are denied. | Propagate with `?` and a real error, or `ok_or(CfuProtocolError::…)`. |
| MSRV job fails on a newly stabilized API | `msrv: "1.79"` in `check.yml`. | Either avoid the API or bump MSRV in the workflow **and** call it out in the PR. |
| `no-std` job fails when adding a dep | The new dep pulls in `std`. | Find a `no_std`-compatible alternative or gate the dep behind a feature that isn't on by default. |
| Windows line-endings sneak in (`^M` in diffs) | `core.autocrlf` defaulted to `true`. | `git config core.autocrlf false` and re-stage. Check with `git diff --check`. |
| `cargo deny` fails on a license | New transitive dep is not on the allowlist. | Update `deny.toml` with justification, or replace the dep. |

---

## 10. Quick reference

```sh
# One-shot local pre-push check (bash):
set -e
cargo +nightly fmt --check
for feat in "" "-F log" "-F defmt"; do
  cargo build $feat
  cargo clippy $feat -- -Dwarnings
  cargo test  $feat
done
RUSTDOCFLAGS="--cfg docsrs" cargo +nightly doc --no-deps --all-features
cargo check --target thumbv8m.main-none-eabihf --no-default-features
```

```powershell
# Same idea in PowerShell:
cargo +nightly fmt --check; if ($LASTEXITCODE) { throw "fmt" }
foreach ($feat in @(@(), @('-F','log'), @('-F','defmt'))) {
  cargo build  @feat; if ($LASTEXITCODE) { throw "build $feat" }
  cargo clippy @feat -- -Dwarnings; if ($LASTEXITCODE) { throw "clippy $feat" }
  cargo test   @feat; if ($LASTEXITCODE) { throw "test $feat" }
}
$env:RUSTDOCFLAGS = "--cfg docsrs"
cargo +nightly doc --no-deps --all-features
Remove-Item Env:RUSTDOCFLAGS
cargo check --target thumbv8m.main-none-eabihf --no-default-features
```

---

## 11. When in doubt

- Read the CFU spec: <https://learn.microsoft.com/en-us/windows-hardware/drivers/cfu/cfu-specification>
- Read `src/protocol_definitions.rs` — it's the wire format.
- Read `.github/workflows/check.yml` — it's the definitive CI contract.
- Read `.github/copilot-instructions.md` — it's the commit-message contract.
- If a convention here conflicts with one of those files, **those files
  win**, and this document should be updated to match.

## Model selection & cost discipline

Premium models (Opus, GPT-5 family, "high"/"xhigh" reasoning variants)
cost an order of magnitude more than standard models (Sonnet, Haiku,
mini). Most steps in a typical task do not need premium reasoning,
and over-using premium models wastes credits without improving
outcomes. The rules below apply to *all* model selection: your own
session, sub-agents launched via the `task` tool, and parallel work
launched via `/fleet`.

### Default posture

- **Default to the cheapest model that can do the job.** Reach for a
  premium model only when one of the escalation triggers below is hit.
- **Plan with premium, execute with cheap.** Spend at most one or two
  premium turns on design / planning, then downshift to a cheaper
  model for mechanical execution of the plan.
- **Never bump the model "just in case."** If you cannot articulate
  *why* a cheaper model would fail, use the cheaper model.

### Escalation triggers (use a premium model)

Reach for a premium model when *any* of these are true:

- Cross-module refactor, architectural design, or API design from
  scratch.
- Subtle correctness reasoning: concurrency, lifetimes, `unsafe`,
  FFI ABI, cryptography, safety-critical control paths.
- Debugging a failure that survived one prior cheap-model attempt.
- Reviewing code on a safety-, security-, or money-critical path.
- The diff cannot be predicted in advance — i.e. there is genuine
  creative or design work to do, not just typing.

### De-escalation triggers (use a cheap model)

Use the cheapest available model when *any* of these are true:

- Searching, reading, summarising files or docs.
- Single-file mechanical edits: rename, format, lint fix, dependency
  bump, boilerplate, scaffolding from a known template.
- Generating tests for code that already works.
- Running builds, tests, linters, or other commands where the model
  only needs to report success/failure.
- Routine commits, PR descriptions, changelog entries.
- The diff is essentially predictable before generation.

### Sub-agent routing (the `task` tool)

When delegating with the `task` tool, set `model:` explicitly. Do not
let sub-agents inherit a premium default for cheap work.

| Sub-agent type    | Default model             | Override to                                     |
|-------------------|---------------------------|-------------------------------------------------|
| `explore`         | cheap                     | keep cheap (`claude-haiku-4.5` or `gpt-5-mini`) |
| `task` (run cmd)  | cheap                     | keep cheap                                      |
| `research`        | cheap for breadth         | premium only for the final synthesis            |
| `general-purpose` | match task                | cheap for mechanical work; premium for design   |
| `rubber-duck`     | premium                   | keep premium — this is where reasoning pays off |
| `code-review`     | premium on critical paths | cheap on cosmetic / mechanical diffs            |

### `/fleet` (parallel sub-agents) rules

- Fleet mode multiplies cost by the fleet width. Apply the rules
  above *per worker*, not in aggregate.
- Split a fleet job along complexity lines: route the cheap,
  parallelisable workers (file edits, test runs, doc updates) to a
  cheap model; reserve premium models for the small number of
  workers that need real reasoning.
- If every worker in a fleet would need a premium model, the work is
  probably not a good fit for fleet mode — reconsider the
  decomposition before paying N× premium.

### Session hygiene

- Keep sessions short and focused. Long premium sessions are the
  single largest source of waste because every turn re-processes the
  full history.
- Use `/compact` when the conversation grows long, and `/new` for
  unrelated work.
- Prefer `/ask` for one-off side questions so they don't extend the
  main session.

### When in doubt

Ask: *"If a cheaper model produced the wrong answer here, would I
catch it in seconds (compiler, tests, my own review) or in
weeks (production incident)?"* If the former, use the cheap model
and let the feedback loop do its job.
