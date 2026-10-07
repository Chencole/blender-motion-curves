# Blender Motion Curves

[中文](README.md) | English

A Codex skill for analyzing and repairing Blender animation curves, especially densely keyed HY-Motion and motion-capture clips. It guides segment-based retiming, local filtering, and corrections defined by a small number of control points instead of manually posing every frame.

This repository contains instructions and reference material for an AI coding assistant. It is not a Blender add-on or a one-click repair program. The skill instructions and detailed references are currently written in Chinese.

## Capabilities

- Diagnose timing, jitter, rotation discontinuities, and transitions in attack/parry clips.
- Distinguish Euler wrapping, quaternion sign changes, IK flips, root-motion issues, and incorrect poses.
- Plan local smoothing, time remapping, low-frequency offsets, and controlled key reduction.
- Protect hand grips, weapon contact, and planted feet through evaluated-pose checks.
- Use Blender 5.x Action slots, channelbags, F-Curve modifiers, and NLA with version-aware guidance.

Curve smoothing cannot replace correct rigging, retargeting, IK, or grip geometry. The workflow calls for diagnosis and the smallest appropriate correction.

## Installation

Requires Git and Codex with local skill support. Reading the skill requires no Python dependencies. Inspecting or editing animation files requires Blender and a way for the assistant to access those files.

Windows PowerShell:

```powershell
$skillsRoot = if ($env:CODEX_HOME) { Join-Path $env:CODEX_HOME 'skills' } else { Join-Path $HOME '.codex/skills' }
New-Item -ItemType Directory -Force -Path $skillsRoot | Out-Null
git clone https://github.com/Chencole/blender-motion-curves.git (Join-Path $skillsRoot 'blender-motion-curves')
```

macOS / Linux:

```bash
skills_root="${CODEX_HOME:-$HOME/.codex}/skills"
mkdir -p "$skills_root"
git clone https://github.com/Chencole/blender-motion-curves.git "$skills_root/blender-motion-curves"
```

If the destination already exists, inspect it before replacing any local customizations. Open a new Codex session after installation; restart Codex if the skill is not listed yet. Companion animation and rigging skills are optional.

## Usage

Analysis only:

```text
Use $blender-motion-curves to analyze these attack and parry clips.
Inspect their curves, timing, and contact intervals. Propose changes without editing the animations.
```

An authorized repair:

```text
Use $blender-motion-curves to repair wrist jitter and attack timing in a copy of this animation.
Preserve the choreography, root motion, and two-handed grip on one sword.
Provide a before/after comparison.
```

Provide the actual `.blend` file and identify the target clips. Screenshots alone support observations and hypotheses, not claims that the underlying curves have been inspected.

## Contents

| File | Purpose |
|---|---|
| [SKILL.md](SKILL.md) | Scope, diagnostic workflow, editing rules, and acceptance criteria |
| [references/curve-repair.md](references/curve-repair.md) | Repair methods and an attack/parry analysis example |
| [references/blender-api.md](references/blender-api.md) | API guidance, validation notes, and official sources |
| [agents/openai.yaml](agents/openai.yaml) | Codex display metadata and default prompt |

## Validation scope

On 2026-10-08, an isolated Blender 5.2.2 LTS factory scene was used to verify Action slot/channelbag access, Smooth modifier fields, Graph Editor operator parameter names, and NLA scaling semantics. Skill structure validation passed. Check compatibility when using another Blender version.

These checks do not establish repair quality for any particular animation. See the references for methodology and links to the official Blender documentation.
