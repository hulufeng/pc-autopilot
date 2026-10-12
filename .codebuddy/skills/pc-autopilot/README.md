# pc-autopilot

用 **clawdcursor**（MCP 桌面自动化）操作本机 Windows 应用（主要是 LabCode / codelab-desktop、TrieCode）做 GUI 验证与点击的**可靠手册 + 实战知识库**。

> 核心理念：**越用越聪明**。每次真实操作电脑后，把新踩的坑、新发现的本机事实、更优的操作顺序追加进 `learnings.md`；过时经验划线标注「已证伪」保留痕迹。所有人的经验汇聚，技能就越用越准。

## 包含什么
- `SKILL.md` —— 操作铁律与可靠流程（截图只能看坐标、先 focus 再点、PowerShell 写 `.ps1`、`a11y` 快照截断、原生对话框走 `openPath` 绕过、Electron 改主进程必须整进程重启…）。
- `learnings.md` —— 实战知识库。开头先读它当默认动作；用完追加新条目。
- `README.md` —— 本文件。

## 安装到你的工作区
```powershell
git clone https://github.com/hulufeng/pc-autopilot.git
cp -r pc-autopilot/.codebuddy/skills/pc-autopilot <你的工作区>/.codebuddy/skills/
```
（也可以只把 `SKILL.md` + `learnings.md` 拷过去。CodeBuddy 会在技能描述命中「操作我的电脑 / 点按钮看界面 / GUI 自动化 / 验证某面板是否崩」时自动加载。）

## 怎么「越用越聪明」
1. 让 AI 在触发 GUI 操作时加载本技能。
2. **每次实战后**，把新经验追加进 `learnings.md`（格式：`- [日期] 一句话结论。细节…`）。
3. 阶段性 `git commit && git push` 同步。

## 欢迎贡献（自主迭代）
你踩到的新坑、新发现的本机事实、更优的操作顺序——直接开 PR 把 `learnings.md` 的新条目合并回来。所有人的实战经验汇聚，技能就自动进化。

## 注意事项
- `learnings.md` 里的「本机事实」是基于原作者机器（LabCode / ESP32-S3 / COM30 / TrieCode pid 27052 …）的基线，**到你机器上请按实际情况核对、覆盖**。
- **不要往本仓库提交任何令牌 / 密钥**。原作者远程已内嵌其 PAT 于本地 `.git/config`，不在此仓库内。
- 截图图像会被模型侧过滤、只能给坐标不能给内容——这条铁律请务必遵守，否则 AI 会凭截图编造 UI 状态。

License: MIT（随意使用、修改、再分发）。
