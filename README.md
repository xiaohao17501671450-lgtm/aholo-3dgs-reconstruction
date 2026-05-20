# aholo-3dgs-reconstruction

Cursor / OpenClaw Agent Skill for **Aholo China Open Platform** 3D tasks: world reconstruction and spatial generation via OpenAPI v1.

- **Gateway:** `https://api.aholo3d.cn`
- **World APIs:** `/world/v1/...`
- **API keys:** [labs.aholo3d.cn/api-keys](https://labs.aholo3d.cn/api-keys)

## Contents

| File | Role |
|------|------|
| `SKILL.md` | Agent instructions (required for Cursor skills) |
| `aholo_reconstruct.py` | CLI: upload assets, create task, poll status |

## Install (Cursor)

```bash
git clone https://github.com/xiaohao17501671450-lgtm/aholo-3dgs-reconstruction.git
```

Copy or symlink the repo folder into your skills directory, e.g.:

- Windows: `%USERPROFILE%\.cursor\skills\aholo-3dgs-reconstruction`
- macOS/Linux: `~/.cursor/skills/aholo-3dgs-reconstruction`

Restart Cursor so the skill is discovered.

## Prerequisites

```bash
pip install -r requirements.txt
```

Set your API key:

```bash
# PowerShell
$env:AHOLO_API_KEY="your_api_key"
```

```bash
# bash
export AHOLO_API_KEY="your_api_key"
```

## Quick test

```bash
python -u aholo_reconstruct.py '{"action":"status","worldId":"<worldId>"}'
```

See `SKILL.md` for full workflows (`create`, `poll`, `scene`, `taskQuality`, etc.).

## Docs

- [Aholo Labs — Aholo World skill](https://labs.aholo3d.cn/skills/aholo-3dgs-reconstruction) (Chinese site docs)
- [SkillHub](https://skillhub.cn/skills/aholo-3dgs-reconstruction)
