# Blender Motion Curves

中文 | [English](README.en.md)

一个面向 Codex 的 Blender 动画曲线分析与修正技能。通过分段时间重映射、局部滤波和少量姿态偏移控制点，辅助修正 HY-Motion、动捕等密集关键帧动画，减少逐帧手动 K 帧的工作。

这是供 AI 编程助手读取的工作方法与参考资料，不是 Blender 插件，也不包含一键自动修复程序。

## 能做什么

- 分析攻击、招架等动作的节奏、抖动、旋转跳变和衔接问题。
- 区分 Euler 绕回、Quaternion 反号、IK 翻转、根位移与错误姿态。
- 规划局部平滑、分段调速、低频偏移与受控减键。
- 在修正时保护握剑、武器接触和脚底支撑，并检查最终求值姿态。
- 优先处理主要可见缺陷；手臂穿插检查实际蒙皮，IK 修正复查肘部翻转，未通过则在交付前继续迭代。
- 指导 Blender 5.x Action slot、channelbag、F-Curve modifier 与 NLA 的正确使用。

曲线平滑不能替代正确的绑定、重定向、IK 或握持几何关系。技能要求先定位问题，再做最小修正。

笼统要求“修这段动作”时，不能只优化容易量化的收势速度而留下明显穿插。明确只要求调速时仍遵守该范围。这套流程减少把漏修交给用户发现的情况，不承诺一次求解就能自动修好所有动作。

## 安装

需要 Git 和支持本地技能的 Codex。分析资料本身不需要安装 Python 依赖；读取或修改 `.blend` 动画时，需要可用的 Blender 与相应文件访问能力。

Windows PowerShell：

```powershell
$skillsRoot = if ($env:CODEX_HOME) { Join-Path $env:CODEX_HOME 'skills' } else { Join-Path $HOME '.codex/skills' }
New-Item -ItemType Directory -Force -Path $skillsRoot | Out-Null
git clone https://github.com/Chencole/blender-motion-curves.git (Join-Path $skillsRoot 'blender-motion-curves')
```

macOS / Linux：

```bash
skills_root="${CODEX_HOME:-$HOME/.codex}/skills"
mkdir -p "$skills_root"
git clone https://github.com/Chencole/blender-motion-curves.git "$skills_root/blender-motion-curves"
```

如果同名目录已经存在，先检查现有内容，避免覆盖自定义修改。安装后重新打开 Codex 会话；若尚未显示，重启 Codex。

其他动画/绑定技能均为可选，本技能可以独立安装。

## 使用示例

只分析，保留原文件：

```text
用 $blender-motion-curves 分析这两段攻击和招架。
先检查曲线、动作节奏和接触区间，提出修改方案，不修改动画。
```

授权修正时：

```text
用 $blender-motion-curves 在副本中修正这段动画的手腕抖动和出招节奏。
保留大体动作、根位移和一把剑的双手握持关系，交付修正前后对比。
```

提供实际 `.blend` 文件及目标片段。只有截图时，技能会区分可见现象与待验证判断，不声称已读取曲线。

## 文件

| 文件 | 用途 |
|---|---|
| [SKILL.md](SKILL.md) | 触发条件、范围、诊断与验收流程 |
| [references/curve-repair.md](references/curve-repair.md) | 曲线修正方法与攻击/招架分析示例 |
| [references/blender-api.md](references/blender-api.md) | Blender API 注意事项、验证记录与官方来源 |
| [references/pose-contact-review.md](references/pose-contact-review.md) | 主要缺陷优先、蒙皮穿插、IK 平面连续性及内部迭代 |
| [agents/openai.yaml](agents/openai.yaml) | Codex 展示名称与默认提示词 |

## 验证范围

2026-10-08：在独立 Blender 5.2.2 LTS 空场景中验证了 Action slot/channelbag 查询、Smooth modifier 字段、Graph Editor operator 参数名与 NLA 缩放语义；技能结构校验通过。不同 Blender 版本仍需检查兼容性。

姿态复审流程另经独立只读场景验证：笼统修复、仅改节奏、骨段分开但蒙皮仍交叉、缺少握持目标四类情形。测试覆盖问题优先级、任务范围和交付判断，不是自动修复成功率评测。

这些检查不代表已验证任何特定用户动画的修复效果。完整方法及 Blender 官方资料链接见上述参考文档。
