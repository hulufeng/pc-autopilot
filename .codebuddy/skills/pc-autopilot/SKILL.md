---
name: pc-autopilot
description: 用 clawdcursor（MCP 桌面自动化）操作本机 Windows 应用（主要是 LabCode / codelab-desktop、TrieCode）做 GUI 验证与点击的可靠手册。覆盖窗口遮挡、原生对话框不渲染、截图被过滤、PowerShell 转义、a11y 截断、Electron 前后端热更新边界等踩坑。当任务涉及「操作我的电脑」「点按钮看界面」「打开工程」「GUI 自动化截图确认」「验证某面板/弹窗是否崩」时使用。本技能带 learnings.md 知识库，越用越聪明。
---

# pc-autopilot（操作我的电脑）

用 clawdcursor 在本机真实操作桌面应用、做 GUI 验证。本机常年有 LabCode（codelab-desktop，pid 会变）、TrieCode（pid 27052）、豆包等窗口。**GUI 验证必须以文本为准，绝不能凭"看到"的截图下结论**——本会话模型无多模态，截图图像会被过滤，只有坐标元数据可用。

## 何时用
- 用户说"操作我电脑""帮我点一下""打开 XXX 看下""截图确认效果""验证某个面板/弹窗没崩"。
- 要对比 TrieCode / LabCode 的 UI 布局、确认重构后的界面渲染正确。

## 铁律（先读这条，能避掉 90% 的坑）
1. **截图只能给坐标，不能给内容**。返回的图像被过滤，凭它描述 UI 会编造。GUI 结构一律用 `smart_read`（a11y 树）/ `compile_ui` 的**文本结果**确认；点击用截图/`get_windows` 提供的坐标。
2. **先 focus 再点**。LabCode 常被豆包/浏览器压住，坐标点到别的窗口。每次操作前 `focus_window(pid)`（用 `get_windows` 拿最新 pid），点击用 `space:"screen"`。焦点被抢时用 Win32 `FindWindow`/`SetForegroundWindow` 强置前。
3. **PowerShell 内联命令靠不住**。`$env:`/`$_`/`$_` 等变量会被工具链转义吞掉。一律写成 `.ps1` 文件再 `powershell -File` 执行。npm 必须用 `cmd.exe /c npm run xxx` 启动，不能直接 `.exe`。
4. **a11y 快照会截断**（约 96 元素），底部面板常被截。小范围看渲染用 `desktop_screenshot_region` 裁小区域；读结构用 `smart_read` 带 `target` 关键词（如 "已安装平台"）定向抓。
5. **图标按钮 a11y 名为空**（`[Button] ""`）。靠相邻有名字的按钮坐标偏移来判断新按钮位置。
6. **Electron 热更新边界**：HMR 只刷渲染层；改了 `src/main/*`、`preload/*`、IPC 注册后必须**整进程重启** dev，`Ctrl+R` 重载不够。

## 可靠操作流程（照着走）
1. `get_windows` 拿目标 pid（LabCode pid 每次重启会变，别记死）。
2. `focus_window(pid)`。
3. 用 `smart_read`（或 `find_element`/`a11y_get_element`）按文本定位要点的元素，拿坐标。
4. `mouse_click`（`space:"screen"`）→ `wait` 短延时 → 再 `smart_read` 验证状态变化（文本出现/消失/数值变化）。
5. 大范围布局验证：`desktop_screenshot_region` 裁小图后**只看坐标是否落在预期区域**，细节读 a11y。

## 打开工程（绕过原生对话框）
原生 `showOpenDialog` 在本机 Shell 视图不渲染、脚本/剪贴板都驱动不了。两条路：
- **首选**：渲染层调 `window.api.project.openPath('<绝对路径>')`（主进程 `project:openPath` IPC，按明确路径打开）。可用 CDP（`relaunch_with_cdp` + `cdp_evaluate`）在渲染上下文执行，或点应用内自带的"打开"路径选择器。
- ❌ **已证伪**：`LABCODE_PROJECT` 环境变量 / `%TEMP%\labcode_project.txt` 都无效——`src/main/index.ts` 里没有这个读取逻辑，别再试。
- 路径尽量纯 ASCII（ESP-IDF 在 Windows 要求 ASCII 工程路径，含中文会静默失败）。

## 重启 LabCode dev（前后端改动后必做）
杀掉 `electron-vite` + `dev` 命令行的 node 进程树，再 `cmd.exe /c npm run dev` 隐藏窗口重启：
```powershell
# restart-labcode-dev.ps1
$procs = Get-CimInstance Win32_Process -Filter "Name='node.exe'" | Where-Object { $_.CommandLine -match 'electron-vite' -and $_.CommandLine -match 'dev' }
foreach ($p in $procs) { taskkill /PID $p.ProcessId /T /F 2>$null }
Start-Sleep -Seconds 2
Start-Process -FilePath 'cmd.exe' -ArgumentList '/c npm run dev' -WorkingDirectory 'D:\work\daimaLABshiyanshi\codelab-desktop' -WindowStyle Hidden
```
启动后等 ~20s 再 `get_windows` 找新 pid。

## 不靠 GUI 的确定性验证（更快更稳）
很多"是否崩/数值对不对"不用开界面，直接跑代码：
- **Node v24 原生支持 TS**：`node x.ts` 可直接 `import` 项目里的 `.ts`（用 `file:///` 绝对 URL 导入，否则 Windows 绝对路径报错 `ERR_UNSUPPORTED_ESM_URL_SCHEME`）。
- 例：Kconfig 解析器修复 → 写一个最小合成 Kconfig（含 `if` 块 + 嵌套 config）喂 `parseKconfig`，确认 `if` 块生成 `menu` 节点、嵌套正确、无 `Cannot read properties of undefined (reading 'push')` 崩溃。
- 例：库搜索版本号 → 直接用 `arduino-cli lib search Wire --format json` 看 JSON 里 `latest.version` 字段，再核对映射函数是否取到。

## 本机已知事实（验证基线）
- **arduino-cli** 路径：`%APPDATA%\codelab-desktop\tools\arduino-cli\arduino-cli.exe`（版本 1.5.2-rc.1）。
- **ESP-IDF 框架**：`C:\Espressif\frameworks\esp-idf-v5.4`（靠 glob 探测，无 `IDF_PATH`）。
- **LabCode 设备面板子标签**：`设备 / 配置 / 目标 / 平台 / 库`（DevicePanel.tsx `activeTab`）；ESP-IDF 面板是主区标签页 `espressif 目标与工具链` / `espressif Kconfig 配置`。
- **已修的真 bug**（静态检查看不出，必须真机/实跑）：分区表保存 ENOENT（`.replace(/^"|"$/g).trim()` 顺序错，CRLF 残留引号）；Kconfig `if` 分支漏 `children:[]` 必崩；`searchLibraries` 未取 `latest.version` 导致版本号全空。
- **接的 ESP32-S3 板**：串口 COM30（CH340 桥 VID:PID 1A86:55D3），非原生 USB；烧录前对齐端口。

## 越用越聪明（知识库机制）
本技能配 `learnings.md`（同目录）。**每次用完**：
- 开头先读 `learnings.md`，把里面的避坑点当默认动作。
- 结束时把你这次新踩到的坑、新发现的本机事实、某流程更优的顺序，**追加**到 `learnings.md`（用追加，不覆盖旧内容）。
- 若发现某条旧经验已过时/错误（如上面的 `LABCODE_PROJECT` 证伪），**划掉并标注"已证伪"**而非删除，保留演进痕迹。
这样技能会随每次实战累积，越来越准。
