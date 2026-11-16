# v2 字段参考

v2 是**以 `canvas` 为地基的全贴图皮肤**：九宫格面板、面板内安全内边距、行距、卡片外挂件、卡片外留白。只有 `schema_version = 2` 且 `base = "canvas"` 时这些字段才会被解析。

v1 的字段（配色、版式、背景图、装饰图、工具栏）在 v2 里同样可用，见 [schema-v1.md](schema-v1.md)。硬性范围与报错文案见 [validation.md](validation.md)，图片与动图规则见 [images.md](images.md)。完整模板见 [templates/skin-v2.toml](templates/skin-v2.toml)。

v2 字段全部写在 `[candidate_window]` 里：`line_gap_dip` 是直接键，其余是子表 / 数组表。

## 什么时候需要 v2

只有下面这些需求需要 v2，其余情况停在 v1：

- 整块面板贴图（九宫格拉伸 / 平铺），不是卡内一张背景图；
- 面板内的四边安全内边距；
- 预编辑行与候选行之间的行距；
- 挂到卡片外沿的立绘 / 挂件（可多层、可动图）；
- 卡片四边之外要预留的透明空白（给卡外挂件用）。

## 与 v1 的绑定规则

- `schema_version = 2` 必须配 `base = "canvas"`。`canvas` 是透明面板、无描边、直角、默认没有阴影的地基，卡片长什么样完全由贴图与挂件决定。
- 写了 `2` 但 `base` 是别的基底：包仍能装、能用，但**按 v1 加载**。`border_image`（含 `.horizontal` / `.vertical`）、`content_insets_dip`（含 `.horizontal` / `.vertical`）、`line_gap_dip`、`outer_insets_dip`（含 `.horizontal` / `.vertical`）、`decorations` 全部被忽略（连同其中的错误），设置页诊断里会多一条说明。
- `schema_version = 1` 配 `base = "canvas"` 会被直接拒：没有面板贴图，它只是一张透明卡片。
- `[animation]` 与装饰图的 `motion` / `motion_period_ms` 是 msime 各平台共用的 v2 键，对任何基底都合法；Windows 按静态图显示，不报错也不提示（见文末）。

`canvas` 已经帮你做好了透明面板、无描边、直角、默认没有阴影。想额外保险，可以在 `[candidate.dark]` / `[candidate.light]` 把 `surface` 给透明色、`corner_radius_dip = 0`、`border_width_dip = 0`；不想要预览里的投影，整段不写 `shadow` 即可（`canvas` 只在写了 `soft` / `strong` 时才画阴影）。

## `line_gap_dip`

| 键 | 范围 | 说明 |
|---|---|---|
| `line_gap_dip` | 0~100 的数字 | 预编辑行与候选行之间的距离。不写沿用默认值 |

皮肤自己写了 `preedit_margin_dip`（`[candidate_window]` 或它的 `.horizontal` / `.vertical` 子表）时以它为准：`line_gap_dip` 不再覆盖预编辑行的下边距，用户在设置页勾「行距」也不会把皮肤调好的位置顶回去。反过来说，想让行距跟着设置页走，就别写 `preedit_margin_dip`。

## `[candidate_window.border_image]`

面板整体贴图：把整块卡片背景换成这张图，按九宫格切片绘制。

| 键 | 必填 | 范围 | 说明 |
|---|---|---|---|
| `source` | 是 | 包内相对路径，`.png` / `.apng` / `.gif` / `.webp` | 面板图 |
| `light_source` | 否 | 同上 | 浅色主题换用的面板图；切片、边宽、`fill` 与 `source` 共用 |
| `slice_dip` | 是 | 1 / 2 / 4 个 0~500 的数值 | 切片线位置，**单位是源图像素** |
| `stretch_dip` | 否 | 1 / 2 / 4 个 0~500 的数值 | 可拉伸区，**源图像素**；给出时它才是可拉伸区 |
| `width_dip` | 否 | 1 / 2 / 4 个 0~500 的数值 | 四边绘制宽度，**单位是屏幕 DIP**；缺省用 `slice_dip` / `stretch_dip` 的值 |
| `fill` | 否 | 布尔，默认 `true` | 是否画九宫格中心块 |
| `repeat` | 否 | `stretch` / `repeat` / `round` / `space`，1 个或 2 个值 | 平铺方式；1 个值两轴相同，2 个值「横向 纵向」 |

顺序都是「上 右 下 左」；1 个值表示四边同值，2 个值表示「纵向（上/下）与横向（右/左）」。

几条语义：

- **九宫格切片**：`slice_dip` 的四条线把图切成九块，四角原样、四边单向拉伸、中心双向拉伸。单位是**源图像素**（96 DPI 下 1 源像素 = 1 DIP），与屏幕 DIP 的 `width_dip` 不是一回事。
- **图里画死的装饰要落在可拉伸区之外**：`stretch_dip` 比 `slice_dip` 直白——它是「可以随意拉伸的区域」。右下角画了一条帆船，就把右侧切片放到船身左边，否则船会被横着拉长。
- **`repeat` 决定四边怎么铺**，语义同 CSS `border-image-repeat`，两轴各自独立：
  - `stretch`（默认）：拉伸，边块被单向拉长；
  - `repeat`：按源图原尺寸一块块平铺，末尾超出面板的部分直接裁掉，所以花纹往往在边缘被切断；
  - `round`：平铺，但把边块等比缩放到「正好整数块」，边缘收口，不留半块也不裁切。花纹周期对不上时用它；
  - `space`：按原尺寸平铺，剩下的空间均分成块间空隙，同样收口；整段放不下两块时退回拉伸。
  写 1 个值两轴相同，写 2 个值是「横向 纵向」，例如 `["stretch", "round"]` 表示横向拉伸、纵向按整数块平铺。四角永远不铺。花纹边框（竖向候选窗两侧的铆钉、重复纹理）用 `round` / `space` 比 `repeat` 更整齐；一次性画死的装饰仍要靠 `stretch_dip` 留在可拉伸区之外，否则会被重复或拉长。
- **`fill = false` 会连中心区一起不画**（露出 `surface` 底色）；`fill = true`（默认）时中心区也按九宫格绘制。
- **`width_dip` 决定内容盒内缩量**，也是贴在屏上的边框厚度：切得越宽，内容盒收得越多（所以它要和贴图里的边框厚度对齐）。`width_dip` 不写就取切片的值当 DIP，给 2 倍图时写成切片的一半，200% 缩放下逐像素清晰、100% 下是缩小一半的同一张图。
- **`light_source` 给浅色主题换一张面板图**：切片、边宽、`fill` 与 `source` 共用，所以两张图要按同一套几何画；不写时深浅主题都用 `source`。

## `[candidate_window.border_image.horizontal]` / `.vertical`

横排 / 竖排各换一套面板图。两张子表都只覆盖写了的键，其余沿用上面共用那份；**两张子表地位相同、互不继承**——只写了 `.vertical` 时横排仍用共用那份，反过来也一样。

| 键 | 是否继承共用那份 |
|---|---|
| `source` | 是；不写就用共用的 `source` |
| `slice_dip` | 是 |
| `stretch_dip` | 是 |
| `width_dip` | 是 |
| `fill` | 是 |
| `repeat` | 是 |
| `light_source` | **跟着 `source` 走**：子表换了 `source` 又没写 `light_source` 时不会沿用共用那张浅色图（它是按另一张图的几何画的）；只改切片、边宽等几何时浅色图照常沿用 |

横竖排图尺寸不同时，`slice_dip` 一般要各写一份。子表必须是表；写了 `source` / `light_source` 就要按同一套规则校验，文件必须存在。

```toml
[candidate_window.border_image]
source = "assets/panel.png"
slice_dip = [16, 44, 16, 44]

[candidate_window.border_image.vertical]
source = "assets/panel-vertical.png"
slice_dip = [26, 10, 26, 10]
repeat = ["stretch", "round"]     # 横向拉伸，纵向按整数块平铺

[candidate_window.border_image.horizontal]
repeat = ["space", "stretch"]     # 只改横排：左右边按原尺寸平铺、空隙均分
```

## `[candidate_window.content_insets_dip]`

文字与内容的安全内边距，从面板外缘量起。

| 键 | 上限 | 说明 |
|---|---|---|
| `top` / `bottom` | ≤200 | 缺省 0 |
| `left` / `right` | ≤500 | 缺省 0 |
| `clamp_to_border` | 布尔，默认 `true` | `true` 时某一边小于九宫格该边的边宽就按边宽算；`false` 时完全按上面的数值 |

- 默认（`clamp_to_border = true`）下，某一边小于九宫格该边的边宽（`width_dip`）时按边宽算，文字不会压在边框上。
- 贴图的边宽里大半是黑底、真正画出来的边框更窄时（比如宽摆件只占左边一段圆角），写 `clamp_to_border = false` 就能把这块黑底也用上，省掉一段白留白。
- 整段不写时是「边宽 + 5 DIP」。
- **空表不等于不写**：只写了表头、四边都没给时，四边按 0 算，那 5 DIP 的默认留白也没了。
- 横排 / 竖排需要不同的边距时另写 `[candidate_window.content_insets_dip.horizontal]` / `.vertical`：只覆盖写了的边，其余沿用共用那份；两张子表互不继承，没写的排布直接用共用那份。`clamp_to_border` 也可以在这里再写一次，只影响那一种排布。

## `[candidate_window.outer_insets_dip]`

卡片四边之外要预留的透明空白，给卡片外的挂件用。四边都 ≤500，缺省 0。

窗口按卡片大小开，不声明外留白，挂到卡片外面的立绘会被窗口边界裁掉；挂在卡片内的挂件不需要它。

横排 / 竖排的挂件不同、需要预留的空白也不同时，另写 `[candidate_window.outer_insets_dip.horizontal]` / `.vertical`：只写要覆盖的边，其余沿用共用那份；写 `0` 可以取消共用那侧的预留。两张子表互不继承，没写的排布直接用共用那份。

```toml
[candidate_window.outer_insets_dip]
top = 95

[candidate_window.outer_insets_dip.vertical]
top = 0            # 竖排没有挂在卡片上方的挂件
```

## `[[candidate_window.decorations]]`

卡片外的自由挂件，数组表：想要几个就写几段，最多 32 层。

| 键 | 必填 | 范围 / 取值 | 说明 |
|---|---|---|---|
| `image` | 是 | 包内相对路径，`.png` / `.apng` / `.gif` / `.webp` | 挂件图 |
| `anchor` | 否 | 九个锚点，默认 `right` | 见下 |
| `width_dip` | 是 | >0 且 ≤2000 | 挂件框宽度 |
| `height_dip` | 是 | >0 且 ≤2000 | 挂件框高度 |
| `offset_x_dip` | 否 | -2000~2000，默认 0 | 对齐后再平移，正数向右 |
| `offset_y_dip` | 否 | -2000~2000，默认 0 | 对齐后再平移，正数向下 |
| `z_index` | 否 | -1000~1000 的整数，默认 2 | 叠放层级 |
| `layouts` | 否 | `["horizontal"]` / `["vertical"]`，可写两个 | 只在哪些排布里显示；不写则横竖都画 |

锚点：`top-left` / `top-right` / `bottom-left` / `bottom-right` / `left` / `right` / `top` / `bottom` / `center`。默认 `right` 是卡片右侧垂直居中。

- **挂件不吃点击**，只作装饰，也不参与布局。
- **面板自身相当于 `z_index = 1`**：`z_index` 小于 1 的挂件画在面板之下（只有伸出面板的部分看得见），其余画在面板与文字之上；同层按 `z_index` 从小到大、相同时按书写顺序叠放。
- 图片在框内 **contain 等比缩放、居中**，所以盒子最好与图片宽高比一致，比例差得多时四周会留白。
- `layouts` 只做筛选：不在当前排布里的挂件不画，也不参与最小宽度计算。横排挂右边、竖排挂下边这类需求就写两张挂件表，各自带上 `layouts`。
- v1 的 `[candidate_window.decoration]`（单数、顶部装饰图）与 v2 的 `[[candidate_window.decorations]]`（复数、卡片外挂件）是两套东西，可以同时用，`layouts` 的写法也一样。

## 与背景图、材质、卡片样式的互斥关系

- **`background.image` 和 `border_image` 是二选一**：写了面板图就不再画背景图（两者在同一处绘制）。
- **有面板图时，卡片自己的底色、描边、圆角和阴影都让给贴图**：写了 `shadow` 也不画，`surface` / `border` / `corner_radius_dip` 也不再决定外观。真正做成「面板即贴图」。
- **有面板图时 `backdrop` 不生效**：不再垫亚克力 / 云母材质。
- 要卡内底图就别写面板图；要系统材质也别写面板图。

## `[animation]` / 装饰图的 `motion`

`[animation]` 和装饰图的 `motion` / `motion_period_ms` 是 msime 各平台共用的 v2 键。Windows 目前按静态图显示：不报错、不提示，也不会动——写了也不能当 Windows 的动图开关。

Windows 上真正会动的是图片本身：把 v2 的图片路径指向 APNG / GIF / 动画 WebP 即可原生播放，具体规则（帧时长、帧数上限、64 MB 预算、系统关闭动画效果时的降级）见 [images.md](images.md#动图)。
