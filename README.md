# Rust Refinement Skill

A systematic workflow for turning locally-correct, agent-generated Rust into a coherent,
idiomatic, efficient codebase. It surveys architecture first, applies mechanical tooling
separately from behavioral refactors, reviews recurring Rust failure modes, and verifies
the result with repository-appropriate gates.

## Principles

- Survey crate ownership, runtime constraints, and house style before editing.
- Keep formatting, lint fixes, dependency pruning, and behavioral refactors separate.
- Fix lints at the cause; do not hide debt behind unexplained `#[allow]` attributes.
- Make invalid states unrepresentable with types where the complexity is justified.
- Audit ownership, clones, allocations, async synchronization, and task lifecycles.
- Choose data structures from access patterns, not habit.
- Treat library API stability, unsafe code, WASM, and async runtimes as distinct risk
  profiles.
- Preserve unrelated work and distinguish focused checks from full local or remote proof.

## Install

Clone the repository once:

```sh
git clone https://github.com/chrischabot/rust-refinement-skill.git ~/Projects/rust-refinement
```

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

Invoke `/rust-refinement` or ask Agent to refine the current Rust workspace. Cursor also
discovers Codex and Claude skill directories, so install only one visible copy.

To update any installation:

```sh
git -C ~/Projects/rust-refinement pull --ff-only
```

## License

MIT
