# 网络、账号、工具与设备

## 设备选择顺序

第 1 步只读盘点设备并判断环境门槛，不按 ADB 返回顺序自动选机。第 2 步明确目标包与版本要求后，在实际导出、安装或运行前按以下顺序选择：

1. **用户指定优先**：遵从用户对当前动作指定的真机、模拟器或 serial，已有指定不重复询问。指定设备不可用或不能满足目标时，说明具体阻塞；已有备用设备授权则按授权继续，否则只询问必要的修复或换机选择，不静默换机。
2. **未指定，只读导出 APK**：在已连接、已授权且可响应的设备中，优先复用已安装目标 App 且满足版本要求的设备，真机和模拟器不另排先后。不为导出启动 App、重新安装或清除数据；没有合适已安装样本则继续下载子流程。此规则只决定设备导出路径内的选机，不改变本地归档等取包渠道顺序。
3. **未指定，安装运行、Mock 与 UI 走查**：默认使用任务专用模拟器。先复用匹配 SDK/ABI、满足所需服务且不占用其他任务的在线实例，再按需启动已有完整、适合本任务的 AVD，并核实 ADB shell 可响应。真机已连接或已安装目标 App，不会因此越过模拟器成为默认运行设备。
4. **模拟器受限时再用真机**：有证据表明模拟器存在 ABI/SDK、应用或 SDK 的兼容性限制，或本次验证需要其无法提供的真实相机、传感器、SIM 等能力时，记录依据并选择能满足要求的真机。仅有登录、网络或空白页面不足以判定反模拟器，先按 [运行故障分类](runtime-and-mock.md) 定位原因。没有可用任务专用模拟器且没有可启动的合适完整 AVD 时，记录环境可用性限制后也可使用已授权真机；不为满足默认偏好自动安装新模拟器。

取包设备和运行设备分别记录。第 3 步设备相关渠道先记录实际取包 profile（ABI、SDK、密度），与预期运行条件分别核对。跨设备运行不适配时，按 [关联取包规则](download-workflow.md#任务级尝试记录与去重) 以新 profile、原尝试 id 和证据理由重新获取；保留历史/搜索预算，不因样本来自真机就直接改用真机运行。多个候选满足当前规则时复用本任务已选设备；只有差异影响结果且无法从请求确定时才询问。

首次选机或换机时，在正文说明用途、设备类型、serial 和选择依据，证据记录关联实际设备。设备选择不扩大操作授权；个人/共享真机的新安装、升级、覆盖及数据处置仍遵循现有授权规则，已有授权不重复请求。

### 启动已配置的 AVD

设备门槛通过只表示“有完整 AVD 可准备启动”，不等于它一定能启动。启动前先看 `preflight` 给出的 `directory`、`directory_writable` 和 `snapshot_state`：

1. `directory_writable=false` 时不要启动。模拟器启动会更新或删除 `bootcompleted.ini` 等快照标记；当前会话写不进 AVD 目录时，日志会先是 `WARNING | Deleting .../bootcompleted.ini failed`，随后 `FATAL | A snapshot operation for '<名字>' is pending and timeout has expired.`。这是权限问题，不是镜像损坏、也不是 AVD 锁坏了。按 `AVD_NOT_WRITABLE` 报出的路径交用户处理：在其自己的终端启动该 AVD，或授予写权限后重跑第 1 步。
2. 需要自己启动且确实有写权限时，用已有 AVD、保留原快照，并让输出可回收：

   ```sh
   emulator -avd <已有AVD名> -no-snapshot-load -no-boot-anim
   adb -s emulator-5554 shell getprop sys.boot_completed
   ```

   `sys.boot_completed=1` 后才能在正文说该 AVD 已在线，并把 `boot_verified` 记为 true；`emulator-5554` 只是示例，以实际 serial 为准。命令要作为后台任务运行并保留日志，不要阻塞主流程；**严禁裸跑无超时的 `adb wait-for-device` 或无限 `while` 循环**（当模拟器卡在 `offline` 或 `/init` 崩溃时，`adb wait-for-device` 会永久挂起），必须使用带次数/时间上限的有界轮询（如最多 45 次 × 2 秒 = 90 秒）同时检查模拟器进程存活、`adb -s <serial> get-state` 与 `sys.boot_completed`，超时或进程退出时立即中止并读取启动日志排错，保留原始 `FATAL`/`WARNING` 行作为证据。

3. 用户已在自己终端启动的 AVD 只通过 `adb -s SERIAL` 交互：不要从受限会话 `kill`、重启或替换它，杀进程后本会话起不回来，只能再请用户启动。收尾时保持它在运行并明确说明交还，不主动关闭。
4. 快照可疑（`snapshot_state.artifacts` 含 `bootcompleted.ini`/`snapshots` 且上次未正常退出）时，加 `-no-snapshot-load` 冷启动一次即可；不要删除快照目录，也不要 `-wipe-data`。
5. 不要为了绕过目录无写权限（`AVD_NOT_WRITABLE`）去新建 AVD：目录不可写属于宿主权限问题，按第 1 条交用户处理。但若第 6/7 步因 PairIP/防重签、证书限制或无 `debuggable` 需要 root + Frida 注入，而当前只有日常未 root 的 `google_apis_playstore` AVD 时，**允许且应当直接复用本机已有系统镜像新建任务专用隔离 AVD，并通过 `-ramdisk` 挂载修补后的 `ramdisk` 副本启动**（不改动原有 AVD 与原系统镜像文件，也不下载新系统镜像；详见下方[设备与运行时工具缺口](#设备与运行时工具缺口)）。

## 网络盘点与 VPN

主流程第 1 步先运行 `python3 scripts/preflight.py --country PH --report <任务目录>/environment.json`（国家按已知目标替换，未知可省略）。先检测 SDK、ADB 设备和 AVD 配置/镜像，并对每个 AVD 目录做一次创建即删除的写探测，判断当前会话是否真能启动它；没有可响应的已授权设备、也没有当前会话可写的完整 AVD 则返回 4 并立即中断，VPN/代理等后续检查跳过。除这个探测文件外不写 AVD 目录，也不切换网络或启动模拟器。通过设备硬门槛后才只读盘点环境代理、macOS 系统代理、VPN 客户端与已知 controller。Python 要求 3.9+。

正文分别报告工具检测、设备门槛、VPN/代理盘点结果。设备/AVD 不存在、目录不可写（`AVD_NOT_WRITABLE`）与权限未就绪应保留明确错误；不能为了继续下载临时忽略这个失败。完整 AVD 的 `boot_verified=false` 意味着尚未实际启动验证，不能说它已在线；`directory_writable=false` 则连启动都不应尝试。

节点发现按以下顺序：

1. 已知客户端优先。按本机实际安装的客户端与进程确认（如 Clash Verge Rev 等 Clash/Mihomo 客户端，bundle ID 常见如 `io.github.clash-verge-rev.clash-verge-rev`）；先核对本机实际客户端，不假定固定服务商或进程名。
2. 有连接器/已授权客户端工具则读取节点。否则通过客户端 UI 读取当前策略组、选中节点及目标国家候选。
3. 已从客户端设置/现有配置确认本地 Clash/Mihomo controller 时，可执行：

   ```sh
   python3 scripts/preflight.py --country PH --controller http://127.0.0.1:9090
   ```

   `9090` 只是示例，不能盲猜本机端口。需要 secret 时使用 `--secret-env <已有私密环境变量名>`；不要把 secret 放命令行、聊天或归档。脚本仅对 loopback 发出 `GET /proxies`，不调用变更接口。若客户端未开启 HTTP `external-controller`（返回 `controller_unavailable`），`preflight.py` 会自动只读扫描本机 Clash/Mihomo 配置目录中的 `profiles.yaml` 与当前激活 profile 的策略组名、当前选中项（`now`）与节点名称列表（输出于 `vpn_local_profile`，过滤服务器地址、端口、UUID、密码与订阅 URL）。
4. 区分“节点名字匹配国家”与“当前请求出口确在该国家”。节点名、全局组选中值都不能证明请求已经过它。当第 3 步取包或第 5–8 步访问目标 App 远端 H5/API 遇到 CloudWAF / GeoIP 地区拦截（如返回 HTTP `418` `The access is blocked` 或 `403` 地区阻断）时，同样按下方流程盘点目标国家/地区或同区域候选节点（见 `preflight.py` 的 `matches_country_name` 与 `matches_region_fallback`）并向用户提出切换确认，不将其误判为应用自身故障或普通未登录（`110001`）。

有候选时，先记录当前组/节点、系统代理或 TUN 状态、脚本代理、模拟器代理；向用户提出具体变更，例如：

> 当前策略组为 A、节点为 B；发现“菲律宾 X”节点。下载请求需要菲律宾出口，是否临时切换该组到此节点？这会影响使用该组的网络请求，完成后恢复 B。

已有用户授权涵盖这一切换则无需再次询问。用户拒绝或无可用节点，继续不依赖切换的渠道。不要替用户改订阅、账号国家或购买 VPN。

执行获准变更后，用同一请求路径核查 HTTPS 连通性与出口国家。可用用户已接受的 IP 地理服务或目标服务证据；调用外部 IP 服务会向该服务暴露当前出口，避免发送其他身份数据。不能只测宿主机浏览器，就声称模拟器或 Python 也走了同一出口。

脚本 `--proxy http://127.0.0.1:<实际端口>` 仅影响该脚本；省略时 urllib 使用环境/系统代理，`--proxy direct` 强制直连。HTTP 代理端口、SOCKS 端口、controller 端口不同。stdlib 下载器不提供 SOCKS 支持，有 SOCKS-only 线路时使用客户端现成 HTTP 入口或其他已安装且支持 SOCKS 的工具，不把 SOCKS URL 当 HTTP 代理传入。

模拟器的 `127.0.0.1` 指模拟器自己；Android 官方 Emulator 访问宿主机常用 `10.0.2.2`，其他虚拟化环境要先确认。设置设备全局代理前读出原值；它也是设备网络变更，须在用户批准范围内。不要为可用的脚本代理额外修改设备代理。

无论成功/失败均在本次临时网络变更结束时恢复原状态。若用户在执行中自行改变了设置，不覆盖其新选择，应说明差异并协调恢复。恢复失败应报告当前残留状态。敏感会话、匿名令牌和带签名直链不写入共享报告。

## 工具依赖

| 能力 | 必需依赖 | 缺失时 |
|---|---|---|
| APKPure/Aptoide/已知直链 | Python 3.9+ 标准库、可用 HTTPS 路径 | 先处理连接；不为此安装大型第三方包 |
| APK 身份与签名校验 | Android SDK Build Tools 中的 aapt、apksigner；可运行的 Java | 优先现有 SDK 与 Android Studio JBR；缺工具时输出未完成校验，不降级成成功 |
| 真机/模拟器导出 | adb、已授权 device 状态 | 复用现有 platform-tools；unauthorized 交用户确认调试 |
| Google Play 模拟器取包 | ABI/SDK 匹配的 AVD、带 Google Play 的镜像、商店登录和应用可用性 | Google APIs 镜像不等于带 Play Store。优先已有合适 AVD，禁止 wipe-data 或修改无关 AVD |
| 匿名 Play getItems/delivery | 当前协议支持的设备 profile、已注册匿名 device ID；versionCode 可由 getItems 自动查询 | 先复用任务已有私密会话/工具生成记录；缺少时使用官方 Aurora 客户端或维护中的 GPlayAPI 工具完成注册，不伪造 device ID |
| 运行时注入与 AOT/Hermes 恢复（按需） | 可 root 任务设备/AVD、`frida`/`frida-server`；Flutter AOT 需 `blutter`（依赖 `cmake`/`clang`/capstone）；RN 需 Hermes 反汇编器 | 只要本机已有匹配的系统镜像（含 `google_apis_playstore`），直接在任务内基于现有镜像新建隔离 AVD、用 Magisk 修补 `ramdisk.img` 副本（`-ramdisk` 启动）并在用户目录补齐匹配 ABI 的 `frida`/`frida-server` 与 `Magisk.apk`，**无需暂停请求用户授权，也禁止为此下载新系统镜像**；仅当本机无任何可用镜像或需改动真机/原日常 AVD 时才向用户申请 |

工具安装先查现有路径（PATH、ANDROID_HOME、ANDROID_SDK_ROOT、常见 SDK 目录、`~/.venv*`）。第 1 步设备/ADB 硬门槛失败时，本次调研直接中断，环境修复须另按用户请求执行。门槛已经通过而任务需要小型用户目录工具（如虚拟环境安装 `frida-tools`、下载匹配 ABI 的 `frida-server` 与 `Magisk.apk`、对独立 AVD 的 `ramdisk.img` 副本做本地修补）时，**直接作为任务内标准步骤自主完成，不停下等待授权**；仅当涉及宿主机 `sudo`、修改系统级配置、下载 GB 级新系统镜像、或改动用户个人真机/原有日常 AVD 时，才先把具体组件、位置、大小/代价说明给用户并等待决策。不要通过安装另一整套 SDK 解决 PATH 问题。

### 设备与运行时工具缺口

当分析推进到第 6/7 步，确认遇 **PairIP/防重签、Android 7+/Flutter 证书限制、或无 `debuggable` 权限**，而当前运行设备不具备 root（例如只有 `user` 构建的 `google_apis_playstore` AVD）或本机尚未安装 `frida`/AOT/Hermes 工具时：

1. **禁止只做浏览器 H5 Mock 就停下交付或索要授权**：不得将其定性为必须真实账号/OTP 的 `real_external_blocker`，也不得以“已完成浏览器 H5 全拦截 Mock”为由跳过 App 壳内走查或停下向用户发 `(A)/(B)` 选择题。只要本机已有匹配的 Android 系统镜像，必须自动完成下方的零镜像下载隔离 root + Frida 壳内走查闭环。
2. **复用本机已有镜像，零新镜像下载、零原环境污染**：
   - 若已有可 `adb root` 的 `google_apis`/`userdebug` AVD 或已授权的 root 测试真机，直接复用；
   - 若本机只有 `google_apis_playstore`（`user` 构建）镜像与日常 AVD（如 `Pixel_10_Pro`），**严禁修改原有 AVD 或原系统镜像目录下的 `ramdisk.img`，也严禁去 `sdkmanager` 下载新的 `google_apis` 系统镜像**；必须直接复用本机已安装的这套 Play 商店镜像新建任务专用隔离 AVD，并对其 `ramdisk.img` 副本做 Magisk 修补后通过 `-ramdisk` 参数启动。
3. **本机已有镜像（含 Play 商店镜像）的 `-ramdisk` 副本 Root 与 Frida 挂载标准流程（跨架构/页大小/版本通用，无需请求授权，直接执行）**：
   - **步骤 1（新建隔离 AVD、修正 `target` 并复制 `ramdisk.img`）**：用 `avdmanager create avd -n <Task_Magisk> -k "<本机已有system-image包名>"` 创建独立 AVD（可按 `avdmanager list device` 选用本机存在的手机 profile，或从已有 AVD 的 `config.ini` 同步屏幕尺寸与密度；非登录 shell 下直接按 `environment.json` 的 `sdk_roots` 调用 `avdmanager` 与 `emulator`）。**创建后立即根据 `environment.json` 中记录的 AVD `ini_file` 与 `config` 路径检查 `target=` 字段**：在 `android-36.1`/`android-37.1` 等带小数点的系统镜像上，`avdmanager` 会写出 `target=android-0`（`preflight.py` 会标记 `target_corrupted=true` 并给出 `expected_target`），导致模拟器（`_avdInfo_getApiLevel < 21`）禁用硬件加速并报 `hvf is not enabled` / `mprotect failed: Permission denied` 卡死；发现 `target_corrupted` 或缺失 `target=` 时，先将其修正为 `expected_target`（即 `target=android-<镜像版本>`），再将 `image_directory` 下的 `ramdisk.img` 复制为 `<Task_Magisk>.avd/ramdisk-stock.img`。
   - **步骤 2（修补模拟器 `ramdisk` 副本，兼容多段拼接 `cpio`、4KB/16KB 页大小与不同 Magisk 版本）**：
     - **4KB 与 16KB（`ps16k`）页大小兼容**：Android 15/16+ 常见 `ps16k`（16KB 页大小，如 `google_apis_playstore_ps16k`，见 `preflight` 的 `page_size_16k`）镜像要求 `/init`（`libmagiskinit.so`）具备 `0x4000`（16KB）ELF LOAD 段对齐；旧版 Magisk（如 `v28.x` 及更早）为 4KB（`0x1000`）对齐，在 `ps16k` 镜像上会因内核拒绝加载 `/init` 导致模拟器永久停留在 `adb: device offline`。**统一优先选用支持 16KB 页对齐的 Magisk `v30.0+`/`v31.1+`（向下兼容 4KB 页大小镜像；优先选用官方 `app-debug.apk` 以便步骤 3 免 UI 交互授权）**。
     - **多段拼接 `ramdisk.img`（Generic + Vendor Ramdisk）拆分与重拼**：Android 13–16+（GKI）模拟器镜像的 `ramdisk-stock.img` 常由**两段或多段拼接的压缩 `cpio` 帧**组成（第 1 段偏移 `0` 为含 `/init` 的通用 ramdisk；第 2 段含 `first_stage_ramdisk/fstab.ranchu` 与 `lib/modules/virtio_*.ko` 等关键虚拟化驱动），而 `magiskboot cpio` 重打包时会在第 1 段 `TRAILER!!!` 处截断，直接丢弃第 2 段驱动与 `fstab.ranchu`，导致模拟器开机无法挂载分区与 `vsock`、永久卡在 `adb: offline`。因此**在解压前必须先检测 `ramdisk-stock.img` 偏移 `4` 之后是否存在第 2 个压缩帧魔数**（`lz4_legacy` 魔数 `\x02\x21\x4c\x18` 或 `gzip` 魔数 `\x1f\x8b\x08`）：若存在多段，先按第 2 段起始偏移拆分为 `ramdisk-generic.img`（首段）与 `ramdisk-vendor.img`（后续段），仅对 `ramdisk-generic.img` 执行下述修补，最后将重压缩生成的首段与原始 `ramdisk-vendor.img` 首尾拼接为 `<Task_Magisk>.avd/ramdisk-magisk.img`（单段镜像则直接修补）。
     - **从 Magisk 安装包提取对应 ABI（`arm64-v8a` 或 `x86_64`）组件并按实际格式修补**：注意模拟器 `ramdisk` 是无 Android boot header 的压缩 `cpio`（不能直接跑 `boot_patch.sh` 的 `unpack`），按以下通用顺序修补首段（或单段）副本：
       1. 运行 `magiskboot decompress <ramdisk-generic.img或ramdisk-stock.img> ramdisk.cpio`，从输出日志读取原始压缩格式 `RAMDISK_FMT`（常见为 `lz4_legacy` 或 `gzip`；未识别时默认 `lz4_legacy`，重打包时保持原格式）；
       2. 执行 `magiskboot compress=xz lib/$ABI/libmagisk.so magisk.xz`，并**动态检测附加组件**（Magisk `v29+`/`v31+` 含 `lib/$ABI/libinit-ld.so` 则压缩为 `init-ld.xz`；Magisk `v28` 及更早含 `assets/stub.apk` 则压缩为 `stub.xz`，哪个存在就注入哪个，不硬编码单一版本文件列表）；
       3. 动态确定 `PREINITDEVICE`：若已有同镜像在线实例可执行 `./magisk --preinit-device` 获取实际分区名（不同镜像/内核可能为 `metadata`、`vdd1` 或 `userdata`），离线或返回空时回退为 `metadata`；生成含 `KEEPVERITY=false`、`KEEPFORCEENCRYPT=false`、`RECOVERYMODE=false`、`PREINITDEVICE=<动态探测值或metadata>` 及 `SHA1=<原cpio的sha1>` 的 `config` 文件；
       4. 执行 `magiskboot cpio ramdisk.cpio`：先完成 `"backup ramdisk.cpio.orig" "mkdir 0750 overlay.d" "mkdir 0750 overlay.d/sbin" "add 0644 overlay.d/sbin/magisk.xz magisk.xz"`，再将实际存在的 `init-ld.xz`（`add 0644 overlay.d/sbin/init-ld.xz init-ld.xz`）或 `stub.xz`（`add 0644 overlay.d/sbin/stub.xz stub.xz`）加入，最后执行 `"patch" "mkdir 000 .backup" "add 000 .backup/.magisk config"`；
       5. 执行 `magiskboot compress=$RAMDISK_FMT ramdisk.cpio ramdisk-patched-generic.img`；若原 `ramdisk-stock.img` 含多段，执行 `cat ramdisk-patched-generic.img ramdisk-vendor.img > <Task_Magisk>.avd/ramdisk-magisk.img`（单段则直接输出到 `ramdisk-magisk.img`）。
   - **步骤 3（有界启动轮询、多路径 `su` 探测与免坐标激活 Shell Root 及 `frida-server`）**：
     1. **有界启动轮询**：后台启动 `emulator -avd <Task_Magisk> -ramdisk <Task_Magisk>.avd/ramdisk-magisk.img -no-snapshot-load -no-boot-anim`，用有界循环（如最多 45 次 × 2 秒 = 90 秒）检查模拟器进程存活、`adb -s <serial> get-state` 与 `getprop sys.boot_completed=1`（**严禁裸跑 `adb wait-for-device`**；若持续 `offline` 或超时未就绪，立即终止该进程并检查日志中的 `/init` 页大小或 `ramdisk` 格式报错）；
     2. **多路径探测 `su` 与激活 Shell（`uid=2000`）Root**：不同 Android 版本或 Root 实现下 `su` 可能位于 `/debug_ramdisk/su`、`/sbin/su`、`/system/bin/su` 或 `/system/xbin/su`，先通过 `SU_BIN=$(adb -s <serial> shell 'for p in /debug_ramdisk/su /sbin/su /system/xbin/su /system/bin/su; do [ -x "$p" ] && echo "$p" && break; done' | tr -d '\r\n')` 动态获取路径。安装 Magisk APK 并启动一次 `com.topjohnwu.magisk/.ui.MainActivity` 完成环境初始化；若 `$SU_BIN -c id` 返回 `Permission denied`（`su: request rejected (2000)`）：
        - **免 UI 坐标优先路径（适用 `app-debug.apk`）**：利用 `com.topjohnwu.magisk` 自身 UID 被 `magiskd` 直接放行的特性，执行 `adb -s <serial> shell "run-as com.topjohnwu.magisk $SU_BIN -c '${SU_BIN%/su}/magisk --sqlite \"REPLACE INTO policies (uid, policy, until, logging, notification) VALUES (2000, 2, 0, 1, 1);\"'"` 直接写入策略表，跨任意机型/分辨率免点击激活 Shell Root；
        - **UI 节点兜底路径（适用非 debuggable 的 release 版 `Magisk.apk`）**：用 `uiautomator dump` 获取界面 XML，按文本/资源 ID 定位授权弹窗或 Magisk **Superuser** 页的 `[SharedUID] Shell`（`com.android.shell`）开关并点击（不硬编码固定分辨率坐标），确认 `$SU_BIN -c id` 返回 `uid=0(root)`；
     3. **开机后推送并启动 `frida-server`**：因 `-no-snapshot-load` 启动会重置未保存的临时状态，须在 `-ramdisk` 正常开机且 `$SU_BIN -c id` 验证通过后，将匹配 ABI（`ro.product.cpu.abilist`）与本机 `frida` 版本的 `frida-server` 推入 `/data/local/tmp/frida-server`，通过 `$SU_BIN -c 'chmod 755 /data/local/tmp/frida-server && /data/local/tmp/frida-server -D'` 启动；收尾时关闭并清理临时隔离 AVD 及转发端口。
4. **何时才需要向用户询问**：仅当本机连任何可匹配的 Android 系统镜像都未安装（确需下载数 GB 镜像）、需要宿主机 `sudo`、或上述隔离 root + Frida 注入实测仍被服务端强校验阻断而只能依赖真实账号/OTP 时，才向用户说明具体阻塞与代价并请求决策。

## 匿名下载的实际边界

匿名下载来源仍是 Google Play。`download.py play` 使用已有配套 profile 和已注册 device ID，以 `X-PS-RH` 请求 Google 的 `/fdfe/getItems` 自动取得匹配包名的 versionCode，再请求 `/fdfe/delivery` 并下载全部返回文件、核对提供的摘要；不需要个人 Google 账号或 Aurora dispenser token。`/fdfe/details` 是账号路径，其 401 不能判定匿名链路失败；dispenser 403 也不应阻止已有注册设备的 getItems/delivery。

脚本不会创建账号、注册设备或接受条款。输入：

- `--device-profile`：实际工具生成的 JSON，包含 Build.VERSION.SDK_INT、Build.ID、Build.VERSION.RELEASE、Build.DEVICE、Build.HARDWARE、Build.MODEL、Build.BRAND、Build.PRODUCT、Build.FINGERPRINT、Vending.versionString。
- `--device-id-file`：与 profile 配套的已注册匿名设备 ID（十进制整数）。从已有资料或注册工具生成，不随机伪造；私密输入仅当前用户可读。
- `--version-code`：可选，只有需要确切版本时提供；省略时脚本实际调用 getItems 查询，不猜测、不将 versionName 转数字。
- `--country`、`--language`：目标地区；仅改参数不保证绕过发行限制。

已有输入先执行完整匿名链路。缺注册设备时按当前上游协议/官方 Aurora 客户端补齐，不要求用户手写 token；确实无法补齐时记为 dependency_missing，而非下载失败。getItems 成功仅证明详情可取，必须继续 delivery 与文件落盘才算取包成功。匿名请求异常时分别记录端点、HTTP/业务结果和未完成步骤；基于新协议证据做有界纠错，不重复错误接口。

新机器缺输入且当前工具不能注册时，这条路径记为“依赖未就绪”，继续其他渠道，不停在要求用户提供 token。已有的 profile 与设备 ID 文件只存于任务或本机私密目录，不打包进 skill 也不写入共享报告。

## 模拟器 Google Play 与真机导出

1. 列出设备：`adb devices -l`；按 [设备选择顺序](#设备选择顺序) 选定本次取包设备，每次命令明确 `adb -s SERIAL`。可只读检查目标包 `shell pm path PACKAGE`。
2. 已安装则直接导出，不必先启动目标 App，不访问其私有数据。用户要求导出 APK 不等于授权导出账号、数据库或业务缓存。
3. 未安装且准备使用任务模拟器时，检查 `ro.product.cpu.abilist`、`ro.build.version.sdk`、Play Store 是否存在及剩余空间；打开已确认包名的商店页：

   ```sh
   adb -s SERIAL shell am start -a android.intent.action.VIEW -d 'market://details?id=PACKAGE' -p com.android.vending
   ```

   上面 SERIAL/PACKAGE 是参数占位；实际执行前替换，不能把占位串当命令运行。
4. 真实账号登录、二次验证等所需输入由用户完成；任务专用环境的常规同意按 [主工作流](guide.md) 连续执行规则处理。用户登录后重新观察实际状态，不以回复“好了”推断安装完成。不收集密码，不自动迁移 Play 账号地区。
5. 免费 App 在已选任务专用模拟器中安装属于取包必要步骤；个人/共享真机的新安装、升级、覆盖需要明确设备修改授权。用户已有相同授权则继续。
6. 商店提示国家不可用时分别检查账号地区与请求出口。菲律宾 VPN 不能自动改变中国区 Play 账号；不要清除 Play/目标 App 数据试错。用户可选择已有合适账号，或改走匿名/镜像。
7. 安装结束后重新查询 `pm path`，运行导出脚本；安装画面完成不等于本机已取得 APK。所有 `base.apk`、ABI、密度、语言及功能 split 都应保存。

   ```sh
   python3 scripts/export_device.py --serial SERIAL --package PACKAGE --out /absolute/sample-dir
   python3 scripts/verify.py --input /absolute/sample-dir --package PACKAGE
   ```

导出脚本只执行设备读取与 `adb pull`，逐文件比较设备和电脑 SHA-256，并复查安装路径集是否改变。它不安装、不启动、不卸载 App。脚本失败时保留不完整目录和明确失败状态，重新导出使用新目录，避免混合样本。
