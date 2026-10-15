# Formatix 项目记忆

> 目标：下次接手任务时，先看这份文档就能直接进入改动，不必重新通读 2000+ 行代码。

## 1. 项目速览

Windows 桌面端批量图片转换/缩放工具，100% 离线、单文件可分发（PyInstaller）。
技术栈：Python 3.10+ / tkinter + tkinterdnd2 / Pillow（含 ImageCms）/ pillow-heif / resvg-py。
入口 `formatix.py`，直接 `python formatix.py` 运行（本机实测 Python 3.12 + Pillow 12.2）。

## 2. 文件职责

| 文件 | 行数 | 职责 |
| :--- | :--- | :--- |
| `formatix.py` | ~2400 | 主窗口 `App`、UI 构建、设置读写、转换线程调度、自定义控件（滑块/分段开关/勾选框） |
| `converter.py` | ~700 | 纯转换逻辑：`convert_one()`、格式可用性探测、文件名模板、目标大小二分找质量 |
| `localization.py` | ~460 | 6 语言字典 `STRINGS`（`en`/`ru`/`uk`/`de`/`fi`/`zh`）+ 系统语言检测 |
| `settings.py` | ~530 | 设置窗口、捐赠窗口（通过 `app` + `colors` 传入，避免循环导入） |
| `compare.py` | ~690 | 转换前后对比窗口 |
| `update.py` | ~105 | GitHub 版本检查 |

模块关系约定：`converter.py` 自给自足、不 import 主模块；`settings.py`/`compare.py` 通过显式传入 `app` 与调色板字典访问主窗口，不反向 import。

## 3. 环境与依赖（uv）

已用 uv 0.12 搭好：`pyproject.toml` + `uv.lock` + `.venv`（Python 3.12.10 **64 位**，`D:\Python\Python312`）+ `.python-version`（3.12）。

```bash
uv sync                      # 安装/同步依赖（含 dev 组）
uv run python formatix.py    # 启动应用（不要直接 python formatix.py，会用到系统解释器）
uv add <pkg> / uv remove <pkg>
uv run black . / uv run isort .
```

- 运行时依赖：`pillow` `pillow-heif` `resvg-py` `tkinterdnd2`；dev 组：`black` `isort` `pytest` `pytest-cov`（black/isort 已在 pyproject 里配好 line-length 88）。
- 依赖装全后 8 种输出格式与 13 种输入扩展全部可用（HEIC/SVG 生效，`HAS_DND=True`），已实测 HEIC 输出、SVG 输入都正常。
- **坑**：本机存在 uv 托管的 **32 位** 3.12，pillow-heif/resvg-py 没有 32 位轮子，uv 会退回源码构建并因缺 MSYS2 失败。pyproject 里已加 `[tool.uv] python-preference = "only-system"` 规避；若换机器仍踩到，用 `uv venv --python <64位解释器路径>` 重建。

## 4. 关键数据流

### 转换链路
`_start()`（GUI 线程校验 + 快照参数 + 冲突询问）→ `threading.Thread(_convert_worker)` → `ThreadPoolExecutor(min(cpu,8))` → `converter.convert_one()`。
后台线程一律用 `_start` 传进去的**参数快照**，禁止在线程里读 `self._fmt` 等 GUI 变量。

### 缓存
`self._converted_cache`（主线程持有）+ `self._data_lock`。键是源文件路径，值是 `(config_str, out_path, size, True, res_str)`；命中条件是 config 一致且结果文件还在。
**坑**：`current_config_str` 在 `_start()` 和 `_convert_worker()` 里各拼了一次，**新增任何影响输出的参数都必须两处同步改**，否则缓存会误命中。当前格式：
`fmt:..|q:..|dir:..|mode:..|w:..|h:..|fn:..|meta:0/1`

### 结果回传
`_convert_worker` 把 `insert_result`/`progress`/`finish` 消息塞进 `self._gui_queue`，每个消息打 `batch_id`；`_listen_queue` 丢弃过期 batch_id，避免用户点「清除」后旧 worker 的结果回填 UI。

### 设置持久化
文件：`~/.formatix_image_converter_settings.json`。
`_save_settings()` 在 `_loading=True` 期间直接 return（防止初始化时 trace 回调覆盖恢复值）；fmt/quality/resize/out_dir/filename/keep_metadata 只在「记住设置」勾选时落盘，`lang`/`theme`/`remember_settings`/`check_updates` 始终落盘。

## 5. UI 结构要点

- 底部面板 `bot` 由三行组成（自上而下）：`folder_row`（保存文件夹 + 路径 + 选取/清除/打开按钮）→ `progress_row`（**单独一行**：进度标签 + 可拉伸进度条 + 右侧状态/错误文本）→ `ctrl_row`（横向一行：`格式 | 质量(+% /大小切换) | 分辨率 | 宽×高 | 保留元数据 | 转换按钮`）。注意：文件大小比对块（was/became）**已移除**，`_total_src_bytes`/`_total_dst_bytes` 现为只写、无界面消费。
- `_ctrl_block()` 生成「上标签 + 下控件」的标准块；无标题的块（宽高输入、保留元数据）用**同高度空白 Label** 占位对齐。
- 自定义控件都是 Canvas 手绘、由 tk 变量驱动（trace 重绘）：`FancySlider`、`SegmentedToggle`、`CheckToggle`（后者为「保留元数据」新增）。切语言靠 `_update_ui_strings()` 逐个 `config(text=...)` / `set_label()`。
- 主题色是模块级常量（BG/BG2/BG3/CARD/FG/FG2/FG3/ACCENT/ACCENT2/BORDER/GREEN 等），主题切换需重启应用。

## 6. 本地化

`app.t(key)` 取当前语言字符串，缺省回退英文。**新增任何 UI 文案都要在 6 个语言字典里同时加 key**，否则非中文界面显示 key 本身。

## 7. 踩坑与约定

- **`img.info` / `getexif()` 会在 `convert()`、`resize()`、`ImageCms.profileToProfile()` 之后丢失**——要取元数据必须在 `Image.open()` 之后立刻抓快照。
- 写盘统一走 `out_path + ".tmp"` + `os.replace()` 的原子替换，异常时清理临时文件。
- ICO 上限 256px；SVG 无内嵌尺寸时需要用户在「自定义」模式给宽高。
- 代码注释：文件原有注释是**俄文**，但用户规则要求新写的注释用**中文**——目前新增代码（含 `converter.py` 元数据段、`CheckToggle`）用的是中文，属于刻意混排，不要「顺手」统一成俄文。
- 用户规则：答复与注释用中文；专注当前需求，不改动未提及的功能；不主动写测试脚本。
- 端到端验证方式：写临时脚本放系统临时目录（不要落工作区），跑完删掉；GUI 冒烟要 `os._exit()` + watchdog 定时器，否则 `mainloop` 会挂住命令行。

## 8. 已完成的改动：选项「保留元数据」

- 位置：主界面底部控制栏（`CheckToggle` + `self._keep_metadata`），默认关闭，随设置持久化。
- 行为：勾选后把源文件 EXIF（含 GPS）、XMP、ICC、PNG 文本块、JPEG 注释复制到结果；有源 ICC 时**跳过强制 sRGB 转换**、保留源配置；无源 ICC 时沿用原兜底（AVIF 仍内嵌 sRGB）。
- 实现落在 `converter.py`：`extract_source_metadata()`（开图即抓快照）→ `fix_exif_dimensions()`（修正尺寸、清 IFD1 缩略图）→ `build_metadata_kwargs()`（按格式白名单）。
- 格式支持矩阵实测：JPEG/WEBP/PNG/AVIF 可写 `exif`+`icc_profile`+`xmp`；PNG 的 XMP 只能走 iTXt 块 `XML:com.adobe.xmp`；BMP/ICO 无元数据槽位；TIFF/HEIF 未实测（本机缺 pillow_heif）。
- 元数据写失败时自动降级：剥掉元数据再存一次，不让转换失败。

## 9. 打包 exe（PyInstaller）

`pyinstaller 6.22.3` 已装在 dev 组。构建命令与仓库 CI（`.github/workflows/build.yml`）保持一致：

```bash
uv run pyinstaller --noconfirm --clean --onefile --noconsole \
  --icon=icon.ico --add-data "icon.ico;." --collect-all tkinterdnd2 formatix.py
# 产物改名（CI 同款命名，VERSION 在 formatix.py:173）
Move-Item dist/formatix.exe dist/formatix-image-converter-v1.17.4.exe
```

- 产物约 **19.9 MB**；`--add-data "icon.ico;."` 是必须的，设置/对比窗口会用 `resource_path("icon.ico")` 读它。
- `libheif-*.dll`、`libtkdnd2.10.3.dll`、`resvg_py.pyd`、`_tkinter.pyd` 都由 hooks 自动收集，**不需要** `--collect-all pillow_heif`。
- UPX 报 `NotCompressibleException` 只是警告（个别系统 DLL 压不动），不影响产物；介意可加 `--noupx`。
- **坑**：onefile 是双进程模型——父进程常驻约 7 MB 只负责解压，真正的界面在同名的**子进程**（约 100 MB，窗口标题 `Formatix Image Converter`）。冒烟验证要按内存/窗口标题找子进程；清理时 `Get-Process -Name "formatix*" | Stop-Process` 才能杀干净，只杀父进程会留下孤儿窗口。
- 构建中间产物：`build/`、`formatix.spec`（后续可用 `uv run pyinstaller formatix.spec` 直接重编）。
