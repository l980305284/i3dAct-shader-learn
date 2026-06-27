# Repository Guidelines

## Project Structure & Module Organization

This repository is a Markdown-based i3dAct shader tutorial project. Article drafts and finished chapters live in `articles/`, named with a two-digit chapter prefix such as `13-fullscreen-shader-intro.md`. Planning and progress documents live at the root: `ARTICLE_PLAN.md`, `PROGRESS.md`, `CLAUDE.md`, and `I3DAct_Engine_Reference.md`. Design specs and implementation plans created during agent workflows live under `docs/superpowers/specs/` and `docs/superpowers/plans/`.

There is no application source tree, asset pipeline, or automated test directory.

## Build, Test, and Development Commands

This is a documentation repository, so there is no build step. Useful local checks:

- `git status --short --branch` checks the current branch and pending changes.
- `rg "Godot|Unity|Unreal" articles/` verifies articles do not mention other engines.
- Search `articles/` for the banned Chinese template phrases listed in `ARTICLE_PLAN.md`.
- `(Get-Content articles/13-fullscreen-shader-intro.md -Encoding UTF8).Count` checks article length in PowerShell.

## Writing Style & Naming Conventions

Use Chinese for future project conversations, article drafts, plans, specs, and contributor-facing documentation unless the user explicitly requests another language.

Write in conversational Chinese, as if teaching a colleague. Prefer code first, then explanation. Keep each article focused on one concept and avoid unnecessary theory.

Use i3dAct naming only. Examples: `MeshRender`, `Item3D`, `.s3`, `iStart()`, `Update()`. Do not introduce other engine names in tutorial text. Shader examples should use practical `glsl` blocks and meaningful comments.

Article filenames should follow `NN-topic-name.md`, for example `12-clipping-animation.md`.

## Testing Guidelines

There is no formal test framework. Validate contributions by reading the rendered Markdown structure, checking article length, searching for banned phrases, and reviewing code snippets for consistency with previous chapters. For shader articles, ensure examples are copyable and use the same naming style as nearby articles.

## Commit & Pull Request Guidelines

History uses concise conventional-style messages, for example `feat: add article 13 - fullscreen shader intro` and `docs: add article 13 implementation plan`. Use `feat:` for new article content and `docs:` for plans, specs, progress, or repository guidance.

Pull requests should summarize changed articles or docs, list validation commands run, and mention any intentional scope exclusions. Screenshots are not required unless visual assets or rendered examples are added.

## Security & Configuration Tips

Local agent settings are ignored via `.gitignore` (`.claude/settings.local.json`). Do not commit personal credentials, local tool permissions, or machine-specific configuration.
