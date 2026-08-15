---
name: rust-refinement
description: Systematically review and refine Rust code for architecture coherence, API and type design, algorithms, allocation, async correctness, error handling, safety, performance, and idiomatic style. Use when asked to review, clean up, harden, optimize, refactor, de-slop, or establish lint and CI policy for a Rust crate or workspace, especially after large agent-generated changes.
---

# Rust Refinement

A playbook for turning agent-generated Rust — locally correct, architecturally unaware —
into a coherent, idiomatic, efficient codebase. Agents make *surgical* edits: they solve
the task in front of them without seeing the whole, so the failure modes are predictable:
duplicated helpers, convention drift, defensive re-validation, clone-to-satisfy-the-
borrow-checker, `unwrap()` in library paths, wrong sync primitives in async code, dead
code hidden behind `#[allow]`, and dependencies added but never removed. This skill
attacks those failure modes in a fixed order: **survey → mechanize → review → refactor →
verify**.

Two rules govern everything below:

1. **Separate mechanical from behavioral changes.** Keep formatting, lint autofixes,
   and dependency pruning in distinct change sets from structural or semantic
   refactors, with tests between them. Commit only when the user asked for commits;
   separation is a review boundary, not authorization to mutate Git history.
2. **Never "fix" a lint by silencing it.** `#[allow]` without a comment is how agent
   debt compounds. Prefer `#[expect(lint_name, reason = "...")]` and ensure
   `unfulfilled_lint_expectations` is enforced so obsolete suppressions are surfaced.

## Operating contract

- Read repository instructions (`AGENTS.md`, `CONTRIBUTING.md`, toolchain files, and
  CI configuration) before choosing commands or policy. Repository rules override this
  generic playbook.
- Inspect `git status`, existing worktrees, configured Cargo target directories, and
  available disk before edits or heavyweight builds. Preserve unrelated dirty work.
- Default to audit-only when the user asks what needs improving. Do not run `--fix`,
  rewrite files, install tools, add policy, create architecture documents, or commit
  unless the request authorizes those mutations.
- Honor an existing external `CARGO_TARGET_DIR`; never create a large target tree in an
  agent worktree. Set `CARGO_INCREMENTAL=0` for verification unless the repository says
  otherwise, and serialize workspace-wide gates when other agents or builds are active.
- Treat missing optional tools as an evidence gap, not permission to install them.
  Prefer repository-pinned commands and CI-equivalent gates; request approval before a
  networked or system-wide installation.
- Keep accepted artifact, focused checks, full local gate, deployed/live proof, and
  remote push as separate claims. Never present one as evidence for another.

---

## Phase 0 — Survey before touching anything

Agents fail because they lack the map. Build the map first; it changes every decision
afterward.

```bash
cargo metadata --format-version=1 | jq '.workspace_members'   # workspace shape
cargo tree -e features --depth 1                              # deps + features actually enabled
cargo tree --duplicates                                       # duplicate versions in the graph
tokei src/ 2>/dev/null || find . -name '*.rs' | xargs wc -l   # where the mass is
rg -n '#\[allow\(' -g '*.rs' -g '!**/tests/**'                # existing suppressions = prior shortcuts
rg -l '\bunsafe\b' -g '*.rs'                                  # unsafe surface area
```

Then read (don't skim) the crate roots — `lib.rs`/`main.rs` of each workspace member —
and answer in writing before proceeding:

- What are the intended module boundaries? Which crates/modules own which concepts?
- Is this a **library** (public API stability matters, structured errors, semver) or an
  **application/service** (opaque errors fine, binary size/latency matter)? Mixed
  workspaces need the distinction per crate.
- What is the async story: tokio multithreaded? single-threaded? WASM (no threads, no
  `spawn_blocking`)? This determines which primitives are even legal.
- Which conventions does the *oldest, most human-written* code follow? That's the house
  style; agent code drifted from it, not the other way around.

Deliverable of this phase: a short architecture note in the response or task artifact
(module map + conventions + crate type per member). Write it into the repository only
when the user requests a durable architecture document.

## Phase 1 — The tooling gauntlet (mechanical, automatable)

Run applicable steps in this order. In audit-only work, use check/report modes and make
no changes. If implementation is authorized, keep each mechanical step in a distinct
change set and commit only when requested. Inventory missing tools first; do not install
them implicitly. `cargo-udeps` and miri require nightly, so treat them as optional unless
the repository already pins or authorizes that toolchain.

```bash
# 1. Format — normalizes agent output variance, makes real diffs visible
cargo fmt --all -- --check               # audit; omit --check only for an authorized formatting change

# 2. Lints at full strength (see lint policy below for the permanent config)
cargo clippy --workspace --all-targets --all-features -- -D warnings
# Authorized mechanical pass only, and only in an isolated/clean owned change set:
cargo clippy --workspace --all-targets --all-features --fix

# 3. Unused dependencies — agents add, never remove
cargo machete --with-metadata            # fast, text-based; false positives on proc-macros → ignore-list them
cargo +nightly udeps --all-targets       # slower, compiler-accurate; run when machete output looks suspicious

# 4. Dependency policy — advisories, licenses, duplicate versions, banned crates
cargo deny check                         # needs deny.toml; `cargo deny init` to create

# 5. Tests, properly
cargo nextest run --workspace            # detects leaky/flaky tests cargo test hides
cargo test --doc                         # nextest doesn't run doctests — don't skip these

# 6. Unsafe verification — only if the codebase has unsafe blocks
cargo +nightly miri test                 # UB detection on the test suite (slow; scope to affected crates)

# 7. Libraries only: API stability
cargo semver-checks                      # catches accidental breaking changes agents introduce silently
```

### Candidate permanent lint policy

Propose this for the workspace `Cargo.toml` only after comparing it with existing CI,
crate types, MSRV, and warning debt. Do not impose it during an audit. Each member crate
then needs `[lints] workspace = true`.

Agents respect lints they can see; encode approved rules in the workspace so every
future agent session inherits them.

```toml
[workspace.lints.rust]
unsafe_code = "warn"                  # forces #[allow] + justification at each site
missing_docs = "warn"                 # libraries; drop for binaries if noisy

[workspace.lints.clippy]
# Groups: priority -1 so individual overrides below win
pedantic  = { level = "warn", priority = -1 }
nursery   = { level = "warn", priority = -1 }

# Cherry-picked restriction lints (never enable the whole restriction group)
unwrap_used                = "warn"   # the single highest-value lint for agent code
expect_used                = "warn"   # allow in tests/main.rs via #[expect(...)]
dbg_macro                  = "warn"
todo                       = "warn"
unimplemented              = "warn"
panic                      = "warn"   # library crates; relax for binaries
indexing_slicing           = "warn"   # slice[i] → .get(i); agents index blindly
undocumented_unsafe_blocks = "warn"   # every unsafe block gets a // SAFETY: comment
allow_attributes           = "warn"   # forces #[expect] over #[allow]
clone_on_ref_ptr           = "warn"   # Arc::clone(&x), not x.clone() — makes refcounting visible
unused_result_ok           = "warn"

# Pedantic/nursery lints that fight real codebases — tune to taste
module_name_repetitions = "allow"
missing_errors_doc      = "allow"     # re-enable for published libraries
must_use_candidate      = "allow"
option_if_let_else      = "allow"     # nursery; often reduces readability
```

For triage on a large legacy surface, ratchet instead of boiling the ocean: fix
`correctness`/`suspicious`/`perf` findings now, convert the rest to `#[expect]` with
reasons, and burn them down per-module.

## Phase 2 — Architectural review (the part tools can't do)

Work module by module against the Phase 0 map. These are the specific pathologies of
surgically-generated code, in priority order.

### 2.1 Duplication and convention drift
- **Search for near-duplicate helpers** before reviewing anything else: agents re-implement
  `parse_x`, `validate_y`, retry wrappers, and serde helpers because they couldn't see the
  existing one. `grep` for suspicious names (`_helper`, `_util`, `format_`, `parse_`,
  `to_`, `from_`) across crates; consolidate into one owner module and delete the rest.
- **One pattern per problem.** If the codebase has three error-conversion styles, two
  builder conventions, or both `log` and `tracing`, pick the incumbent (from Phase 0) and
  migrate the minority. Consistency beats local optimality.
- **Defensive re-validation:** agents re-check invariants already guaranteed upstream
  (null checks on non-optional types, re-parsing validated input). Delete the re-checks
  and instead *encode the invariant in a type* — see 2.2.

### 2.2 Type-driven design (make illegal states unrepresentable)
- **Newtype the primitives.** `UserId(u64)`, `Bytes(usize)`, `Ratio(f64)` — agent code
  passes bare `String`/`u64` everywhere, which is how arguments get swapped silently.
  Newtypes cost nothing at runtime and turn a class of bugs into compile errors.
- **Replace boolean/Option soup with enums.** `is_ready: bool, error: Option<String>,
  result: Option<T>` → a three-variant state enum. Look for structs where field
  combinations are mutually exclusive; that's a sum type wearing a product-type costume.
- **Parse, don't validate:** functions should accept the already-refined type
  (`&ValidatedConfig`), not raw input plus a prayer. Push parsing to the boundary once.
- **Typestate for protocols** (builders, connection lifecycles) only where misuse has
  actually occurred or is plausible — don't cargo-cult it onto everything.

### 2.3 API and ownership design (Rust API Guidelines, condensed)
- Arguments: accept `&str` not `&String`, `&[T]` not `&Vec<T>`, `impl AsRef<Path>` for
  paths. Take ownership only when you store the value; otherwise borrow.
- Returns: return `impl Iterator` instead of collecting to `Vec` when the caller might
  not need a `Vec`; return the concrete smart pointer only when it's part of the contract.
- Derive eagerly: `Debug, Clone, PartialEq` on data types; `Copy` where cheap;
  `Default` where a zero-value is meaningful; `#[must_use]` on builders and on functions
  whose ignored return value is a bug.
- Visibility: agents mark things `pub` to make errors go away. Audit every `pub` — demote
  to `pub(crate)` or private unless it's deliberately part of the crate's API surface.
  For libraries, `cargo semver-checks` then guards what remains.
- `impl` blocks: constructors named `new`/`with_*`/`from_*`; conversions via
  `From`/`TryFrom` impls rather than ad-hoc `to_other()` methods; no
  "manager"/"handler"/"processor" god-structs — if a struct's methods cluster into two
  unrelated groups, it's two types.

### 2.4 Error handling by intent (not by dogma)
The rule isn't "thiserror for libraries, anyhow for apps" — it's: **will the caller
branch on the failure mode?**
- Caller branches → structured enum error, `thiserror` for the boilerplate, variants
  grouped by *what the caller does about them* (not one variant per underlying cause).
- Caller only reports/logs → opaque error (`anyhow`/`eyre`), with `.context("...")` at
  each level that adds information. Bare `?` chains that propagate `io::Error` from six
  layers down are agent signature; add context where meaning is added.
- **`unwrap()`/`expect()`:** allowed only where panic is the correct behavior (truly
  impossible states, test code, `main` startup). Everywhere else: `?`, `unwrap_or_else`,
  or restructure so the impossible case doesn't type-check. The lint policy above
  enforces the audit; each surviving `expect` gets a message stating *why it can't fail*.
- Never `let _ = fallible()`; if discarding a `Result` is genuinely right, write
  `drop(fallible())` with a comment, or handle it.

### 2.5 Algorithms and data structures
Agents default to `Vec` + linear scan + `clone`. Standard substitutions:
- Membership tests / dedup in a loop → `HashSet`. Keyed lookup → `HashMap`; use the
  **`entry()` API** instead of the agent-typical `contains_key` + `insert` double lookup.
- Ordered iteration or range queries → `BTreeMap`, not `HashMap` + sort-every-time.
- Queue semantics → `VecDeque`; `Vec::remove(0)` in a loop is O(n²) and a classic agent
  artifact. Priority → `BinaryHeap`.
- Growing a `Vec`/`String` in a loop with known size → `with_capacity`. Building a
  `String` with `format!` in a loop → `push_str`/`write!` into one buffer.
- Nested loops doing matching → index one side into a map first (O(n²) → O(n)).
- Small hot collections (< ~16 elems in hot paths) → consider `smallvec`; interned/shared
  strings → `Arc<str>`; only after profiling shows it matters.
- **Iterator chains over index loops** — but don't fetishize it: a chain nobody can read
  loses to a clear `for` loop. `needless_collect`, `map().flatten()` → `flat_map`,
  `filter().next()` → `find` are covered by the lint config; review catches the rest.

### 2.6 Allocation and clone hygiene
- Every `.clone()` gets classified: (a) semantically required (shared ownership), (b)
  cheap-and-clear (small `Copy`-ish data), or (c) **borrow-checker appeasement** — the
  agent cloned to make lifetimes compile. Category (c) is the refactor target: restructure
  ownership, pass `&`/`&mut`, split structs so disjoint fields borrow independently, or
  use `Cow<'_, str>` for the borrow-or-own cases.
- `Arc<Mutex<T>>` as the default sharing strategy is an agent reflex. Ask: does this need
  shared *mutable* state at all? Often message passing (a task owning the state + mpsc)
  or `RwLock` (read-heavy) or plain ownership transfer is correct and faster.
- `Box<dyn Trait>` where generics would do (hot paths) — or the inverse, generics
  monomorphizing large functions where `dyn` would shrink the binary. Direction depends
  on the profile: services usually favor generics, WASM favors `dyn` (see 2.8).

### 2.7 Async correctness (services: tokio/axum)
- **Mutex choice is about the critical section, not the surrounding code:** short,
  non-await-holding sections use `std::sync::Mutex` even inside async fns; only use
  `tokio::sync::Mutex` when the guard must live across an `.await`. Agent code gets this
  backwards constantly. If contention is real, the fix is a state-owning task + channels
  or sharding — not swapping mutex brands.
- **Never hold a `std` guard across `.await`** (deadlock + `!Send` futures). Clippy's
  `await_holding_lock` catches most; scope guards with a block: `{ let g = m.lock(); ... }`.
- **Blocking in async:** `std::fs`, `reqwest::blocking`, heavy CPU, `std::thread::sleep`
  inside async fns starve the runtime. Move to `tokio::fs`/async clients, or
  `spawn_blocking` for CPU-bound work. Grep targets: `std::fs::`, `.blocking_`,
  `thread::sleep`, `block_on(` inside `async fn`.
- **Cancellation safety:** any `select!` arm's future may be dropped mid-poll. State
  machines built across awaits inside select loops lose data on cancel; restructure so
  each arm is cancel-safe or buffer state outside the future.
- **Task lifecycle:** `tokio::spawn` with a discarded `JoinHandle` is fire-and-forget —
  panics vanish and shutdown can't drain. Use `JoinSet` (or `TaskTracker`) and join on
  shutdown; propagate panics deliberately.
- Bounded channels by default; an agent-chosen `unbounded_channel` is an OOM with extra
  steps unless backpressure is handled elsewhere.
- Timeouts at the boundary: every outbound network call wrapped in `tokio::time::timeout`
  or client-level timeout config; agents omit them because the happy path works.

### 2.8 WASM / Cloudflare Workers profile
When the crate targets `wasm32` (Workers, Durable Objects, browser):
- **No threads:** `spawn_blocking`, `std::thread`, rayon are unavailable or traps.
  `Send + Sync` bounds are often unnecessary noise — single-threaded executors accept
  `!Send` futures; removing bounds also removes `Arc`/`Mutex` that existed only to
  satisfy them (`Rc<RefCell<T>>` is legitimate here).
- **Binary size is a feature:** `cargo bloat --release --crate-name <name>` to find the
  mass. Usual wins: trim serde derives to what's used, `default-features = false` on
  deps then re-add needed features (the `tokio = { features = ["full"] }` agent habit is
  the first target), prefer `dyn` dispatch over monomorphization bloat in cold paths,
  audit for accidental `regex`/`chrono` pulls where a hand-rolled parse suffices.
  Release profile: `opt-level = "z"`, `lto = true`, `codegen-units = 1`, `panic = "abort"`,
  plus `wasm-opt -Oz` in the pipeline.
- Time/random/entropy come from the host: `std::time::Instant`, `SystemTime`, and
  `rand`'s default features misbehave — use the platform bindings (`js-sys`, `worker`
  crate APIs, `getrandom` with the wasm feature).
- Durable Objects: state mutations must complete before returning; agent code that
  spawns detached work after the response loses it.

### 2.9 Unsafe audit
- Every `unsafe` block: (1) is it *needed*? Agents copy unsafe pointer patterns from
  training data where safe code exists (`vec.first().copied()` beats a raw deref). Try
  deleting it. (2) If needed: `// SAFETY:` comment stating the invariant, enforced by
  the `undocumented_unsafe_blocks` lint. (3) Run miri over the covering tests. (4) FFI
  boundaries validate inputs on the Rust side.

## Phase 3 — Refactor mechanics

- Order refactors **leaf-first** (types/helpers before the modules that use them) so each
  commit compiles and tests green.
- For each behavioral refactor: characterize current behavior with a test *first* if
  coverage is thin — agent code is undertested precisely where it's wrong.
- After ownership/API refactors, re-run the full gauntlet (Phase 1) — refactors reopen
  lint findings and can orphan dependencies.
- When the user wants durable guidance, update the Phase 0 architecture note as you go
  (`ARCHITECTURE.md` or `AGENTS.md`) and include it in an explicitly authorized change
  set. Otherwise keep the map in the report. The lint config constrains the small stuff;
  the architecture note constrains the big stuff.

## Phase 4 — Verify and report

```bash
CARGO_INCREMENTAL=0 cargo fmt --all -- --check
CARGO_INCREMENTAL=0 cargo clippy --workspace --all-targets --all-features -- -D warnings
CARGO_INCREMENTAL=0 cargo nextest run --workspace
CARGO_INCREMENTAL=0 cargo test --doc
cargo machete && cargo deny check                       # when installed/configured
cargo semver-checks                                     # published libraries with a valid baseline
CARGO_INCREMENTAL=0 cargo build --release --timings     # when proportional to the requested scope
```

Report format — keep it decision-oriented:

```
## Rust refinement report
### Mechanical (committed separately)
- fmt/clippy-fix: N files; unused deps removed: <list>; advisories: <status>
### Structural changes
- <module>: <what changed and why, one line each, linked to the pathology (2.x)>
### Deferred (needs owner decision)
- <finding>: <options + tradeoff>   # e.g. error-type unification across crates
### Guardrails installed
- workspace lints, deny.toml, ARCHITECTURE.md updated, CI additions
```

Findings that change public API, alter error variants callers may match on, or trade
readability for performance go in **Deferred** — surface them, don't decide unilaterally.
