# EazyProxy

> 面向 Android TV 与平板的系统级代理客户端 · A system-wide proxy client for Android TV and tablets

[English](#english) · [简体中文](#简体中文)

---

## English

### What it is

EazyProxy turns a SOCKS5 or HTTP CONNECT proxy into a system-wide VPN on Android. Built on `VpnService`, it captures IP packets from every app — or only the apps you choose — reassembles TCP/UDP sessions in userspace, and forwards them through an upstream proxy **you provide**. Apps with no proxy support of their own end up going through the tunnel anyway.

This edition is a TV-focused rework of the open-source [free-proxy](https://github.com/xVanTuring/free-proxy) project. The phone-oriented bottom navigation, theme switcher and floating action buttons have been removed; a single left navigation rail, a one-screen home dashboard and complete D-pad control take their place. The VPN engine and its pure-Kotlin userspace TCP/IP stack are unchanged. No features were added beyond what the original already did. Android tablets are supported by the same APK and layout: they use the regular `LAUNCHER` entry and touch input, while the TV-tuned overscan margins and 64 dp focus rows simply leave a little more whitespace. Portrait phones are not a target.

### At a glance

| | |
|---|---|
| App name | EazyProxy |
| Package | `com.easyproxy.tv` |
| Version | 1.0.4 (versionCode 5) |
| Platform | Android 7.0+ (minSdk 24), targetSdk / compileSdk 36 |
| Primary target | **Android TV and Android tablets** (one APK; the layout assumes a landscape large screen, not a portrait phone) |
| Interface | English and Simplified Chinese; fixed dark theme |
| License | Apache License 2.0 |
| Proxy servers | **Not included — bring your own** |

### What you can do

**Proxy & tunnel**
- SOCKS5 and HTTP CONNECT profiles with optional username/password authentication
- UDP over SOCKS5 (UDP ASSOCIATE) per profile; when unavailable, DNS automatically falls back to DNS-over-TCP
- Per-profile DNS mode: resolve through the proxy (private) or directly (fast)
- IPv6 leak blocking, private-network (LAN) bypass, adjustable DNS server and MTU

**Profiles**
- Add, edit, duplicate, delete and switch between multiple profiles
- "Test connection" performs a real proxy handshake before you save
- Foreground-service notification with live upload/download counters and one-tap disconnect

**Per-app routing**
- Three modes: all apps, allow list (only selected apps use the proxy), bypass list (selected apps go direct)
- Search installed apps, show/hide system apps, select all or clear; changes apply after reconnecting

**DNS blocking**
- Block matching DNS queries locally with a configurable response IPv4
- Domain rules: `example.com` covers its subdomains, `*.example.com` matches subdomains only
- Rules can be added by hand, imported from a file, or added straight from a log entry; block by app is supported too

**Activity log**
- Live, colour-coded stream of DNS lookups and connection results, showing resolved hostnames instead of bare IPs and (on Android 10+) the app that opened the connection
- Search, filter by app, pause/resume recording, clear, jump to latest, export to CSV

**Automation**
- Start on boot (reconnect the last profile), stop automatically when the underlying network drops, reset all settings to defaults

### How it works

There is no NDK code and no bundled `tun2socks` binary — the TCP/IP stack is written in plain Kotlin, so one build covers every ABI.

```
app → system routing → TUN device
                         ↓  reader thread
                   IPv4 / TCP / UDP header parsing
                         ↓
             TcpSession (userspace state machine) / UdpSession
                         ↓  protect()-ed socket
              ProxyClient (SOCKS5 handshake / HTTP CONNECT)
                         ↓
                     upstream proxy → destination
```

Return traffic is re-encapsulated by `PacketBuilder` and written back to the TUN device. A few details worth knowing: the proxy hostname is resolved **before** `establish()` because the tunnel depends on that lookup; LAN bypass is done by declaring only the complement of private ranges to the TUN (routing, not dropping); and `HostRegistry` learns hostnames from DNS responses passing through, which is what makes the log readable.

### Designed for the big screen

- **Two launcher entries**: `LEANBACK_LAUNCHER` for TV plus `LAUNCHER`, with a TV banner; touchscreen and leanback are both declared as *not required*
- **Tablets work too**: launch from the regular `LAUNCHER` icon on an Android tablet and use it by touch; the same rail, two-column home and dialogs apply. Overscan padding and 64 dp focus rows are TV-tuned, so a tablet simply gets a bit more whitespace — every function stays available. Portrait phones are not supported.
- **One-level hierarchy**: a 200 dp left rail with four destinations (Home, Profiles, Logging, Settings); per-app routing and DNS blocking are second-level pages reached from Settings, with BACK returning to where you were
- **Home fits one screen**: status dial on the left, active profile / traffic / sessions / routing and the connect button on the right; scrolling is only a fallback for unusually short screens or enlarged system fonts
- **Overscan-safe**: a 5% safe area (48 dp horizontal / 27 dp vertical) is applied once at the root and inherited by every screen; content is capped at 960 dp width on very large logical displays
- **Everything reachable by remote**: 64 dp minimum focus rows, a 3 dp focus outline drawn above any Material surface, focus restoration when switching destinations, explicit focus bridges inside dialogs, and pre-filled text fields that select all on focus — no long-press, no popup menus, no edge-hugging floating buttons
- **Density-independent**: all sizes use dp/sp and all artwork is vector XML; 720p, 1080p and 4K share one layout, differing only in sharpness, because Android normalises physical pixels via density

### Install & requirements

- Requires Android 7.0 or newer and a user-supplied SOCKS5 / HTTP CONNECT proxy. The first connection prompts for VPN authorisation; Android 13+ may additionally ask for the notification permission (declining it only hides the status notification).
- **The package name changed to `com.easyproxy.tv`.** It cannot overwrite an older FreeProxy build: the two APKs install side by side, and profiles, per-app rules and VPN authorisation must be set up again. Remove the old build first if you no longer need it.
- On an Android tablet, install the same APK and open it from the app drawer — no TV launcher is needed. VPN authorisation works the same way as on a TV.
- No account, no ads, no bundled servers, no Google Play services dependency. VPN permission is requested only when you press Connect.

### Known limitations

| Limitation | Detail |
|---|---|
| IPv4 only | "Block IPv6" captures IPv6 into the tunnel and drops it so it cannot leak around the proxy |
| No ICMP | Proxy protocols cannot carry it; `ping` inside the tunnel failing is expected |
| UDP depends on the proxy | Needs SOCKS5 with UDP ASSOCIATE; otherwise only DNS works (over TCP) |
| No TCP window scaling | Per-connection throughput is capped by the 64 KB window |
| No fragment reassembly | Fragmented IP packets are dropped; the app sets the TUN MTU so normal traffic never fragments |
| Credentials in plaintext | Proxy passwords live unencrypted in the app's private directory, excluded from cloud backup |
| App attribution needs Android 10+ | Relies on `getConnectionOwnerUid`; older systems leave that column empty |
| ROM-dependent behaviour | Overscan is a fixed safe area rather than a value read from the TV; system font scaling and vendor TV keyboards may change the visual result |

### Tech stack & verification

Kotlin 2.2.21 · Jetpack Compose Material 3 (BOM 2026.06.01) · Navigation Compose · DataStore · kotlinx.serialization; no Hilt, no Room, no NDK. Toolchain: AGP 8.13.2, Gradle 9.1.0, Java 21 bytecode target (the release was built and verified on JDK 25).

The release build enables R8 code shrinking, resource shrinking and `lintVitalRelease`, and is v2-signed and zipalign-verified. All 29 JVM unit tests pass (route complement calculation, IP/TCP/UDP packet round trips, checksums, sequence wraparound, DNS message parsing and DNS block rules). Static checks confirm every interactive element has focus handling, with no long-press, popup-menu or floating-button-only paths. **Physical TV / emulator verification is still outstanding** — VPN authorisation dialogs, notification permission, file import/CSV export on TV, and vendor IME behaviour need a real device.

---

## 简体中文

### 它是什么

EazyProxy 把 SOCKS5 / HTTP CONNECT 代理转换成 Android 的系统级 VPN。它基于 `VpnService`，把全部应用（或你指定的应用）的流量捕获成 IP 包，在用户态还原为 TCP / UDP 会话，再经**你自备的**上游代理转发出去 —— 本身不支持代理的应用也能因此走代理。

这个版本是开源项目 [free-proxy](https://github.com/xVanTuring/free-proxy) 面向电视端的精简改造：移除了手机端的底部导航、主题切换与悬浮按钮，改为单层左侧导航、一屏展示的首页和完整的遥控器操作；VPN 引擎与纯 Kotlin 的用户态 TCP/IP 协议栈保持原样，没有新增原项目之外的功能。同一个 APK 与同一套布局也适用于 Android 平板：走常规 `LAUNCHER` 入口、用触摸操作，按电视调校的过扫描留白与 64 dp 焦点行在平板上只是多留一些空隙。竖屏手机不是目标设备。

### 一览

| | |
|---|---|
| 应用名称 | EazyProxy |
| 包名 | `com.easyproxy.tv` |
| 版本 | 1.0.4（versionCode 5） |
| 系统要求 | Android 7.0+（minSdk 24），targetSdk / compileSdk 36 |
| 主要适配 | **Android TV 与 Android 平板**（同一个 APK；布局按横屏大屏设计，不适合竖屏手机） |
| 界面 | 中英双语，固定深色主题 |
| 许可证 | Apache License 2.0 |
| 代理服务器 | **不提供，需要自备** |

### 能做什么

**代理与隧道**
- SOCKS5 与 HTTP CONNECT 配置，支持用户名 / 密码认证
- 每条配置可选 SOCKS5 的 UDP ASSOCIATE 转发；代理不支持时，DNS 自动降级为 DNS over TCP
- 每条配置独立选择 DNS 解析方式：走代理（隐私）或本地直连（速度）
- 阻断 IPv6 泄漏、内网（局域网）直连、可调整 DNS 服务器与 MTU

**配置管理**
- 多套配置的新增、编辑、复制、删除与切换
- 「测试连接」在保存前用真实握手验证代理可达性
- 常驻通知实时显示上下行流量，并可一键断开

**分应用代理**
- 三种模式：全部应用 / 白名单（仅选中走代理）/ 黑名单（选中的绕过）
- 支持搜索已安装应用、显示或隐藏系统应用、全选与清空；改动在重新连接后生效

**DNS 拦截**
- 命中规则的 DNS 查询由隧道直接伪造应答，应答 IPv4 可自定义
- 域名规则：`example.com` 覆盖其子域，`*.example.com` 仅匹配子域
- 规则可手工添加、从文件导入，或直接从日志条目加入；也支持按应用拦截全部查询

**活动日志**
- 实时、按事件着色的日志流，记录 DNS 查询与每条连接结果；显示解析出的域名而非裸 IP，Android 10+ 还能标出发起连接的应用
- 支持搜索、按应用过滤、暂停 / 继续记录、清空、跳到最新、导出 CSV

**自动化**
- 开机自启（重连上次配置）、底层网络断开时自动停止、一键恢复默认设置

### 工作原理

不含 NDK 代码，也没有打包 `tun2socks` 之类的二进制：协议栈用纯 Kotlin 实现，一次构建通吃所有 ABI。

```
应用 → 系统路由 → TUN 设备
                    ↓  读线程
              IPv4 / TCP / UDP 头部解析
                    ↓
        TcpSession（用户态状态机）/ UdpSession
                    ↓  protect() 过的 socket
           ProxyClient（SOCKS5 握手 / HTTP CONNECT）
                    ↓
                上游代理 → 目标服务器
```

回程数据由 `PacketBuilder` 重新封包后写回 TUN。几个关键处理：代理域名必须在 `establish()` **之前**解析，因为隧道本身依赖这次查询；内网直连靠「只把私有网段的补集声明给 TUN」实现，是路由而非丢包；`HostRegistry` 从流经的 DNS 应答里学习域名，日志因此能显示可读的主机名。

### 为电视大屏而做

- **双启动入口**：TV 的 `LEANBACK_LAUNCHER` 与常规 `LAUNCHER`，并带 TV 横幅；触摸屏与 leanback 均声明为*非必需*
- **平板同样可用**：在 Android 平板上从常规 `LAUNCHER` 图标启动、用触摸操作即可，侧栏、双列首页与弹窗是同一套实现；过扫描留白与 64 dp 焦点行按电视调校，平板上只是多留一些空隙，功能一个不少。竖屏手机不在支持范围。
- **单层层级**：200 dp 左侧导航栏，四个主目的地（首页、配置、日志、设置）；分应用代理与 DNS 拦截是从设置进入的二级页，按返回键回到原处
- **首页一屏展示**：左侧状态圆环，右侧依次是当前配置、流量 / 会话数、路由模式与连接按钮；只有在屏幕异常矮或系统字体放大时才滚动兜底
- **过扫描安全区**：根布局统一留出 5%（水平 48 dp / 垂直 27 dp），所有页面继承；逻辑宽度特别大的屏幕上，内容宽度封顶 960 dp
- **全遥控器可达**：最小 64 dp 焦点行、绘制在 Material 背景之上的 3 dp 焦点环、切换页面时恢复焦点、弹窗内显式焦点桥接、预填输入框聚焦即全选；不依赖长按、弹出菜单或贴边悬浮按钮
- **与分辨率无关**：尺寸一律用 dp/sp，图标与横幅都是矢量 XML；720p、1080p、4K 共用一套布局，差别只是清晰度 —— 物理像素由 Android 按屏幕密度归一化

### 安装与须知

- 需要 Android 7.0 及以上，并自备 SOCKS5 / HTTP CONNECT 代理。首次连接会弹出 VPN 授权；Android 13+ 还会请求通知权限（拒绝只影响状态通知，不影响连通性）。
- **包名已改为 `com.easyproxy.tv`**，无法覆盖安装旧的 FreeProxy 包：两者会在设备上并存，配置、分应用规则与 VPN 授权都需要重新设置；不再需要旧版时请先卸载。
- Android 平板装同一个 APK，从应用列表打开即可，不需要 TV 启动器；VPN 授权流程与电视端一致。
- 无需账号、无广告、不内置服务器、不依赖 Google Play 服务。只有点击连接时才会申请 VPN 权限。

### 已知限制

| 限制 | 说明 |
|---|---|
| 仅支持 IPv4 | 「阻断 IPv6」会把 IPv6 收进隧道并丢弃，避免绕过代理泄漏 |
| 不支持 ICMP | 代理协议承载不了，隧道内 `ping` 不通属正常现象 |
| UDP 依赖代理能力 | 需要 SOCKS5 且支持 UDP ASSOCIATE，否则只有 DNS 可用（走 TCP） |
| 无 TCP 窗口缩放 | 单连接吞吐受 64 KB 窗口限制 |
| 不做分片重组 | 分片 IP 包会被丢弃；TUN 的 MTU 由应用设定，正常流量不会分片 |
| 凭据明文存储 | 代理密码未加密，存于应用私有目录并已排除出云备份 |
| 应用归属需 Android 10+ | 依赖 `getConnectionOwnerUid`，更早系统上该列留空 |
| 受 ROM 影响 | 过扫描是固定安全区而非读取电视实际设置；系统字体缩放与厂商 TV 输入法可能改变实际观感 |

### 技术栈与验证情况

Kotlin 2.2.21 · Jetpack Compose Material 3（BOM 2026.06.01）· Navigation Compose · DataStore · kotlinx.serialization；未使用 Hilt、Room 与 NDK。工具链：AGP 8.13.2、Gradle 9.1.0，字节码目标 Java 21（本次发布包在 JDK 25 环境下构建验证）。

发布包开启 R8 代码混淆、资源压缩与 `lintVitalRelease`，并通过 v2 签名与 zipalign 校验。29 项 JVM 单元测试全部通过（路由补集计算、IP/TCP/UDP 报文往返、校验和、序列号回绕、DNS 报文解析与 DNS 拦截规则）。静态检查确认所有交互项都有焦点处理，不存在只能靠长按、弹出菜单或悬浮按钮触达的路径。**真机 / 模拟器验证仍待完成** —— VPN 授权弹窗、通知权限、电视上的文件导入与 CSV 导出、厂商输入法行为，都需要在真实设备上确认。

---

## 致谢 / Acknowledgements

**中文**：EazyProxy 并非从零开始。它建立在 **xVanTuring** 的开源项目 [free-proxy](https://github.com/xVanTuring/free-proxy) 之上 —— 系统级 VPN 的完整实现、纯 Kotlin 的用户态 TCP/IP 协议栈、SOCKS5 / HTTP CONNECT 握手、DNS 处理、日志与分应用路由，都来自原作者的工作。本次所做的只是面向 Android TV 与平板等大屏的裁剪与适配：收敛导航层级、重排大屏布局、补齐遥控器焦点与输入体验、更换名称与包名，核心能力一行未重写。由衷感谢原作者以 **Apache License 2.0** 开放源代码，让这样的二次改造成为可能；也感谢 Kotlin、Jetpack Compose 与 AndroidX 等开源项目的贡献者。上游仓库与许可证链接一并保留，以示来源与尊重。

**English**: EazyProxy did not start from scratch. It stands on **xVanTuring**'s open-source project [free-proxy](https://github.com/xVanTuring/free-proxy) — the complete system-wide VPN implementation, the pure-Kotlin userspace TCP/IP stack, SOCKS5 / HTTP CONNECT handshakes, DNS handling, logging and per-app routing all come from the original author. This edition only trims and adapts it for large screens (Android TV and tablets): a flatter navigation hierarchy, a large-screen layout, full D-pad focus and input handling, plus a new name and package. None of the core networking code was rewritten. Sincere thanks to the original author for releasing the work under the **Apache License 2.0**, which makes a derivative like this possible, and to the contributors behind Kotlin, Jetpack Compose and AndroidX. The upstream repository and its license are linked here to keep the provenance clear.
