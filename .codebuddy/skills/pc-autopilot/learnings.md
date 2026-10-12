# learnings.md — pc-autopilot 实战知识库

> 越用越聪明：每次操作电脑后把新坑/新发现**追加**到末尾，旧经验过时了**划掉标注"已证伪"**而非删除。
> 加新条目用格式：`- [日期] 一句话结论。细节…`

---

## 已验证的避坑点（默认动作）

- [2026-10-12] 截图图像被过滤，模型无多模态。GUI 结构只用 smart_read/a11y/compile_ui 文本确认；点击只用坐标。凭截图描述 UI 必编造。
- [2026-10-12] 每次操作前 get_windows 拿最新 pid 再 focus_window，点用 space:"screen"。LabCode 常被豆包/浏览器压住导致坐标点错窗口。
- [2026-10-12] PowerShell 内联 `$env:`/`$_` 会被工具链转义吞掉。一律写 .ps1 再 `powershell -File`；npm 用 `cmd.exe /c npm run xxx`。
- [2026-10-12] a11y 快照约 96 元素上限会被截断，底部面板看不全。看渲染用 desktop_screenshot_region 裁小图（只看坐标落点），读结构用 smart_read 带 target 关键词定向抓。
- [2026-10-12] 图标按钮 a11y 名为空 [Button] ""，靠相邻有名字按钮的坐标偏移定位。
- [2026-10-12] Electron HMR 只刷渲染层；改 src/main/*、preload/*、IPC 注册后必须整进程重启 dev（杀 electron-vite dev node 树再 npm run dev），Ctrl+R 不够。
- [2026-10-12] 原生 showOpenDialog 在本机不渲染、脚本驱动不了。打开工程用 window.api.project.openPath('<绝对路径>')（主进程 project:openPath IPC），或点应用内路径选择器。路径尽量纯 ASCII（ESP-IDF 要求）。
- [2026-10-12] Node v24 原生跑 .ts：`node x.ts` 可 import 项目 .ts；跨文件 import 必须用 `file:///` 绝对 URL，否则 Windows 绝对路径报 ERR_UNSUPPORTED_ESM_URL_SCHEME。
- [2026-10-12] 不靠 GUI 的确定性验证更快：Kconfig 修复用最小合成 Kconfig 喂 parseKconfig 确认 if 块生成 menu 节点且无 undefined.push 崩溃；库版本号修复直接看 arduino-cli lib search 的 latest.version 字段。
- [2026-10-12] 设备面板「库」页验证：版本号在 a11y 树里是**独立文本节点**——标签 "· 最新" 与真版本号（如 "1.0.0"）被 JSX 拆开渲染。搜 "最新" 只命中标签，要确认版本号得 find_element 搜 "1." 抓具体版本文本。已修 searchLibraries 未取 latest.version 导致版本号全空（之前每行都显示"最新 —"）。
- [2026-10-12] 设备面板操作路径已跑通：活动栏点「设备」图标 → 子标签 设备/配置/目标/平台/库；平台页确认 esp32:esp32 已安装，库页搜 Wire 出结果且带版本号。说明平台/库页渲染 + 库搜索修复 OK。
- [2026-10-12] 重启 LabCode dev 用 `C:\Users\Administrator\AppData\Local\Temp\restart-labcode-dev.ps1`（`powershell -File` 执行），杀 electron-vite+dev node 树再 `cmd.exe /c npm run dev` 隐藏窗重启；启动后等 ~20s 用 get_windows 找新 pid（pid 每次变，别记死）。
- [2026-10-12] 本机到 GitHub 的网络：git push（HTTPS→github.com:443）能通，但 `gh` 的 OAuth 令牌交换端点（/login/oauth/access_token）间歇性 TCP 超时，`gh auth login` 设备流常卡在最后换令牌那步（"failed to authenticate via web browser: ... connectex ... failed to respond"）。最稳推送法：去 GitHub 生成 classic PAT（勾 `repo`，**无期限**），用 PowerShell `Invoke-RestMethod -Uri https://api.github.com/user/repos -Method Post -Headers @{Authorization="Bearer $token"} -Body (@{name=...;private=$true;description=...}|ConvertTo-Json)` 建私有仓库；再 `git -c credential.helper= push "https://${token}@github.com/<user>/<repo>.git" master`，推完 `git remote set-url origin https://github.com/<user>/<repo>.git` 把令牌从配置去掉。⚠️ 别用 `curl.exe -d` 传 JSON——PowerShell 转义会破坏导致 "Problems parsing JSON"，且 `@file` 读文件在本机报 "error reading file"；`Invoke-RestMethod` 最稳。
- [2026-10-12] pc-autopilot 仓库的 `origin` 已**内嵌 PAT**（仅存本地 `.git/config`，不进被推送的仓库），故以后直接 `git push origin master` 即可、无需 gh 登录/再要令牌。PAT 为**无期限** classic token（repo 权限），长期有效；若将来在 GitHub 撤销/轮换该 token，重新生成后执行 `git remote set-url origin "https://<新TOKEN>@github.com/hulufeng/pc-autopilot.git"` 即可。
- [2026-10-12] 【自测标记】本行由 auto-merge 测试 PR 自动合入，用于验证自主迭代闭环可用；验证通过后已清理删除。

## 本机事实（验证基线）

- [2026-10-12] arduino-cli：`%APPDATA%\codelab-desktop\tools\arduino-cli\arduino-cli.exe`（v1.5.2-rc.1）。不在 PATH，LabCode 内部靠此路径解析。
- [2026-10-12] ESP-IDF 框架：`C:\Espressif\frameworks\esp-idf-v5.4`，无 IDF_PATH，靠 glob 探测。
- [2026-10-12] LabCode 设备面板子标签：设备/配置/目标/平台/库；ESP-IDF 面板是主区标签页 espressif 目标与工具链 / espressif Kconfig 配置。
- [2026-10-12] 接的 ESP32-S3 板：串口 COM30（CH340 桥 VID:PID 1A86:55D3），非原生 USB；烧录前对齐端口。
- [2026-10-12] TrieCode pid 稳定 27052，工程目录 C:\Users\Administrator\Documents\TrieCodeProjects；活动栏=文件/源代码管理/插件/Arduino CLI 工具链/ESP-IDF 工具链。

## 已证伪（踩过，记下来别再试）

- [2026-10-12] ~~`LABCODE_PROJECT` 环境变量 / `%TEMP%\labcode_project.txt` 让 LabCode 启动即开工程~~ → 已证伪：src/main/index.ts 无此读取逻辑，写入后"尚未打开项目"依旧。改用 project:openPath IPC。

## 待验证 / 存疑

- [2026-10-12] ESP-IDF Kconfig 面板在 GUI 里尚未用真实工程跑通（只做了合成 Kconfig 确定性验证）。下次开一个含 Kconfig 的 ESP-IDF 工程（如 D:\work\projects\s3-ov2640-face-detect——但它根目录无 Kconfig，需确认真正的 ESP-IDF 配置型工程路径）再 GUI 复核一次。
