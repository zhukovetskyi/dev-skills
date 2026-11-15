# dev-skills

AI agent skills that bring models with older knowledge cutoffs up to date. Each skill carries only the knowledge a stale model gets wrong — versions, syntax, APIs, and deprecations that post-date typical cutoffs — and deliberately excludes general best practices models already know.

Format: universal [Agent Skills](https://agentskills.io) layout (`<name>/SKILL.md` with `name`/`description` frontmatter), understood natively by **OpenCode** and **Claude Code**.

## Skills

| Skill | What it fixes for stale models |
| --- | --- |
| [`postgres-18`](skills/postgres-18/) | Native `uuidv7()`, async I/O (`io_method`), B-tree skip scan, virtual generated columns (now the default), `RETURNING OLD/NEW`, temporal constraints, `NOT NULL NOT VALID`, moved Docker data dir |
| [`object-storage`](skills/object-storage/) | MinIO is dead (images deleted from Docker Hub) — RustFS / Silo compose configs, env vars, ports, healthcheck paths, storage-agnostic S3 clients |
| [`rust-web-api`](skills/rust-web-api/) | Axum 0.8 (`{id}` route syntax, no `#[async_trait]`), SQLx 0.9 (`SqlSafeStr`, `sqlx.toml`, `.sqlx` offline cache), utoipa 6 (OpenAPI 3.1), tower-http 0.7 |
| [`go-modern`](skills/go-modern/) | Go 1.23–1.27: range-over-func iterators, `tool` directive, generic type aliases, `os.Root`, `testing/synctest`, `new(expr)`, `errors.AsType`, `sync.WaitGroup.Go`; deprecations like the no-op `rand.Seed` |
| [`react-19`](skills/react-19/) | React 19.2/19.3 APIs (`useEffectEvent`, `<Activity>`), React Compiler 1.0 + Vite 8 setup, React Router v8, TanStack Query v5 syntax, RSC version-pin security |

## Install — workstation-wide (global)

Works for **both** OpenCode and Claude Code from one checkout: both read `~/.claude/skills/` (OpenCode also reads `~/.config/opencode/skills/` and `~/.agents/skills/` as compatibility sources).

```sh
git clone <this-repo-url> ~/dev-skills     # any permanent location
mkdir -p ~/.claude/skills
for s in postgres-18 object-storage rust-web-api go-modern react-19; do
  ln -s ~/dev-skills/skills/$s ~/.claude/skills/$s
done
```

- Symlinks are explicitly supported by Claude Code, and OpenCode reads the same directory — one install serves both.
- Updates are then just `git -C ~/dev-skills pull`.
- If `~/.claude/skills` is otherwise empty and you prefer not to symlink per skill, `ln -s ~/dev-skills/skills ~/.claude/skills` works too (skip if you already have other skills there).
- OpenCode-only alternative: the same symlinks under `~/.config/opencode/skills/`, or register the checkout via the `skills` array in `opencode.json`.

## Install — per-repo (project-level)

For a single project, so teammates and CI get the skills automatically. Put them in the repo and commit:

```sh
mkdir -p <repo>/.claude/skills
cp -r ~/dev-skills/skills/postgres-18 <repo>/.claude/skills/
cp -r ~/dev-skills/skills/object-storage <repo>/.claude/skills/
```

- Pick only the skills matching the repo's stack.
- All three directory names work for both agents: `.claude/skills/`, `.opencode/skills/`, `.agents/skills/`.
- A git submodule of this repo is a good option for staying updatable without vendoring files.
- Project skills override global ones with the same name.

## Verify a skill is discovered

- **Claude Code:** run `/skills` (lists discovered skills and their sources), or `claude --debug` and search the log for skill names.
- **OpenCode:** the skill appears in the model's available-skills list; you can also force-load one by mentioning `@postgres-18` in a prompt.

## Skill format

Each skill is `skills/<name>/SKILL.md`:

```markdown
---
name: kebab-name
description: One paragraph saying what it corrects and when to load it. ≤ 1024 chars —
  this is what the model sees when deciding whether to load the skill.
---
Body: stale → current correction table, minimal working snippets, gotchas, dated sources.
```

Rules that keep the format portable: directory name is kebab-case and matches frontmatter `name`; only standard fields (`name`, `description`); everything lives in the single `SKILL.md`.

## Maintenance

Fast-moving facts (crate versions, image tags, current Go/React minors) go stale. Each skill ends with dated source URLs — re-verify a pinned version against them before trusting it, and bump the skill when a source contradicts it. Rebuild knowledge when a new major line lands (e.g. PostgreSQL 19, Go 1.28, React 20).
