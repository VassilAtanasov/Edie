# Disabled skills

`refine-feature` and `build-feature` are disabled while they have known issues. They live outside `skills/`, so neither Claude Code nor Cursor loads them. Their files are kept here unchanged for later rework.

To re-enable one, move its folder back under `skills/` and restore it in the manifests' keywords, `agents/edie.md`, `rules/edie.mdc` and `README.md`.
