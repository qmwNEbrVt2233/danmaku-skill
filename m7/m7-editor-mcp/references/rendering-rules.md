# 渲染规则

本文依据 `src/components/player/DanmakuLayer.vue`。弹幕字段最终映射到 DOM 内联样式和容器 CSS；需要精确复现观感时，可按 CSS 变换、锚点和画布坐标思考。坐标和字号均按像素处理。

## 元素结构与基础样式

每条弹幕对应 `.danmaku` 绝对定位元素，内部 `.danmaku-content` 保存文字。画布层 `.layer` 铺满画布并设置 `perspective: 500px`；弹幕元素使用 `transform-style: preserve-3d`，变换原点为 `0% 0%`，即文字框左上角。

| MCP 字段 | DOM/CSS 映射 | 结果 |
| --- | --- | --- |
| `content.text` | 写入内容；换行符转成 `<br />` | 保留空格，长内容允许断词；默认不换行，文本中的显式换行除外。 |
| `content.font` | `font-family: "字段值", Arial, Helvetica, sans-serif` | 指定字体不可用时回退到后续字体。 |
| `content.size` | `font-size: Npx` | 字号是像素大小，不随画布宽高缩放。 |
| `content.color` | `color: 字段值` | 文字前景色。 |
| `content.stroke` | 开启时设置八方向 `1px` 黑色 `text-shadow` | 这是黑色文字阴影描边，不是独立 SVG 或 CSS `-webkit-text-stroke`。 |
| `opacity.from`, `opacity.to` | 内联 `opacity` | 在整段 `animation.duration` 生命周期中线性插值，与运动缓动分开。 |
| `transform.start.{x,y}`, `transform.end.{x,y}` | `translate3d(xpx, ypx, 0)` | 从起点到终点插值，锚点始终是文字框左上角。 |
| `transform.zRotate` | 非 0 时追加 `rotateZ(Ndeg)` | 绕左上锚点在画面平面内旋转。 |
| `transform.yRotate` | 非 0 时追加 `rotateY(360 - N deg)` | 绕左上锚点作 3D 翻转；父层的 500px 透视会影响投影。 |
| `layer`, `startTime` | 用于排序，排序后的序号分配 `z-index` | 排序键先是 `startTime` 升序，再是 `layer` 升序；序号越大，CSS 层级越高。 |

## 时间与运动映射

弹幕在 `[startTime, startTime + duration]` 区间内可见。设 `t = 当前时间 - startTime`：

- `t <= delay` 时运动进度为 0，弹幕留在起点。
- `moveDuration > 0` 时，运动进度为 `clamp((t - delay) / moveDuration, 0, 1)`。
- `easing = speedup` 时进度再变为 `p²`（慢到快）；`speeddown` 时为 `1 - (1 - p)²`（快到慢）。
- 位置按缓动后进度在 start/end 坐标间线性插值。因而单条弹幕沿直线坐标移动，但时间到位移的变化为二次曲线。
- `moveDuration <= 0` 时直接将进度设为 1，代码不会再检查 `delay`。需要延迟后瞬移或用延迟控制层级时，把 `moveDuration` 设为 `1`，不要设为 `0`。
- `duration` 控制可见寿命，不等于运动耗时；运动结束后弹幕仍可停留到寿命结束。

## 画布、坐标和 3D 透视

MCP 的 `transform.start/end` 使用 CSS 像素坐标。画布宽高可在生成表达式中通过 `width`、`height` 读取，用它们计算居中或边缘位置；坐标字段本身仍是像素，不会因为修改画布尺寸自动按比例缩放。

`yRotate` 不是 0 或 180 度时，文字平面有一部分进入/离开透视深度，最终投影受到元素相对左上锚点的位置、画布尺寸与构图影响。同一个角度在不同宽高画布上的视觉位置和透视比例可能不同。0/180 度没有这种侧向深度分量，通常不会出现同样的透视偏移。制作斜角 Y 轴翻转时，应按目标画布检查 `get_frame`，不要只凭角度推算。

Z 轴旋转也以左上角为原点。工具栏测量会根据原始文字框宽高和 `zRotate` 计算旋转后的轴对齐包围盒；这用于居中/测量，不代表弹幕锚点改到了中心。

## 性能含义

每条可见弹幕对应一个 DOM 节点。日常创作一般不必为性能限制设计；同屏数千条移动弹幕，或约一秒内创建、销毁数千个节点时，预览可能出现卡顿。逐帧导出逐帧计算会花更久，但不会降低最终画面质量。一般制作时尽量优化即可。
