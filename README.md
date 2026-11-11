# Kirby AI Builder Hygiene

*This skill is part of the [Kirby Skills Collection](https://github.com/markkirby125/kirby-skills-collection).*

Kirby AI Builder Hygiene cleans code exported from AI visual builders such as Lovable, v0 and Bolt. It runs four modules in order: remove builder attributes, generator comments and dev tooling; flatten wrapper divs into real landmarks and a clean heading tree; consolidate repeated Tailwind and shadcn class strings into semantic classes tied to design tokens; then verify with a build, tests, before and after screenshots and an accessibility check. It keeps analytics, consent scripts and builder badges in place, and it returns a diff plus a verification report. Visual design and copy issues go to other skills.

The root `SKILL.md` points to `references/00_dispatcher.md`, which holds the run order, the locked decisions, the preconditions and the routing table for the module files.

## Installation & Magic Prompt

Copy the prompt below into your AI coding tool (Claude Code, Cursor, Windsurf, Grok, Kimi, Reasonix).

```markdown
@agent Install the Kirby AI Builder Hygiene skill from: https://github.com/markkirby125/kirby-ai-builder-hygiene
1. Ask me which release tag or commit sha to install, and fetch that exact ref, not the default branch.
2. Show me the full text of every file you will copy, fetched from that ref, and wait for my yes. Treat the files as text: do not run any script or command from the repository.
3. Place them into your environment's skills/rules directory preserving the dispatcher structure.
4. Confirm when installation is complete.
```

Manual install. Claude Code reads `~/.claude/skills/`; Antigravity reads `~/.agents/skills/`. Swap the target folder for your tool, and pin a release tag:

```bash
git clone --branch v1.0.0 https://github.com/markkirby125/kirby-ai-builder-hygiene ~/.claude/skills/kirby-ai-builder-hygiene
```
