# 原包运行与按需 Mock

业务调研和 UED 提取共同使用本文件准备原实现与状态；待打开的页面/路由、H5 覆盖策略和采集状态来自 [共用页面提取模块](ui-extraction.md)。本文件不另维护页面列表。场景覆盖及停止条件遵循必读的 [调研闭环与完成条件](completion.md)。

## 第 6 步：先取得原包基线

先按 [设备选择顺序](network-and-devices.md#设备选择顺序) 确定运行设备，不能直接沿用取包设备；明确其 serial、SDK/ABI、屏幕与 WebView 环境，核对第 4 步样本与该设备适配。所选运行设备已安装且能核对为第 4 步同一原包时直接复用，需要时才安装完整分包；不要为采集基线重复卸载或覆盖用户已有状态。先看未修改 App 的启动、声明、权限、登录与首页，记录版本/哈希、前台 Activity、截图、日志和实际网络状态。

改变安装/权限/代理/调试能力或测试缓存前，记录原状态、授权范围及恢复动作；在任务专用环境隔离场景。变更会删除用户数据且无法无损恢复时先交用户决定，不把“结束后再装回 APK”等同于恢复了数据。

第 6 步发现运行 profile 与取包条件不一致时，按关联取包规则返回第 3/4 步，保留原证据和搜索预算。使用调试副本前检查条目比对结论；`requires_manifest_review` 须先完成 [Manifest 语义验收](package-analysis.md#manifest-语义验收)，再将其页面当作原实现证据。

业务调研/UED 提取授权包含任务专用环境中的首次隐私同意、一般服务说明、用途告知和测试权限操作，记录后直接推进，不逐按钮询问；用户已有明确授权继续有效。无个人资料的隔离测试环境可按场景测试允许/拒绝分支，不将设备弹窗一律当成用户阻塞。真实 OTP、身份资料和生产交易改用隔离场景，缺少用户输入不妨碍继续代码和 Mock 分析。

通过自然点击记录入口。原生导航、Deep Link、Bridge 打开 H5 和直接 URL 访问是不同进入方式；后两者可用于定位，但不能替代真实入口证据。用户没有账号时，先留下登录门槛与匿名可见内容，再准备本地场景。

原包无法运行时，先按证据区分：分包/ABI/SDK、网络或账号地区、签名改变、设备能力、SDK 授权、资源缺失、应用自身错误。不要一看到登录或空白就默认反模拟器，不修改未知开关碰运气。模拟器本身起不来时，先按[启动已配置的 AVD](network-and-devices.md#启动已配置的-avd) 核对目录写权限与快照标记；`A snapshot operation ... is pending` 是权限问题，换 AVD 或清数据都不解决。

## 第 7 步：按阻塞所在层选择 Mock

| 所在层 | workspace 中验证过的方式 | 成功条件 |
|---|---|---|
| App 内 WebView | ADB 转发实际 WebView DevTools socket，Playwright/CDP 连接；在该 context 拦截数据请求，使用原 Bridge/存储 API 准备演示会话、配置、草稿；遇 PairIP/防重签无法改 `debuggable` 副本时，保持原包（安装加 `-i com.android.vending`），通过任务模拟器网络代理（含系统 CA/TLS 绕过）或**在 root 测试环境用 Frida 运行时调用 `WebView.setWebContentsDebuggingEnabled(true)`** 挂载调试 | 使用原文档/JS/CSS，目标位于前台、visible/attached；页面行为由原组件处理 |
| Android 原生状态卡/表单 | 定位 ViewModel/Repository/状态数据入口，在原运行时提供结构一致的数据；可调试副本可用 JDI 调用原 `LiveData.setValue`；**遇 PairIP/不可重签时改为在 root 测试环境用 Frida 注入原包，Hook `SharedPreferences`/`MMKV`/Repository/`OkHttp` 拦截器或 `ViewModel` 数据入口** | 原 RecyclerView/布局实际显示状态，不能拿网页 Mock 或直接布局预览代替 |
| RN/Hermes | 在实际 Hermes 运行时提供本地服务数据/条件，保留原组件和 Store API；遇 PairIP/不可重签时保持原包，通过 root 测试环境 + 系统 CA/Frida Hook `AsyncStorage`、`NativeModule` 或底层 `OkHttp`（`NetworkingModule`）注入输入并等待原 UI 队列稳定 | 原组件产生状态/弹层；原始服务合同和边界条件一致 |
| 独立 H5 原页面 | 当已取得 H5 源码快照时，可在独立浏览器加载原 HTML 与静态依赖，全拦截 API 并模拟当前 App 实际 Bridge 辅助排错与枚举路由；**注意把“业务 API 鉴权失败（如 `110001`）”与“静态 CDN 资源可访问性”分开核查**。**但独立浏览器 H5 绝不能替代 App 壳内 WebView 实走**：只要本机存在可用系统镜像，必须继续按[设备与运行时工具缺口](network-and-devices.md#设备与运行时工具缺口)通过隔离 AVD `-ramdisk` root + Frida 挂载原包壳内 WebView 完成走查，不得只跑完浏览器 H5 就停下交付 | 可证明该 H5 的交互与状态流转；只作为 `browser` 模式辅助或兜底证据单列，不能跳过 App 壳内 WebView 实测，也不能声称是 App 内实拍 |
| Flutter 原生页面 | 先定位 `libapp.so` 与平台通道合同（可用 `Blutter`/反汇编，或版本不匹配时先用符号/字符串检查配合 Frida 只读观察平台插件读取的键名）；**若无 PairIP/防重签限制可直接按调试副本路径运行，遇 PairIP/不可重签时保持原包，基于本机已有镜像启动 `-ramdisk` root 隔离 AVD，用 Frida Hook 未混淆的 Java 平台通道/存储插件（如 `LegacySharedPreferencesPlugin.getAll`、`SharedPreferencesPlugin`、`flutter_secure_storage`、自定义 MethodChannel）注入隔离会话/状态**，在断网或隔离代理下由原 Flutter 代码完成渲染与跳转；需要拦截 Dart HTTPS 请求时须使用系统 CA 或 Frida TLS/BoringSSL Hook | 原 Flutter 页面实际完成初始化、状态渲染与自然跳转；不能以重写 Flutter 页面、H5 替代或零散字符串推断冒充实测；仅当实测隔离 root + Frida 注入仍被强反调试/服务端硬校验阻断后才记限制 |
| 相机/OCR/活体/评分等外部 SDK | 尽量观察原组件与实际错误；需要业务继续时可提供本地回调继续下游 | 回调成功只验证客户端下游，不证明 SDK 真正完成识别、活体或评分表单展示 |

### Mock 的不变量

1. 只模拟服务响应、会话、配置、时间/缓存或业务数据；不重画界面，不改原 JS/CSS/DOM 布局、原生布局资源或按钮处理逻辑来制造目标结果。
2. 按第 5 步的范围/状态候选矩阵选场景，再从原代码读取条件和接口合同；fixture 列出输入、预期页面、预期下一跳和反例，避免“先想一个结论，再把结果注入”。第 8 步回填实际观测，第 9 步独立回查原依据。
3. 尽量在数据输入层复现场景。直接注入终态、直接打开路由只能证实状态可渲染，不能独自证明上游流程、提交或条件成立。
4. 接入假会话/资料前准备网络隔离。真实账户、OTP、借款、还款、征信查询、外部活体等操作不得因为测试触发到生产；只放行已确认需要的静态资源，其他业务请求默认拦截/阻止。
5. 未实现 API 要明确报错和记录；不要全局返回 `{}` 或统一成功后宣称路径走通。缺字典、配置、枚举和异步初始化必须补合同而非强改界面。
6. 使用合成资料，不从个人设备复制真实短信、联系人或身份信息来填 fixture。不把生产 token 放入脚本、日志或报告。
7. 单个 scenario 隔离会话/缓存和响应状态；动态后端 fixture 随原点击/请求推进，保留完整请求序列和未处理项。先跑主线，再按闭环规则补必要的正反、失败、返回与续办分支；主线走通不能关闭整项调研。
8. 等待可观察条件（目标 Activity/文本、请求完成、渲染稳定、UI 队列空闲），不要仅固定 sleep 后截图。保留初次失败与最终断言，不能删除失败记录装成一次成功。

## 防重签、证书限制与运行时注入底座

原生 Android、RN 与 Flutter 在现代样本上常同时叠加三类障碍：**PairIP/防重签（不能改 `debuggable=true`）**、**证书不信任用户 CA**、**未登录接口拦截**。这三者不构成“必须真实账号/OTP”的充分条件，通用应对底座如下：

1. **区分“重签改包”与“运行时注入”**：无 PairIP/强签名校验时仍可按调试副本或代理流程推进；仅当确认存在 PairIP/防重签保护时才切入免重签路径——**运行时 Frida 注入不需要修改或重签原 APK**。遇 PairIP 校验安装来源时保持原包并用 `-i com.android.vending` 安装；若当前 AVD 为未 root 的 `user` 构建（如 `google_apis_playstore`），直接按[设备与运行时工具缺口](network-and-devices.md#设备与运行时工具缺口)复用本机已有系统镜像新建隔离 AVD、用 Magisk 修补 `ramdisk.img` 副本（`-ramdisk` 启动）并运行匹配 ABI 的 `frida-server`，**无需暂停请求用户授权，也不要下载新系统镜像**。启动原包通过首次完整性检查后，优先使用 `frida -U -p <pid>` 附加（避免 `-f` spawn 阶段触发 PairIP 崩溃）。
2. **证书信任限制与离线存储绕过**：Android 7+ 默认所有应用（含原生 `OkHttp` 与 RN `NetworkingModule`）均不信任用户安装的代理 CA（除非配置了 `network_security_config`），Flutter Dart `HttpClient` 更进一步仅读取系统内置根证书路径。需要拦截 HTTPS 业务接口时，必须在可写/root 测试 AVD 写入系统 CA 或用 Frida Hook TLS/`TrustManager`/BoringSSL；若目标页只需登录会话驱动（如本地 `token`/`phone`/`custNo`），优先在**设备断网隔离下 Hook 客户端本地存储/平台通道读取返回值**，绕过登录 HTTPS 接口。
3. **跨栈寻找“低混淆边界”**：业务代码无论混淆成 ProGuard、Hermes 字节码还是 Flutter AOT（`libapp.so`），持久化存储、系统桥接与网络发送通常仍经过标准类/插件边界：
   - **Flutter**：优先查 `LegacySharedPreferencesPlugin.getAll`、`SharedPreferencesPlugin`、`flutter_secure_storage` 及自定义 MethodChannel；
   - **原生 Android**：优先查 `SharedPreferences`、`MMKV`、`OkHttp` 拦截器或 `ViewModel`/`Repository` 入口（替代无 `debuggable` 时失效的 JDI）；
   - **RN**：优先查 `AsyncStorageModule`、`SharedPreferences`、自定义 `NativeModule` 与 `OkHttp` 网络模块。

## WebView 的具体经验

- 先定位当前 PID、WebView 调试 socket 和 `/json/list` 目标；App 原版未开放调试时，按 package-analysis 准备任务分析副本；若原包带 PairIP/防重签保护不能重签，保持原包（`-i com.android.vending`），直接按[设备与运行时工具缺口](network-and-devices.md#设备与运行时工具缺口)在 `-ramdisk` root 隔离 AVD 中先启动 App，再用 `frida -U -p <pid>` 附加并在主线程调用 `android.webkit.WebView.setWebContentsDebuggingEnabled(true)` 暴露 `@webview_devtools_remote_<pid>`；独立浏览器全拦截仅用于辅助验证 H5 逻辑，**不能代替原版 App 壳内 WebView 的挂载与实拍**。
- 若使用 Playwright `chromium.connectOverCDP` 连接 Android WebView DevTools 遇到 `Browser.setDownloadBehavior` 等上下文管理不支持报错，或 CDP `Fetch.requestPaused` 在 `Page.reload` 时发生死锁，不要误判 WebView 不可调试；改用原生 CDP WebSocket 在页面内直接 Hook `XMLHttpRequest.prototype.open`/`send`、`window.fetch` 与 Bridge（如 `window.flutter_inappwebview.callHandler`）。
- **壳内 WebView 启动时序、Bridge 锁定与加密开关重置**：与空白浏览器不同，原版 App 壳内 WebView 在首次加载文档时，前端状态库（如 Pinia/Vuex/Redux）会在挂载前调用原生 Bridge（如 `getUserInfo`、`isHttpEncrypt`）完成一次性内存初始化（例如将未登录态或接口加密标志写入响应式 Store 与 `localStorage`）。因此：
  1. **不能只在页面挂载后改 `localStorage` 或切路由 hash**：已挂载的内存 Store 不会重新读取 `localStorage`，且单纯 `Page.reload()` 会让原生 WebView 插件重新注入原生 Bridge 覆盖普通赋值；
  2. **必须在初始化阶段锁定 Bridge 或从 Java 层 Hook**：要么在 Frida Java 层直接 Hook 宿主 Bridge/存储插件返回值，要么在 CDP `Page.addScriptToEvaluateOnNewDocument` 中用 `Object.defineProperty` 锁定 Bridge 调用入口（防止重载时被原生插件覆盖）并写入登录态 `localStorage`，再执行一次重载使前端从启动阶段就拿到模拟会话；
  3. **注意布尔/字符串标志的真值陷阱**：前端从 `localStorage` 读取加密等开关时（如 `localStorage.getItem("crypt") || ""`），写入字符串 `"false"` 在 JS 中仍为真值（`Boolean("false") === true`），会导致前端继续按密文发送请求并在收到明文 Mock 时报解析/请求错误；关闭此类开关必须置为空串 `""` 或 `removeItem`（同时令 Bridge 加密查询返回关闭值），并清理首次启动残留的未登录缓存（如 `homeData`）及处理壳内重复底栏。
- 多个 WebView 可能同时存在，旧 page 对象可能指向后台。每个场景都核对前台包/Activity、目标 URL，以及 DevTools description 的 visible、attached；不能选择列表第一个目标就截图。
- 使用 `window.Android`、`window.mocamocaJs` 等**当前 App 实际存在**的 Bridge，不把历史方法名写死为新 App 通用 API。
- 保留原始静态依赖和哈希。浏览器原组件预览可能缺原生标题栏、安全区、字体、键盘和桥接行为；报告明确它是浏览器证据，优先在 App WebView 补实拍。

## 原生与 RN 的具体经验

FT 首页 F9 曾需要 JSON 卡片列表进入原 HomeViewModel 数据层；WebView 的 route.fulfill 无法影响该页面。应先识别 UI 宿主和数据模型，再决定工具。模板布局存在只能证明资源存在，不能直接算真实页面。

Duxi 的 RN/Hermes 试验保留原 Store/组件，用 fixture 控制状态、开关、订单、次数和冷却期。需要校验缺订单、开关关闭、重复订单、不同评价选择等反例。调用了 Native Review API 但没有 Google 评分表单，只能记录 API 已调用；不能把两者等同。

## 复现记录与恢复

记录每个状态：来源包/版本/代码哈希、设备/宿主、真实或合成数据、fixture、拦截层、入口操作、响应序列、断言、截图、未覆盖项。分别标记：

- UI 来源：App 原始实现 / 原 H5 浏览器运行 / 组件或布局预览 / 重绘示意。
- 执行模式：真实只读观察 / 本地合成数据 / SDK 预览或回调替代。
- 代码资源一致性：哈希/条目比对结果；尚未比对则继续核对，不能作为已通过场景。
- 后台事实：是否有真实服务证据；Mock 的审核成功绝不意味着真实批准/到账。

恢复按执行前快照进行，不能套用历史 `emulator-5554`、代理 `10.0.2.2:7897` 或端口值。只移除本任务创建的 CDP/JDWP 转发和测试副本；需要重装原包时先确认设备数据处置授权，再比对设备与原包哈希。保留截图、fixture 和证据日志。
