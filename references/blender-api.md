# Blender 5.x API 与官方依据

研究日期：2026-10-08。本地以正式安装的 Blender **5.2.2 LTS** 独立后台空场景验证了下列 channelbag 查询、Smooth modifier 字段、operator 参数名与 NLA 缩放语义。没有加载、修改或保存用户动画。版本升级后重新检查 RNA，不把开发分支 API 自动当成本机支持。

## Action 与 slot：读取当前所有者的曲线

Blender 5.0 移除了旧 `Action.fcurves`、`Action.groups` 和 `Action.id_root`。不能抄旧版本的遍历代码，也不能总取第一个 slot。[5.0 官方 API 变更](https://developer.blender.org/docs/release_notes/5.0/python_api/)

```python
from bpy_extras.anim_utils import action_get_channelbag_for_slot

# owner 必须是已定位到的动画所有者；不要依赖当前鼠标所选对象。
ad = owner.animation_data
if ad is None or ad.action is None or ad.action_slot is None:
    raise ValueError("当前所有者没有活动 Action/slot；继续审计 NLA 或实际驱动对象")
action = ad.action
slot = ad.action_slot
bag = action_get_channelbag_for_slot(action, slot)
curves = list(bag.fcurves) if bag is not None else []
for fc in curves:
    print(fc.data_path, fc.array_index, len(fc.keyframe_points),
          len(fc.sampled_points), len(fc.modifiers))
```

这是活动 Action 的定位起点，**不是完整场景求值器**。NLA strip 可能引用其他 Action/slot；多所有者、共享 Action、驱动器和约束也需要分别审计。无 channelbag 时不要在只读分析中调用 `action_ensure_channelbag_for_slot`，因为 ensure 会创建数据。

本机合成测试确认：`bag.fcurves.find('location', index=2)` 与 `fc.evaluate(5.5)` 可用；`hasattr(action, 'fcurves')` 为 False。

## 最终姿态与曲线值

- `fc.evaluate(frame)` 接受该曲线时间域的帧，可含小数，返回标量曲线值（含该曲线 modifier）。它不代表经过整个角色约束/NLA 后的世界空间变换。
- 场景采样需要 `scene.frame_set(integer_frame, subframe=fraction)`，然后更新/取得依赖图并读取 evaluated object/pose。骨骼的 `pose_bone.matrix` 是骨架对象空间；世界矩阵通常为 `evaluated_armature.matrix_world @ evaluated_pose_bone.matrix`。
- 查询骨骼时先核实其名称与语义，不能把名为 `root` 的对象、root 骨骼、IK 辅助骨当成同一实体。
- 记录帧率与 `fps_base`；Action time 和 Scene time 在 NLA 缩放、循环或重映射后不同。
- 采样若在用户会话中改变当前帧，需要恢复原帧及子帧；研究参数优先在隔离副本进行。

## Smooth (Gaussian) modifier：本机已核实接口

下面代码会修改一条曲线，**只用于用户已授权修正的副本**。其中 `fc`、区间和参数需要来自诊断，不能照抄到全骨架。

```python
# 先检查 fc.modifiers：Smooth 必须第一位，不能与 Cycles 共存。
# 这个最小示例仅处理没有其他 modifier 的曲线，避免静默改变堆栈。
if len(fc.modifiers):
    raise ValueError("先审计现有 modifier 堆栈，再决定修正方式")
smooth = fc.modifiers.new(type='SMOOTH')
smooth.sigma = sigma_in_frames
smooth.filter_width = filter_width_in_frames
smooth.use_influence = True
smooth.influence = blend_factor
smooth.use_restricted_range = True
smooth.frame_start = start_frame
smooth.frame_end = end_frame
smooth.blend_in = blend_in_frames
smooth.blend_out = blend_out_frames
```

已核实存在 `sigma`、`filter_width`、`use_influence`、`influence`、`use_restricted_range`、`frame_start/end`、`blend_in/out`。不存在该示例需要的 `blend_type='GAUSSIAN'`；不要使用 `modifiers.new('Smooth', 'SMOOTH')` 这类未核实签名。

限制：Smooth 必须堆栈第一位，与 Cycles 不兼容；子帧采用线性插值，可能影响运动模糊；大量曲线使用时可能慢。restricted range 与 influence 混合不是“接触完全不变”的证明，仍需最终姿态检查。[5.1 新功能说明](https://developer.blender.org/docs/release_notes/5.1/animation_rigging/)、[5.2 modifier 手册](https://docs.blender.org/manual/es/5.2/editors/graph_editor/fcurves/modifiers.html)

## Graph Editor operator 与上下文

本机读取 RNA 确认参数名如下，没有对用户曲线执行这些 operator：

| Operator | 参数名 |
|---|---|
| `bpy.ops.graph.gaussian_smooth` | `factor`, `sigma`, `filter_width` |
| `bpy.ops.graph.butterworth_smooth` | `cutoff_frequency`, `filter_order`, `samples_per_frame`, `blend`, `blend_in_out` |

operator 依赖编辑器上下文、可编辑曲线及所选关键帧。不得假定后台执行 `bpy.ops.graph.*` 就会命中目标；使用正确 context override，或采用显式数据处理并独立验证其数学语义。可以用 `operator.get_rna_type().properties` 检查当前版本的参数、类型、范围和说明。不要根据 UI 标签猜 Python 名称。

Euler Filter 要同时处理 XYZ；Quaternion 不适用。Gaussian 会影响局部细节。Butterworth 在急变处可能产生波纹，边界混合不保证武器接触不变。详见 [修正方法](curve-repair.md)。

`Clean` 的 Channels 模式可能处理曲线中未选中的键，甚至删掉只剩单键的曲线；不能用这个模式声称保留其他区间。`Decimate` 的 Allowed Change 是标量曲线偏差，不是刀尖空间误差。

## 时间重映射与数据修改注意项

- 本机合成 Action 帧范围 `[0, 10]`，NLA strip 的 `scale=2` 后帧范围为 `[0, 20]`，确认为时长加倍、速度减半。
- Animated Strip Time 的值对应 Action 帧。Tweak Mode 对这种时间映射的 UI 支持有限，不依赖其画面来验证完整栈。
- NLA Apply Scale 会写入被引用的 Action，先检查共享引用并制作可追溯副本。
- 手动修改 `keyframe_points` 的时间/数值后，手柄可能需要随之移动或重新计算，调用 `fc.update()`；检查自动手柄是否重算、边界是否改变。重排时间后检查排序与重复帧。
- `sampled_points` 与 `keyframe_points` 是不同表示；不要假定每条曲线都能直接逐键编辑。需要转换时在副本中进行，并验证重采样误差。
- Quaternion 分量应成组处理并归一化；Euler 使用原轴序。将旋转换一种表示需要保留方向与连续性，不是换一个 enum 即完成转换。
- 修改后重新读取和求值，确认改动确实作用于目标 slot，且没有被约束/驱动器覆盖。

## 官方来源

以下是工具事实依据。接触保护、事件分段和误差验收是本技能据其能力边界制定的策略，不是官方滤波器的保证。

| 主题 | 一手来源 |
|---|---|
| Blender 5.0 移除旧 Action API、slot helper | [5.0 Python API](https://developer.blender.org/docs/release_notes/5.0/python_api/) |
| Action slot 的组织 | [Slotted Actions 迁移](https://developer.blender.org/docs/release_notes/4.4/upgrading/slotted_actions/)、[ActionChannelbag API](https://docs.blender.org/api/5.1/bpy.types.ActionChannelbag.html) |
| Euler Filter 操作与算法边界 | [5.2 Graph Channels](https://docs.blender.org/manual/ja/5.2/editors/graph_editor/channels/editing.html#discontinuity-euler-filter)、[Euler Filter 算法说明](https://developer.blender.org/docs/release_notes/2.92/animation_rigging/#euler-filter) |
| 旋转表示和模式一致性 | [Rotation Modes](https://docs.blender.org/manual/en/4.3/advanced/appendices/rotations.html) |
| Gaussian/Butterworth 操作 | [Graph Editor 平滑](https://docs.blender.org/manual/de/latest/editors/graph_editor/fcurves/editing.html#smooth)、[5.1 Graph operators API](https://docs.blender.org/api/5.1/bpy.ops.graph.html#bpy.ops.graph.butterworth_smooth) |
| Gaussian modifier 的引入及限制 | [5.1 Animation & Rigging](https://developer.blender.org/docs/release_notes/5.1/animation_rigging/)、[5.2 F-Curve Modifiers](https://docs.blender.org/manual/es/5.2/editors/graph_editor/fcurves/modifiers.html) |
| Modifier 通用 influence、范围与混合 | [FModifier API 5.2](https://docs.blender.org/api/5.2/bpy.types.FModifier.html) |
| Clean 与 Decimate | [Clean Keyframes](https://docs.blender.org/manual/ka/5.0/editors/graph_editor/fcurves/editing.html#clean-keyframes)、[Decimate 手册](https://docs.blender.org/manual/en/5.3/editors/graph_editor/fcurves/editing.html#decimate) |
| NLA 缩放、Strip Time | [NLA Sidebar 5.2](https://docs.blender.org/manual/uk/5.2/editors/nla/sidebar.html) |
| Apply Scale 与 Tweak Mode | [NLA Strip Editing](https://docs.blender.org/manual/en/4.4/editors/nla/editing/strip.html#apply-scale)、[NLA 开发说明](https://developer.blender.org/docs/features/animation/nla/) |

部分引用是旧版稳定概念或较新手册内容，不能据此跳过本机兼容性检查。未运行的 API/算法应标明未验证，不能声称已完成动画修复。
