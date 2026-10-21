# 通用制作技巧到 MCP 的映射

本文说明如何用 m7-editor MCP 完成常见动画制作任务。工具参数的完整结构见 [mcp-api.md](mcp-api.md)，CSS 映射见 [rendering-rules.md](rendering-rules.md)。

## 制作规范

- **按效果需要选择数量。** 一条弹幕是一个 DOM 元素。高密度逐帧会频繁创建和移除节点，同屏大量移动弹幕也会增加预览负担；通常不必因此限制设计，若预期会达到极端数量请进行优化。
- **先规划定位与层级。** MCP 坐标是画布像素，字号也不会按宽高比例缩放。弹幕都以文字框左上角作为变换锚点。建立完整构图前确认画布尺寸、坐标约定、前后层级和 3D 翻转效果。
- **每个阶段及时保存或备份。** 完成一段打轴、一个镜头/构图或一批生成并检查后，立即调用 `save_folder_project({})` 或 `backup_folder_project({})`。这两项仅适用于文件夹工程；本地工程也需要恢复点。若是单文件工程，应在制作完成后提醒用户导出工程以避免工作丢失
- **批量生成并检查。** 规则重复时用 `create_generated_danmakus`，减少逐条调用；完成后查看操作结果，必要时用 `list_danmakus`、`get_frame` 检查数量、字段和目标时刻画面。单个生成批次作为一次编辑/撤销操作。

## 连续运动与路径近似

一条 MCP 弹幕只有一组 start/end 坐标，沿一条直线运动，时间进度按 `speedup` 或 `speeddown` 二次缓动。需要多段路径时，在段与段之间安排多个弹幕：上一段的终点和下一段的起点相同，后段 `startTime` 接续前段的到达时间（前段 `startTime + delay + moveDuration`）。按 `animation.duration` 覆盖各段运动时间，必要时让两段生命周期轻微重叠，并用 `get_frame` 检查接缝。

段与段之间不需要固定增加额外寿命；先检查接缝，只有画面确实出现空隙时才调整 `duration` 或时间安排。路径可用少量直线段近似：将各段端点放在期望曲线上，按关键点时间安排 `startTime`，必要时用生成表达式计算坐标。单段运动本身带二次缓动，许多柔和弯曲可以用少量采样表达；需要更细腻的曲线时也可以增加采样密度。

## 逐帧动画

逐帧可以通过 `create_generated_danmakus` 创建一组短时弹幕，每条代表一个采样帧。用 `startTime` 的 `relative` 规则设置帧间距（例如 `"step": "+50"` 代表约 20 帧/秒）；用 `range.expression` 计算每帧的位置、字号或颜色，用 `content.text` 模板生成帧文字；用 `animation.duration` 设为帧间隔附近的时长。若帧只需静止显示，令每条弹幕 `transform.start` 与 `transform.end` 相同。

示意结构（具体坐标表达式按作品调整）：

```json
{
  "quantity": 20,
  "relativeToCurrentTime": false,
  "rules": {
    "startTime": { "mode": "relative", "start": 1000, "step": "+50" },
    "animation.duration": { "mode": "range", "start": 50, "end": 50 },
    "animation.moveDuration": { "mode": "range", "start": 1, "end": 1 },
    "transform.start.x": { "mode": "range", "start": 100, "end": 800 },
    "transform.end.x": { "mode": "range", "start": 100, "end": 800 },
    "transform.start.y": { "mode": "range", "start": 200, "end": 500 },
    "transform.end.y": { "mode": "range", "start": 200, "end": 500 },
    "content.text": { "mode": "assign", "value": "●" }
  }
}
```

逐帧需要保证同一索引的 start/end 坐标相同；示例中对应字段使用相同的 `range` 首尾值，因此每条都是静止采样点。若需要每帧的状态由复杂公式决定，可对 start/end 对应字段使用相同表达式，并为两条规则提供相同的 `start`、`end`。

帧间隔可按动作需要选择，例如 50ms（20fps）或约 33ms（30fps）。逐帧会频繁创建和移除 DOM 元素，可能影响预览流畅度；一般规模无需刻意降帧，若短时间内达到数千次节点更新，再考虑降低密度或拆分预览。逐帧导出会逐帧计算，耗时较长但不影响成片质量。

## 近似匀速

MCP 的缓动只有 `speedup`（`p²`）和 `speeddown`（`1 - (1-p)²`），没有线性匀速选项。要做近似匀速，可让运动的大部分阶段发生在画布外，把画面内可见区域放在二次速度曲线相对平缓的一段；用画布尺寸计算屏外起终点，再用 `startTime`、`delay` 和 `duration` 控制可见时段。对需要精确速度的短路径，可改用多个短段配合时间采样并检查 `get_frame`。

## 控制层级与延迟瞬移

需要利用延迟预先布置对象时，渲染器对 `moveDuration <= 0` 会直接把进度设成 1，跳过 `delay` 判断；要保留延迟，应设置 `animation.moveDuration: 1`（毫秒），并把 `animation.delay` 设成所需等待时间。`duration` 要足够长以覆盖这段延迟和之后的可见时间。

例如一条弹幕要延迟 2000ms 后从画布外瞬移到目标点：设 start 为画布外坐标、end 为目标坐标、`delay: 2000`、`moveDuration: 1`，并令 `duration > 2001`。延迟期间弹幕留在起点；到延迟结束后约 1ms 到达终点。若屏外起点仍可能进入画面，应使用目标画布尺寸检查坐标。

MCP 中实际前后顺序由渲染排序控制：先按 `startTime` 升序，再按 `layer` 升序，序号写入 CSS `z-index`，所以较晚 `startTime` 或同一 `startTime` 下较大的 `layer` 会被分配更大的 `z-index`。优先显式规划这些字段；延迟瞬移不会改变排序键本身。

## 颜色相关

需要制作变色效果时，首选使用两条弹幕叠加的方法制作，一条作为起始色，一条作为终点色，两条弹幕使用相同的运动变换，叠加透明度变化即可做出变色效果。在使用 `create_generated_danmakus` 制作逐帧动画时可通过颜色字段自动混色功能

遮罩颜色滤镜可使用遮罩模板制作，如#FFFFFF 透明度 0.2 的遮罩可使低层级弹幕更亮。
如果需要更精细的控制，建议使用 `split_danmakus` 时间分割后再使用 `select_by_command` 再行筛选，最后使用 `apply_field_update` 操作颜色字段，如：FFFFFF@0.2


## 常用模板

遮罩模板，可作为背景、滤镜等使用，原理是 `█` 字符在微软雅黑下近乎无缝

```json
{
  "layer": 0,
  "startTime": 0,
  "content": {
    "text": "███████████\n███████████\n███████████\n███████████",
    "font": "Microsoft YaHei",
    "size": 127,
    "color": "#FFFFFF",
    "stroke": false
  },
  "transform": {
    "start": { "x": 0, "y": 0 },
    "end": { "x": 0, "y": 0 },
    "zRotate": 0,
    "yRotate": 0
  },
  "opacity": { "from": 1, "to": 1 },
  "animation": {
    "duration": 1000,
    "moveDuration": 500,
    "delay": 0,
    "easing": "speedup"
  }
}
```