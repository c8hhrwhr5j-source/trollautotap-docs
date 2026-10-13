# TrollAutoTap Lua API 文档

**TrollAutoTap** 是基于 **Lua 5.4** 的 iOS 自动化脚本引擎（TrollStore 环境运行），提供**找色、找图、OCR 文字识别、触摸模拟、UI 自动化、HTTP 网络、文件系统**等完整的脚本能力。

## 目录

- 简介 —— 运行机制、目录结构、全局变量
- screen —— 屏幕与找色（findColor / findImage / OCR / snapshot）
- dotocr —— 点阵 OCR（模板字库）
- touch —— 触摸与手势
- script —— 脚本控制
- sys —— 系统与提示
- http —— 网络请求与下载
- device —— 设备控制
- app —— 应用管理
- appNode —— 本进程 UI 树节点
- uiacc —— 跨进程节点助手
- file —— 文件与目录
- str / json —— 字符串与 JSON
- pasteboard / key —— 前贴板与按键
- ui —— 脚本设置界面
- logWindow —— 浮动日志窗口
- storages —— 本地键值存储
- 附录 —— 全局函数速查表、常见问题

---

## 简介

### 目录结构

App 首次启动自动创建以下目录（TrollStore 沙盒外，稳定可写）：

```
/var/mobile/touch/
├── lua/            ← Lua 脚本目录（.lua / .tas / 项目文件夹）
├── log/            ← 日志目录（debug.log、snapshot_*.png）
├── res/            ← 资源目录（图片等）
└── storages/       ← 键值存储目录（storages 模块自动创建与读写）
```

### 运行方式

| 方式 | 说明 |
|---|---|
| 单文件脚本 | 将 `.lua` 文件放入 `/var/mobile/touch/lua/`，配置页点击运行；`.tas` 为加密脚本，运行时自动解密 |
| 文件夹项目 | 多个 `.lua` 与资源文件打包为文件夹项目运行，支持 `require()` 加载模块 |
| 音量键快速运行 | 设置中开启 TAS 服务后：空闲按音量键弹出运行确认，运行中按音量键弹出暂停/停止菜单 |

文件夹项目的入口文件查找顺序：`main.lua` → `init.lua` → `index.lua` → `app.lua` → 按字母排序的第一个 `.lua` 文件。

### 脚本停止机制

- 脚本调用 `mSleep` / `sleep` 时检查停止标志；
- 每 5000 条 Lua 指令执行一次停止钩子检查（死循环也能被中断）；
- 停止入口：App 内停止按钮、悬浮窗停止按钮、音量键菜单。

### 后台自动恢复

设备重启或进程被杀后，App 通过 iOS **SLC（Significant Location Changes）** 监听在后台自动重新拉起，TAS 服务与 8080 远程端口自动恢复，无需手动打开 App。SLC 为宿主系统级行为，不提供 Lua API。用户从主屏上划强杀 App 后，需手动打开一次 App 才能恢复自启。

---

## screen - 屏幕与找色

屏幕取色、找色、找图与文字识别。所有坐标均为**脚本坐标系**（由 `screen.init` 决定方向），颜色格式统一为 `0xRRGGBB`。

### findColor
<hr style="margin: 0;">

在屏幕上查找指定颜色，返回首个命中点坐标。

- `color` {number} 目标颜色 `0xRRGGBB`。
- `sim` {number} 相似度 0~1，默认 `0.9`。
- `x, y, w, h` {number} 查找区域左上角坐标与宽高。
- `rect` {table} 区域表 `{x=, y=, width=, height=}`。

**返回值**：找到返回 `x, y`；未找到返回 `nil`。

```lua
findColor(color)                       -- 全屏，sim = 0.9
findColor(color, sim)                  -- 全屏，指定相似度
findColor(color, x, y, w, h, sim)      -- 区域
findColor(color, rect, sim)            -- 区域（table 形式）

local x, y = findColor(0xFF0000, 0.85)
if x then tap(x, y) end

local x, y = findColor(0x00FF00, {x=100, y=200, width=300, height=400}, 0.9)
```

### findColors
<hr style="margin: 0;">

多点找色（偏移表形式）：以主色为基准，查找一组固定相对位置的颜色点，适合定位按钮、图标等组合特征。

- `mainColor` {number} 主色 `0xRRGGBB`。
- `offsets` {table} 偏移点数组，支持两种写法：`{dx=, dy=, color=}` 或 `{dx, dy, color}`。
- `sim` {number} 主色相似度，默认 `0.9`。
- `x, y, w, h` {number} 查找区域左上角坐标与宽高。
- `rect` {table} 区域表 `{x=, y=, width=, height=}`。
- `offSim` {number} 偏移点相似度，默认等于 `sim`。

**返回值**：找到返回主色点坐标 `x, y`；未找到返回 `nil`。

```lua
local offsets = {
    { dx = 10, dy = 0,  color = 0x00FF00 },
    { dx = 0,  dy = 10, color = 0x0000FF },
}

local x, y = findColors(0xFF0000, offsets, 0.9)                          -- 全屏
local x, y = findColors(0xFF0000, offsets, 100, 100, 200, 200, 0.9, 0.85) -- 区域 + 双相似度
local x, y = findColors(0xFF0000, offsets, {x=100, y=100, width=200, height=200}, 0.9)
```

### findColors（颜色模板字符串）
<hr style="margin: 0;">

多点找色（字符串形式）：用一行字符串同时描述区域、主色与所有偏移点，格式与 AutoGo `images.FindMultiColors` 一致。

- `x1, y1` {number} 查找区域左上角坐标。
- `x2, y2` {number} 查找区域右下角坐标，传 `0` 表示使用屏幕最大宽高。
- `colorsStr` {string} 颜色模板字符串：`"主色,偏移x,偏移y,颜色,偏移x,偏移y,颜色,..."`；第 1 个元素为主色（6 位 hex），之后每 3 个元素为一组偏移点（偏移可为负数）；颜色可带 `-偏色` 后缀，偏色被忽略、统一由 `sim` 控制。
- `sim` {number} 相似度 0.1~1.0，默认 `0.9`。

**返回值**：找到返回主色点坐标 `x, y`；未找到返回 `nil`。

```lua
-- "4a9a10,1,-1,429a10,2,-1,4a9e10" 解析为：
--   主色 4a9a10，偏移点 (1,-1) 颜色 429a10，偏移点 (2,-1) 颜色 4a9e10
local x, y = findColors(378, 547, 402, 569,
    "4a9a10,1,-1,429a10,2,-1,4a9e10,3,-1,4a9a10,4,-1,4aa608,5,-1,429a10", 0.9)
if x then tap(x, y) end

-- 右下角传 0 = 全屏
local x2, y2 = findColors(100, 200, 0, 0, "bd2c31,-10,13,732429,0,22,732421", 0.9)
```

### getColor
<hr style="margin: 0;">

获取屏幕指定点的颜色值。

- `x` {number} 屏幕横坐标。
- `y` {number} 屏幕纵坐标。

**返回值**：颜色值 `0xRRGGBB`；截屏失败返回 `0`。

```lua
local c = getColor(100, 100)
logStr(string.format("颜色 = 0x%06X", c))
```

### screen.getColorRGB
<hr style="margin: 0;">

获取屏幕指定点的 R、G、B 三个分量，省去脚本内位运算。

- `x` {number} 屏幕横坐标。
- `y` {number} 屏幕纵坐标。

**返回值**：`r, g, b` 三个 0~255 整数；截屏失败返回 `0, 0, 0`。与 `getColor(x, y)` 等价。

```lua
local r, g, b = screen.getColorRGB(100, 100)
logStr(string.format("R=%d G=%d B=%d", r, g, b))

if r > 200 and g < 60 and b < 60 then
    logStr("接近纯红色")
end
```

### findImage
<hr style="margin: 0;">

模板找图：在屏幕上查找指定图片（模板匹配），常用于定位按钮、图标、物品。

- `path` {string} 图片文件完整路径。
- `accuracy` {number} 相似度 0~1，默认 `0.8`。
- `x, y, w, h` {number} 查找区域左上角坐标与宽高。

**返回值**：找到返回图片中心点坐标 `x, y`；未找到返回 `nil`。

```lua
local x, y = findImage("/var/mobile/touch/res/button.png", 0.85)          -- 全屏
local x, y = findImage("/var/mobile/touch/res/icon.png", 0.85, 100, 100, 200, 200) -- 区域
local x, y = findImage("/var/mobile/touch/res/icon.png", 100, 100, 200, 200)       -- 区域（默认相似度）
if x then tap(x, y) end
```

### findText
<hr style="margin: 0;">

OCR 查找指定文字的位置，底层为 `screen.paddleOcr` 的封装。

- `text` {string} 要查找的文字。
- `x1, y1, x2, y2` {number} 可选，查找区域左上角 + 右下角，默认全屏。

**返回值**：找到返回文字框中心坐标 `x, y`（可直接用于 `tap`）；未找到返回 `nil`。

```lua
local x, y = findText("开始游戏")
if x then tap(x, y) end

local x, y = findText("附近", 0, 0, 1334, 400)   -- 区域查找
```

### screen.paddleOcr
<hr style="margin: 0;">

对屏幕全屏或指定区域做 OCR，返回所有识别到的文本块。底层基于 Apple Vision Framework，默认识别简体中文、繁体中文、英文。

- `x1, y1` {number} 可选，区域左上角，默认 `0, 0`。
- `x2, y2` {number} 可选，区域右下角，默认屏幕右下角。**注意：参数是对角点（左上 + 右下），不是宽高**。
- `color` {string} 可选，按字体颜色过滤后再识别，格式同找色：`"RRGGBB"` / `"#RRGGBB"` / `"RRGGBB-偏色"`。识别前将不在目标色 ± 容差内的像素涂白，只输出该色系文字。

**返回值**：数组，每个元素为 `{string, x, y, w, h, confidence}`：

| 字段 | 类型 | 说明 |
|---|---|---|
| `string` | string | 识别到的文本 |
| `x, y` | number | 文本框左上角坐标（脚本坐标系） |
| `w, h` | number | 文本框宽高 |
| `confidence` | number | 置信度 [0,1] |

调用 `screen.init(1)` 横屏后，区域参数与返回坐标均自动换算为横屏脚本坐标系，与 `tap` / `findColor` 一致，无需手动换算。

> 打印结果请使用 `print(...)`（自动连接所有参数）；`log()` 只显示第一个参数。

```lua
local result = screen.paddleOcr()                     -- 全屏
local result = screen.paddleOcr(126, 2, 281, 35)      -- 区域
local result = screen.paddleOcr(0, 0, 540, 300, "563A24-303030")  -- 按颜色过滤

for i, v in ipairs(result) do
    print(string.format("[%d] %s 框(%.0f,%.0f,%.0f,%.0f) 置信度%.3f",
        i, v.string or "", v.x or 0, v.y or 0, v.w or 0, v.h or 0, v.confidence or 0))
end

-- 找到某文字并点击其中心
for _, v in ipairs(result) do
    if v.string == "附近" then
        tap(v.x + v.w / 2, v.y + v.h / 2)
        break
    end
end
```

### screen.visionOcr
<hr style="margin: 0;">

多语言屏幕 OCR。与 `screen.paddleOcr` 结构完全一致（参数、返回值、遍历方式相同），区别是支持自定义识别语言。

- `lang` {string} 可选，识别语言代码，默认 `zh-Hans`。未支持的语言会被引擎自动忽略。
- `x1, y1` {number} 可选，区域左上角，默认 `0, 0`。
- `x2, y2` {number} 可选，区域右下角，默认屏幕右下角。

支持的语言代码：`en-US`、`fr-FR`、`it-IT`、`de-DE`、`es-ES`、`pt-BR`、`zh-Hans`、`zh-Hant`、`yue-Hans`、`yue-Hant`、`ko-KR`、`ja-JP`、`ru-RU`、`uk-UA`。

**返回值**：同 `screen.paddleOcr`。

```lua
local result = screen.visionOcr()                        -- 全屏（默认中文）
local result = screen.visionOcr(100, 100, 200, 200)      -- 区域
local result = screen.visionOcr("ko-KR", 100, 100, 200, 200)  -- 韩文
```

### snapshot
<hr style="margin: 0;">

保存当前屏幕截屏为 PNG。

- `path` {string} 可选，保存路径；省略时自动命名保存到 `/var/mobile/touch/log/snapshot_yyyyMMdd_HHmmss.png`。

**返回值**：成功返回保存的完整路径；失败返回 `nil`。

```lua
local path = snapshot("/var/mobile/touch/log/my_snap.png")
local p2 = snapshot()
```

### screen.keep / screen.unkeep
<hr style="margin: 0;">

缓存当前屏幕像素帧。缓存期间 `findColor` / `findColors` / `getColor` / `findImage` 全部复用该帧，不再重复截屏，画面静止的脚本性能显著提升。三种等价写法：`screen.keep()`、`keep()`、`keepScreen(true)`（取消：`screen.unkeep()`、`unkeep()`、`keepScreen(false)`）。

> 缓存期间画面被"冻结"——脚本读到的始终是开始保持那一刻的画面。屏幕内容变化后必须 `screen.unkeep()` 释放或重新 `screen.keep()` 刷新，否则会一直按旧画面找色。

```lua
while true do
    screen.keep()                         -- 保持当前画面
    local x, y = findColor(0x00FF00)
    if x then
        tap(x, y)
        screen.unkeep()                   -- 画面已变化，释放缓存
    end
    mSleep(500)
end
```

### getScreenSize
<hr style="margin: 0;">

获取屏幕尺寸。等价写法：`screen.getSize()` / `sys.screenSize()`。

**返回值**：`w, h`（逻辑分辨率，单位：点；如 iPhone 15 Pro Max 为 `430 x 932`）。引擎所有坐标均使用同一套点坐标，无需换算。

```lua
local w, h = getScreenSize()
logStr(string.format("屏幕 %.0f x %.0f", w, h))
```

### screen.init
<hr style="margin: 0;">

设置脚本坐标系方向。设置后 `tap` / `touchDown` / `swipe` / `findColor` / `findImage` / `getColor` 等所有坐标均按该方向解释，引擎自动旋转到设备当前实际方向后执行；`getScreenSize`、`findText`、`appNode` 等返回值也统一换算回脚本坐标系。

- `dir` {number} 方向：`0` = home 在下（竖屏，默认），`1` = home 在右，`2` = home 在左。

```lua
screen.init(1)   -- 横屏脚本（home 在右）
```

---

## dotocr - 点阵 OCR

模板字库点阵匹配识别文字：逐像素位图匹配，只认识字库中已有的字，速度快、不依赖模型、坐标精确，适合游戏等通用 OCR 难以处理的场景。字库格式与 nodeaid（AutoGoRunner）`libdotocr.dylib` 完全兼容。API 大小写均可（`dotocr.SetDict` / `dotocr.setDict`、`dotocr.Ocr` / `dotocr.ocr`）。

### dotocr.SetDict
<hr style="margin: 0;">

设置（或整体覆盖同名字库）点阵字库。字库常驻内存，脚本重启无需重新设置（除非 App 被杀）。

- `name` {string} 字库名称，为空字符串时使用 `"default"`。
- `dictContent` {string} 字库内容，按行分割，每行一条模板记录：`文字$hex点阵$占位$占位$宽.高`。

**返回值**：`true`。

字库行格式说明：

| 字段 | 说明 |
|---|---|
| 文字 | 识别命中后返回的标签，任意 UTF-8 文本（如 `"攻"`、`"HP"`） |
| hex 点阵 | 模板位图十六进制串，每 2 个 hex 字符 = 1 字节，每字节按**位**展开为 8 像素（LSB 位序在前，1 = 白 / 0 = 黑），行优先排布 |
| 第 3、4 字段 | 占位字段（解析时忽略，通常写 `0`） |
| 宽.高 | 模板尺寸，如 `8.12`；缺 `.` 时高度按 `位数/宽+1` 推算（兼容触动精灵字库 `文字$hex$1$0.0.xx$宽` 格式） |

`#` 开头为注释行，空行忽略，解析失败的行自动跳过，**行首空白（空格/Tab）自动剥离**（字库在 `[[ ]]` 长字符串中可自由缩进）。字库一般由取字工具从截图生成，或直接使用 nodeaid / 触动精灵生态的现成字库。

```lua
local dictContent = [[
# 我的字库
┌$e700$0$0$3.3
]]
dotocr.SetDict("游戏字库", dictContent)
```

### dotocr.Ocr
<hr style="margin: 0;">

在指定区域内用字库做点阵匹配识别。

- `x1, y1` {int} 区域左上角（脚本坐标系）。
- `x2, y2` {int} 区域右下角，为 `0` 时取屏幕最大宽高。
- `threshold` {string} 颜色串（见下表）。
- `sim` {float} 相似度 0.0~1.0；`sim > 0.95` 进入精确匹配模式（不容任何失配，速度最快），常用 `0.8`~`0.9`。
- `asJSON` {bool} `true` 返回 JSON 数组字符串，`false` 返回纯文本。
- `dictName` {string} 字库名称，为空时使用 `"default"`。
- `displayId` {int} 屏幕 ID，单屏设备固定传 `0`。

threshold 颜色串格式（"目标色±容差"语义）：

| 写法 | 含义 |
|---|---|
| `"ffffff-101010"` | 目标色 ffffff，容差每通道 ±16 |
| `"ffffff"` | 单色，默认容差 `0a0a0a`（每通道 ±10）；`""` / `nil` 等效 |
| `"a1b2c3-0a0f14\|d4e5f6-101010"` | `\|` 分隔多组颜色，任一命中即算前景 |
| `"auto"` | 灰度化 + Otsu 自动阈值二值化 |

**返回值**：

- 纯文本模式：按阅读顺序（先上后下、先左后右）直接拼接所有识别到的字符。
- JSON 模式：数组字符串，每个元素为 `{"x", "y", "width", "height", "text", "sim"}`，坐标为脚本坐标系。
- 屏幕抓取失败返回 `nil`。

```lua
dotocr.SetDict("游戏字库", file.read("/var/mobile/touch/res/my.dict"))

-- 纯文本识别
local text = dotocr.Ocr(100, 200, 400, 260, "ffffff-101010", 0.85, false, "游戏字库", 0)
if text == "开始战斗" then
    tap(250, 230)
end

-- JSON 模式：拿到每个字的坐标与相似度
local arr = json.decode(dotocr.Ocr(100, 200, 400, 260, "auto", 0.8, true, "游戏字库", 0))
for i, v in ipairs(arr) do
    print(i, v.text, v.x, v.y, v.width, v.height, v.sim)
end
```

---

## touch - 触摸与手势

多点触摸模拟，最多支持 10 根手指（索引 0~9）。

### tap
<hr style="margin: 0;">

模拟单击（内部为按下与抬起）。

- `x, y` {number} 点击坐标。
- `durationMs` {number} 按压时长（毫秒），默认 `50`。
- `pressure` {number} 压力值 0~1，默认 `1.0`。
- `radius` {number} 触摸半径，默认 `0`。

```lua
tap(x, y)                 -- 单击
tap(x, y, 200)            -- 按压 200ms（长按）
tap(x, y, 200, 1.0, 5)    -- 完整参数
```

### touchDown / touchMove / touchUp
<hr style="margin: 0;">

多点触摸低层接口。模块形式 `touch.down` / `touch.move` / `touch.up` 与全局形式等价。

- `index` {number} 手指索引 0~9。
- `x, y` {number} 坐标。
- `pressure` {number} 可选，压力值（仅 down / move），默认 `1.0`。
- `radius` {number} 可选，触摸半径（仅 down / move），默认 `0`。

```lua
-- 双指操作
touchDown(0, 100, 200)
touchDown(1, 300, 400)
touchMove(0, 120, 220)
touchMove(1, 280, 380)
touchUp(0, 120, 220)
touchUp(1, 280, 380)
```

### swipe
<hr style="margin: 0;">

模拟直线滑动。

- `x1, y1` {number} 起点坐标。
- `x2, y2` {number} 终点坐标。
- `duration` {number} 时长（毫秒），默认 `300`。
- `steps` {number} 采样步数，省略时按 `时长(秒)×60` 自动计算（约 60Hz）。
- `pressure` {number} 可选，压力值，默认 `1.0`。
- `radius` {number} 可选，触摸半径，默认 `0`。

```lua
swipe(160, 300, 160, 100, 500)        -- 向上滑 500ms
swipe(100, 200, 200, 200, 200, 10)    -- 向右滑 200ms，10 步采样
```

### stroke
<hr style="margin: 0;">

多点轨迹滑动。

- `points` {table} 坐标点表 `{x1, y1, x2, y2, ...}`，偶数长度，最多 256 个点。
- `duration` {number} 总时长（秒），默认 `0.3`。

```lua
stroke({100, 200, 150, 250, 200, 200, 250, 150}, 0.8)
```

### touchStatus
<hr style="margin: 0;">

**返回值**：当前触摸状态的描述字符串，如 `"touches: 2, [0]=(100,200) down, [1]=(300,400) down"`。

```lua
logStr(touchStatus())
```

---

## script - 脚本控制

查询宿主脚本引擎自身状态（区别于 `app.isRunning(bid)`——后者查询被操作的目标 App 进程）。

### script.isRunning
<hr style="margin: 0;">

**返回值**：`boolean`，脚本是否正在运行。仅在脚本真正开始执行后为 `true`、真正退出后变回 `false`，与 UI 上的"运行中"状态同步。同步直读状态位，无锁无阻塞，可高频调用。

### script.isPaused
<hr style="margin: 0;">

**返回值**：`boolean`，脚本是否被暂停（音量键菜单 / 悬浮球 / HUD 暂停按钮触发）。用于在循环内精确判定暂停状态并等待恢复。

```lua
-- 等待恢复运行，单次上限 15 秒
function waitWhilePaused(deadline)
    while script.isPaused() and sys.mtime() < deadline do
        mSleep(200)
    end
end
```

### restartScript
<hr style="margin: 0;">

重新启动当前正在运行的脚本：调用后当前执行立即中止（后面的代码不会执行），脚本从头重新运行。支持单文件 `.lua`、文件夹项目、加密项目 `.tas`、网页远程下发代码。

> 注意：重启前需将持久数据写入磁盘（`file.addText` 等），全局变量与局部状态会全部丢失；用户主动"停止"优先于重启请求；请避免"一进循环就立刻重启"造成无限重启。

```lua
restartScript()
```

---

## sys - 系统与提示

### mSleep / sleep
<hr style="margin: 0;">

脚本延时，期间响应停止标志（可被用户中断）。

- `mSleep(ms)` {number} 延时毫秒数。
- `sleep(sec)` {number} 延时秒数（支持小数）。

```lua
mSleep(500)    -- 延时 500 毫秒
sleep(1.5)     -- 延时 1.5 秒
```

### logStr / print
<hr style="margin: 0;">

输出日志，写入 `/var/mobile/touch/log/debug.log` 并显示在日志面板。`logStr(s)` 接受单参数；`print(...)` 接受任意多参数并自动连接。`log()` 只显示第一个参数，多参数调试请使用 `print`。

```lua
logStr("这是一条日志")
print("结果数量:", 32, "耗时:", 1.2)
```

### sys.alert
<hr style="margin: 0;">

显示阻塞式提示框，等待用户操作。

- `msg` {string} 提示内容。
- `timeout` {number} 显示时间（秒），默认 `0`；`0` = 永久显示带"确定"按钮，`>0` = 自动消失（右上角实时倒计时）。
- `title` {string} 标题，默认 `"提示"`。

```lua
sys.alert("任务完成")
sys.alert("3 秒后消失", 3)
sys.alert("错误", 0, "错误提示")
```

### sys.alertButtons
<hr style="margin: 0;">

显示带多个按钮的提示框，阻塞等待用户点击。

- `msg` {string} 提示内容。
- `buttons` {table} 按钮文本数组，如 `{"确定", "取消"}`。
- `title` {string} 可选，标题。
- `timeout` {number} 可选，超时秒数（右上角倒计时）。

**返回值**：用户点击返回按钮文本（string）；超时未点击返回 `nil`。

```lua
local clicked = sys.alertButtons("选择操作", {"确定", "取消", "重试"})
local result = sys.alertButtons("继续?", {"是", "否"}, "确认", 10)
if result == nil then logStr("超时未点击") end
```

### sys.inputAlert
<hr style="margin: 0;">

显示带**文本输入框**的提示框，阻塞等待用户输入。仅在 TrollAutoTap 处于前台时可用（输入框依赖系统键盘）；App 在后台（脚本运行于其它 App 之上）时立即返回 `nil`，不阻塞脚本。

- `msg` {string} 提示内容。
- `title` {string} 可选，标题，默认 `"提示"`。
- `defaultText` {string} 可选，输入框预填文本。
- `timeout` {number} 可选，超时秒数（超时自动关闭）。

**返回值**：用户点击"确定"返回输入文本（string）；点击"取消" / 超时 / App 不在前台返回 `nil`。

```lua
local times = sys.inputAlert("请输入要刷的次数", "任务设置", "10")
if times then
    print("本次运行:", tonumber(times) or 0, "次")
else
    logStr("用户取消或 App 不在前台，使用默认值")
end
```

### sys.toast
<hr style="margin: 0;">

在任意前台 App 之上短暂显示一条提示，**非阻塞**（调用后立即返回，脚本继续执行；由系统级 HUD 托管窗口显示，脚本在后台运行也能看到）。全局函数 `toast(...)` 完全等效。

- `msg` {string} 提示内容。
- `ms` {number} 显示时间（毫秒），默认 `1000`。
- `hidden` {boolean} `true` = 弱化模式（屏幕顶部小字），默认 `false`。

```lua
sys.toast("任务完成")
sys.toast("倒计时 3 秒", 3000)
sys.toast("正在找色...", 500, true)
```

### sys.setFloatBallPoint
<hr style="margin: 0;">

将悬浮球本体中心移动到指定坐标（未显示时自动显示后再移动）。坐标使用脚本坐标系，与 `tap` / `findColor` 同源并随 `screen.init` 方向自适应，无需手动换算。

- `x` {number} 脚本坐标系横坐标。
- `y` {number} 脚本坐标系纵坐标。

```lua
local w, h = getScreenSize()
sys.setFloatBallPoint(w / 2, h / 2)
```

### sys.mtime
<hr style="margin: 0;">

**返回值**：`number`，毫秒级 UTC 时间戳（自 1970-01-01 起的毫秒数）。

```lua
local t1 = sys.mtime()
-- ... 执行任务 ...
print(string.format("耗时: %.0f ms", sys.mtime() - t1))
```

### sys.availableMemory / sys.processUsedMemory / sys.usedMemory
<hr style="margin: 0;">

内存查询，均基于 `mach` API，无阻塞。

- `sys.availableMemory()` **返回值**：`number`，系统可用物理内存（字节）= `(free + inactive + speculative) × pageSize`。
- `sys.processUsedMemory()` **返回值**：`number`，本进程实际占用的物理内存（字节，resident_size）。
- `sys.usedMemory()` **返回值**：`number`，系统已用物理内存（字节）= `(active + wire) × pageSize`。

```lua
print(string.format("可用: %.2f MB, 进程占用: %.2f MB",
    sys.availableMemory() / 1048576, sys.processUsedMemory() / 1048576))
```

### sys.version
<hr style="margin: 0;">

**返回值**：`string`，App 版本号（CFBundleShortVersionString）。可用于功能探测与降级。

```lua
print("当前 App 版本: " .. sys.version())
```

### sys.palyAudio
<hr style="margin: 0;">

异步播放本地音频文件（不阻塞，可重复调用切换音频）。函数名保留原版拼写 `palyAudio` 以兼容旧脚本。

- `path` {string} 音频文件本地路径，支持 `.mp3` / `.wav` / `.m4a` 等系统原生格式。

**返回值**：`boolean`，失败（文件不存在、格式不支持）返回 `false`。

```lua
sys.palyAudio(file.resDir() .. "/alert.mp3")
```

### sys.isVPNConnected / sys.vpnState
<hr style="margin: 0;">

检测设备是否连接 VPN（WireGuard、IKEv2、代理类工具等），同步立即返回、不联网。

- `sys.isVPNConnected()` **返回值**：`boolean`，是否连接 VPN。直接读缓存，可高频调用。
- `sys.vpnState()` **返回值**：`table` 检测详情：

| 字段 | 类型 | 说明 |
|---|---|---|
| `connected` | boolean | 是否连接 VPN |
| `method` | string | 命中的判定层：`path`（Network.framework 路径存在 utun 隧道，权威判定）/ `utun`（getifaddrs 枚举兜底）/ `proxy`（系统全局代理兜底）/ `unknown`（监听尚未上报首帧） |
| `interface` | string | 隧道接口名（如 `utun4`），代理模式为空串 |

```lua
if not sys.isVPNConnected() then
    sys.toast("未连接 VPN，请先开启 VPN 再运行脚本")
    return
end

local st = sys.vpnState()
logStr(string.format("VPN=%s method=%s interface=%s",
    tostring(st.connected), st.method, st.interface))
```

### sys.info / sys.osVersion / sys.model / sys.getIP / sys.battery
<hr style="margin: 0;">

系统信息查询。

- `sys.info()` **返回值**：`table`，设备信息集合。
- `sys.osVersion()` **返回值**：`string`，系统版本。
- `sys.model()` **返回值**：`string`，设备型号。
- `sys.getIP()` **返回值**：`string`，WiFi IP。
- `sys.battery()` **返回值**：`number`，电量 0~1。

```lua
print(sys.osVersion(), sys.model(), sys.getIP(), sys.battery())
```

---

## http - 网络请求与下载

基于系统 `NSURLSession` 的同步阻塞 HTTP 模块，UTF-8 解码。三个函数均**不会对 URL 自动编码**，需自行处理。

### http.get
<hr style="margin: 0;">

发送 GET 请求。

- `url` {string} 完整 URL。
- `timeout` {number} 可选，超时秒数，默认 `60`。
- `headers` {table} 可选，请求头键值表（如 `{["User-Agent"]="...", ["Cookie"]="..."}`），省略使用默认 UA。

**返回值**：`状态码, 返回头, 内容` ——

| 字段 | 类型 | 说明 |
|---|---|---|
| 状态码 | integer | HTTP 响应码（如 200、404）；网络错误时为 `0` |
| 返回头 | table | 响应头键值表 `{Server="nginx", ...}` |
| 内容 | string | 响应体 UTF-8 字符串 |

```lua
local code, header, body = http.get("https://www.baidu.com", 30, {
    ["User-Agent"] = "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36"
})
if code == 200 then
    print(header)
    print(body)
end
```

### http.post
<hr style="margin: 0;">

发送 POST 请求。参数同 `http.get`，第四参为请求体：

- `body` {string} 可选，form-urlencoded 字符串（如 `"username=test&password=123"`）；无需显式设置 `Content-Type`，引擎自动加 `application/x-www-form-urlencoded`。

**返回值**：同 `http.get`。

```lua
local code, header, body = http.post("https://www.baidu.com", 30, {
    ["User-Agent"] = "Mozilla/5.0 ..."
}, "username=1237489&password=237348")
if code == 200 then
    print(body)
end
```

### http.download
<hr style="margin: 0;">

下载文件到本地路径。

- `url` {string} 文件 URL。
- `savePath` {string} 本地绝对路径，父目录不存在时自动创建。
- `timeout` {number} 可选，超时秒数，默认 `60`。
- `callback` {function} 可选，进度回调 `function(totalLength, currentLength, downloadSpeed)`，约每 200ms 触发一次，三个参数均为整数（字节），`downloadSpeed` 为瞬时速率。

**返回值**：`下载是否成功, 错误信息` ——

| 字段 | 类型 | 说明 |
|---|---|---|
| 下载是否成功 | boolean | 成功 `true`；网络错误、写文件失败、超时、URL 无效均 `false` |
| 错误信息 | string | 成功为空串；失败为可读简述（同时 NSLog 完整 NSError） |

模块行为说明：三函数均同步阻塞，脚本在网络完成前不继续执行；不自动跟随 3xx 重定向；HTTPS 走系统证书链；响应体按 UTF-8 解码（失败回退 ISO-8859-1）；请求体仅 form-urlencoded，JSON 需自行放入 `body` 并在 `headers` 覆盖 `Content-Type`。

```lua
local savePath = "/var/mobile/Media/svip/res/55.mp4"
local ok, err = http.download("http://example.com/video/5.mp4", savePath, 10,
    function(totalLength, currentLength, downloadSpeed)
        print(string.format("总大小:%.2fMB 已完成:%.2fMB 速度:%.2fMB/s",
            totalLength/1024/1024, currentLength/1024/1024, downloadSpeed/1024/1024))
    end)
if ok then
    print("下载成功: " .. savePath)
else
    print("下载失败: " .. tostring(err))
end
```

---

## device - 设备控制

### device.udid / device.serialNumber
<hr style="margin: 0;">

获取设备 UDID 与序列号。仅 TrollStore 安装的 App 可用（依赖 MobileGestalt 私有 API 与对应 entitlement）；沙盒 App Store 安装返回 `nil`。

**返回值**：`string / nil`。

```lua
local udid = device.udid()
local sn = device.serialNumber()
if udid then print("UDID: " .. udid) end
```

### device.turnOnAssistiveTouch / device.turnOffAssistiveTouch
<hr style="margin: 0;">

启用或停用 iOS 辅助触控（小白点）。仅 TrollStore 安装的 App 可用。异步执行（约 200ms 内生效）；写入后通过 Darwin 通知刷新系统，并在需要时重启 `assistivetouchd`。

**返回值**：`boolean`，`true` = 成功修改 plist 并已发送通知。

```lua
device.turnOnAssistiveTouch()
device.turnOffAssistiveTouch()
```

### device.isScreenLocked / device.unlockScreen
<hr style="margin: 0;">

查询屏幕锁定状态 / 唤醒并解锁屏幕。

- `device.isScreenLocked()` **返回值**：`boolean`，`true` = 锁定中（基于 Darwin 通知 `com.apple.springboard.lockstate` 查询）。
- `device.unlockScreen()` **返回值**：`boolean`，`true` = 复查确认已解锁；`false` = 解锁失败（设备设置了锁屏密码时必然失败，需用户在挂机前关闭密码；失败原因写入 `touch.log`）。

```lua
if device.isScreenLocked() then
    device.unlockScreen()
    sys.msleep(1000)
    print(device.isScreenLocked() and "解锁失败" or "已解锁")
end
```

### device.name / device.type
<hr style="margin: 0;">

**返回值**：`string` —— `device.name()` 为设备名；`device.type()` 为设备类型（`iPhone` / `iPad` / `TV` / `CarPlay` / `Mac` / `Unspecified`）。

```lua
print("设备名: " .. device.name())
print("设备类型: " .. device.type())
```

### device.backlightLevel / device.setBacklightLevel
<hr style="margin: 0;">

读取或设置屏幕亮度（基于 `UIScreen.brightness` 公开 API）。

- `device.backlightLevel()` **返回值**：`number`，当前亮度 [0, 1]。
- `device.setBacklightLevel(n)` `n` {number} ∈ [0, 1]。

```lua
device.setBacklightLevel(0.3)   -- 调暗省电
if device.backlightLevel() < 0.5 then
    device.setBacklightLevel(1.0)
end
```

### device.lockScreen / device.vibrator
<hr style="margin: 0;">

- `device.lockScreen()` 锁定屏幕（等同电源键）。
- `device.vibrator()` 触发系统震动反馈。

```lua
device.lockScreen()
device.vibrator()
```

### device.setVolume
<hr style="margin: 0;">

设置系统音量（通过临时 `MPVolumeView` 滑块实现）。异步执行，调用后约 200ms 内生效。

- `n` {number} ∈ [0, 1]。

```lua
device.setVolume(0.5)   -- 50%
device.setVolume(0.0)   -- 静音
```

---

## app - 应用管理

### app.frontBid
<hr style="margin: 0;">

获取当前前台 App 的 bundle id。引擎依次尝试三条通路（FrontBoard 显示布局 → SpringBoardServices 前台查询 → 本 App 自身），全部失败时按停在桌面处理。**恒返回字符串，不会是 `nil`**，可直接比较。

**返回值**：`string`，前台 App bundle id；停在桌面时为 `com.apple.springboard`。

```lua
local bid = app.frontBid()
local APP = "com.xxx.game"
if bid ~= APP then
    app.open(APP)
end
```

### app.isInstalled / app.isRunning
<hr style="margin: 0;">

- `app.isInstalled(bid)` **返回值**：`boolean`，指定 bundle id 的 App 是否已安装。
- `app.isRunning(bid)` **返回值**：`boolean`，指定 App 进程是否正在运行。

### app.open / app.close
<hr style="margin: 0;">

- `app.open(bid)` 打开指定 App。**返回值**：`boolean`。
- `app.close(bid)` 关闭指定 App。**返回值**：`boolean`。参数为 **bundle id**（如 `com.tencent.xin`），不是显示名；失败原因写入 `touch.log`（`未找到运行进程` = 未运行或 bundle id 不对；`Operation not permitted` = 无权发信号）。

```lua
app.open("com.xxx.game")
app.close("com.xxx.game")
```

### app.inputText
<hr style="margin: 0;">

向当前聚焦的输入框输入文本。

- `text` {string} 要输入的文本。

**返回值**：`boolean`。

```lua
app.inputText("hello")
```

---

## appNode - 本进程 UI 树节点

> 仅能遍历**本应用进程**的视图树（TrollStore App 以普通 App 身份运行，无法跨进程遍历其他 App 的 UI；跨进程需求请使用 uiacc 模块）。

### appNode.info
<hr style="margin: 0;">

**返回值**：`string`，完整视图树 JSON。

```lua
logStr(appNode.info())
```

### appNode.findByText
<hr style="margin: 0;">

按文本查找节点。

- `text` {string} 要查找的文本。

**返回值**：`table`，匹配节点列表，每个节点含 `class`（视图类名）、`text`（文本）、`centerX` / `centerY`（中心坐标）、`frame`（`{x, y, width, height}`）字段。

```lua
local nodes = appNode.findByText("开始游戏")
if nodes[1] then
    tap(nodes[1].centerX, nodes[1].centerY)
end
```

### appNode.tapByText
<hr style="margin: 0;">

点击第一个匹配文本的节点。

- `text` {string} 节点文本。

```lua
appNode.tapByText("确定")
```

### appNode.keep / appNode.unKeep
<hr style="margin: 0;">

缓存 / 释放当前视图树，避免多次查找时重复遍历。

```lua
appNode.keep()
local n1 = appNode.findByText("按钮1")
local n2 = appNode.findByText("按钮2")
appNode.unKeep()
```

---

## uiacc - 跨进程节点助手

跨进程读取**任意前台 App** 的控件树，按文本 / 描述 / 类名 / ID / 坐标查找节点并点击，1:1 复刻 AutoGoRunner「节点助手」。适合游戏、第三方 App 自动化（`appNode` 在这些场景取不到节点）。首次查询自动初始化（FKA 预热，约 0~3 秒）；AX 返回物理像素坐标，接口坐标一律为脚本坐标系，模块内部自动换算。

### uiacc.setup / uiacc.lastError
<hr style="margin: 0;">

- `uiacc.setup()` 手动初始化（通常无需调用，首次查询自动初始化）。
- `uiacc.lastError()` **返回值**：`string`，最近一次失败的原因。

### uiacc.new
<hr style="margin: 0;">

创建选择器对象（`uiacc.New()` 同义）。返回的 selector 支持链式调用，全部选择器如下（小写 = Lua 惯例，大写 = 原版兼容；`text` 系先按 label 匹配、未命中回退按 value 匹配）：

| 类别 | 方法 |
|---|---|
| 文本 | `text` / `textContains` / `textStartsWith` / `textEndsWith` / `textMatches`（正则） |
| 描述 | `desc` / `descContains` / `descStartsWith` / `descEndsWith` / `descMatches` |
| ID | `id` / `idContains` / `idStartsWith` / `idEndsWith` / `idMatches` |
| 类名 | `className` / `classNameContains` / `classNameStartsWith` / `classNameEndsWith` / `classNameMatches` |
| 坐标 | `bounds(l,t,r,b)` 精确 / `boundsInside(l,t,r,b)` 节点在范围内 / `boundsContains(l,t,r,b)` 节点包含范围 |
| 布尔 | `clickable(b)` / `selected(b)` / `enabled(b)` / `scrollable(b)` / `editable(b)` / `checked(b)` |
| 其它 | `index(i)` 父节点中的序号 |

终端方法：

| 方法 | 说明 |
|---|---|
| `findOnce()` | 第一个匹配，没有返回 `nil` |
| `find()` | 全部匹配，返回数组（可能为空表） |
| `waitFor(ms)` | 轮询等待出现（默认 3000ms），返回节点或 `nil` |
| `waitUntilGone(ms)` | 等待消失，返回 true / false |
| `click(text)` | 查找并点击：无参 = 点击当前条件首个匹配；带参 = 按文本点击 |

```lua
-- 等待"进入游戏"按钮出现并点击（最长 30 秒）
local btn = uiacc.new():textContains("进入游戏"):waitFor(30000)
if btn then
    btn:click()
else
    toast("超时未找到进入游戏按钮")
end
```

### UiObject 节点方法
<hr style="margin: 0;">

`findOnce()` / `waitFor()` 等返回的节点对象支持：

```lua
obj:click(durMs)          -- 点击（默认 50ms 按压）
obj:longClick(durMs)      -- 长按（默认 800ms）
obj.text / obj:GetText()  -- 文本
obj.value / obj:GetDesc() -- 描述
obj.id / obj:GetId()      -- ID
obj:GetClassName()        -- 类名
obj:GetBounds()           -- 返回 l, t, r, b 四值
obj:GetChildCount()       -- 子节点数
obj:GetIndex()            -- 在父节点中的序号
obj:toString()            -- 摘要字符串
obj.x, obj.y, obj.w, obj.h -- 节点框（脚本坐标系）
obj.cx, obj.cy            -- 中心点
```

### uiacc.click
<hr style="margin: 0;">

快捷写法：查找并点击，等价于 `uiacc.new():text(v):click()`。

- `text` {string} 目标文本。

```lua
uiacc.click("确定")
```

### uiacc.dump / uiacc.info
<hr style="margin: 0;">

调试接口。

- `uiacc.dump([x, y])` dump 指定点（逻辑坐标，缺省屏幕中心）元素的原始属性表（属性 ID → 值）。
- `uiacc.info([depth])` 前台 App 根节点树（嵌套 children，默认 12 层，递归深度上限默认 30，`uiacc.new(40)` 可提高）。

### uiacc 命令行命令
<hr style="margin: 0;">

命令行脚本（`.tas` / 命令行模式）可直接使用以下命令，与 Lua API 一一对应：

| 命令 | 等价 Lua | 说明 |
|---|---|---|
| `uiaccinfo [depth]` / `uinfo` | `uiacc.info(depth)` | 整树 JSON（默认 12 层，1~50） |
| `uiaccdump [x y]` / `udump` | `uiacc.dump(x,y)` | 原始属性表（缺省屏幕中心） |
| `uiaccfind <sel> <val>` / `ufind` | `...:sel(val):findOnce()` | 找首个并输出中心坐标（不点击） |
| `uiaccclick <sel> <val>` / `uclick` | `uiacc.click(val)` | 找首个并模拟点击 |
| `uiacclongclick <sel> <val>` / `ulongclick` | `obj:longClick()` | 找首个并长按 800ms |
| `uiaccwaitfor <sel> <val> [ms]` / `uwait` | `...:waitFor(ms)` | 轮询等待出现（默认 3000ms） |

支持的选择器（8 个）：`text` / `textContains` / `desc` / `descContains` / `id` / `idContains` / `className` / `classNameContains`。value 支持引号（如 `uiaccclick text "确定 按钮"`）。

```
uiaccclick text "确定"
uiaccwaitfor text "进入游戏" 30000
uiaccclick id loginBtn
uiaccfind textContains "游戏"
```

### uiacc HTTP 端点
<hr style="margin: 0;">

App 内置 HTTP 服务（默认 `http://<设备IP>:8080`，与网页配置同一服务）暴露节点查询端点，供 PC / 浏览器 / 中控对接：

| 端点 | 说明 |
|---|---|
| `GET /api/uiacc/ready` | AX 通道状态 + `lastError` |
| `GET /api/uiacc/dump?depth=N&x=..&y=..` | 指定点所在 App 的整棵节点树 JSON |
| `GET /api/uiacc/element?depth=N&x=..&y=..` | 指定点处元素及其子树 JSON |

参数：`depth`（递归深度 1~50，缺省 12）、`x` / `y`（逻辑点坐标，缺省屏幕中心）、`raw=1`（额外输出 `rawAttrs` 原始属性字典，体积大，调试用）。返回 JSON 中 `frame` 为**物理像素**（AX 原始值）；字符串字段为空时省略键。

```bash
curl http://192.168.1.20:8080/api/uiacc/ready
curl "http://192.168.1.20:8080/api/uiacc/dump?depth=8&x=300&y=400" | python -m json.tool
```

---

## file - 文件与目录

### file.read / file.write
<hr style="margin: 0;">

- `file.read(path)` **返回值**：`string / nil`，文件内容。
- `file.write(path, content)` **返回值**：`boolean`，写入成功与否。

```lua
file.write("/var/mobile/touch/log/test.txt", "hello world")
local content = file.read("/var/mobile/touch/log/test.txt")
```

### file.exists / file.delete
<hr style="margin: 0;">

- `file.exists(path)` **返回值**：`boolean`，文件是否存在。
- `file.delete(path)` **返回值**：`boolean`，删除成功与否。

### 目录路径函数
<hr style="margin: 0;">

| 函数 | 返回值 |
|---|---|
| `file.documentsDir()` | App Documents 目录 |
| `file.touchDir()` | `/var/mobile/touch` |
| `file.luaDir()` | `/var/mobile/touch/lua`（脚本目录） |
| `file.logDir()` | `/var/mobile/touch/log`（日志目录） |
| `file.resDir()` | `/var/mobile/touch/res`（资源目录） |
| `file.scriptDir()` | 当前脚本 / 项目所在目录 |

```lua
local imgPath = file.scriptDir() .. "/images/button.png"
local x, y = findImage(imgPath, 0.85)
```

### file.readImage
<hr style="margin: 0;">

读取图片的像素尺寸。

**返回值**：`w, h`（**物理像素**；注意与 `getScreenSize` 返回的逻辑分辨率不同）。

```lua
local w, h = file.readImage(file.resDir() .. "/button.png")
```

### file.addText
<hr style="margin: 0;">

追加文本到文件末尾（文件不存在则创建）。

- `path` {string} 文件路径。
- `text` {string} 追加内容。

**返回值**：`boolean`。

```lua
file.addText(file.logDir() .. "/run.log", os.date("[%Y-%m-%d %H:%M:%S] 任务完成\n"))
```

### file.size
<hr style="margin: 0;">

**返回值**：`number`，文件字节数；不存在返回 `-1`。

### file.list
<hr style="margin: 0;">

列出目录下所有条目（不含路径前缀，不递归）。

- `dirPath` {string} 目录路径。

**返回值**：`table` `{name1, name2, ...}`；失败返回 `nil`。

```lua
for i, name in ipairs(file.list(file.luaDir())) do
    print(i, name)
end
```

### file.md5
<hr style="margin: 0;">

**返回值**：`string`，文件 MD5（32 位十六进制小写）；失败返回 `nil`。

### file.getLines / file.lineCount / file.getLineText / file.resetLineText / file.insertLineText
<hr style="margin: 0;">

按行读写文件（1-based 索引）。行分隔符统一为 `\n`；`resetLineText` 在 `n > 总行数` 时不操作返回 `false`；`insertLineText` 在 `n > 总行数` 时自动追加到末尾。适合配置文件、日志索引等小文件（每次操作全量读写）。

- `file.getLines(path)` **返回值**：`table`，所有行；失败返回 `nil`。
- `file.lineCount(path)` **返回值**：`number`，总行数（失败为 `-1`）。
- `file.getLineText(path, n)` **返回值**：`string / nil`，第 `n` 行。
- `file.resetLineText(path, n, text)` **返回值**：`boolean`，替换第 `n` 行。
- `file.insertLineText(path, n, text)` **返回值**：`boolean`，在第 `n` 行前插入。

```lua
local path = file.scriptDir() .. "/config.txt"
local lines = file.getLines(path)
print("共 " .. #lines .. " 行")
file.resetLineText(path, 2, "新内容")
file.insertLineText(path, 1, "插入的首行")
```

---

## str / json - 字符串与 JSON

### str 字符串工具
<hr style="margin: 0;">

| 函数 | 说明 |
|---|---|
| `str.md5(s)` | MD5 摘要 |
| `str.sha1(s)` | SHA1 摘要 |
| `str.split(s, sep)` | 按分隔符拆分 → `{"a", "b", "c"}` |
| `str.trim(s)` | 去首尾空白 |
| `str.random(n)` | `n` 位随机字符串 |
| `str.urlEncode(s)` | URL 编码（`"a b c"` → `"a%20b%20c"`） |
| `str.urlDecode(s)` | URL 解码 |

### json 编解码
<hr style="margin: 0;">

- `json.encode(obj)` **返回值**：`string`，table 编码为 JSON 文本。
- `json.decode(s)` **返回值**：`table`，JSON 文本解码为 table。

```lua
local jsonText = json.encode({a = 1, b = {c = 2, d = "hello"}})
local obj = json.decode(jsonText)
print(obj.b.d)   -- hello
```

---

## pasteboard / key - 剪贴板与按键

### pasteboard
<hr style="margin: 0;">

- `pasteboard.get()` **返回值**：`string`，剪贴板内容。
- `pasteboard.set(s)` 写入剪贴板。

### key 物理按键
<hr style="margin: 0;">

| 函数 | 说明 |
|---|---|
| `key.pressHome()` | Home 键 |
| `key.pressLock()` | 锁屏键 |
| `key.pressVolumeUp()` | 音量 + |
| `key.pressVolumeDown()` | 音量 - |
| `key.inputText(s)` | 模拟键盘输入文本 |

```lua
pasteboard.set("新的内容")
key.pressHome()
```

---

## ui - 脚本设置界面

TrollAutoTap 提供两套并存的设置 UI，共用同一份 settings.json（`/var/mobile/touch/lua/<脚本名>.settings.json`）与同一个全局 `settings` 表：

| 维度 | HTML 网页 UI（`index.html`） | UIKit 原生 UI（`schema.lua`） |
|---|---|---|
| 文件路径 | `ui/<name>/index.html` | `ui/<name>/schema.lua` |
| 渲染方式 | WKWebView + 内嵌 HTTP 服务 | UITableView + UIKit 控件 |
| 后台渲染 | App 在后台时 WKWebView 内容无法提交 → 需切回前台 | 可在任意前台 App 之上直接显示（SBS 系统级承载） |
| 布局自由度 | 完全自由（HTML/CSS/JS） | 受 UIKit 控件约束 |
| 适用场景 | 复杂排版、富媒体 | 普通设置项、需要在游戏运行时弹出 |

### ui.open
<hr style="margin: 0;">

打开脚本设置界面。

- `name` {string} 脚本名（`ui/<name>/` 目录名）；也可直接传入 HTML 内容字符串。
- `type` {string} 可选，`"auto"`（默认，优先 `schema.lua`，缺则回退 `index.html`）/ `"html"` / `"native"`。
- `opts` {table} 可选，见下表。

**返回值**：`boolean`，`true` = 用户点了"保存运行"（`settings` 表已注入，可读 `settings.xxx`）；`false` = 取消或失败。

| opts 字段 | 类型 | 说明 |
|---|---|---|
| `orientation` | string | HUD 承载时表单方向：`"auto"`（默认，跟随脚本坐标系）/ `"portrait"` 强制竖屏 / `"landscape"` 强制横屏 |
| `autoCloseAfter` | number | 倒计时秒数，归零自动触发"运行"流程；默认 `0` 不自动关闭 |
| `headerTitle` | string | 导航栏下方渲染的顶部大标题（24pt bold 居中，自动缩放） |

```lua
local ran = ui.open("myScript")              -- auto 模式
if ran then
    logStr("[设置] autoStart=" .. tostring(settings.autoStart))
end

ui.openForm("myScript", schema, { orientation = "portrait", autoCloseAfter = 30 })
```

### ui.openForm
<hr style="margin: 0;">

动态 schema 方式：不写文件，直接用 Lua 表声明表单。参数与返回值同 `ui.open(name, "native", opts)`。

```lua
local ran = ui.openForm("myScript", {
    title = "我的脚本设置",
    sections = {
        {title = "基础", rows = {
            {type="switch", key="autoStart", label="自动启动", default=true},
            {type="slider",  key="speed",     label="速度", min=0.5, max=2.0, step=0.1, default=1.0},
        }},
    },
})
```

### schema.lua 行类型
<hr style="margin: 0;">

`schema.lua` 返回一个描述表单结构的 Lua 表。全部行类型：

| type | 必填字段 | 常用可选字段 | 控件 / 写入值类型 |
|---|---|---|---|
| `switch` | key, label | default | UISwitch → bool |
| `checkbox` | key, label | default | 色块复选框（App ≥ 1.1.0）→ bool |
| `checkGroup` | key, label, options | default（数组）, columns（默认 3） | 一行 N 个色块按钮（App ≥ 1.2.0）→ string 数组 |
| `stepper` | key, label, min, max | default, step | UIStepper → number |
| `slider` | key, label, min, max | default, step, format, showValue | UISlider → number |
| `segmented` | key, label, options | default | UISegmentedControl → string（超过 5 项被截断，改用 select） |
| `select` | key, label, options | default | 子表选择 → string |
| `text` | key, label | default, placeholder, keyboard, validator, validatorMessage | UITextField → string |
| `textLong` | key, label | default, placeholder | UITextView → string |
| `number` | key, label, min, max | default, step, keyboard | 数字键盘 UITextField → number |
| `date` | key, label | default（Unix 秒）, mode（`date`/`time`/`datetime`） | UIDatePicker → number（秒） |
| `duration` | key, label | default（秒）, min, max, unit | UIDatePicker.countDownTimer → number（秒） |
| `color` | key, label | default（`#RRGGBB`） | UIColorWell（iOS 14+）→ string |
| `multi` | key, label, options | default（数组） | 子表多选 → string 数组 |
| `action` | key, label | onTap（function） | UIButton，触发 Lua 回调 |
| `info` | text | — | 纯文字，不写 settings |

高级字段：`visibleWhen` + `visibleWhenValue` 依赖显示（某 key 的值等于指定值时才显示本行，比较为 `==`）；`validator`（`url` / `email` / `number` / `decimal`）+ `validatorMessage` 保存校验；`onTap` 回调参数为当前所有行 `currentValue` 的字典（回调内修改不回写 settings.json）。

```lua
-- /var/mobile/touch/lua/ui/myScript/schema.lua
return {
    title = "我的脚本设置",
    sections = {
        {
            title = "基础",
            rows = {
                {type="switch",    key="autoStart", label="自动启动", default=true},
                {type="checkGroup", key="taskSel", label="要做的任务",
                 options={"师门任务","宝图任务","捉鬼任务"}, columns=3,
                 default={"师门任务"}},
                {type="slider",    key="speed", label="运行速度",
                 min=0.5, max=2.0, step=0.1, default=1.0, format="%.1fx", showValue=true},
                {type="text",      key="webhook", label="通知地址",
                 placeholder="https://...", keyboard="url",
                 validator="url", validatorMessage="URL 必须以 http/https 开头",
                 visibleWhen="mode", visibleWhenValue="自定义"},
                {type="info",      text="提示: 启动后可在悬浮球暂停/恢复"},
            },
        },
    },
}
```

### HTML 设置页 JavaScript 桥
<hr style="margin: 0;">

把 `index.html` 放到 `ui/<脚本名>/` 目录，引擎通过内嵌 HTTP 服务加载。网页端可用接口：

| API | 说明 |
|---|---|
| `ts.save(obj)` | 保存配置到 settings.json，触发关闭并启动脚本 |
| `ts.cancel()` | 取消配置，关闭设置页并停止脚本，不启动 |
| `ts.get(key)` | 读当前已保存的值 |
| `GET /api/ui/settings/<脚本名>` | 读整个配置表（JSON） |
| `PUT /api/ui/settings/<脚本名>` | 写整个配置表 |

```html
<script>
    // 页面加载时填充已保存的值
    fetch('/api/ui/settings/myScript').then(r => r.json()).then(s => {
        if (s.username) document.getElementById('username').value = s.username;
    });
    function onSave() {
        ts.save({ username: document.getElementById('username').value });
    }
    function onCancel() { ts.cancel(); }
</script>
```

底部按钮行为：`取消`（红色）= 停止脚本、不写 settings.json、返回 `false`；`运行`（蓝色）= 写 settings.json + 注入 `settings` 表 + 启动脚本。用户随时可按音量键 / 悬浮球"停止"强制中断设置页。

---

## logWindow - 浮动日志窗口

在屏幕上显示一块实时日志面板，可配置颜色、字体、位置、透明度。

### logWindow.init
<hr style="margin: 0;">

创建并立即显示一个日志窗口。

- `x, y` {number} 窗口左上角坐标（脚本坐标系，随 `screen.init` 自动旋转；超出屏幕时自动夹回并记日志）。
- `w` {number} 可选，宽度，默认 `500`（注意：是宽高，不是右下角坐标）。
- `h` {number} 可选，高度，默认 `35`。
- `bgAlpha` {number} 可选，背景透明度，默认 `0.5`。
- `bgColor` {number} 可选，背景色 `0xRRGGBB`，默认 `0x000000`。
- `textColor` {number} 可选，字体色 `0xRRGGBB`，默认 `0x00ff00`。
- `fontSize` {number} 可选，字体尺寸，默认 `12`。
- `singleLine` {boolean} 可选，单行模式，默认 `false`；`true` 时每次 `addLog` 只显示最新一条（整体替换，不滚动），适合状态栏式提示。

**返回值**：日志窗口对象；失败返回 `nil`。

```lua
local w, h = getScreenSize()
local lw = logWindow.init(0, 200, w, h - 400, 0.5, 0x000000, 0x00ff00, 15)
lw:addLog("第一条日志")
lw:addLog("红色日志", 0xff0000)
lw:addLog("大号字", 0xffff00, 20)
sleep(10)
lw:release()
```

### 日志窗口对象方法
<hr style="margin: 0;">

| 方法 | 说明 |
|---|---|
| `lw:addLog(text [, color, size])` | 追加一行日志（可指定颜色 `0xRRGGBB` 与字号） |
| `lw:show()` | 显示被隐藏的窗口（不重建，位置与内容保留） |
| `lw:hide()` | 临时隐藏窗口（不销毁；隐藏期间 `isVisible()` 返回 `false`） |
| `lw:isVisible()` | 窗口存在且未隐藏返回 `true`，否则 `false` |
| `lw:release()` | 释放（销毁）该窗口 |

```lua
local lw = logWindow.init(0, 0, 300, 30, 0.6, 0x000000, 0x00ff00, 14, true)
lw:addLog("运行中...")
lw:hide()           -- 暂时收起（如截图留档时）
sleep(2)
if not lw:isVisible() then
    lw:show()       -- 重新显示，位置内容不变
end
lw:release()
```

### logWindow.setHideWindowMode
<hr style="margin: 0;">

设置隐藏模式，影响后续创建的所有窗口。

- `hidden` {boolean} `true`（推荐）：面板肉眼可见但**不会被截进画面**——每次截屏时面板被临时摘除，`findColor` / `findImage` / `getColor` / `snapshot` 拿到的画面不含该面板；`false`（默认）：面板被一起截进画面。

```lua
logWindow.setHideWindowMode(true)
```

### logWindow.releaseAll
<hr style="margin: 0;">

一次关闭所有日志窗口。脚本结束（正常结束 / 停止 / 报错）时引擎自动调用，不会留残影。

补充说明：所有日志窗口挂在同一个全屏透明 UIWindow 上，`userInteractionEnabled=NO`，完全不参与触摸；App 退到后台时走 SBS 系统级托管（与悬浮球同一机制），在其它 App / 桌面上依旧可见；面板内部最多保留最近 400 行。

---

## storages - 本地键值存储

跨脚本运行与设备重启持久保存的键值存储，采用"表（table）+ 键（key）"两级命名空间，自动落盘为 `/var/mobile/touch/storages/<表名>.json`。适合记录任务进度、每日次数限制、累计统计等场景，无需手写文件与 JSON。值统一按**字符串**存储（`put` 支持 string / number / boolean），取出数字请用 `tonumber` 还原。

### storages.put
<hr style="margin: 0;">

写入一条数据并立即落盘。

- `table` {string} 表名（自动清洗：仅保留字母 / 数字 / `_` / `-`，其余字符替换为 `_`）。
- `key` {string} 键名。
- `value` {string|number|boolean} 值，落盘时统一转为字符串。

**返回值**：`boolean`，是否保存成功。

```lua
storages.put("task", "todayCount", 5)        -- 数字自动转 "5"
storages.put("task", "userName", "玩家一")
storages.put("task", "vip", true)            -- 布尔转 "true"
```

### storages.get
<hr style="margin: 0;">

读取一条数据。

- `table` {string} 表名。
- `key` {string} 键名。

**返回值**：`string`；键不存在返回 `nil`。

```lua
local count = tonumber(storages.get("task", "todayCount") or "0")
```

### storages.remove
<hr style="margin: 0;">

删除一条数据。

- `table` {string} 表名。
- `key` {string} 键名。

**返回值**：`boolean`，键原本存在且已删除返回 `true`。

```lua
storages.remove("task", "todayCount")
```

### storages.contains
<hr style="margin: 0;">

查询键是否存在。

- `table` {string} 表名。
- `key` {string} 键名。

**返回值**：`boolean`。

```lua
if storages.contains("task", "todayCount") then
    print("今天已有进度记录")
end
```

### storages.getAll
<hr style="margin: 0;">

读取整张表。

- `table` {string} 表名。

**返回值**：`table`（键→字符串值），表不存在时返回空表。

```lua
local all = storages.getAll("task")
for k, v in pairs(all) do
    print(k, "=", v)
end
```

### storages.clear
<hr style="margin: 0;">

清空整张表（保留空表文件）。

- `table` {string} 表名。

**返回值**：`boolean`，是否成功。

```lua
-- 每日首次运行时清零
local day = os.date("%Y-%m-%d")
if storages.get("task", "lastDay") ~= day then
    storages.clear("task")
    storages.put("task", "lastDay", day)
end
storages.put("task", "todayCount", (tonumber(storages.get("task", "todayCount") or "0") + 1))
```

---

## 附录 A：全局变量

| 变量 | 说明 |
|---|---|
| `_SCRIPT_PATH_` | 当前脚本 / 入口文件的完整路径 |
| `_SCRIPT_DIR_` | 项目目录完整路径（仅项目运行时存在） |
| `_PROJECT_DIR_` | 项目目录完整路径（同 `_SCRIPT_DIR_`） |
| `settings` | 设置界面注入的配置表（来源于 `<script>.settings.json`） |

项目运行时 Lua `package.path` 自动包含 `项目目录/?.lua;项目目录/?/init.lua`，可直接 `require('module')` 加载项目内其他 Lua 文件。运行脚本时引擎按"设备目录 → 脚本同目录"的顺序查找同名 `<script>.settings.json` 并注入为 `settings` 表。

## 附录 B：全局函数速查表

### 找色找图

| 函数 | 说明 |
|---|---|
| `findColor(color[, sim])` | 全屏单点找色 → x,y / nil |
| `findColor(color, x,y,w,h[, sim])` | 区域单点找色 |
| `findColor(color, rect[, sim])` | 区域单点找色（table） |
| `findColors(mainColor, offsets[, sim])` | 全屏多点找色（偏移表） → x,y / nil |
| `findColors(mainColor, offsets, x,y,w,h[, sim][, offSim])` | 区域多点找色 |
| `findColors(mainColor, offsets, rect[, sim])` | 区域多点找色（table） |
| `findColors(x1,y1,x2,y2, colorsStr[, sim])` | 多点找色（颜色模板字符串） → x,y / nil |
| `findImage(path[, accuracy])` | 全屏找图 → x,y / nil |
| `findImage(path, accuracy, x,y,w,h)` | 区域找图 |
| `findImage(path, x,y,w,h)` | 区域找图（省略 accuracy） |
| `getColor(x, y)` | 取色 → 0xRRGGBB |
| `screen.getColorRGB(x, y)` | 取色 RGB 分量 → r, g, b（0~255） |
| `findText(text)` | OCR 找文字 → x,y / nil |
| `screen.paddleOcr([x1,y1,x2,y2][, color])` | 屏幕 OCR（默认中英文） → 文本数组 |
| `screen.visionOcr([lang][,x1,y1,x2,y2])` | 屏幕 OCR（多语言） → 文本数组 |
| `dotocr.SetDict(name, dict)` | 设置点阵字库 → true |
| `dotocr.Ocr(x1,y1,x2,y2, threshold, sim, asJSON[, dictName, displayId])` | 点阵识别 → 文本 / JSON 字符串 / nil |

### 截屏与缓存

| 函数 | 说明 |
|---|---|
| `snapshot([path])` | 保存截屏 → 路径 / nil |
| `screen.keep()` / `keep()` / `keepScreen(true)` | 保持屏幕（缓存当前帧） |
| `screen.unkeep()` / `unkeep()` / `keepScreen(false)` | 取消保持 |

### 触摸与手势

| 函数 | 说明 |
|---|---|
| `tap(x, y[, dur][, pressure][, radius])` | 点击 |
| `touchDown(i, x, y[, pressure][, radius])` | 手指按下 |
| `touchMove(i, x, y[, pressure][, radius])` | 手指移动 |
| `touchUp(i, x, y)` | 手指抬起 |
| `swipe(x1,y1,x2,y2[, dur][, steps][, pressure][, radius])` | 滑动 |
| `stroke({x1,y1,...}[, dur])` | 多点轨迹 |
| `touchStatus()` | 触摸状态描述 |

### 延时与日志

| 函数 | 说明 |
|---|---|
| `mSleep(ms)` | 延时毫秒 |
| `sleep(sec)` | 延时秒 |
| `logStr(s)` / `print(...)` | 日志输出 |
| `toast(msg[, ms][, hidden])` / `sys.toast(...)` | 屏幕悬浮提示（非阻塞） |
| `sys.alert(msg[, timeout][, title])` | 阻塞弹窗 |
| `sys.alertButtons(msg, {btns}[, title][, timeout])` | 带按钮弹窗 → 按钮文本 / nil |
| `sys.inputAlert(msg[, title][, defaultText][, timeout])` | 带输入框弹窗 → 输入文本 / nil（仅前台可用） |
| `sys.setFloatBallPoint(x, y)` | 移动悬浮球（脚本坐标系，中心点） |
| `restartScript()` | 重启当前脚本（后面的代码不会执行） |

### 屏幕与方向

| 函数 | 说明 |
|---|---|
| `getScreenSize()` / `screen.getSize()` / `sys.screenSize()` | 屏幕尺寸 → w,h（逻辑分辨率） |
| `screen.init(dir)` | 设置脚本坐标系方向（0/1/2） |

### 系统信息

| 函数 | 说明 |
|---|---|
| `sys.info()` | 设备信息 → table |
| `sys.osVersion()` | 系统版本 → string |
| `sys.model()` | 设备型号 → string |
| `sys.getIP()` | WiFi IP → string |
| `sys.battery()` | 电量 0~1 → number |
| `sys.mtime()` | 毫秒级时间戳 → number |
| `sys.availableMemory()` | 系统可用内存（字节） → number |
| `sys.processUsedMemory()` | 进程内存（字节） → number |
| `sys.usedMemory()` | 系统已用内存（字节） → number |
| `sys.version()` | App 版本 → string |
| `sys.palyAudio(path)` | 播放音频文件 → boolean |
| `sys.isVPNConnected()` | 是否连接 VPN → boolean |
| `sys.vpnState()` | VPN 检测详情 → table（connected / method / interface） |

### HTTP / 下载

| 函数 | 说明 |
|---|---|
| `http.get(url, [timeout], [headers])` | HTTP GET → 状态码, 返回头, 内容 |
| `http.post(url, [timeout], [headers], [body])` | HTTP POST（form-urlencoded） → 状态码, 返回头, 内容 |
| `http.download(url, savePath, [timeout], [callback])` | HTTP 下载 → 是否成功, 错误信息；callback(total, current, speed) |

### 设备与控制

| 函数 | 说明 |
|---|---|
| `device.udid()` | 设备 UDID → string / nil |
| `device.serialNumber()` | 设备序列号 → string / nil |
| `device.turnOnAssistiveTouch()` | 启用辅助触控 → boolean |
| `device.turnOffAssistiveTouch()` | 停用辅助触控 → boolean |
| `device.isScreenLocked()` | 屏幕是否锁定 → boolean |
| `device.unlockScreen()` | 唤醒 + 解锁屏幕 → boolean |
| `device.name()` | 设备名 → string |
| `device.type()` | 设备类型 → string（iPhone/iPad/...） |
| `device.backlightLevel()` | 屏幕亮度 [0,1] → number |
| `device.setBacklightLevel(n)` | 设置屏幕亮度 |
| `device.lockScreen()` | 锁定屏幕 |
| `device.vibrator()` | 系统震动 |
| `device.setVolume(n)` | 设置系统音量 [0,1] |

### 应用与脚本状态

| 函数 | 说明 |
|---|---|
| `app.frontBid()` | 前台 App bundle id → string（桌面时为 `com.apple.springboard`，恒有值） |
| `app.isInstalled(bid)` | 是否安装 → boolean |
| `app.isRunning(bid)` | 目标 App 是否在运行 → boolean |
| `app.open(bid)` | 打开 App → boolean |
| `app.close(bid)` | 关闭 App → boolean |
| `app.inputText(text)` | 输入文本 → boolean |
| `script.isRunning()` | 本引擎脚本是否在运行 → boolean |
| `script.isPaused()` | 脚本是否被暂停 → boolean |

### UI 树节点与 uiacc

| 函数 | 说明 |
|---|---|
| `appNode.info()` | 完整视图树 JSON → string |
| `appNode.findByText(text)` | 按文本查找节点 → 节点列表 |
| `appNode.tapByText(text)` | 点击文本节点 |
| `appNode.keep()` / `appNode.unKeep()` | 缓存 / 释放视图树 |
| `uiacc.new()` | 创建跨进程节点选择器 |
| `uiacc.click(text)` | 查找并点击（快捷写法） |
| `uiacc.setup()` / `uiacc.lastError()` | 初始化 / 最近错误 |
| `uiacc.dump([x,y])` / `uiacc.info([depth])` | 调试接口 |

### 文件与目录

| 函数 | 说明 |
|---|---|
| `file.read(path)` | 读文件 → string / nil |
| `file.write(path, content)` | 写文件 → boolean |
| `file.exists(path)` | 是否存在 → boolean |
| `file.delete(path)` | 删除 → boolean |
| `file.documentsDir()` | App Documents 目录 |
| `file.touchDir()` | `/var/mobile/touch` |
| `file.luaDir()` | `/var/mobile/touch/lua` |
| `file.logDir()` | `/var/mobile/touch/log` |
| `file.resDir()` | `/var/mobile/touch/res` |
| `file.scriptDir()` | 当前脚本 / 项目目录 |
| `file.readImage(path)` | 图片尺寸 → w,h（物理像素） |
| `file.addText(path, text)` | 追加文本 → boolean |
| `file.size(path)` | 文件大小（字节） → number / -1 |
| `file.list(path)` | 目录列表 → table / nil |
| `file.md5(path)` | 文件 MD5 → string / nil |
| `file.getLines(path)` | 所有行 → table / nil |
| `file.lineCount(path)` | 总行数 → number |
| `file.getLineText(path, n)` | 第 n 行 → string / nil |
| `file.resetLineText(path, n, text)` | 替换第 n 行 → boolean |
| `file.insertLineText(path, n, text)` | 插入到第 n 行前 → boolean |

### 键值存储

| 函数 | 说明 |
|---|---|
| `storages.put(table, key, value)` | 写入键值（string/number/boolean） → boolean |
| `storages.get(table, key)` | 读取键值 → string / nil |
| `storages.remove(table, key)` | 删除键 → boolean |
| `storages.contains(table, key)` | 键是否存在 → boolean |
| `storages.getAll(table)` | 整张表 → table |
| `storages.clear(table)` | 清空表 → boolean |

### 字符串 / JSON / 剪贴板 / 按键 / UI

| 函数 | 说明 |
|---|---|
| `str.md5(s)` / `str.sha1(s)` | 摘要 |
| `str.split(s, sep)` / `str.trim(s)` / `str.random(n)` | 拆分 / 去空白 / 随机串 |
| `str.urlEncode(s)` / `str.urlDecode(s)` | URL 编解码 |
| `json.encode(obj)` / `json.decode(s)` | JSON 编解码 |
| `pasteboard.get()` / `pasteboard.set(s)` | 剪贴板读写 |
| `key.pressHome()` / `key.pressLock()` | Home / 锁屏键 |
| `key.pressVolumeUp()` / `key.pressVolumeDown()` | 音量键 |
| `key.inputText(s)` | 模拟键盘输入 |
| `ui.open(name[, type[, opts]])` | 打开设置界面 → boolean |
| `ui.openForm(name, schema[, opts])` | 动态 schema 表单 → boolean |
| `logWindow.init(...)` | 创建浮动日志窗口 → 对象 / nil |
| `logWindow.setHideWindowMode(b)` | 日志窗口隐藏模式 |
| `logWindow.releaseAll()` | 关闭所有日志窗口 |
| `lw:addLog(text [, color, size])` | 追加一行日志 |
| `lw:show()` / `lw:hide()` / `lw:isVisible()` | 日志窗口显示 / 隐藏 / 可见性查询 |
| `lw:release()` | 释放日志窗口 |

### 模块列表

| 模块 | 说明 |
|---|---|
| `touch` | 触摸手势 |
| `screen` | 屏幕相关 |
| `sys` / `device` | 系统信息与设备控制 |
| `app` | 应用管理 |
| `appNode` | 本进程 UI 树节点 |
| `uiacc` | 跨进程节点助手 |
| `http` | HTTP 请求与下载 |
| `dotocr` | 点阵 OCR |
| `json` / `str` | JSON 与字符串工具 |
| `file` | 文件操作 |
| `storages` | 本地键值存储 |
| `pasteboard` / `key` | 剪贴板与按键 |
| `ui` / `logWindow` | 设置界面与日志窗口 |

## 附录 C：常见问题

| 问题 | 解决 |
|---|---|
| `attempt to call a nil value` | 函数名拼错，或该函数未注册 |
| `findColor` 找不到 | 降低相似度（如 0.7~0.8）；确认颜色格式是 `0xRRGGBB` |
| 颜色取不到 | 用 `getColor(x, y)` 确认目标点颜色，再写入脚本 |
| 脚本卡死 | 点击 App 内"停止"按钮；脚本应避免 `while true do end` 无延时循环 |
| 项目 `require` 失败 | 确认项目以文件夹形式运行（不是单文件）；模块文件后缀必须是 `.lua` |
| 横屏坐标错乱 | 在脚本开头调用 `screen.init(1)` 或 `screen.init(2)` |
| `screen.keep()` 后找不到目标 | 缓存的是旧画面，画面变化后需 `screen.unkeep()` 释放再重新 keep |
| OCR 结果打印为空 | `log()` 只显示第一个参数，改用 `print(...)` |
| 区域 OCR 识别不到 | 参数是对角点 `(x1,y1,x2,y2)` 不是宽高；先全屏 OCR 打印坐标再圈区域 |
