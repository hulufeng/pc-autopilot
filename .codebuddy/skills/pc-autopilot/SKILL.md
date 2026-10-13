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
4. **a11y 快照会截断**（约 120 元素，按 y 从上到下、超出即截），底部面板常被截。小范围看渲染用 `desktop_screenshot_region` 裁小区域；读结构用 `smart_read` 带 `target` 关键词（如 "已安装平台"）定向抓；**输出/日志类自定义滚动面板 UIA 读不到时，用 `compile_ui` 带 `max_cost:"ocr_ok"` + `target_text` 融合 OCR 拿文本**（见下）。
5. **图标按钮 a11y 名为空**（`[Button] ""`）。靠相邻有名字的按钮坐标偏移来判断新按钮位置。
6. **Electron 热更新边界**：HMR 只刷渲染层；改了 `src/main/*`、`preload/*`、IPC 注册后必须**整进程重启** dev，`Ctrl+R` 重载不够。

## 可靠操作流程（照着走）
1. `get_windows` 拿目标 pid（LabCode pid 每次重启会变，别记死）。
2. `focus_window(pid)`。
3. 用 `smart_read`（或 `find_element`/`a11y_get_element`）按文本定位要点的元素，拿坐标。
4. `mouse_click`（`space:"screen"`）→ `wait` 短延时 → 再 `smart_read` 验证状态变化（文本出现/消失/数值变化）。
5. 大范围布局验证：`desktop_screenshot_region` 裁小图后**只看坐标是否落在预期区域**，细节读 a11y。

## 打开工程 / 原生对话框
- **原生 `showOpenDialog` 在本机其实能被 UIA 驱动（已证伪"驱动不了"）**：TrieCode「文件 → 打开项目」弹的是标准 Win32 `IFileDialog`，`find_element controlType:ListItem` 列出全部项目、`controlType:Button` 列出「选择文件夹」(id:1)/「取消」(id:2)，`smart_click` 按名 invoke 即可完成打开；也能看到 `帮助(&H)`/`组织`/`新建文件夹`/`后退`(id `Item 256`)。LabCode 是否同款待验证，但**别再假设原生对话框"读不到/点不了"**——先用 UIA 探一下。
- **首选**：若渲染层暴露 `window.api.project.openPath('<绝对路径>')`（主进程 `project:openPath` IPC），直接按明确路径打开最稳；或点应用内自带的"打开"路径选择器。
- ❌ **已证伪**：`LABCODE_PROJECT` 环境变量 / `%TEMP%\labcode_project.txt` 让 LabCode 启动即开工程——`src/main/index.ts` 无此读取逻辑。
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

## TrieCode 专属（GUI 取证 + 逆向要点）

TrieCode v1.3.1（`C:\Program Files\TrieCode\TrieCode.exe`，常驻 5 进程；**pid 每次变，别记死 27052**）是带 AI 编程智能体的 Arduino/ESP-IDF/STM32 IDE。完整静态+动态逆向见 `D:\work\TrieCode-分析报告.md`（A–N 十四块）。GUI 取证踩过的坑：

1. **窗口关了会退居托盘**：`MainWindowHandle=0`、`MainWindowTitle` 空。恢复**不要杀进程**——`Start-Process 'C:\Program Files\TrieCode\TrieCode.exe'` 二次启动即可（`requestSingleInstanceLock` + `second-instance → showOrCreateMainWindow`，pid 不变、不起第二实例）。别用 `start "" "路径"`（PowerShell 把 `start` 当 Start-Process 且空 FilePath 报错）。
2. **🔴 账号 token 过期 → UI 假"已登录"（真 UX 缺陷）**：`auth:restoreSession` 先把 config 缓存的 `accountUser` 立即填上并 `return`（UI 立刻显示邮箱/积分/退出按钮），**再异步** `/user/profile` 校验；校验失败才清 `accountToken`。结果：UI 显示"已登录"，但 `ai:getBuiltinModels` 因 `!accountToken` 返回 `[]`（模型下拉**只剩标题零条目**）、发消息回"登录已过期"、点刷新无用。**判据：模型下拉空 + 聊天报过期 + 账户页却有数据 = token 死了**。**干净重启应用若直接停在登录界面，即证明 config 无有效 token**（曾实测：重启后停 `登录 TrieCode AI` 界面）。修复只能**应用内重登**（设置→账户与计费→退出，再用用户名密码/邮箱验证码），**网页端 triecode.com 登录无效**（应用只认自己存的 token，无 refresh_token）。另一条路：`设置 → 模型 → 自定义模型` 配自有 API Key（BYOK，不依赖账号）。
3. **🔴 真 bug：arduino-cli daemon 跨会话"收养"导致 gRPC 通道从未建立**：设备面板选板卡报 `Error: arduino-cli 守护进程未连接。`，但平台/库面板正常、状态栏仍 `1/1 后端就绪`。根因 `connectToBackend()` 只在 `onReady` 调，而 `onReady` 仅当本会话亲自 spawn 后端时触发；收养上个会话 `--daemonize` 残留的 daemon 不触发 → 编译/上传/板卡/库全失效。**一键修复：强杀 arduino-cli daemon（pid 见 `%APPDATA%\TrieCode\backend-pids\arduino-cli-toolchain-default.pid`），看门狗 ~1s 后重 spawn，onReady 触发即恢复**。
4. **输出/日志面板 UIA 读不到**：自定义滚动区，`find_element`/`smart_read`/`copy_all_text` 全失败，`ocr_read_screen` 本机不可用。**唯一可靠：`compile_ui` 带 `purpose:"find_text"` + `target_text:"编译"` + `max_cost:"ocr_ok"`**（a11y+OCR 融合）。
5. **🔴 chat 输入框 = React 受控值自动化盲区（2026-10-13 实测收口）**：`type_text`(粘贴)/`smart_type`(键击)/`set_field_value`(原生 setter)都写不进受控值；点预设快捷指令(`检查串口输出`等)只改了**显示层文本**(a11y 里一个独立 [text] 节点)，底层受控 `<input>` 仍空 → `发送` 按钮 / `Enter` 读到空值、对话永不启动。`mouse_click 发送`/`key_press Enter`/`smart_click 发送(invoke_element)` 全无效。真人点击正常，纯自动化触发不了。**要真机观测一次完整 AI turn，只能 `relaunch_with_cdp` 进渲染层用 `cdp_evaluate` 直接调发送逻辑**（会重启丢登录态）。token 有效性可旁证：模型下拉能拉到 `Minimax MiniMax M2.7` 即 token 活。坐标坑：`mouse_click` 把入参 ×2 当屏幕坐标，传 a11y 坐标的一半（输入框中心 a11y 1303,1205 → 651,602）；`smart_click` 直接吃 a11y 坐标。
6. **编译不在 sketch 目录进行**：复制到 `%LOCALAPPDATA%\arduino\sketches\<32位哈希>\` 再编，**别用"有没有 build/ 目录"判断编译是否发生**。判据：`compile_ui` 读 `编译成功`/`上传成功`，或看 arduino-cli daemon CPU 增量（一次 ESP32-S3 编译约 +16s）。
7. **状态栏暴露后端就绪**：`1/1 后端就绪` + `arduino-cli-toolchain:default=ready`（`<pluginId>:<backendId>=<state>`）。确认后端是否活，读状态栏比 netstat 快。
8. **CDP 进不去**：`DevToolsActivePort` 可能是陈旧端口；clawdcursor 的 cdp_* 默认连 9223（代理自己浏览器）。要 CDP 进 TrieCode 得带 `--remote-debugging-port` 重启，代价大；用 UIA 定向读即可。

## 本机已知事实（验证基线）
- **arduino-cli（两套，别混）**：LabCode 用 `%APPDATA%\codelab-desktop\tools\arduino-cli\arduino-cli.exe`（v1.5.2-rc.1）；**TrieCode 用 `%APPDATA%\TrieCode\plugins\bin\arduino-cli\arduino-cli.exe`（v1.5.1，由 TrieCode 托管、动态端口）**。
- **ESP-IDF 框架**：`C:\Espressif\frameworks\esp-idf-v5.4`（靠 glob 探测，无 `IDF_PATH`）。
- **LabCode 设备面板子标签**：`设备 / 配置 / 目标 / 平台 / 库`（DevicePanel.tsx `activeTab`）；ESP-IDF 面板是主区标签页 `espressif 目标与工具链` / `espressif Kconfig 配置`。
- **已修的真 bug**（静态检查看不出，必须真机/实跑）：分区表保存 ENOENT（`.replace(/^"|"$/g).trim()` 顺序错，CRLF 残留引号）；Kconfig `if` 分支漏 `children:[]` 必崩；`searchLibraries` 未取 `latest.version` 导致版本号全空。
- **接的 ESP32-S3 板**：串口 COM30（CH340 桥 VID:PID 1A86:55D3），非原生 USB；烧录前对齐端口。
- **TrieCode pid 每次启动都变**（见过 27052/15264/26444…），arduino-cli daemon pid 也随之变；**永远用 `get_windows`/`Get-Process` 拿最新 pid，别记死**。工程目录 `C:\Users\Administrator\Documents\TrieCodeProjects`；活动栏=文件/源代码管理/插件/Arduino CLI 工具链/ESP-IDF 工具链。

## 越用越聪明（知识库机制）
本技能配 `learnings.md`（同目录）。**每次用完**：
- 开头先读 `learnings.md`，把里面的避坑点当默认动作。
- 结束时把你这次新踩到的坑、新发现的本机事实、某流程更优的顺序，**追加**到 `learnings.md`（用追加，不覆盖旧内容）。
- 若发现某条旧经验已过时/错误（如上面的 `LABCODE_PROJECT` 证伪），**划掉并标注"已证伪"**而非删除，保留演进痕迹。
- **自动同步社区经验**：若本技能目录自身是 git 仓库（即用 `git clone` 安装、而非复制文件），加载时先 `git -C <本目录> pull --ff-only` 拉取全网最新 `learnings.md`；别人提的 PR 被机器人自动合并后，你打开电脑即拿到他人经验。
- **元迭代（经验升方法）**：当某类坑在 `learnings.md` 反复出现（≥3 次）或同型问题高频，主动把它**提炼成上面「铁律」的新条目**——从零散经验升级为通用规则，技能不只堆数据、还会自我抽象。
- **社区贡献零摩擦**：其他人把新坑开 PR 到本仓库，GitHub Action 会自动合并「仅追加 `learnings.md`、且不含密钥」的 PR；改动 `SKILL.md` 或含密钥的 PR 不自动合、留人审。
这样技能会随每次实战累积，越来越准。
