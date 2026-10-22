# google-rust-style

An [Agent Skill](https://agentskills.io) that teaches your coding agent the [Google Rust Style Guide](https://google.github.io/styleguide/rustguide.html). It runs in Claude Code, Codex, Cursor, Gemini CLI, GitHub Copilot, OpenCode and any other agent that reads `SKILL.md`.

A style guide in prose gets half-remembered. This skill turns each rule a tool can check into a rustfmt or clippy setting, so your CI enforces it. Prose covers the rest: the calls only a reviewer can make.

## What your agent gets

- **Lint config to paste.** A `rustfmt.toml`, a `clippy.toml` and `[lints]` tables for `Cargo.toml`, with 21 rows mapping each setting to the guide section it enforces. I checked every lint on clippy 0.1.98 against code that breaks its rule and watched it fire.
- **40 judgement rules** for what no linter sees: module layout, import style, when to return an iterator, `Option` vs `Result` vs `panic!`, accessor and constructor names, unsafe hygiene. Each links to its section of the guide.
- **A library/application switch.** Google binds its API rules for library code and relaxes them for application code. The skill makes your agent decide which one it is in before applying anything.
- **9 overrides** where Google disagrees with common Rust advice, such as `imports_granularity = "Module"` over `"Crate"`, and `expect("invariant")` over a blanket ban on panics.

## Install

The [Skills CLI](https://skills.sh) installs to whichever agents it finds on your machine.

Current project:

```bash
npx skills add 0xMassi/google-rust-style-skill
```

All your projects:

```bash
npx skills add 0xMassi/google-rust-style-skill -g
```

Pick agents and skip the prompts:

```bash
npx skills add 0xMassi/google-rust-style-skill -g -a claude-code -a codex -a cursor -y
```

### Without Node

Copy `skills/google-rust-style` into your agent's skills folder:

```bash
git clone --depth 1 https://github.com/0xMassi/google-rust-style-skill.git
cp -R google-rust-style-skill/skills/google-rust-style ~/.claude/skills/
```

| Agent | Global folder | Project folder |
|---|---|---|
| Claude Code | `~/.claude/skills/` | `.claude/skills/` |
| Codex, Cline, Warp, Zed | `~/.agents/skills/` | `.agents/skills/` |
| Cursor | `~/.cursor/skills/` | `.agents/skills/` |
| Gemini CLI | `~/.gemini/skills/` | `.agents/skills/` |
| GitHub Copilot | `~/.copilot/skills/` | `.agents/skills/` |
| OpenCode | `~/.config/opencode/skills/` | `.agents/skills/` |
| Windsurf | `~/.codeium/windsurf/skills/` | `.windsurf/skills/` |
| Amp | `~/.config/agents/skills/` | `.agents/skills/` |

Paths come from the Skills CLI [agent table](https://github.com/vercel-labs/skills/blob/main/docs/agents.md), which lists the other agents.

## Use it

Ask in plain words. Your agent loads the skill when the request mentions Google style or Rust lint setup:

```
review this crate against Google Rust style
```

```
set up rustfmt and clippy the Google way for this workspace
```

```
is this public API library-grade by Google's rules?
```

## Requirements

- Stable Rust with clippy for the lint config.
- Nightly rustfmt for three options (`imports_granularity`, `hex_literal_case`, `format_macro_matchers`). Stable rustfmt prints a warning and skips them, and the guide allows a stable-only codebase to do the same.

## Known limits

A run on a 63,000-line, seven-crate workspace surfaced three rough edges in the config:

- `unsafe_code = "forbid"` breaks edition 2024 tests that call `std::env::set_var`, which is now `unsafe`. Use `deny` and put `#[expect(unsafe_code, reason = "...")]` on those tests.
- `allow-unwrap-in-tests` covers `#[test]` functions and `#[cfg(test)]` modules. Helper functions in `tests/*.rs` still trip `unwrap_used`.
- The library lint block assumes outside users. On an internal workspace crate, `missing_docs`, `must_use_candidate` and `exhaustive_*` flag hundreds of items that Google's "used by other teams" scope would skip. Apply that block to published crates.

The skill leaves out rules Google tags for Chromium, Fuchsia or Android only, such as flat crate layout, `chromium::import!` and `fuchsia_sync::Mutex`.

## Credits and license

The rules adapt the [Google Rust Style Guide](https://google.github.io/styleguide/rustguide.html) by Google, licensed under [CC BY 3.0](https://creativecommons.org/licenses/by/3.0/). The skill tracks the guide at commit `2ac1584` of [google/styleguide](https://github.com/google/styleguide).

This project is licensed under [CC BY 4.0](LICENSE). Google does not endorse or maintain it.
