# app-analysis 使用文档

## 一、作用与功能

`app-analysis` 是一个面向 **海外 Android App** 的调研与取包 Skill（不支持 iOS），支持三类任务：

1. **取包与校验（`download`）**：通过多渠道获取或导出完整 APK / APKS / XAPK，校验包名、版本、分包与签名。
2. **业务调研（`research`）**：分析指定功能或整款 App 的业务链路、功能入口、条件分支、字段权限与对应原版 UI，输出离线报告与 Mermaid 流程图（可同时输出图册）。
3. **UED 界面提取（`ued`）**：采集原生 / RN / Flutter / H5 原始页面、弹窗与状态截图，滚动页面拼接完整长图，生成可离线浏览的图册 `ui/index.html`。

---

## 二、环境依赖

- **Node.js** `>=22.12`
- **Python** `>=3.9`（长图拼接需 `Pillow`）
- **Android SDK**：`adb`、`emulator`、`aapt`、`apksigner`（及 `java`）
- **Android 设备或模拟器（必选其一）**：
  - 已授权且可响应的 Android 真机 / 在线模拟器；
  - 配置与系统镜像完整、且目录可写的 AVD。
- **按需分析工具**（工作流按技术栈自动检测或补齐）：`jadx`、`frida` / `frida-server`、`cmake` / `clang`（Flutter AOT `Blutter`）

---

## 三、安装

执行默认安装命令（默认安装到 `~/.agents/skills/app-analysis/SKILL.md`）：

```sh
npx -y --prefer-online app-analysis@latest skill install
```

如需安装到指定 AI 客户端目录，使用 `--dir` 传入目标绝对路径：

```sh
# Codex
npx -y --prefer-online app-analysis@latest skill install --dir "$HOME/.codex/skills/app-analysis"

# Claude Code
npx -y --prefer-online app-analysis@latest skill install --dir "$HOME/.claude/skills/app-analysis"

# Cursor / Gemini / Kiro 等同理，替换为对应客户端的 skills/app-analysis
```

> 安装后在后续新任务执行时会自动同步更新脚本与工作流资源；若本地 `SKILL.md` 有更新，按客户端要求重新加载 Skill 即可。

---

## 四、使用

在 AI 客户端中直接给出 **国家/市场 + App 名称**（有包名/链接也可以直接附带）和想做的事即可，AI 会自动补齐其余信息并按流程执行：

### 1. 只下载 / 导出 APK
> 下载菲律宾的 Cashola APK。

> 从我连接的手机导出 Cashola 的安装包。

### 2. 业务调研
> 分析墨西哥的 HolaFina 借款流程、入口和字段权限。

> 对比菲律宾的 Cashola 和 Cashify 征信授权流程（顺便整理 UI 图册）。

### 3. UED 界面提取
> 提取印尼 Kredivo 的全部界面截图，做成 UED 图册。
