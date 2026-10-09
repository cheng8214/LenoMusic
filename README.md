# LenoMusic — 音乐下载器（LenoLang 版）

酷我音源的音乐搜索 / 试听 / 下载工具，**GUI 与 CLI 两个入口共用同一套引擎**。

> 本仓库是从 [LenoLang](https://github.com/cheng8214/LenoLang) 的
> `leno_gui/应用/音乐下载器/` 用 `git subtree` 切出来的**镜像**：
> 源码与提交历史都在，但**开发仍在主仓库进行**，这里只做同步。
> 提 issue / PR 请到主仓库。

> **只想直接用？** 到 [Releases](../../releases) 下载 `LenoMusic.exe` —— 单文件，
> 图标 / 原生库 / 资源全内嵌，目标机器**不需要装 LenoLang**（也不需要装 SDL3），
> 双击即用（首次运行会自动把依赖解包到用户缓存目录）。
> 下面是"从源码跑"与"自己打包"的方式。

## 一、先看这段：运行需要 LenoLang 运行时

本仓库**只有源码**，不含运行时。`leno.exe` / `leno_vm.exe` 与 `leno_module/`
（LenoSDL3、LenoWeb、LenoMusic、LenoCrypto 等，含原生 DLL）都在 LenoLang 发行包里，
这里不重复发布（那部分几十 MB）。

1. 到 [LenoLang Releases](https://github.com/cheng8214/LenoLang/releases) 下载 **v0.1.1 或更新**的发行包
2. 把本目录放到任意位置（不必和发行包放在一起）
3. 用发行包里的编译器运行：

```bat
:: GUI（主入口）
<发行包>\leno.exe <本目录>\musicdl_gui.leno

:: CLI
<发行包>\leno.exe <本目录>\musicdl.leno 周杰伦 晴天
```

模块查找是**按 `leno.exe` 所在目录**扫 `leno_module/` 的 ⇒ 两者放哪儿都行，
只要用那个 exe 跑即可 ✓

## 二、功能

> 界面是**两页签**：`搜索` / `榜单`（★ 2026-10-09，参考 `leno_gui/应用/数据看板/dashboard.leno`
> 的 `TabControl` 用法；**播放条在页签之外** ⇒ 切页不打断播放 ✓）。
> 自检：`DBANG=1` 启动即切到榜单页，`DBANGSRC=<1..4>` 顺带选中站点，配合 `DSHOT` 可无头截图 ✓；
> `DCTX=1`（配 `DBANG=1`）往榜单表注入**真实右键**、`DCTX=2` 往搜索表空白处注入、
> `DCTX=3`（配 `DBANG=1`）直接调「搜索这首歌」这个动作 —— 三者都会打印 `[dctx] 通过 / **不通过**` ✓
> 双击自检：`DSEARCH=<关键词> DDBL=1` ⇒ 往搜索表第 0 行注入**真双击**；`DBANG=1 DDBL=2` ⇒ 往榜单表第 0 行注入；
> 断言"起的是试听（`gPreviewCh` + 试听行）、**没**开下载窗口"⇒ 打印 `[ddbl] 通过 / **不通过**` ✓
> 「关于」弹窗自检：`DABOUT=1` ⇒ 第 3 拍自动打开它 ✓
> ⚠ **DSHOT 截不到这个弹窗**：DSHOT 读的是**主窗口渲染器**的像素，而模态是**另一个窗口** ✗
> ⇒ 想看它的样子得用**系统截屏**（`Win32.captureScreen` / 截图工具）✓

- **榜单**（v0.1.6 ✓，移植自二创并改成独立页）：选站点（酷我 / 网易云 / 酷狗 / QQ）⇒
  选榜单 ⇒ 取该榜歌曲（默认 100 首）；**切到本页才加载**（懒加载，不拖慢启动 ✓）
  - 酷我 **31 个**榜单 + QQ **9 个**榜单**内置成常量表**（免网络、瞬时 ✓）
  - 网易云 / 酷狗的榜单菜单**现拉接口**（各自榜单接口，免登录 ✓）
  - 四个源的榜单接口与坑（`rankid ≠ id`、缺 `singername`、URL 结尾多个 `?` 会让首个参数失效…）
    全部记在各自 core 的注释里；离线可测的部分钉在 `test/test_bang_recipe.leno`（38 项 ✓）
  - **「格式 / 大小」列会逐首补上**（★ 2026-10-09 修）：榜单接口**确实不给体积**（实测：酷我
    `kbangserver` 的 `musiclist` 只有 `id/name/artist/album/formats/duration/param…`；官方取直链的
    `mobi.s` 也只回 `format/bitrate/url`）⇒ 取到行后由 `dl_engine.fill_sizes` 逐首补：
    ① 有第三方聚合通路的源先走 `proberow`（与搜索同口径）；② 仍没有体积 ⇒ 取**官方直链**
    再 `HEAD`（探不到就自动退 `Range: bytes=0-0`）探真实字节数，`ext` 按直链后缀回填 ✓
    切片是**连续段**（不是搜索用的轮转片 ✗）：**榜单排名就是顺序**，轮转会把顺序打乱 ✗
    ⇒ 8 路并行，实测 100 首约 **5~20 s**（状态栏显示"正在逐首补…"→"已补到格式/大小 N 首" ✓）
  - ⚠ **榜单行仍不做「有没有体积」过滤**：补不到的行**照样留着**（不按体积筛掉 ✓
    这是与搜索页有意的差别：搜索页"没体积 = 下不了 ⇒ 不进表"，榜单页"下不了也让你先看见" ✓）；
    真下的时候按 `Range` 头算真实总量 ✓
  - 自检：`DBANG=1 DBANGSIZE=1` ⇒ 等补完打印「补到 N/M 首 + 耗时 + 首行有没有变（顺序 ✓）」再退出 ✓
  - **右键菜单与搜索页同一套**（★ 2026-10-09 补）：两张表共用 `buildMenu`/`onMenu`，
    只靠 `menuCtx` 分辨作用在哪张表上（见 `ctx_table/ctx_len/ctx_row/ctx_picked` ✓）⇒
    试听 / 下载这一首 / 只下这一首的歌词 / 复制「歌名-歌手」/ 下载勾选的 N 首 / 只下勾选的歌词 /
    全选 / 全不选 / 在资源管理器里定位 / 删除本地文件 / 打开下载目录 —— 两个页面**完全一致** ✓
    （⚠ 原先榜单表只接了双击 ⇒ 想"选一批再下"没法做 ✗；格式过滤只属于搜索页 ⇒ 榜单页不套它 ✓）
  - **双击 = 试听**（★ 2026-10-09 改）：两张表都走同一个 `previewRow`（与右键「试听（同时后台下载整首）」
    是**同一条入口** ⇒ 拉前一段先出声 + 静默下整首、下完自动切到完整文件接着放 ✓）。
    原先双击是"下载这一首"，但下载本就有「下载勾选」与右键两条更明确的路，而"点一下先听听"没快捷方式 ✗
    ⇒ 把最顺手的双击给试听（回车 `RETURN` 也走这条回调 ✓）；想下载用右键或按钮 ✓
  - 自检：`DSEARCH=<词> DDBL=1`（搜索表）/ `DBANG=1 DDBL=2`（榜单表）⇒ 注入真双击并断言 ✓
  - **榜单页独有的一项**：「**搜索这首歌（跳到搜索页）**」（★ 2026-10-09）——
    关键词取 `"歌名 歌手"`（歌手为空就只搜歌名；榜单接口常缺 `singername` ✓），
    填进搜索框 ⇒ `tab.set_active(0)` 切到搜索页 ⇒ 立刻发起搜索（上一次还在搜就只填词不乱发 ✓）。
    为什么只在榜单页给：榜单行只带**一个站点** ⇒ 想看别家有没有、比对音质就得换个关键词再搜 ✓；
    搜索页本来就在搜、词就是你刚输入的 ⇒ 再加一项纯属重复 ✗
- **搜索**：关键词 + 每源条数；**多音源**（酷我 / 网易云 / 酷狗 / QQ）一起搜、结果表按音源混排；
  音源复选框按注册表**动态生成**（加新源只需在注册表加一行 ✓）
- **结果表**：勾选列（表头可全选）/ 排序 / 多选 / **双击试听**（同时后台下整首）/ 右键菜单
  （下载勾选项、复制「歌名-歌手」、在资源管理器里定位、删除本地文件、打开下载目录）
- **下载**：`Range` 分片（1 MB/片）+ **断点续传**（重下同一首走 `416` 快路径，秒回）
  + 暂停 / 继续 / 取消 + 多首并行；音频落盘时顺带下同名 `.lrc`
- **播放**：播放条（转盘 / 图标 / 点击快进）+ 歌词浮层（自绘画布、当前句高亮放大、
  邻近句按距离淡出、随播放平滑上滚、边缘渐隐、滚轮手翻、点句跳转、
  **逐字时间轴**（有逐字就用，没有退回标准 LRC）、译文行）
- **设置**：标题栏齿轮进入（下载目录 / 格式过滤 / 并行数等）
- **关于**（★ 2026-10-09 新增）：标题栏最右那个 ⓘ（`images/about.png`，群友「小半头砖」提供的素材 ✓）
  ⇒ 弹窗给出**版本 / 致谢 / 免责声明**（后两块的口径与本文第八、九节**同一套** ✓）
  - 致谢名单与免责文案都在 `dl_about_win.leno` 顶部那两个常量里 ⇒ **要加人就加一行** ✓
  - 画法照 `dl_settings_win.leno`（自绘标题栏 + `runModal` 模态 ✓）：ESC / 回车 / ✕ 都能关 ✓

### 音源差异（务必知道）

| | 酷我 | 网易云 | 酷狗 |
| --- | --- | --- | --- |
| 协议 | 官方 mobi 接口 + **DES 变体**加密 | **weapi**：AES-128-CBC 两轮 + 1024 位 RSA（`netease_crypto.leno`）| 移动端搜索 + CDN 取链：`key = md5(hash + "kgcloudv2")`，**免签名**|
| 歌词 | `newlyric.lrc`（zlib + gb18030；译文行混在原文里）| `weapi/song/lyric`（译文**单独一份** ⇒ 按时间戳并回原文）| `krcs` 搜 ID → `lyrics.kugou.com/download`（base64 LRC，自带 BOM）|
| 列表体积来源 | `MINFO` 字段 | 搜索后再发**一次批量**请求补齐 | 搜索响应**直接带三档体积**（不用额外请求 ✓）|
| 最高音质 | 接口给到什么就是什么（含 flac）| 官方只有 320k mp3；**flac 走第三方聚合接口**（实测 113 MB / 3.5 Mbps ✓）| **免费曲直接给 flac**（实测 52.8 MB ✓）；付费曲走第三方兜底 |
| 拿不到直链时 | 有第三方兜底接口 | 付费/原唱曲 `code=404` ⇒ **第三方兜底**（至少可下 ✓）| 付费曲 CDN 回 `status=2` ⇒ **第三方兜底** |
| 金标回归 | `kuwo_des.leno` 文件内向量 | `test/test_netease_crypto.leno` | `test/test_kugou_cdn.leno` |

> **无损（flac）统一走 `thirdparty.leno`**（网易 `music.qinglvai.top` / QQ `api.vkeys.cn` /
> 酷狗 `api.baka.plus`）—— 官方接口在**未登录 / 未开会员**时一档无损都拿不到，这是实测结论
> （第三方通路的八个坑见 `docs/移植音乐下载器实录与待办.md` 的 A8；配方回归在 `test/test_thirdparty.leno`）。
> **QQ 音乐**（第 4 个音源，未进上表）：官方 `musicu.fcg` 免登录时 `F000`(flac) 的 `purl` 为空
> ⇒ 无损同样走第三方（实测 178 MB / 5.5 Mbps 的臻品母带 ✓）。
>
> 无损是**搜索时逐首去问第三方**的（上游 musicdl 也是这个做法）⇒ 逐首请求是搜索耗时的大头；
> 所以搜索拆成**两阶段并行**（每个音源一条线程 + 无损直链按行切片多线程），
> 实测 4 源 × 10 条从 **~26 s 降到 ~6 s** ✓。拿不到的行**保持官方结果**（不会因此变成"下不了"），
> 且直链会**随搜索结果带着走** —— 落盘时不再问一次 ✓

> **列表里"格式空白 + 大小 0" = 这首下不了**（付费/原唱）——三源统一这个口径：
> 网易在搜索后靠一次批量请求判出来，酷狗直接读搜索响应里的 `pay_type*` / `*privilege` ✓
> 点了下载也会如实报"取直链失败"，不会假装成功 ✓

## 三、文件职责

| 文件 | 职责 |
| --- | --- |
| `musicdl_gui.leno` | GUI 主程序：**两页签**（`搜索` / `榜单`，`TabControl`）+ 标题栏动作；**播放条在页签之外**（切页不打断播放 ✓）|
| `songrow.leno` | **歌曲行的字段契约**：10 个下标常量（`F_*`）+ 具名存取（`rowname` / `rowext` / `rowbytes` …）。全仓唯一事实来源 —— 引擎转发一份 `eng.F_*` 保兼容，GUI/下载窗口零改动 |
| `parsing.leno` | **严格解析**工具（纯逻辑）：`parse_secs` / `parse_millis` / `parse_word_span` / `parse_uint` 一律用**多返回值**`(值, 是否成功)`，不用 -1 哨兵；外加越界安全取值 `at` / `at_int` / `tail_from` |
| `dl_engine.leno` | **引擎层**（无 UI、无 print、无 main）：音源注册表 + 分片下载（进度 / 续传 / 暂停取消）+ 跨线程 worker 入口；对外一律传「纯字符串数组行」（struct 不能跨线程 ✗） |
| `dl_window.leno` | 下载窗口（非模态、不重复打开：任务表 + 总进度 + 按钮 + 120ms 刷新） |
| `player.leno` | 播放器**纯逻辑**：扫目录 / 播放索引 / 循环三态 / 歌词解析（标准 LRC + 逐字时间轴） |
| `player_bar.leno` | 播放条 UI（嵌在主界面表格下方、状态栏上方） |
| `dl_settings.leno` | 设置**纯逻辑**（读写配置 + 取值钳位；不 import SDL3 ⇒ 可单测 ✓） |
| `dl_settings_win.leno` | 设置弹窗（SDL3） |
| `kuwo_core.leno` | 酷我核心：搜索 / 取直链 / **榜单**（内置 31 榜菜单 + `kbangserver` 老接口）|
| `kuwo_des.leno` | 酷我官方接口要用的 DES 变体（`encryptquery`） |
| `kuwo_lyric.leno` | 酷我歌词：`newlyric.lrc` 接口 + zlib 解压 + gb18030 解码 + 落盘 |
| `netease_core.leno` | 网易核心：`weapi/search/get` 搜索 + 批量取直链（顺带补齐体积与"能不能下"）|
| `netease_crypto.leno` | 网易 weapi 参数加密：AES-128-CBC 两轮 + 1024 位 RSA（用任意精度 `int` 直接做模幂）|
| `netease_lyric.leno` | 网易歌词：`weapi/song/lyric` + 译文按时间戳并回原文 + 落盘补 BOM |
| `kugou_core.leno` | 酷狗核心：移动端搜索（三档体积 + 付费标记一次带回）+ CDN 取链（`kgcloudv2`，免签名）|
| `kugou_lyric.leno` | 酷狗歌词：`krcs` 搜 ID → `lyrics.kugou.com/download`（base64 LRC）；**入口收整行**（命中需要歌名+时长 ✓）|
| `qq_core.leno` / `qq_crypto.leno` | QQ 核心（`musicu.fcg` 搜索 + vkey 取链，需设备指纹 QIMEI36）/ 指纹注册与兜底值 |
| `thirdparty.leno` | **第三方聚合接口取直链**（无损的唯一通路）：网易 `qinglvai` / QQ `vkeys` / 酷狗 `baka meting` + 体积探测（HEAD ⇒ `Range: 0-0`）+ 302 只读 `Location`；对外只有 `resolve(client, src, id)` |
| `musicdl.leno` | CLI 入口（与 GUI 共用同一套引擎） |
| `resource.toml` | 单文件打包配置（`onefile` / `images/**` 资源 / `app.ico` 图标） |
| `test/` | 可直接运行的回归脚本（见第五节） |
| `images/`、`app.ico` | 播放器/标题栏图标、**右键菜单项图标**（`select_all` / `select_none` / `download_*` / `delete_file` / `open_folder` / `audition` / `search`…）；侧栏两个入口图 `sousuo`/`bangdan` 是**白色蒙版**（描边改白、形状不变 ⇒ 由 `image_tint` 染成任意色，见 `musicdl_gui` 里 `onTabChanged` ✓）；打包用应用图标 `app.ico` |
| `downloads/` | 默认输出目录（**不入库**） |

## 四、CLI 用法

```bat
leno.exe musicdl.leno 周杰伦 晴天              :: 搜索并下载第 1 首（默认连歌词一起下）
leno.exe musicdl.leno -n 3 周杰伦 晴天         :: 下载前 3 首
leno.exe musicdl.leno -s 周杰伦 晴天           :: 只搜索、列出全部，不下载
leno.exe musicdl.leno --lyric-only 周杰伦 晴天 :: 只下歌词（音频不动）
leno.exe musicdl.leno --no-lyric 周杰伦 晴天   :: 只下音频，不要歌词
leno.exe musicdl.leno -o D:\Music 晴天        :: 指定输出目录
leno.exe musicdl.leno --help
```

| 选项 | 说明 |
| --- | --- |
| `-n <数量>` | 下载前 N 首（默认 1） |
| `-l <数量>` | 每个关键词搜索多少条（默认 10） |
| `-o <目录>` | 输出目录（默认 `<脚本目录>/downloads`） |
| `-s, --search` | 只搜索并列出，不下载 |
| `-y, --lyric` | 下载音频时同时保存同名 `.lrc`（默认开） |
| `--no-lyric` | 只下音频，不存歌词 |
| `--lyric-only` | 只下歌词，不碰音频 |
| `-h, --help` | 显示帮助 |

## 五、跑测试

11 个脚本都能直接运行，**全部离线**、不需要音频设备（`test_parallel_dl` 用的是本机 HTTP 服务端）：

```bat
<发行包>\leno.exe test\test_player_state.leno      :: 播放器规则：认哪些扩展名 / 下一首是谁 / 循环三态 / 失败要安全
<发行包>\leno.exe test\test_bang_recipe.leno        :: 榜单配方金标：31/9 榜常量表 + 酷我 URL 配方 + formats→ext + 脏行丢弃（38 项 ✓）
<发行包>\leno.exe test\test_parsing.leno           :: parsing/songrow 的边界：at_int 脏数据**不抛** / 解析严格性 / 老行兼容（43 项断言 ✓）
<发行包>\leno.exe test\test_lrc_parse.leno         :: 歌词解析金标：标准 LRC / 逐字时间轴 / 译文行 / 非法标签（22 项断言 ✓）
<发行包>\leno.exe test\test_parallel_dl.leno       :: 多首歌并行下载（离线服务端）
<发行包>\leno.exe test\test_settings.leno          :: 设置的真文件读写与钳位
<发行包>\leno.exe test\test_dl_window_open.leno    :: 下载窗口「不重复打开」的规则
<发行包>\leno.exe test\test_titlebar_actions.leno  :: 标题栏动作按钮：解析 + 内置图标名判定
<发行包>\leno.exe test\test_netease_crypto.leno  :: 网易 weapi 加密金标回归（AES 两轮 + RSA，8 项断言 ✓）
<发行包>\leno.exe test\test_kugou_cdn.leno       :: 酷狗 CDN 取链配方回归（key 算式 + URL 形态 + 大小写敏感，6 项断言 ✓）
<发行包>\leno.exe test\test_thirdparty.leno      :: 第三方聚合接口：URL 配方 + 档位顺序 + 四类响应解析（25 项断言 ✓）
```

## 六、打包成单文件 exe

`resource.toml` 已经写好（`onefile = true`、资源 `images/**`、图标 `app.ico`）：

```bat
<发行包>\leno.exe -p --onefile musicdl_gui.leno
```

产物在 `dist\` 下，只有一个 exe，原生 DLL 与资源都内嵌其中，首次运行解包到用户缓存目录。

## 七、约定

- **字段契约只有一处**：歌曲行的下标一律用 `songrow.leno` 的 `F_*`（或它的具名存取
  `srow.rowname(row)` 等），**不要写裸下标** `row[1]` —— 加字段只能在**末尾**追加，
  中间插入会让已缓存的行含义错位且**编译器不报错** ✗
- **严格解析走 `parsing.leno`**：需要"认不出就失败"时用 `parse_secs` / `parse_uint`
  这类接口（多返回值 `(值, 是否成功)`），**不要拿 -1 当失败哨兵** —— `-1` 与合法负数
  永远分不开 ✗（`player.lrc_tag_secs` 就是这么被替掉的）
- **跨线程只传 `Array[string]`**：struct / 多返回值都不能过 channel（收到的是 `any`，
  静态检查会拒绝解构）⇒ 行协议与进度协议保持"纯字符串数组"是有原因的，不是偷懒 ✓
- `downloads/` 是默认输出目录（音频 + 同名 `.lrc`），**已在 `.gitignore` 里**
  ⇒ 别把下下来的内容提交上来（体积大，而且那不该由这个仓库分发）
- `.lenocache/`、`*.lenb`、`dist/` 同样是产物，不入库

## 八、免责

本工具只做一件事：**按你的要求，从各音乐平台的网页端 / 移动端接口取直链并落到你的磁盘**
（官方接口拿不到无损时，会按 `thirdparty.leno` 的通路去问公开的第三方聚合接口）。
不提供任何内容分发，仓库里也不含任何音频。下载内容的版权归各平台与权利人所有，
请仅用于个人学习与已购内容的备份，并遵守平台的服务条款。

## 九、许可

本目录**大部分**代码与 [LenoLang](https://github.com/cheng8214/LenoLang) 主仓库一致，采用 MIT。