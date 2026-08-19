# laelaps-skills

Claude Code skills for the Laelaps team, packaged as a plugin.

## Install

```
/plugin marketplace add ErikLit005/laelaps-skills
/plugin install laelaps-skills@laelaps
```

Later changes: `/plugin update laelaps-skills@laelaps` (restart to apply).

## Skills

- **humanize** — strip AI writing patterns out of drafted copy and add real voice. Source: [agent-foundations-starter](https://github.com/JamsusMaximus/agent-foundations-starter/blob/main/skills/humanizer/SKILL.md).
- **grilling** — relentless interview to stress-test a plan or decision before committing to it.
- **handoff** — compact the current conversation into a handoff doc for another agent/session to pick up.

## Layout

```
.claude-plugin/
  plugin.json         plugin manifest
  marketplace.json    marketplace manifest — points at this repo root
skills/<name>/SKILL.md
```

Validate after editing: `claude plugin validate .`
