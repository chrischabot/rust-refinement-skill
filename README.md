# Rust Refinement Skill

Rust Refinement is a skill for anyone who lets Codex, Claude Code, Pi or Cursor change Rust code and then has to live with the result: it tells the agent how to give a crate or workspace a careful second pass. Agents edit surgically, solving the task in front of them without seeing the rest of the codebase, so each change can be correct on its own while the whole drifts. Helpers that already exist get written again, conventions multiply, invariants guaranteed upstream get checked a second time, values are cloned to satisfy the borrow checker, `unwrap()` appears in library paths, async code reaches for the wrong synchronization primitive, dead code hides behind `#[allow]`, and dependencies are added but never removed. Because those failures are predictable, the skill has the agent look for them deliberately and in a fixed order.

## How it works

The skill is one Markdown playbook, [`skill/SKILL.md`](skill/SKILL.md), with no program of its own, so all of the work is done by your agent and the Rust tools on your machine. The playbook takes the agent through five phases, starting with the map of the codebase it would otherwise lack:

1. **Survey.** The agent maps the workspace with `cargo metadata` and `cargo tree`, reads each crate root, and writes down the module boundaries, which crates are libraries or applications, what the async runtime permits, and the house style of the oldest human-written code.
2. **Mechanize.** It runs `cargo fmt`, Clippy with warnings denied, `cargo machete` or `cargo udeps`, `cargo deny`, `cargo nextest` and doctests, then Miri for `unsafe` code and `cargo semver-checks` for libraries.
3. **Review.** Module by module, it looks for what the tools cannot see: duplicated helpers and convention drift, states that types could rule out, API and ownership design, error handling, data structures, clones made to appease the borrow checker, async correctness in tokio and axum services, WASM and Cloudflare Workers constraints, and `unsafe` blocks.
4. **Refactor.** It changes leaf types and helpers first, writes a characterization test where coverage is thin, and reruns the tools afterward, since refactors reopen lint findings and can orphan dependencies.
5. **Verify and report.** It runs the final checks and reports mechanical changes, structural changes, new guardrails and deferred findings separately.

Mechanical changes stay in separate change sets from structural or semantic ones, with tests between them, so that a reviewer can tell them apart. Lints get fixed rather than silenced, and where a suppression is justified the playbook prefers `#[expect(lint_name, reason = "...")]`, which gets flagged once it is obsolete.

## Install

Clone the repository once:

```sh
git clone https://github.com/chrischabot/rust-refinement-skill.git ~/Projects/rust-refinement
```

Then link its `skill` directory into your agent's skills folder.

### Codex

```sh
mkdir -p ~/.agents/skills
ln -s ~/Projects/rust-refinement/skill ~/.agents/skills/rust-refinement
```

Invoke with `$rust-refinement`.

### Claude Code

```sh
mkdir -p ~/.claude/skills
ln -s ~/Projects/rust-refinement/skill ~/.claude/skills/rust-refinement
```

Invoke with `/rust-refinement` or ask Claude to refine the current Rust workspace.

### Pi

```sh
mkdir -p ~/.pi/agent/skills
ln -s ~/Projects/rust-refinement/skill ~/.pi/agent/skills/rust-refinement
```

Invoke with `/skill:rust-refinement` or ask Pi to use the skill.

### Cursor

```sh
mkdir -p ~/.cursor/skills
ln -s ~/Projects/rust-refinement/skill ~/.cursor/skills/rust-refinement
```

Invoke `/rust-refinement` or ask Agent to refine the current Rust workspace.

Cursor also reads `~/.agents/skills` and `~/.claude/skills`, and Pi also reads `~/.agents/skills`, so if you use several of these agents, avoid giving any one of them two copies.

### Updating

Each install is a symlink into the clone, so updating the clone updates every agent:

```sh
git -C ~/Projects/rust-refinement pull --ff-only
```

### Rust tools

The agent needs a Rust toolchain, plus `jq` and `ripgrep` for the survey. It installs nothing on its own: a missing tool such as `cargo-nextest` or `cargo-deny` is reported as a gap in its evidence, and it asks before any networked or system-wide installation. `cargo udeps` and Miri also need a nightly toolchain.

## Usage

The playbook ties what the agent may change to what you ask for:

- **Audit.** Ask what needs improving, and the agent works in check and report modes, without running `--fix`, rewriting files, installing tools, adding policy or committing.
- **Refine.** Ask it to refine or clean up the code, and it may edit, though it commits only when asked. Findings that change a public API, alter error variants that callers may match on, or trade readability for performance come back to you as decisions.
- **Lint policy.** Ask for one, and it compares candidate lints such as `unwrap_used` and `undocumented_unsafe_blocks` with your CI, crate types, MSRV and warning debt before proposing them for the workspace `Cargo.toml`, where later agent sessions inherit them.

These are instructions to the agent rather than enforced limits, and your repository's own rules in `AGENTS.md`, `CONTRIBUTING.md`, toolchain files and CI configuration take precedence over them.

The default prompt the skill ships for Codex takes the refine path:

```text
Use $rust-refinement to survey and systematically improve this Rust workspace.
```

## Development

`skill/` is the whole installable skill: `SKILL.md` holds the playbook, and `agents/openai.yaml` holds the display name, short description and default prompt for OpenAI's skill interfaces. There is no build step or test suite, so a change to the skill is an edit to that Markdown.

## License

MIT, copyright 2026 Chris Chabot. See [LICENSE](LICENSE). The playbook's API and ownership section condenses the [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/).
