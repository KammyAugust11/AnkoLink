# 更新日志

[简体中文](CHANGELOG.md) · [English](CHANGELOG.en.md) · [返回 README](README.md)

本文件记录 MuseLink / Muse连接器 的功能更新、安全加固与问题修复，随版本发布与推送同步维护。

## [Unreleased]

### 安全与稳定性加固
- **发布产物增加构建来源证明**：发布工作流为 Windows / Linux / macOS 三个平台的发布资产生成 GitHub 构建来源证明（build provenance attestation），用户可通过 `gh attestation verify` 校验下载的二进制确实由本仓库源码在 GitHub Actions 中构建，未被替换或篡改。
- **扩大默认读取保护范围**：`.npmrc`、`.pypirc` 由仅写保护升级为读 / 搜索 / 写全面保护；新增对 `.docker`、`.kube`、`.azure`、gcloud 凭证目录以及各平台浏览器凭证库的默认保护（Windows：Chrome / Edge 的 `User Data` 与 Firefox 的 `Profiles`；macOS：Application Support 下的 Chrome / Edge / Firefox，配合既有 `~/Library/Keychains` 保护；Linux：`google-chrome` / `chromium` / `microsoft-edge` 与 `.mozilla/firefox`），避免磁盘授权或主目录授权时令牌与登录数据被读取。

## [v0.3.1] - 2026-11-27

### 问题修复与跨平台适配
- **修复 macOS / Linux「复制专用令牌」必定失败以及 Windows 剪贴板短暂占用导致复制失败的问题**：
  - **macOS 原生隐私剪贴板**：通过 AppKit `NSPasteboard` 实现令牌复制，写入 `org.nspasteboard.ConcealedType`、`org.nspasteboard.TransientType` 标记并启用 `CurrentHostOnly` 选项，防止令牌被剪贴板历史工具记录或通过通用剪贴板跨设备同步。
  - **Linux 剪贴板支持**：支持通过 `wl-copy` / `xclip` / `xsel` 标准输入安全写入系统剪贴板（不经命令行参数传递令牌，并隔离后台守护进程输出管道）。
  - **Windows 剪贴板稳定性与隐私加固**：为 `OpenClipboard` 增加短退避重试（缓解刚复制完接入说明后剪贴板被系统剪贴板历史/第三方工具瞬时占用的竞态失败），并补全 4 字节 `DWORD(0)` 的 `CanIncludeInClipboardHistory` 与 `CanUploadClipboardData` 隐私标记。
  - **透出真实错误诊断**：新手指引与接入说明卡片复制令牌失败时展示具体失败原因（同步中/英/日三语），同时保留「定位令牌文件」兜底入口。
- **修复 macOS / Linux「管理本机磁盘（整机授权）」全部返回 403 且无法读取桌面的问题**：
  - **实现 Unix 本地根目录发现与校验**：`fixed_local_roots()` 在 macOS 与 Linux 上返回本机系统根目录 `root`（`/`），更新磁盘别名校验与前端磁盘选择器显示（「系统根目录（/）」），并在后端校验中拒绝空的整机根映射，彻底修复 `/agent/v1/roots` 为空与 `/agent/v1/locations/desktop` 返回 403 的问题。
  - **补齐 macOS / Linux 安全保护与搜索索引边界**：新增 macOS `~/Library/Keychains` 读写搜全保护，以及 macOS/Linux 系统目录、开机自启项（`~/Library/LaunchAgents`、`.config/autostart`）、登录 Shell 脚本（`.zprofile` / `.zshenv` / `.bash_profile`）与 `.app` 应用包的默认写保护；后台搜索索引自动跳过 macOS `/System`（避免重复扫描 `/System/Volumes/Data`）、`/Library`、`/Applications`、`/usr`、`/bin`、`/sbin`、`/private`、`~/Library`（避免触发 iCloud 云盘与第三方 App 容器等延迟隐私弹窗）、照片图库、`/dev`、虚拟目录及非本地挂载卷，以及 Linux `/proc`、`/sys`、`/dev`、`/run`（均不影响直接按路径读取）。
  - **优化 macOS 隐私权限（TCC）体验与接口报错**：预热索引前主动探测桌面/文稿/下载以消除弹窗间的长时间等待；在 macOS 整机授权界面新增「打开完全磁盘访问权限设置」一键直达按钮；更新三语接入说明中的跨平台桌面表述，并在操作系统拒绝目录访问（`PermissionDenied`）时返回带操作引导的 HTTP 403 响应。

## [v0.3.0] - 2026-11-27

### 新增功能
- **应用内自动更新与同步更新日志**：
  - **一键检查与重启更新（Windows 安装版 & Linux AppImage）**：支持启动静默检查与「关于 → 检查更新」手动检查，通过 GitHub Releases 静态 `latest.json` 与编译期固化的 Minisign 公钥验签（不走受限 REST API）；发现新版本时在右下角弹出可关闭的新版本提醒浮窗，下载与验签阶段保持现有连接不断流并展示绿色实时进度条，确认安装后安全停服（含释放 Windows 随包 `Everything.exe` 文件锁）并自动重启完成升级；Windows 免安装便携版、Linux `.deb` 与 macOS/源码构建统一通过静态 `latest.json` 比对版本号，正常提供新版本提醒、更新详情与直达 GitHub Releases 下载。
  - **内嵌更新日志与更新详情弹窗**：「关于」页按钮调整为纯文字「更新日志」、「GitHub」与「检查更新」；点击「更新日志」或右下角浮窗的「更新详情」均在应用内弹窗展示，随界面语言自动切换（日语回退英文日志）、自动过滤未发布草稿与跨语言导航头，采用本地内嵌秒开 + 后台静默同步 GitHub 最新日志（并以 `latest.json` 更新说明兜底），标题栏附外链图标直达 GitHub。
  - **三平台自动发布流水线**：新增 `.github/workflows/release.yml` 与 `generate-updater-manifest.ps1`，推送 `v*` 标签后并行在 Windows、Ubuntu 22.04 与 macOS 上运行测试，自动打包 Windows 便携版、签名 NSIS 安装包、Linux 签名 AppImage / `.deb` 与 macOS Universal `.dmg` / `.app.zip`，统一生成双平台自动更新清单 `latest.json` 与全产物 `SHA256SUMS.txt` 挂载至 Draft Release。
- **Linux 支持**：MuseLink 现已可在 Linux 上构建和运行，覆盖文件 API（列出 / 读取 / 写入 / 新建文件夹）与 Tailscale / Cloudflare 隧道配置。本机命令执行仍仅支持 Windows。
- **macOS 支持**：MuseLink 现已可在 macOS（Apple Silicon & Intel）上运行，覆盖文件 API（列出 / 读取 / 写入 / 新建文件夹）与 Tailscale / Cloudflare 隧道配置（自动发现 Tailscale 应用包与 Homebrew 安装），并提供 Universal `.dmg` 与 `.app.zip` 预构建包。本机命令执行仍仅支持 Windows；macOS 包未做 Apple 签名/公证，首次打开需在「系统设置 → 隐私与安全性」中放行。
- **日语本地化**：新增日语（`ja`）语言选项，界面全部文本（含自动更新与更新日志弹窗）、托盘与接入说明均支持日语显示，可在「设置 → 常规」中切换；并新增日语 README 与界面预览图。

## [v0.2.2] - 2026-11-27

### 优化与新增
- **大幅提升 `/agent/v1/read` 读取上限并支持分片读取与原始二进制直传（解决 #4）**：
  - **整文件 JSON 读取上限提升至 10 MiB**：原 2 MiB 上限提升至 10 MiB（与写入上限对齐），老连接器无需修改配置即可直接读取常见高分辨率图片；超限返回 HTTP `413` 并附带 `size`、`max_bytes`、`chunked_supported`、`recommended_limit`、`raw_supported` 与 `recommended_raw_limit` 引导分片重试；旧版 `/api/read` 仍保持 2 MiB 兼容。
  - **支持 `offset` / `limit` 分片读取（无总文件大小硬上限）**：在同一已校验操作系统句柄上按字节偏移分段读取，JSON 分片默认 2 MiB、单片上限 4 MiB，返回 `offset`、`bytes_read`、`total_size`、`next_offset`、`has_more` 与 `etag`；支持 `if_match` 查询参数或 `If-Match` 请求头（兼容带引号的 `ETag`），若读取中途文件发生变更返回 HTTP `409`。
  - **支持 `raw=1` 原始二进制直传**：直接返回对应 `Content-Type` 的原始字节流（无 Base64 33% 膨胀，实测经公网隧道提速约 40%，单次上限 32 MiB，支持与 `offset`/`limit` 组合分片），通过 `ETag` 与 `X-File-*` 响应头提供完整分片元信息。
  - **更新 Muse 接入说明（Prompt）**：同步更新中英文接入说明，指导 Muse 读取图片/二进制文件时优先使用 `raw=1`、大文件按 4 MiB 分片（或 JSON 2 MiB 分片）读取并按片重试。

## [v0.2.1] - 2026-11-25

### 优化与新增
- **桌面界面全面改版与信息架构重构**：
  - **组件视觉升级**：`Switch` 开启态升级为鲜明绿色（`#48C52C`）+ 白色滑块；`Input`、`Select` 与概览公网地址框统一调整为透明底仅描边风格，悬停时描边平滑提亮；日志搜索框新增内置放大镜图标与「查找记录…」占位提示；危险操作（撤销令牌、收回会话授权）使用红色语义强调；全站卡片与表单间距统一校准。
  - **连接页与首次配置重构**：顶部独立「指引 / 开始指引」卡片，设备名称改为清爽的行内布局，将「Muse 接入说明」（复制说明/复制专用令牌）整合至连接页底部，操作按钮左主右次排布更清晰；首次配置浮窗第 3 步将「复制接入说明」与「复制专用令牌」并排置于正文操作区，底部专注流程推进。
  - **文件权限页聚焦**：将「保存授权」提至卡片标题栏右侧，移除原索引与令牌运维杂项，专注权限范围与本机命令执行开关。
  - **日志页精简**：将时间范围筛选提至一级筛选条，与结果、类型、带图标搜索框单行排布，右上角保留单一「暂停同步 / 继续同步」主按钮。
  - **设置页重构为「常规」与「数据」两大板块**：统一收纳本机数据、本机文件索引（状态·占用、清除/重建、打开索引目录）、日志数据（实时磁盘占用、一键清除日志、筛选导出、打开目录）与本机令牌（撤销、轮换、打开目录）。
  - **优化启动与托盘行为**：开机自启默认关闭，开启自启后登录 Windows 时通过 `--autostart` 静默启动到托盘；平时手动打开程序正常弹出主窗口；「设置 → 常规」改为可配置的「关闭时最小化到托盘」（默认开启，关闭该项后点窗口关闭按钮直接安全退出程序）。
- **移除 Cloudflare 临时隧道（Quick Tunnel）**：该路径在实际接入 Muse 时会被边缘策略拦截且地址不固定，现已从前端与后端彻底移除，统一聚焦 Tailscale Funnel 与 Cloudflare 命名隧道（自有域名）；旧配置中若曾保存 `cloudflare_quick` 会自动平滑回退到 Tailscale Funnel。
- **命令审批与执行状态实时长轮询**：`GET /agent/v1/exec/{job_id}` 新增可选参数 `?wait_ms=`（默认 `0` 保持兼容，上限 `25000` ms，推荐 `20000` ms），异步等待期间不占用请求槽位与权限锁；在本机弹窗点击「允许」或「拒绝」、审批超时或命令结束时毫秒级即时唤醒返回，无需再切回聊天手动通知 Muse。
- **更新 Muse 接入说明（Prompt）**：同步更新中英文接入说明，明确指导 Muse 在收到 `awaiting_approval` 或 `running` 状态时保持工作流不中断，直接使用 `?wait_ms=20000` 持续轮询至终态，并内置 `expired` / `denied` 处理与连续轮询上限兜底（已有连接器用户可重新复制接入说明发送给 Muse 更新指令）。
- **审批弹窗倒计时归零防误点**：审批倒计时归零后自动禁用操作按钮，避免超时瞬间误触发。
- **可筛选日志导出与日志目录直达**：在「设置 → 数据 → 日志数据」提供「清除日志」「导出日志」弹窗与「打开目录」入口；支持按日志范围（全部日志 / 文件操作 / 命令执行 / 全部访问 / 运行日志）、结果、时间范围与关键词筛选导出为 `JSONL` 或 `CSV`（UTF-8 BOM），经系统保存对话框写入并做双重令牌脱敏兜底；「打开目录」直接定位本机日志文件与 10 MiB 轮转历史。
- **开源许可证与合规补齐**：新增仓库根目录 [LICENSE](LICENSE)（MIT License），同步 `package.json`、`Cargo.toml`、中英文 README、第三方组件声明及安装包/便携版随附许可证。

### 问题修复
- **修复后台孙进程继承管道导致任务卡在 `running` 的问题**：命令主进程退出后、等待 `stdout`/`stderr` 读取线程结束前立即通过 Windows `JobObject` 清理残留子进程树，避免被继承的管道写端未关闭导致读取线程永久阻塞。

## [v0.2.0] - 2026-11-25

### 新增功能
- **受控本机命令执行（默认关闭）**：在「文件权限 → Muse 文件工具」新增总开关与风险说明确认窗；开启后支持 Muse 通过 `/agent/v1/exec`、`/agent/v1/exec/{job_id}` 与 `/agent/v1/exec/{job_id}/cancel` 在已授权项目子目录内运行开发工具（`direct` 模式）及受控 `powershell` / `cmd`（禁止直接对盘符根目录执行）。
- **每工作区独立 Windows 写受限沙箱**：基于 `CreateRestrictedToken(WRITE_RESTRICTED)` 与按工作区规范路径 SHA-256 派生的独立 SID，仅允许写入当前项目子目录与单任务隔离临时目录，彻底隔离不同项目与父子目录之间的写权限；配合 Windows `JobObject` 施加内存、进程数与超时限制，失败即拒、绝不静默降级无沙箱。
- **三档风险分流与自动置顶审批弹窗**：
  - 常见只读与项目构建/测试命令自动放行，快速命令短等待内直接返回 HTTP 200；纯只读直调命令跳过工作区写 ACL 毫秒级返回。
  - 首次在项目内使用 Shell、安装/变更依赖或 `git push` 时，自动从托盘唤醒并临时置顶本机审批窗（120 秒未确认自动拒绝），支持「仅允许这一次」或按项目 4 小时会话免打扰（可在「文件权限 → 收回会话授权」一键撤销）。
  - 触碰敏感目录、命令含跨目录路径（`..`）、强制推送、递归/强制删除、系统管理工具、混淆脚本或请求 `unsandboxed` 时，强制单次确认且隐藏会话免打扰。
- **设备命名与改名回执**：支持在「连接」页为多台电脑设置独立设备名称，并自动生成改名说明与 Muse 回执确认流程。
- **日志筛选与关于页更新日志入口**：日志页新增「命令执行」筛选；「关于」页在「前往 GitHub」左侧新增「更新日志」按钮，直达仓库最新更新日志。

### 安全与稳定性加固
- **句柄级文件安全校验**：所有 Windows 文件读取与原子写入均经打开的操作系统句柄复核真实路径与属性，支持 Windows 短路径别名并防范符号链接/联接点逃逸与校验后路径替换（TOCTOU）。
- **默认敏感路径分级保护**：默认屏蔽凭证目录（`~/.muse-bridge`、`~/.ssh`、`~/.aws`、`~/.gnupg`），并对系统目录、启动项、Shell Profile、`.gitconfig` 及程序自身目录启用默认写保护。
- **托管子进程树自动回收**：通过 Windows `JobObject` 托管 `Everything.exe`、`cloudflared` 与命令子进程，退出或关闭开关时自动强杀整棵进程树，杜绝孤儿进程残留。
- **令牌脱敏与权限保留**：审计与运行日志自动脱敏 `mfa_` 专用令牌；「文件」日志默认包含 `/agent/v1/*` 请求；切换文件夹/磁盘授权模式时准确保留已有权限边界；修复 base64 写入 10 MiB 大小限制校验；收紧应用 CSP；升级 `url` 依赖修复 `RUSTSEC-2026-0592`。
- **关闭命令执行时返回明确错误码**：未开启命令执行时统一返回 HTTP 403 与 `{"error": "exec_disabled", "message": "command execution is disabled"}`。

### 工程与测试
- 新增 Windows GitHub Actions 构建与测试工作流（含格式化门禁、前端构建、Rust 测试、符号链接/联接点逃逸与写受限沙箱内核测试）。

## [v0.1.0] - 2026-11-24

### 首个版本发布
- **首次配置引导与文件授权**：支持选择指定文件夹或本机固定磁盘授权，后台建立 Everything 文件名索引并支持按需重建索引。
- **Muse 专用凭证与文件接口**：提供独立于旧版 File Bridge 的 `/agent/v1` 接口与专用 `agent-token`（支持生成、复制、定位、轮换与撤销），支持搜索、列目录、读取、写入与建目录。
- **公网隧道与托盘管理**：默认支持 Tailscale Funnel，可选 Cloudflare 临时隧道与命名隧道；支持系统托盘启停连接、访问统计与日志查看、简体中文 / English 双语界面切换。
- **Windows 分发形态**：提供免安装便携版 ZIP、NSIS 安装程序及 `SHA256SUMS.txt` 校验文件。
