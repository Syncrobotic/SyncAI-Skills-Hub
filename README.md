# SyncAI-Lib-OrbieSkill

[Agent Skills](https://agentskills.io/) for the **Orbie** video analytics platform. Each skill in this repo teaches AI coding agents a different aspect of working with Orbie.

## Skills

| Skill | Description |
|-------|-------------|
| [orbie-api](./orbie-api/) | Integrate with the Orbie REST API — devices, scenes, alerts, analysis, webhooks, usage |

## Install

```bash
npx skills add Syncrobotic/SyncAI-Lib-OrbieSkill
```

This auto-detects your installed agents (GitHub Copilot, Claude Code, Cursor, etc.) and symlinks the skills to the right directories. See [skills CLI docs](https://www.npmjs.com/package/skills) for options like `--global`, `--agent`, `--skill`.

<details>
<summary>Manual install</summary>

```bash
ln -s "$(pwd)/orbie-api" ~/.agents/skills/orbie-api
```

</details>

Once installed, skills activate automatically or can be invoked via `/skill-name`.

## License

Proprietary — SyncAI
