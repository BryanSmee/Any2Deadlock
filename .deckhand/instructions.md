# Any2Deadlock

Converts a humanoid character from another game into a Deadlock hero skin. It ships a
recipe (scripts, mappings, templates), never a mod. Design record:
`docs/design-discussion.md`; read it before structural changes.

## Tooling

- Python, managed with uv (on PATH): `uv sync`, `uv run pytest`.
- Add dependencies with `uv add <pkg>` and commit `uv.lock`.
- Only PyPI and GitHub are reachable. Blender, FModel/CUE4Parse, Source 2 Viewer, Valve's
  resourcecompiler and the games themselves are not here.

## Rules

- Never commit game assets or anything derived from them (meshes, weights, textures,
  compiled `.vmdl_c`/`.vtex_c`, VPKs). Profiles are data only: bone maps, asset paths,
  material hints, settings, and the game version they were made for.
- Tests use small synthetic fixtures (a generated skeleton, a few-vertex mesh, a
  hand-written glTF), never ripped files.
- Code paths that need Valve's tools or a game install stay behind an interface, with a
  fake in tests; say in the pull request what could not be run.
- LLM calls (later phases) are bring-your-own-key: mock them in tests and never add a key.
- Extraction only reads installed files offline, never a running game.

## CI

Workflows under `.github/workflows/` must have `permissions: read-all` (or only `read`
scopes), no `secrets`, no `environment:`, and no reusable workflows from other
repositories, or the push is refused. Use `astral-sh/setup-uv` and run the same checks
as below.

## Checks

Before pushing, once they exist: `uv run pytest`, plus the linter and formatter declared
in `pyproject.toml` (ruff suggested). Keep this list in sync when tooling is added.
