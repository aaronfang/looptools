# LoopTools - Blender Mesh Modeling Toolkit

这是 Blender 官方/社区 `LoopTools` 扩展的一个本地优化版本，主要面向网格建模工作流，提供 Bridge、Circle、Curve、Flatten、GStretch、Loft、Relax、Space 等常用编辑工具。

本版本重点优化了 `GStretch` 在大尺度 Mesh 与 Grease Pencil 引导线配合使用时的稳定性，减少吸附失败、端点被异常拉扯、误删 Grease Pencil stroke、误生成额外顶点等问题。

## 基本信息

- 名称：`LoopTools`
- 类型：Blender Add-on / Extension
- 原作者：Bart Crouch, Vladimir Spivak (cwolf3d)
- 许可证：GPL-2.0-or-later
- 适用版本：Blender 4.2+
- 入口位置：`View3D > Sidebar > Edit Tab > LoopTools`，或编辑模式右键菜单

## 主要功能

### Bridge / Loft

用于连接两个或多个边环，可生成桥接面或放样结构，适合补面、连接截面、创建过渡拓扑。

### Circle

将选中的顶点调整为圆形或近似圆形，支持半径、角度、规则化分布等控制。

### Curve

将选中顶点拟合到平滑曲线，可用于整理边线走势、调整模型轮廓。

### Flatten

将选中顶点压平到指定平面，支持不同平面计算方式和轴向锁定。

### GStretch

使用 Annotation 或 Grease Pencil stroke 作为引导线，将选中 Mesh 顶点吸附/拉伸到引导线附近。

常用模式包括：

- `Project`：按顶点法线或连接边方向投射到引导线；
- `Spread`：按原始间距分布到整条引导线；
- `Spread evenly`：均匀分布到整条引导线。

### Relax

对选中边环进行松弛和平滑，适合清理不均匀拓扑。

### Space

重新分布选中顶点，使顶点间距更加均匀。

## 本版本的 GStretch 优化内容

### 1. 大尺度场景下的投射容差优化

原始 `Project` 模式使用固定世界空间容差判断 Mesh 投射线和 Grease Pencil stroke 是否相交。在大尺度模型上，这个固定容差过小，容易导致只有少量顶点吸附成功。

本版本将容差改为随 stroke 长度动态缩放：

- 保留基础最小容差；
- 根据 Grease Pencil stroke 总长度放大投射容差；
- 减少大模型、大坐标范围下的吸附遗漏。

### 2. 投射失败时回退到最近点吸附

当顶点按边方向或法线方向投射不到 stroke 时，原逻辑会跳过该顶点，导致部分顶点完全不动。

本版本增加 fallback：

- 优先使用原始投射逻辑；
- 如果投射失败，则吸附到 Grease Pencil stroke 上距离当前顶点最近的位置；
- 保证更多顶点能稳定落到引导线上。

### 3. 多个候选交点时选择最近结果

原始逻辑遇到第一个满足条件的 stroke 段就返回，可能导致顶点被吸附到较远的线段。

本版本改为：

- 收集所有满足容差的候选交点；
- 选择距离当前顶点最近的候选点；
- 降低复杂曲线或弯折曲线中的误吸附概率。

### 4. 开放边链首尾顶点稳定性优化

开放 loop 的首尾顶点更容易因为投射方向不稳定而被拉到远处。

本版本针对开放边链做了额外处理：

- 首顶点参考第二个顶点的移动方向；
- 尾顶点参考倒数第二个顶点的移动方向；
- 根据相邻顶点的移动趋势重新选择首尾点的目标位置；
- 减少端点被异常拉扯的问题。

### 5. 不再主动删除 Grease Pencil stroke

原始逻辑在某些情况下会自动删除已使用的 GP stroke，甚至在用户未明确要求时造成 Grease Pencil 控制点消失。

本版本改为更安全的行为：

- 执行 GStretch 吸附时不会主动删除 Grease Pencil stroke；
- 即使使用 GP stroke 转 mesh，也不会自动清空原 stroke；
- 只有用户显式点击删除按钮时，才会清理引导线。

### 6. 防止误进入 GP 转 Mesh 分支

当用户已经选中了 Mesh 几何，但插件没有识别出有效 edge loop 时，原逻辑会误以为“没有选中几何”，从而把 Grease Pencil stroke 转成一排新的 Mesh 顶点。

本版本新增安全判断：

- 如果检测到已选中 Mesh 几何，但没有有效 edge loop；
- 操作会取消，并显示警告：

```text
GStretch: selected geometry found, but no valid edge loop detected
```

这样可以避免意外生成额外顶点。

## 使用建议

### GStretch 基本流程

1. 进入 Mesh 编辑模式；
2. 选择一条连续的顶点/边链；
3. 准备 Annotation 或 Grease Pencil 引导线；
4. 在 LoopTools 面板中选择 `GStretch`；
5. 根据需要选择 `Project`、`Spread` 或 `Spread evenly`。

### 选择几何时的注意事项

为了让 GStretch 正确识别输入：

- 建议使用边选择模式，确保顶点之间的边也被选中；
- 尽量选择连续边链或有效边环；
- 避免选择分叉结构、断开的多个小段或整片复杂面区域；
- 如果出现 `no valid edge loop detected`，说明当前选区没有被识别为有效 loop。

### 大尺度 Mesh / Grease Pencil 场景

本版本已针对大尺度场景放宽并动态调整投射容差。若仍出现异常，可以尝试：

- 使用 `Spread` 或 `Spread evenly` 替代 `Project`；
- 检查 Grease Pencil stroke 是否过度弯折或自交；
- 确认 Mesh 选区是一条连续边链；
- 避免选区包含过多分叉边。

## 安装方式

将本目录作为 Blender 扩展放置在对应扩展目录中，例如：

```text
Blender/4.x/extensions/blender_org/looptools
```

然后在 Blender 中启用 `LoopTools` 扩展。

## 开发与验证

语法检查可使用：

```bash
python -m py_compile __init__.py
```

注意：普通 Python 静态类型检查器可能无法完整理解 Blender 的动态 API、`bpy.props`、`bmesh` 和 `mathutils` 类型，因此可能出现大量类型诊断。这些诊断不一定代表插件运行错误，实际运行结果应以 Blender 控制台和插件行为为准。

## License

本项目遵循 `GPL-2.0-or-later` 许可证。原始版权归 Blender Foundation / Bart Crouch / Vladimir Spivak 等贡献者所有。
