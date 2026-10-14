# Native Feel Mobile

A checklist and playbook for making a responsive web app (Capacitor, Cordova or installed PWA) behave like a native mobile app: back gesture/button, pull-to-refresh, safe areas, keyboard, touch feel, offline and app lifecycle.

It is plain markdown, so it works with any AI coding assistant or IDE, or on its own as a human checklist.

## Files

- [SKILL.md](SKILL.md): the workflow, checklist and rules.
- [REFERENCE.md](REFERENCE.md): code snippets and a manual test script.

## Use it

Copy the files into your project, then ask your assistant to "audit this app with native-feel-mobile" and point it at the files.

| Tool | Where to put it |
| --- | --- |
| Claude Code / Claude apps | `~/.claude/skills/native-feel-mobile/` (or `.claude/skills/` in a project). Loads automatically. |
| Cursor | Copy into `.cursor/rules/native-feel-mobile.mdc` |
| GitHub Copilot | Add to `.github/copilot-instructions.md` |
| Codex, Windsurf, Zed, others | Add to `AGENTS.md` or the tool's rules file |
| No AI tool | Use `SKILL.md` as a review checklist |

The YAML block at the top of `SKILL.md` is only read by Claude. Other tools can leave it in or delete it.

## Note

Items marked **[device]** can't be proven in a desktop browser. Check them on a real device or emulator.
