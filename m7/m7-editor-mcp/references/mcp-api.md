# m7-editor MCP 调用参考

此参考描述仓库 `src/mcp/server.ts`、`src/utils/danmakuGenerator.ts`、`src/core/danmaku.ts`、`src/components/player/DanmakuLayer.vue` 和 `src/components/preference/about.vue` 中的行为。实际调用时以 MCP 当前注册的 Zod schema 为准。

## 会话与常用流程

- 编辑器必须正在运行并连接 MCP 本机桥。`get_editor_state` 返回离线错误时，先让用户启动或重新连接桌面应用；MCP stdio 服务与桌面应用可分别运行。
- 先 `get_editor_state` 确认工程、播放头、画布、选择范围；需要检查具体对象时用 `list_danmakus`。
- 普通字段编辑和工具栏操作默认作用于当前选择。可先用 `select_by_command` 按 `/s` 筛选，也可在读取弹幕时用 ID 列表缩小检查范围。
- 创建或编辑后检查操作结果的新增、修改、删除和选中 ID；需要视觉检查时调用 `get_frame`。创建成功会自动分配新 ID 并选中新弹幕。
- 不要把 `get_frame` 当作截图当前播放头：它渲染传入的 `timeMs`，不会改变主窗口播放头或工程。
- `save_folder_project` 与 `backup_folder_project` 只适用于文件夹工程；单文件工程会返回错误。

## 弹幕对象结构

`create_danmakus` 接收单个对象、对象数组或本机 JSON 文件。对象使用下面的字段结构；输入的 `id` 会忽略并重新分配。坐标单位为像素，时间单位为毫秒，角度单位为度。

```json
{
  "layer": 0,
  "startTime": 1000,
  "content": {
    "text": "示例",
    "font": "Microsoft YaHei",
    "size": 60,
    "color": "#FFFFFF",
    "stroke": false
  },
  "transform": {
    "start": { "x": 0, "y": 0 },
    "end": { "x": 1280, "y": 720 },
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

调用结构：

```json
{
  "danmaku": { "...": "一个弹幕对象" },
  "relativeToCurrentTime": false
}
```

或：

```json
{
  "jsonPath": "C:/path/to/danmakus.json",
  "relativeToCurrentTime": true
}
```

`danmaku` 与 `jsonPath` 必须二选一。文件可包含一个弹幕对象、弹幕数组，或包含 `danmakus` 数组的工程 JSON；相对文件路径按 MCP 服务工作目录解析。`relativeToCurrentTime=false` 保留输入的绝对 `startTime`，只分配新 ID 后添加。设为 `true` 时把整批最早的弹幕对齐到当前播放头，并按插入规则分配层级。

## `create_generated_danmakus`

结构：

```json
{
  "quantity": 4,
  "relativeToCurrentTime": false,
  "rules": {
    "startTime": { "mode": "relative", "start": 1000, "step": "+250" },
    "transform.start.x": { "mode": "range", "start": 0, "end": 800 },
    "transform.end.x": {
      "mode": "range",
      "start": 1280,
      "end": 1280,
      "expression": "S + (E - S) * t"
    },
    "content.text": { "mode": "assign", "value": "`第 ${i + 1} 条`" },
    "content.color": { "mode": "range", "start": "#FFFFFF", "target": "#FF6600" },
    "animation.easing": { "mode": "assign", "value": "speeddown" }
  }
}
```

`quantity` 是 1–1000 的整数；`rules` 至少提供一个字段。未指定的字段继续使用 creationTools 默认值。所有规则对象都严格校验，未知字段、未知模式或额外属性会被拒绝。

### 支持字段和模式

数值字段允许 `range`、`relative`、`cycle`：

| 字段路径 | `range` | `relative` | `cycle` |
| --- | --- | --- | --- |
| `layer`, `startTime`, `content.size` | `{ "mode":"range", "start":0, "end":100, "expression":"..." }` | `{ "mode":"relative", "start":0, "step":"+1" }` | `{ "mode":"cycle", "values":[0,1,2] }` |
| `transform.start.x`, `transform.start.y`, `transform.end.x`, `transform.end.y` | 同上 | 同上 | 同上 |
| `transform.zRotate`, `transform.yRotate` | 同上 | 同上 | 同上 |
| `opacity.from`, `opacity.to` | 同上 | 同上 | 同上 |
| `animation.duration`, `animation.moveDuration`, `animation.delay` | 同上 | 同上 | 同上 |

`expression` 只用于 `range`，省略时按首尾值线性均分。`relative.step` 推荐使用带运算符的字符串：`"+20"`、`"-10"`、`"*1.1"`、`"/2"`。按当前解析规则，正向增量要写 `"+20"`；裸值 `20` 表示把后续项设为 20，而非增加 20。`cycle.values` 按数组循环，数值项可为表达式字符串。

颜色字段 `content.color` 允许：

```json
{ "mode": "range", "start": "#FFFFFF", "target": "#FF6600" }
{ "mode": "relative", "start": "#FFFFFF", "target": "#FF6600", "alpha": 0.2 }
{ "mode": "cycle", "values": ["#FFFFFF", "FF6600"] }
```

`range` 从起始色逐步混合到目标色，最后一项到达目标色；`relative` 使用每步 Alpha 增量，索引 `i` 对应的混合比例为 `min(1, alpha * i)`；`cycle` 按颜色列表循环。颜色使用十六进制色值。

直接字段允许 `assign` 或 `cycle`：

| 字段 | 赋值示例 | 循环示例 |
| --- | --- | --- |
| `content.text` | `{ "mode":"assign", "value":"你好" }` | `{ "mode":"cycle", "values":["你好","再见"] }` |
| `content.font` | `{ "mode":"assign", "value":"Microsoft YaHei" }` | `{ "mode":"cycle", "values":["Arial","Microsoft YaHei"] }` |
| `content.stroke` | `{ "mode":"assign", "value":true }` | `{ "mode":"cycle", "values":[true,false] }` |
| `animation.easing` | `{ "mode":"assign", "value":"speedup" }` | `{ "mode":"cycle", "values":["speedup","speeddown"] }` |

文本和字体值为字符串；描边值为布尔值；缓动仅接受 `speedup` 或 `speeddown`。循环长度可以小于 `quantity`，到末尾后从第一项重复。

未覆盖字段的默认值来自 creationTools：Layer 从 0 开始每条加 1；开始时间从当前播放头开始每条加 100 ms；字号 60；起终点坐标、旋转、延迟为 0；颜色从 `#FFFFFF` 到 `#FFAA00`；透明度从 1 到 1；生存时间 1000 ms；运动时间 500 ms；字体 `Microsoft YaHei`；描边关闭；缓动 `speedup`；默认文本为“欢迎使用高级创建工具”。明确传入规则可覆盖对应默认值。

### 表达式与可用变量

创建规则的数值 `range.expression` 和 `cycle.values` 使用 math.js 表达式，不要加 `$`。每条生成项都会求值一次：

- `S`、`E`：该数值规则的起始值和结束值。
- 在 `cycle.values` 和文本模板中，`S`、`E` 为 0；它们主要用于 `range` 表达式。
- `i`：当前生成索引，从 0 开始。
- `n`：生成数量。
- `t`：归一化进度；数量大于 1 时为 `i / (n - 1)`，否则为 0。
- `width`、`height`：当前工程画布尺寸。
- `pi`、`e`：math.js 常量。
- `bezier(x1, y1, x2, y2, t)`：三阶贝塞尔缓动函数。
- 其他生成字段路径：例如 `transform.start.x`、`startTime`。引用的是同一索引的生成值；数值计算应引用数值字段。字段引用会自动确定计算依赖，直接自引用或循环引用会报错。

基础运算包括 `+ - * / ^ () %`；math.js 的标准数学函数可用，例如 `sin`、`cos`、`abs`、`sqrt`、`min`、`max`、`floor`、`ceil`、`round`、`log`、`exp`、`random()`。例如：

```json
{
  "mode": "range",
  "start": 0,
  "end": 1000,
  "expression": "S + (E - S) * bezier(0.42, 0, 0.58, 1, t)"
}
```

`content.text` 的 `value` 只有在整个值以反引号包住时才按模板解析，`${...}` 内使用同一生成表达式作用域，例如 `` `进度 ${round(t * 100)}%` ``。普通字符串不会被插值。

## 字段编辑表达式

`apply_field_update` 输入为 `{ "field": "字段路径", "method": "输入值" }`，编辑字段使用带点的路径，例如 `transform.start.x`、`content.text`、`opacity.from`。数值字段支持直接赋值 `1000`、相对运算 `+100` / `-50` / `*2` / `/2`、随机范围 `r+50` / `r-30` / `r100`，以及以 `$` 开头的 math.js 表达式。`$` 后不写为空格分隔语法，例如 `"$startTime + duration / 2"`。表达式按当前选择中的每条弹幕分别求值。

字段编辑表达式使用 `about.vue` 中的别名：`layer`、`startTime`、`size`、`startX`、`startY`、`endX`、`endY`、`zRotate`、`yRotate`、`opacityFrom`、`opacityTo`、`duration`、`moveDuration`、`delay`。这些别名与创建规则用的带点字段路径作用域不同。颜色可使用普通十六进制色值或 Alpha 混合格式，例如 `FFFFFF@0.5`。

`execute_edit_command` 可用分号合并多条数值赋值，例如 `startX = startX + 20;zRotate = 90`。同一命令中的表达式都以执行前的原字段值计算。`select_by_command` 使用 `/s` 语法：`;` 分隔 OR 规则组，`,` 分隔 AND 条件；数值范围写 `100~56000`，严格匹配用单引号，正则用尖括号，例如 `/s startTime:"100~56000",text:"<^sample>"`。筛选字段名见 `about.vue` 的“高级工具 / 命令工具”。

## 渲染规则

- 坐标是画布像素。弹幕在 `startTime <= 当前时间 <= startTime + animation.duration` 时属于可见时间范围；`duration` 是生存总时长。
- 运动进度基于 `elapsed = 当前时间 - startTime`。若 `moveDuration <= 0`，运动进度立即为 1；若 `elapsed <= delay`，进度为 0；其余情况使用 `clamp((elapsed - delay) / moveDuration, 0, 1)`。
- 缓动后的位置在起终点间插值：`x = start.x + (end.x - start.x) * progress`，Y 同理。`speedup` 为 `progress²`，`speeddown` 为 `1 - (1 - progress)²`。运动结束后位置停在终点，弹幕仍可在剩余生存时间内显示。
- 透明度按生存进度线性插值：`lifeProgress = clamp(elapsed / duration, 0, 1)`，`opacity = from + (to - from) * lifeProgress`。透明度进度从 `startTime` 开始，不受运动 `delay` 影响。
- `content.size` 是像素字号，`content.color` 是文字颜色，`content.stroke` 控制描边；`transform.zRotate` 为 Z 轴角度，Y 轴渲染使用 CSS `rotateY(360 - yRotate)`。
- 渲染排序先按 `startTime` 升序，同一开始时间再按 `layer` 升序。规范化会把字号限制在 10–127、透明度限制在 0–1、旋转角归一到 `[0, 360)`，并将数值字段取整；坐标、开始时间、运动时间和延迟是否允许负数取决于工程的“允许负值”设置。

`get_frame` 接收 `{ "timeMs": 1000, "maxDimension": 1280 }`，最长边范围 16–1920，默认 1280。它通过独立渲染窗口复用视频导出画面管线，返回 MCP 图片内容；含视频工程需有可访问的本地媒体文件。它不移动主界面播放头，也不修改工程。

## 工具索引

以下功能名对应 MCP 工具名；可选参数使用当前工具 schema 的默认值。

| 功能 | 工具 / 参数 | 用途 |
| --- | --- | --- |
| 状态与弹幕 | `get_editor_state({})`；`list_danmakus({selectedOnly?, ids?, offset?, limit?})` | 读取会话状态；分页读取当前选择或全部弹幕。 |
| 字段编辑 | `apply_field_update({field, method})` | 编辑当前选择字段；数字支持赋值、相对值、随机值和 `$` 表达式。 |
| 创建 | `create_danmakus({danmaku XOR jsonPath, relativeToCurrentTime})`；`create_generated_danmakus({quantity, rules, relativeToCurrentTime?})` | 直接创建对象或按规则生成。 |
| 删除与选择 | `delete_danmakus({ids?})`；`select_by_command({command})` | 删除 ID 列表或当前选择；用 `/s` 命令重置选择范围。 |
| 播放头与画布 | `update_current_time({currentTime})`；`update_editor_settings({screenWidth?, screenHeight?, maxLayers?})` | 时间单位毫秒；画布宽高至少 10，最大层数不能低于现有弹幕层级。 |
| 保存与备份 | `save_folder_project({})`；`backup_folder_project({})` | 保存当前文件夹工程，备份写入工程 `backup` 目录。 |
| 日志与历史 | `get_editor_logs({type?, offset?, limit?})`；`list_history_snapshots({})`；`get_history_snapshot({snapshotId, offset?, limit?})` | 读当前会话日志；先列快照再按 ID 分页读快照中的弹幕。 |
| 命令与描边 | `execute_edit_command({command})`；`apply_stroke({width, color})` | 运行工具栏数值赋值命令；生成描边副本。 |
| 坐标工具 | `center_danmakus({axis, scope?})`；`copy_coordinates({direction?})`；`swap_coordinates({})`；`mirror_danmakus({axis})` | 居中、复制或互换起终点坐标、按画布镜像。居中 `axis=x` 表示垂直居中，`axis=y` 表示水平居中；scope 为 `S` 起点、`E` 终点、`B` 两端。镜像轴为 `horizontal` 或 `vertical`。 |
| 运动计算 | `calculate_rotation({})`；`set_movement_length({length, angle?})`；`lock_movement_angle({changedAxis, angle?})` | 按坐标计算角度、按长度/角度设置终点、修改一轴时保持运动角。 |
| 分割与历史操作 | `split_danmakus({mode})`；`undo_edit({})`；`redo_edit({})` | 按 `line`、`letter` 或 `time` 分割；撤销/重做最近操作。字分割和居中需要界面测量能力。 |
| 视觉检查 | `get_frame({timeMs, maxDimension?})` | 获取指定时间的画面图片。 |

工具栏操作默认对当前选择执行。遇到测量失败时，居中或按字分割会整批取消，不会留下部分修改。
