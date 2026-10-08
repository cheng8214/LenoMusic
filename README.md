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

- **搜索**：关键词 + 每源条数；音源复选框按注册表**动态生成**（当前「酷我」可用，其余源为占位）
- **结果表**：勾选列（表头可全选）/ 排序 / 多选 / 双击下载这一首 / 右键菜单
  （下载勾选项、复制「歌名-歌手」、在资源管理器里定位、删除本地文件、打开下载目录）
- **下载**：`Range` 分片（1 MB/片）+ **断点续传**（重下同一首走 `416` 快路径，秒回）
  + 暂停 / 继续 / 取消 + 多首并行；音频落盘时顺带下同名 `.lrc`
- **播放**：播放条（转盘 / 图标 / 点击快进）+ 歌词浮层（自绘画布、当前句高亮放大、
  邻近句按距离淡出、随播放平滑上滚、边缘渐隐、滚轮手翻、点句跳转、
  **逐字时间轴**（有逐字就用，没有退回标准 LRC）、译文行）
- **设置**：标题栏齿轮进入（下载目录 / 格式过滤 / 并行数等）

## 三、文件职责

| 文件 | 职责 |
| --- | --- |
| `musicdl_gui.leno` | GUI 主程序：主窗口（搜索栏 / 音源勾选 / 结果表 / 右键菜单）+ 标题栏动作 |
| `dl_engine.leno` | **引擎层**（无 UI、无 print、无 main）：音源注册表 + 分片下载（进度 / 续传 / 暂停取消）+ 跨线程 worker 入口；对外一律传「纯字符串数组行」（struct 不能跨线程 ✗） |
| `dl_window.leno` | 下载窗口（非模态、不重复打开：任务表 + 总进度 + 按钮 + 120ms 刷新） |
| `player.leno` | 播放器**纯逻辑**：扫目录 / 播放索引 / 循环三态 / 歌词解析（标准 LRC + 逐字时间轴） |
| `player_bar.leno` | 播放条 UI（嵌在主界面表格下方、状态栏上方） |
| `dl_settings.leno` | 设置**纯逻辑**（读写配置 + 取值钳位；不 import SDL3 ⇒ 可单测 ✓） |
| `dl_settings_win.leno` | 设置弹窗（SDL3） |
| `kuwo_core.leno` | 酷我核心：搜索 / 取直链 / 下载 |
| `kuwo_des.leno` | 酷我官方接口要用的 DES 变体（`encryptquery`） |
| `kuwo_lyric.leno` | 歌词：`newlyric.lrc` 接口 + zlib 解压 + gb18030 解码 + 落盘 |
| `musicdl.leno` | CLI 入口（与 GUI 共用同一套引擎） |
| `resource.toml` | 单文件打包配置（`onefile` / `images/**` 资源 / `app.ico` 图标） |
| `test/` | 可直接运行的回归脚本（见第五节） |
| `images/`、`app.ico` | 播放器与标题栏图标、打包用应用图标 |
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

5 个脚本都能直接运行，**全部离线**、不需要音频设备（`test_parallel_dl` 用的是本机 HTTP 服务端）：

```bat
<发行包>\leno.exe test\test_player_state.leno      :: 播放器规则：认哪些扩展名 / 下一首是谁 / 循环三态 / 失败要安全
<发行包>\leno.exe test\test_parallel_dl.leno       :: 多首歌并行下载（离线服务端）
<发行包>\leno.exe test\test_settings.leno          :: 设置的真文件读写与钳位
<发行包>\leno.exe test\test_dl_window_open.leno    :: 下载窗口「不重复打开」的规则
<发行包>\leno.exe test\test_titlebar_actions.leno  :: 标题栏动作按钮：解析 + 内置图标名判定
```

## 六、打包成单文件 exe

`resource.toml` 已经写好（`onefile = true`、资源 `images/**`、图标 `app.ico`）：

```bat
<发行包>\leno.exe -p --onefile musicdl_gui.leno
```

产物在 `dist\` 下，只有一个 exe，原生 DLL 与资源都内嵌其中，首次运行解包到用户缓存目录。

## 七、约定

- `downloads/` 是默认输出目录（音频 + 同名 `.lrc`），**已在 `.gitignore` 里**
  ⇒ 别把下下来的内容提交上来（体积大，而且那不该由这个仓库分发）
- `.lenocache/`、`*.lenb`、`dist/` 同样是产物，不入库

## 八、免责

本工具只做一件事：**按你的要求，从酷我网页端接口取直链并落到你的磁盘**。
不提供任何内容分发，仓库里也不含任何音频。下载内容的版权归各平台与权利人所有，
请仅用于个人学习与已购内容的备份，并遵守平台的服务条款。

## 九、许可

本目录**大部分**代码与 [LenoLang](https://github.com/cheng8214/LenoLang) 主仓库一致，采用 MIT。