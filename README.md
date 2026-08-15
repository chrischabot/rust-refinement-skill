# Rust Refinement Skill

A skill for giving Rust code a careful second pass: understand the workspace, improve
the code, run the Rust tooling, and keep the changes that genuinely make it better.

## How it works

- Read the workspace and understand how its crates fit together.
- Refactor code that is awkward, wasteful, inconsistent, or unsafe.
- Run the formatter, linter, tests, and any relevant Rust checks.
- Keep the changes only when the code is clearer and the checks pass.

## Principles

- Use Rust's type system to make mistakes harder to express.
- Prefer clear ownership over cloning around the borrow checker.
- Choose abstractions and data structures because they fit the problem.
- Treat async code, unsafe code, and public APIs with extra care.

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
