---
name: devils-advocate-review
description: STUB -- references the published Devil's advocate review skill. This stub exists so the plugin can ship without bundling the skill body. The actual skill body is published in the `anthropic-skills` plugin (or wherever the team has published it). Use this skill via the `triage-reviewer` subagent which loads the published version.
---

# Devil's advocate review (stub reference)

This file is a placeholder. The plugin does NOT bundle the Devil's advocate skill body -- it lives in the published Anthropic skills plugin (or the myPOS-internal skills marketplace, depending on where it was published).

## How this works

When `/triage` calls the `triage-reviewer` subagent (`agents/triage-reviewer.md`), that subagent calls `Skill(skill: "devils-advocate-review")` which Claude Code resolves to the published skill regardless of which plugin published it. The plugin user must have access to that skill in their environment.

## To replace this stub with a bundled copy

If at any point we want the plugin to ship its own copy of the Devil's advocate skill (e.g., to pin a known-good version, or to operate without the published one):

1. Replace this `SKILL.md` with the full skill body (frontmatter + instructions).
2. Update `name:` to keep `devils-advocate-review` (so callers don't change).
3. Update `description:` to whatever the bundled version describes.
4. Bump the plugin version in `.claude-plugin/plugin.json`.

## Verifying availability

`/setup-copilot` runs a smoke test that confirms the skill resolves. If it doesn't, the user is told to either:
- Install the publisher plugin (e.g., `anthropic-skills`)
- Or ask for a bundled version of the legal copilot

## TODO before v0.1.0 ship

Confirm with the legal copilot maintainers (Atanas / Emil) where the Devil's advocate skill is published and whether plugin users will reliably have it available. If yes, this stub is fine. If publishing is uncertain, replace with a bundled copy.
