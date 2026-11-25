---
title: "一辆 2022 Highlander 的全屏 CarPlay 研究：从 iPhone 日志到 GVIF、TIS 和产品化"
date: 2026-11-25
permalink: /blogs/highlander-fullscreen-carplay-research/
lang: zh-CN
read_time: true
read_time_minutes: 35
excerpt: "为什么一块 12.3 英寸超宽屏，CarPlay 只能用大约 62%？从实车测量、iPhone sysdiagnose、丰田 DCU/ICS/MEU 架构，到维修手册、GVIF、Beat-Sonic 和产品化路线，一份仍在进行中的研究记录。"
tags:
  - carplay
  - reverse-engineering
  - automotive
  - toyota
  - embedded-systems
  - product-engineering
---

一辆 2022 款 Toyota Highlander Limited，原厂 12.3 英寸 JBL 多媒体系统，支持 Apple CarPlay。听起来挺好，直到我把手机接上去：**这么宽的一块屏幕，CarPlay 只能占左边大约六成，剩下的部分永远留给丰田自己的界面。**

一开始我想解决的就是这个恼人的 UI 问题。后来越查越深，牵扯出 iPhone 的 CarPlay 视频协商、Panasonic 主机、DENSO 多显示屏控制器、Toyota TIS 维修资料、GVIF 高速串行视频，以及一条可能利用国内供应链做成产品的路线。我甚至专门买下了这辆车，把它作为后续实验平台。

这篇是截至 **2026 年 11 月 25 日**的完整研究记录，而不是一份已经成功解锁的教程。写得比较细，一方面方便以后自己接着做，另一方面也想把那些“看起来很合理，后来却发现证据不够”的判断原样保留下来。

**当前状态：DCU-only 暂停，等 Toyota 诊断设备到货后继续验证；外部视频接入 R-EXT 保留为主线备选。尚未实现原生全屏 CarPlay。**

**目录**

1. [问题究竟是什么：全屏不等于拉伸](#goal)
2. [车上的硬件：DCU、MEU、ICS 与屏幕](#hardware)
3. [最初的实车实验：哪些页面能全屏](#field)
4. [从 iPhone sysdiagnose 抓到的 CarPlay 参数](#ioslog)
5. [Smart Display Zoom 的一次误导](#zoom)
6. [拿尺子量：62.07% 从何而来](#geometry)
7. [沿着丰田维修手册还原视频链路](#topology)
8. [H1a、H1b、H1c：我们为什么反复修改架构判断](#hypotheses)
9. [第一次路线收敛：从 DCU-only 转向 R-EXT](#route-shift)
10. [Beat-Sonic HVXT06：一个很有用但不是 CarPlay 的盒子](#hvxt06)
11. [R-EXT 的完整技术方案与未解难题](#rext)
12. [为什么不直接重写原厂固件](#firmware)
13. [第二次转折：TIS 审计重新打开 DCU-only](#audit)
14. [S0 实车 Service Menu：把三个打码项找出来](#service-menu)
15. [下一轮 D0：Techstream、MFC3 与停止条件](#diagnostics)
16. [从改装一辆车到产品：ODM 与国内供应链](#product)
17. [证据表、已知问题与项目时间线](#status)
18. [资料来源、术语与下一步](#references)

---

<a id="goal"></a>
## 1. 问题究竟是什么：全屏不等于拉伸

这辆车的屏幕横向很长。正常使用时，中间是一个主内容区，旁边是丰田的侧栏。CarPlay、原厂地图、音乐等内容可以在左右两边交换，但无法真正占据整块屏幕。

这里有三种容易混淆的“全屏”：

| 类型 | 看起来发生了什么 | 是否满足目标 |
|---|---|---|
| 把 16:9 或近似普通比例的视频横向拉宽 | 元素变胖、圆形变椭圆 | **不满足** |
| 对窄幅图像裁切放大，强行铺满 | 比例可能正常，但文字、边缘内容丢失 | **不满足** |
| iPhone/CarPlay 接收端一开始就按照超宽区域生成 UI | 导航、Dock、地图等元素重新布局 | **这才是目标** |

最终希望拿到的不是一张“看起来占满屏幕的 HDMI 画面”，而是一个可以日常使用的系统：原厂屏幕保留，CarPlay 原生超宽布局，触控准确，音乐、电话、Siri、方向盘按键可用，原厂空调和倒车影像正常，设备出现异常时还能安全回到丰田原始界面。

更长远地说，目标也不只是给我自己的车装一个盒子，而是探索一套能利用国内电子方案商和线束供应链量产、价格合理的产品。买车是为了获得可重复实验的环境，不代表最终每辆客户车都要做同样复杂的逆向工作。

**研究边界**也很明确：优先诊断、被动监听和可逆信息娱乐接口；不向动力或 ADAS CAN 注入未知消息，不把绕过安全启动或分发 Toyota 专有固件作为产品路线；所有改装都需要验证倒车和必要车辆提示的回退。

<a id="hardware"></a>
## 2. 车上的硬件：DCU、MEU、ICS 与屏幕

先认清我们究竟在和哪几块电脑打交道。我最初以为“车机”就是屏幕后面的一块主板。实际不是。

| 组件 | 在这辆车中的角色 | 已知信息 |
|---|---|---|
| DCU / Radio Receiver | 主多媒体主机，处理 CarPlay、音源和车辆多媒体交互 | Panasonic，Model ID `17TDTENA-DA30`；P/N `86120-0E420`；Audio `1225` |
| MEU / Navigation ECU | 原厂导航扩展模块 | Model ID `17TMTENA-DA50`；P/N `86840-0E075`；Nav `1225` |
| ICS / Integration Control Sub-assembly | TMOT / Multi-display 系统中的图像合成、操作界面控制模块 | DENSO 体系；T-SB-0055-22 出现更换件号 `840C2-48010`，尚非本车实物标签确认 |
| DISP / Multi-display | 真正的 12.3 英寸 LCD/显示组件 | 实际 LCD 像素数仍未知 |
| JBL 放大器 | 原车音频输出 | DCU 与功放之间涉及 MOST |

DCU、MEU 的具体零件号后来在 Service Menu 的 `Product Information` 中重新拍照核实。**没有把芯片唯一序列号、VIN、个人设备 ID 或未经脱敏的 sysdiagnose 上传到公开博客。**

还有两个常见误认需要记下来：

- iPhone 识别到的 `CV-RT29N04J` 是 CarPlay 端点的 modelName，不是已核实的 Toyota 维修零件号。
- 某处 HID 信息中的 `20.2.14` 不能当作整套车机的固件版本；车机自己显示的是 Audio/Nav `1225`。

这套架构大致属于 2019 年末至 2022 年 10 月生产期间使用的老平台；后来的 Toyota Audio Multimedia 是另一代设计，不能直接把 2023+ 的软件更新方式或改装产品套过来。

<a id="field"></a>
## 3. 最初的实车实验：哪些页面能全屏

最开始做的事情很朴素：在停车状态下把每种界面都打开、左右交换、观察边界，然后用 iPhone 生成截图。

| 测试 | 观察 | 直接说明了什么 |
|---|---|---|
| CarPlay 正常连接 | 画面只占约六成宽 | 有稳定的固定内容窗口 |
| 左右交换 CarPlay 和 Toyota 侧栏 | 左右位置改变，尺寸不变 | 换边不是全屏 |
| Trip Information | 可以分屏、左右交换，也可以占满整块屏幕 | 车上确实存在全屏显示模式 |
| Trip 全屏遮住 CarPlay 后在手机上切歌 | iPhone 侧 CarPlay 截图仍能反映状态变化 | 会话和 UI 状态仍在更新 |
| 原厂导航 / Audio / Settings | 都能左右交换，却不能全屏 | 限制不是 CarPlay 独有 |
| 原厂普通设置页面 | 没找到全屏 CarPlay 开关 | 用户菜单没有显式入口 |

这里有一个非常重要的逻辑区别：**Trip 能全屏，不能直接证明 CarPlay 所在的 DCU 视频通路也能全屏；但它证明显示器和原厂系统至少有一种全宽工作模式。**

同样，CarPlay 在 Trip 后面还能更新状态，不等于我已经测量到隐藏期间每一帧 H.264 都持续传输。前者是实测，后者是还需要抓包或日志的假设。

<a id="ioslog"></a>
## 4. 从 iPhone sysdiagnose 抓到的 CarPlay 参数

单靠目测争不出底层事实。我在 **2026-11-24 19:35 PST** 左右生成了一份 iPhone sysdiagnose，在 Mac 上用统一日志工具解码，得到约 7,581 行的 CarPlay 相关日志。第一份解码文本保存在本地 `~/Desktop/carplay_logs.txt`。

这一轮真正关键的是下面几组记录。时间均为太平洋标准时间，行号对应当时保存的解码文本。

| 时间 / 行号 | 记录 | 意义 |
|---|---|---|
| 19:34:58.230，约 2422–2435 行 | `ScreenPixelSize = 800×480`；`ScreenPhysicalSize = 183×109` | Toyota 配件向 iPhone 声明的 CarPlay 显示区域 |
| 19:34:58.233，约 2488–2534 行 | `OverrideCanvasSize = 972×584`，`preferredUIScale = 2` | iPhone 某层 UI 画布覆盖值 |
| 约第 2562 行 | `CarPlay ViewAreas: Disabled` | 动态 View Areas 并未启用 |
| 约第 3091 行 | `visibleFrame = safeFrame = 800×480` | iPhone 认为整个可见 CarPlay 区域就是 800×480 |
| 19:35:02.476–.534，约 6902–6945 行 | H.264 硬件编码器以 `800×480` 初始化 | 这次会话的实际视频编码尺寸有独立证据 |

一些便于复查的原文：

<pre>
Original Display Characteristics:
  ScreenPixelSize    Width=800  Height=480
  ScreenPhysicalSize Width=183  Height=109

OverrideCanvasSize = { Width=972; Height=584; }
preferredUIScale = 2
CarPlay ViewAreas: Disabled

visibleFrame = {x=0, y=0, w=800, h=480}
safeFrame    = {x=0, y=0, w=800, h=480}

H.264-legacy: Init hw video encoder:
dimension = 800 x 480
</pre>

另有这些连接线索：

- 有线 USB CarPlay，枚举到 Toyota Multimedia；USB VID/PID 为 `0x04DA / 0x3210`。
- USB 2.0 High Speed 链路能力为 480 Mbps；这**不是**实际 CarPlay 码率。
- iPhone 暴露的虚拟触控 HID 坐标上限约为 X=800、Y=480；**这并不告诉我们车内真实触摸数据走的是 AVC-LAN 还是 GVIF 反向通道**。
- 该显示端点标称最高 30 FPS；不要把初始化参数当成每秒实际帧数的统计。
- 日志 FeatureKey 里出现过 `viewAreas` 字样，但最终 `CarPlay ViewAreas: Disabled`，不能只截取前一句就宣称 Toyota 已经支持动态分区。

这一步把一个很容易犯的错误排除了：**iPhone 屏幕截图的像素数、UI 逻辑画布、H.264 编码分辨率、车上 LCD 的原生像素数，是四个不同层次。** 其中 LCD 原生像素数直到现在都没有直接量到。

<a id="zoom"></a>
## 5. Smart Display Zoom 的一次误导

我们还测试了 iOS 的 Smart Display Zoom。它刚好说明为什么“截图尺寸变了”不等于“车屏显示区域变了”。

| Smart Display Zoom | CarPlay 截图尺寸 | 车上 CarPlay 物理窗口 |
|---|---:|---|
| ON | `972×584` | 不变 |
| OFF | `800×480` | 不变 |

我最初看到 972×584，差点以为车机其实能接收更大的画面。随后核对 H.264 初始化日志，发现 **Zoom ON 时编码器仍是 800×480**。Zoom 调的是 iPhone 内部 UI 密度和缩放效果，并没有突破 Toyota 预留的物理窗口。

严谨一点：我们对 OFF 状态保留了截图和车屏观察，没有独立采到第二次 OFF 状态 H.264 初始化日志，因此不能凭推测把它写成第二次编码器实测。

这条路就此排除：**在 iPhone 上折腾显示缩放，不是 Toyota 全屏解锁。**

<a id="geometry"></a>
## 6. 拿尺子量：62.07% 从何而来

随后做了一个很有帮助、但完全不需要高级设备的实验：尺量发光区域。

| 区域 | 实测 |
|---|---:|
| 全部发光区域 | **290 × 110 mm** |
| CarPlay 有效窗口 | **180 × 110 mm** |
| 其余 Toyota 侧栏 | 约 110 mm 宽（由两者相减） |

于是 CarPlay 占屏幕宽度：

$$
\frac{180}{290}\times100\%\approx\boxed{62.07\%}
$$

而 sysdiagnose 中的物理尺寸声明是 **183×109**，与尺量的 180×110 mm 只有几毫米误差。很自然的解释是：DCU 告诉 iPhone 的物理显示区域，就是我们肉眼看到的窄窗口，而不是整个 LCD。

我们曾提出一个漂亮的几何猜想：

<pre>
假设 LCD 像素是 1920 × 720
CarPlay 视频是  800 × 480
各向放大 1.5 倍 → 1200 × 720
1200 / 1920 = 62.5%
实际尺量占宽 = 62.07%
</pre>

两个数字非常接近，但我不把它记成“LCD 已确认 1920×720”。它只是一个能解释现象的模型。真实像素数、谁在缩放、是不是严格 1.5 倍，仍需面板标签、像素计数或视频抓取。

值得注意的是，我们用 290/110 得到的物理宽高比约为 **2.636:1**，很接近通常说的 **8:3（2.667:1）**。后面的外部接收器设计会多次用到这个比例，但“8:3 是目标布局”和“1920×720 是已知面板原生像素”不能画等号。

<a id="topology"></a>
## 7. 沿着丰田维修手册还原视频链路

为了确定该改哪块设备，我们整理了此前通过 Toyota TIS 订阅合法取得的维修资料：Repair Manual（RM）、Electrical Wiring Diagram（EWD）、New Car Features（NCF）与 Technical Service Bulletin（TSB）。这些原件保存在研究项目里，下面列出的是文档编号与工程摘要，不转载原厂受版权保护的大段内容。

从资料看，系统主要视频路径是：

<pre>
                         MEU / 原厂导航
                               │ GVIF
                               ▼
iPhone ── USB CarPlay ──► DCU / Radio Receiver
                               │ GVIF（b1 → b2）
                               ▼
                        ICS / TMOT 控制器
                               │ GVIF（d6 → d7）
                               ▼
                        12.3" Multi-display

停车辅助 ECU ── 独立摄像头输入 ──► Display
</pre>

几条值得精确存档的连接（**只作研究索引，不作为免核图直接接线的指令**）：

| 链路 | 已整理的端子 | 原件索引 |
|---|---|---|
| DCU → ICS | DCU `b1-1 GV+ / b1-2 GV- / b1-3 GVG` → ICS `b2-1 GV*+ / b2-2 GV*- / b2-3 GVG*` | EWD `EM36Q0U`、`RM100000001M50V` |
| ICS → DISP | ICS `d6-1 GV+ / d6-2 GV- / d6-3 GVG` → DISP `d7-2 GVI+ / d7-1 GVI- / d7-3 GVG1` | `RM100000001M50X` |
| MEU → DCU | MEU `d1` → DCU `d5`，GVIF | E16 |
| DCU ↔ ICS | AVC-LAN，DCU `H6-17/18` ↔ ICS `H8-36/35` | E23/E24 |
| ICS ↔ DISP | AVC-LAN，ICS `H8-18/40` ↔ DISP `H7-7/19` | E23/E24 |
| 摄像头 → DISP | Parking Assist ECU `H103` → Display `H104` | `RM100000001M519` |
| Visual Mute | DCU `H5-1` → ICS `H8-39 / H8-17` → DISP `H7-11` | `RM100000001M532` |

NCF 文档 `NM100000001MTZI` 对 ICS 职能的描述是：把 Radio Receiver 送来的画面与空调、座椅加热等操作界面合成，然后将结果送往 Multi-display。这是 **ICS 参与最终合成** 的直接文档证据。

ICS 的安装位置在中央扶手箱下方，而不是我最开始想象的“全部都在显示屏背后”。相关拆装文档为 `RM100000001LTMH`。T-SB-0055-22 中出现过 Highlander 的 ICS 更换零件号 `840C2-48010`；官方还警告二手 ICS 可能存在绑定/通信问题，所以未来若买台架备件不能假设插上就能用。

另外，触控的传输路径尚未确认。AVC-LAN 的确连接这些模块，但 GVIF 的反向控制机制也不能凭空排除。倒车摄像头有独立输入也**不意味着**改接 GVIF 后所有倒车场景一定安全，最终必须逐项验证。

<a id="hypotheses"></a>
## 8. H1a、H1b、H1c：我们为什么反复修改架构判断

整个项目最值得记录的，不是某个“确定结论”，反而是我们如何几次发现自己推论得太早。

| 假设 | 核心说法 | 需要什么证据才能坐实 |
|---|---|---|
| **H1a** | DCU 本身输出一幅宽画面，ICS 主要转发/辅助合成 | 抓到 DCU→ICS 的真实宽幅视频 |
| **H1b** | DCU 提供窄主画面，ICS 负责左右分屏与全宽合成 | 视频抓取或明确显示模式控制归属 |
| **H1c** | DCU 与 ICS 分工协作，DCU 可能请求不同布局，ICS 负责最终显示 | 抓到相应控制消息与输出变化 |

### 第一阶段：偏向 H1b

最早我们看到：CarPlay、原厂导航、Audio、Settings 全都受窄窗口限制，而 Trip 可以全屏。再叠加 NCF 明确写 ICS 会合成画面，很自然地得出“ICS 管最终布局”的工作模型，于是把 DCU-only 暂时归档。

后来回看，**这一步把“ICS 参与合成”扩大解释成“DCU 不可能拥有全屏视频输出或模式请求能力”**。后半句并没有得到证明。

### 第二阶段：Trip 的归属让 DCU-only 重新浮出水面

2026-11-25 的一次 TIS 审计发现：

- `RM100000001M537` 中，Trip、Energy Monitor、History 等功能故障诊断多次指向 Navigation ECU（MEU）。
- `RM100000001M50V` 显示 MEU 的视频经 GVIF 先进 DCU。
- 原厂新车功能资料 `NM100000001MHH8` 明确描述全屏与分屏两种 screen mode。

于是出现了一个相反的可能：既然 Trip 关联 MEU，而 MEU 视频通往 DCU，或许 **DCU 本身输出的是宽画面**，或者 **DCU 有权请求 ICS 将某种内容全屏**。

这条思路足以把 H1a、H1c 重新放回桌面，但也有一个逻辑漏洞：维修手册的“Trip 功能故障要检查 MEU”，**不严格等于**“Trip 页面上的每个像素都由 MEU 绘制，再完整作为视频穿过 DCU”。也可能存在数据与最终 UI 绘制分离。

所以，我现在不想再靠“某 ECU 在故障排查表里出现几次”决定架构。真正可靠的答案，最终要来自诊断状态、控制消息或 GVIF 视频测量。

**当前结论：H1a/H1b/H1c 都未被排除；ICS 参与合成是有文字依据的，DCU 是否可以驱动全屏仍未知。**

<a id="route-shift"></a>
## 9. 第一次路线收敛：从 DCU-only 转向 R-EXT

讨论中逐渐形成了两条路线。

**DCU-only**：不更换原车显示链路，只通过 DCU 的配置、现有控制协议或必要的软件改动，使原系统把 CarPlay 改成全宽输出。如果存在合法、稳定、可逆的配置入口，这条路线最优雅：硬件 BOM 很低，安装也可能最容易。

**R-EXT（External Video Injection）**：把外部 CarPlay 接收器产生的超宽视频，在靠近 LCD 的最后一段视频链路接入，避开原厂 DCU 固定的 CarPlay 窗口。代价是必须解决高速视频转换、触控、音频、模式切换、故障回退。

在我们一度偏向 H1b 时，DCU-only 被归档，R-EXT 成为主线。后来 TIS 审计重新打开了 DCU-only 的可能性，所以目前两条路线的关系是：

> **先用有边界的 D0 排查低成本原厂控制入口；没有可用入口，再把研发资源集中投入 R-EXT。**

这里的“暂停”只是研发排期，既不意味着已经证实 DCU-only 不可能，也不是要无限期逆向两套 Toyota 固件。

<a id="hvxt06"></a>
## 10. Beat-Sonic HVXT06：一个很有用但不是 CarPlay 的盒子

R-EXT 最初最大的不确定性是：Toyota 原厂 GVIF 视频有没有现成的外部输入接口？

我们查到 [Beat-Sonic HVXT06](https://www.beatsonicusa.com/pages/manuals)，面向特定 Toyota/Venza/Highlander 12.3 英寸原厂显示屏，官方标价当时约 **649 美元**。这个价格对做一个大众化产品并不理想，但作为研发参照物非常有价值。

后来最关键的一步，是实际取得并阅读它的[官方三页安装手册](https://drive.google.com/file/d/1zQh9EW9msPQ6yVCuKz2qJVeekw1Eeo1g/view)。

### 10.1 我们从手册里真正确认了什么

- 两条附带 GVIF Cable 用于把 HVXT06 **串入车辆视频线与 Display Monitor 之间**，不是简单地从 iPhone USB 里改几个 CarPlay 参数。
- 另外的 `Cable A` 插在 Radio Receiver 背面指定的**最左侧 28-pin 接头**，用于配合原车交互/音频集成；误接到其他看似相同的接头，手册警告可能损坏设备。
- 选择原车 `AUDIO → AUX` 后，HDMI 视频可以显示，音频通过原车扬声器播放。
- `HOME`、`TUNE/TRACK` 等原厂按键可用于在原厂画面与外部 HDMI 间切换、选择显示模式。
- HDMI 显示有三档：**Normal 16:9、Broad 20:9、Full 24:9**。
- 设备 USB 口只提供 **5 V / 2.4 A 电源**，不提供 USB 数据；手册明说原厂方向盘按钮**不能控制外部 HDMI 信号源的播放操作**。

这解决了一个很大的技术疑问：**确实有商业设备能够在这代原车显示器前面插入 HDMI 视频，并且利用原车音响。**

但手册不能证明的东西也很多：

| 问题 | 到目前为止 |
|---|---|
| 本车实装 HVXT06 后真的铺满 290 mm 吗 | 尚未实测 |
| Full 24:9 是否接受原生 8:3 HDMI 时序 | 未确认 |
| 还是把普通 16:9 放大、拉伸或裁切 | 未确认 |
| 原厂屏幕的触摸能否以 USB HID 等形式转发给外部设备 | 没有这项公开保证 |
| 电话、Siri 和麦克风是否完整集成 | 未验证 |
| 断电后能否以硬件方式无损直通原车 GVIF | 未验证 |

有两个细节特别容易被误用。第一，**24:9 的“显示模式”不等于“输入端支持 1920×720 原生时序”**。第二，从 HVXT06 的 HDMI 端读到 EDID，读出的是 HVXT06 作为 HDMI 接收端向源设备声明的模式，不一定是 Toyota LCD 的真实原生像素数。

### 10.2 HVXT06 能运行我们写的程序吗？

不能把它理解成一台可刷自己应用的小电脑。按照公开资料，它是 HDMI/GVIF 视频接入、切换与原厂集成设备，没有开放的应用开发环境，也没有证据表明能在上面部署自己的 CarPlay 服务。

完整系统至少还需要：

<pre>
iPhone
  │ USB / Wi-Fi CarPlay 协议
  ▼
外部 CarPlay 接收器（负责接收、渲染、输出 HDMI）
  │ HDMI
  ▼
HVXT06 或自研等效的 GVIF 视频桥接/切换硬件
  │ 原车 GVIF
  ▼
Toyota 12.3" Display
</pre>

如果我们使用成熟商用 CarPlay 接收器，视频原型不一定需要自己编程；如果要实现真正的超宽协商、原厂触控、Siri/电话和智能回退，就可能需要接收器固件、外部 MCU 或嵌入式 Linux 开发。

**HVXT06 是参考和验证工具，不进入未来产品 BOM；截至本文也尚未购买或安装它。**

<a id="rext"></a>
## 11. R-EXT 的完整技术方案与未解难题

R-EXT 拆成两个分支：

- **R-EXT-A**：先利用 HVXT06 这类成熟 HDMI 输入设备做端到端验证，避免过早自研 GVIF。
- **R-EXT-B**：确认协议后，用国内供应链设计自己的 GVIF 视频接入、切换、触控和音频板卡。

概念系统如下：

<pre>
[外部 CarPlay 接收器] ─ HDMI ─► [视频桥接 / GVIF]
                                        │
原车 ICS d6 ─► [安全的原车 / 外部视频切换] ─► DISP d7
                                        ▲
                  [控制器：切换逻辑、触控适配、状态监测]

音频：外部接收器 ─► 原车可用音频输入路径 ─► DCU / JBL
触控：DISP ─► 确认真实传输协议 ─► 控制器 ─► 接收器
</pre>

这一张图是**架构提案，不是已经验证的接线图**。尤其不能假定拿一个通用高速 2:1 开关就具有断电直通能力。

### 11.1 视频：GVIF 不是 HDMI

这代 Toyota 的视频用的是 GVIF。我们查到 Sony 有 GVIF/GVIF2 Serializer/Deserializer 产品家族，但没有确认这辆车实际用了哪一代、哪一颗芯片。

需要解决的是真实视频分辨率、时序、SerDes 代际、初始化与反向控制通道，以及是否有额外认证/保护。市场上存在某些旧款 Toyota/Lexus 的 GVIF 视频接口，**不等于**它们直接适配本车 ICS d6 → DISP d7。

最终还需一块独立的安全切换系统。断电默认直通原厂画面必须通过硬件实验验证；不能把它当作一句产品宣传语。

### 11.2 超宽：比 1920×720 更重要的是正确比例

如果最后确认 LCD 为 1920×720，那么原生 1920×720 的 CarPlay UI 是一个自然方案。另一种思路是接收器向 iPhone 声明 **1280×480（同样是 8:3）**，后级再按比例放大到目标像素。

关键是 iPhone 从一开始知道它面对的是超宽区域。以正确比例生成 UI 后适当缩放，和拿普通 16:9 内容硬拉成超宽，是两回事。

目前的难点在于：现成 CarPlay HDMI 盒子通常公开的是常规 16:9 模式；我们还没有确认具有合法商业授权、且支持自定义 8:3 输出的成熟模块。必须区分原生输出、非等比变形和裁切。

### 11.3 触控：不能先假定一定走 AVC-LAN

DISP 与 ICS/DCU 间存在 AVC-LAN，也存在 GVIF，某些 GVIF 芯片家族还支持反向控制通道。因此不能凭线束数量断言每一个触摸坐标都在 AVC-LAN 上。

后续 X3 应该先被动监听并识别物理路径，再谈解包触摸坐标、映射到 USB HID，最后解决外部模式下“手指点 CarPlay，后台 Toyota UI 也跟着误触”的问题。

有一个旧式 Toyota AVC-LAN 研究项目 [AVCLAN-Mockingboard](https://github.com/halleysfifthinc/AVCLAN-Mockingboard)，可用于理解 IEBus 物理层与抓包思路，但**它不是 2022 Highlander 触控协议现成答案**。

### 11.4 音频、方向盘按键和车身提示

HVXT06 手册说明音频可通过原厂音响播放，这给了音频接入的商业先例；但是 `Cable A` 28-pin 到底承载模拟 AUX、控制信号还是组合接口，还未确认。我们不能先把它写死成“已经找到隐藏 AUX 的若干引脚”。

电话和 Siri 还需要双向麦克风路径；方向盘按键本来进入 DCU，也不会自动传到外部 CarPlay；空调弹窗、原厂菜单与必要提示在外部全屏时如何展示同样未解。

倒车摄像头虽然有独立路径，产品仍必须对倒车、启动、断电、热重启、故障切换逐项验收。这比“HDMI 能亮起来”重要得多。

<a id="firmware"></a>
## 12. 为什么不直接重写原厂固件

另一个很自然的问题是：既然有 DCU、ICS 两块电脑，为什么不直接逆向它们，写一套新的全屏系统？

技术上当然可以研究，**但“能够读取/分析某份固件”与“能够安全、合法、低成本地向客户销售修改后的车机固件”是完全不同的问题**。

Toyota 的旧平台存在通过 TIS 提供的 `.kwi` 软件包对 DCU/MEU 更新的维修流程；ICS 故障的 TSB 以更换件为处理方式，现有材料没有列出同等现场升级流程。**没有公开更新步骤不等于已经证明 ICS 硬件不可能更新。** 同样，没有实测安全启动验证之前，不应该断言“肯定有签名”或者“肯定能绕过”。

如果直接把 DCU 系统整体替换掉，还要接管收音机、MOST 功放、AVC-LAN 主机、车辆设置、方向盘、电信与麦克风、导航 MEU、CarPlay 认证等职责。即使软件团队完成这些内容，也没自动解决 ICS 的布局权限。

产品层面更麻烦：车型与版本兼容、刷写失败的恢复、客户车辆保修、后续 Toyota 更新，以及固件知识产权与 DMCA 反规避条款。美国法律对合法车主的诊断、维修和部分互操作研究有特定例外，但**不能推导出企业可以无条件分发破解工具或 Toyota 专有镜像**；商业化必须单独做法律审查。

所以我把“自有备用 ECU 上的只读研究”保留为技术储备，不把“给每个客户刷两块 Toyota 固件”当作第一代产品的默认路线。

<a id="audit"></a>
## 13. 第二次转折：TIS 审计重新打开 DCU-only

在准备买外部视频设备时，我还是不甘心：如果原车已经有 Trip 全屏模式，会不会存在一个可以通过诊断、配置或者合法控制命令打开的宽屏模式？先把这个便宜而优雅的方向彻底检查一次，才好意思放弃。

因此在 **2026-11-25**，我们对已保存的 TIS RM/NCF/EWD/TSB 进行了 D0-1 离线审计。特别要记住：**当年下载的 TIS 资料包明确不是完整镜像**，有些章节没有保存。审计的“没找到”只代表这份资料包或相应可见菜单没有，不代表 Toyota 内部系统绝对不存在。

### 13.1 DCU 的可见诊断项

`RM100000001M5AW` 里列出的 Data List 包括：

| 项目 | 类型 | 对全屏问题的帮助 |
|---|---|---|
| Battery Voltage | 只读 | 无直接关系 |
| Active Dimmer、Tail Light SW | 只读 | 亮度与灯光状态 |
| Reverse Signal、Parking Brake、Ignition | 只读 | 车辆状态 |
| Vehicle Speed | 只读 | 车速 |
| HDMI Line (Blu-ray Disc Player) | 只读，车辆显示 Not available | 说明软件有 RSE 视频设备概念，不代表本车有 HDMI 可用接口 |

文档标题虽然出现 `DATA LIST / ACTIVE TEST`，却没有提供具体可执行 Active Test 项。Service Menu 中可见 Panel & Steering Switch、Touch Switch、Microphone Check、Vehicle Signal、HF Voice Quality Setting 等，没见显示面积参数。

**真正关键的遗漏：DCU/Navigation System 的 Customize Parameters 专页没有保存在现有资料包里。** 这也是为什么接下来仍然要上 Techstream，而不能凭 PDF 结束研究。

### 13.2 ICS / D-SEAT SW 的公开配置

对应 `RM100000001LQC3`：

| 选项 | 允许值 | 相关性 |
|---|---|---|
| Touch Panel Reaction Time Adjust | 40 / 140 / 170 / 200 ms | 触控响应时间 |
| Slide Operation | ON / OFF | 滑动操作 |

`RM100000001LQC1` 的 Data List 则只明确列出 `Number of Trouble Codes`。已取得的 ICS Customize 列表里**没有**全屏、窗口宽度、CarPlay ViewArea 等配置。但这同样只说明“受支持的可见配置未暴露”，不是“硬件能力不存在”。

还有一条重要的整车说明来自 `NM100000001MHH8`：这套系统确实存在 `full screen mode` 与 `split screen display mode`。可惜文档没有定义哪些内容允许全屏、切换逻辑在哪个 ECU。

### 13.3 Trip 归属问题必须留有余地

Trip / Energy Monitor 的症状排查条目指向 MEU；MEU 的图像线路又通向 DCU。这个发现足以推翻我们曾经非常武断的“Trip 一定完全由 ICS 自己画出来”的说法。

但它**还不足以证明 DCU 已经输出整块屏幕的全宽像素**。MEU 也可能提供内容或数据，由 ICS 做最后的绘制与布局。要区分两种可能，最终还是得看 DCU→ICS 的 GVIF 视频帧或相关模式请求。

D0-1 的结论可以简洁地写成：**没有找到公开的软件全屏开关，但也没有找到证明 DCU-only 技术上不可能的证据。**

<a id="service-menu"></a>
## 14. S0 实车 Service Menu：把三个打码项找出来

离线资料的某些维修界面截图有被遮盖的菜单项。既然车已经买下来了，我们决定亲自进 Service Menu 看一遍。

2026-11-25 的实车照片确认：

| 页面 | 实际出现的项目 |
|---|---|
| Service Menu 主页面 | Failure Diagnosis / Function Check/Setting / Service Information / Product Information |
| Function Check/Setting 第 1 页 | Panel & Steering Switch、Touch Switch、Microphone Check、Vehicle Signal、HF Voice Quality Setting、**Camera Setting** |
| Function Check/Setting 第 2 页 | **MEU Inspection**，且没有 Next Page |
| Service Information | **MEU Information、Repro info、Version** |
| Product Information | DCU `86120-0E420`、MEU `86840-0E075`，以及未公开的芯片序列号 |

这样，维修资料中之前看不清的三项就补齐了：**Camera Setting、Repro info、Version**。没有一个是明显的 CarPlay Full Screen 设置。`Repro info` 的具体作用不清楚，不会因为这个名字就去尝试刷机或改配置。`Camera Setting` 也不随便碰，以免影响摄像头功能。

实验过程中还发生了一次值得记录的小纠错：有人根据几张照片里的白色菜单框长短，推断不同 Service Menu 页面可能对应不同视频输出宽度。后来我明确确认，**这些页面实际是同样大小，差异来自拍照角度和透视**。所以这里绝不能再写成“菜单宽窄发生过变化”。

S0 的有效结论非常有限、但很扎实：**可见的原厂 Service Menu 已检查，没有显式的 CarPlay 全屏开关；DCU 的内部宽帧能力仍未知。**

<a id="diagnostics"></a>
## 15. 下一轮 D0：Techstream、MFC3 与停止条件

S0 结束后，下一步是用真正的 Toyota 诊断工具读取 ECU 配置。

普通 ELM327 型 OBD-II 读码器面向标准诊断数据，无法假定能把私有多媒体 ECU 的 Customize、Data List 和 Utility 全部读出来。我们选择 Toyota 官方推荐生态中的 **Opus IVS Mongoose-Plus Toyota3 / MFC3**，USB 有线 J2534 接口。

在 2026-11-25 查询时，Amazon 上由 OPUS IVS 销售的 Toyota 有线版标价 **575 美元**，页面显示 **11 月 28 日星期六送达**。我们已经决定按这个型号采购并等待到货；**本文写作时尚未接入实车完成诊断**。设备本身不包含 Toyota Techstream/GTS 的软件使用许可，还需要兼容的 Windows 电脑与适当授权。相关入口：[Toyota TIS](https://techinfo.toyota.com/)。

### 15.1 到货后的只读实验

| 顺序 | 操作 | 要保存的证据 |
|---|---|---|
| D0-2A | Health Check / ECU 清单 | 所有可识别 ECU、Part No.、软件/标定信息 |
| D0-2B | Navigation System（DCU） | **Customize Parameters 全部项**，尤其补齐 TIS 缺页；Data List、Utility |
| D0-2C | D-SEAT SW（推定 ICS，需按实际识别确认） | Customize、Data List、可见 Active Test 列表 |
| D0-2D | 其他识别到的 MEU / Multi-display 条目 | 版本、配置和关联状态 |
| D0-2E | 状态对照：CarPlay 左、CarPlay 右、Trip 全屏、原厂导航 | 是否有与布局切换同步变化的只读状态参数 |

全程**只读**：不执行 Active Test、不清 DTC、不修改 Customize、不做 Initialize/Registration、不试未知的刷写或控制命令。若发现看起来和 Display Mode 有关的可写项，先保存原值和维修说明，再单独设计可回退的受控实验。

### 15.2 怎么决定什么时候停

- **Go**：发现受支持的全屏配置项，或明确的 DCU→ICS 正常模式请求接口；继续研究其是否允许 CarPlay 全宽。
- **No-Go（针对低成本配置解锁）**：DCU 与 ICS 的真实可见参数都无相关项，且一次定向日志测试也找不到具体可用入口。停止继续投入“点几下菜单就解锁”的路线。
- **证据不足**：诊断软件识别不了某模块、资料缺页、或只有间接推断。老实标注未知，不把“没读到”写成“技术上不支持”。

后续可选实验包括 MMR Diagnosis Recorder 导出 `17DISP_DREC.zip`、定向的 AVC-LAN 被动监听，以及台架上的两段 GVIF 视频测量。但它们不是同一个成本等级，也不适合在没有目标时无休止地展开。

尤其要避免一个看似简单却有风险的建议：**“用一根跳线绕过 ICS，直接把 DCU GVIF 接到 LCD”**。GVIF 是高速串行视频链路，可能有特定时序、初始化、反向通道和设备约束。没有确认电气条件与专用线束之前，不把实车当作随手插拔的实验板。

因此我们目前把 **DCU-only 暂时 hold 住，等 MFC3 到货，跑完 GTS/Techstream 的只读诊断再做决定**。

<a id="product"></a>
## 16. 从改装一辆车到产品：ODM 与国内供应链

花 649 美元买一个日本品牌 HDMI 输入盒子，再买一个 CarPlay 接收器，给自己车上试验也许没问题。但如果目标是性价比高、可大规模销售的产品，不能把这一套零售价直接堆成 BOM。

我更感兴趣的是能否找到深圳或其他国内方案商，直接做一块满足需求的板卡。

| 子系统 | 可考虑的供应链方向 | 我们真正需要掌握的能力 |
|---|---|---|
| CarPlay/Android Auto | 有相应授权与 ODM 能力的成熟模块商 | 8:3 原生画布、稳定连接、麦克风/电话、SDK 或定制协议 |
| GVIF 视频桥接 | 做 Toyota/Lexus 原屏改装接口的工程团队 | 本车型视频时序、SerDes 代际、切换和故障恢复 |
| 触控 | MCU、车载接口与 HID 开发团队 | 找到真实触控路径，不误触原厂后台 |
| 音频 | 原厂音频输入兼容方案 | 音乐、Siri、电话分别测通，不猜 28-pin 引脚 |
| 线束 | 原厂插头匹配的国内线束厂 | 不剪线、可逆、接触可靠性与批量测试 |
| 质量 | 车载电子测试与认证合作方 | 温度、EMC、电源瞬态、倒车/启动/故障场景 |

一个可能最有效率的路线是**找已经掌握这代 Toyota GVIF 的 ODM 方案商**，由我们给出“原生 8:3、不拉伸、触控全宽、音频、原车回退”等验收规格，而不是从 Sony SerDes 芯片原理图开始把所有东西重新造一遍。

Nifty City、某些 Lexus/Toyota 旧款 CarPlay 改装盒提供了供应链线索，但**截至本文，没有经过实测、能够确认某款现售盒子已经在 2022 Highlander 旧平台 12.3 英寸屏幕上实现真正全屏 CarPlay**。网页写着“Full Screen”不等于用实车照片、输入分辨率和触控测试证明了它。

还有商业合规问题：CarPlay 相关硬件属于 Apple MFi 生态。**买到所谓 MFi 认证芯片，不等于自己的品牌自动取得产品销售与协议授权**。最终必须与有合适许可、可承接 ODM 和品牌交付的主体合作，核实其实际授权范围。

我希望未来的第一版产品至少满足：

1. 保留 2020–2022 目标平台的原厂 12.3 英寸屏幕。
2. CarPlay UI **原生超宽布局**，不是把 16:9 画面拉伸。
3. 原厂触摸、边缘点击、手势、必要方向盘按键工作正常。
4. 音乐、电话、Siri、麦克风分别验证。
5. 原厂导航、空调、车辆提示、倒车影像安全切换。
6. 原车插头免剪线、可恢复；失电与异常场景有经过验证的恢复路径。
7. 能按 50、200、1000 台三个规模得到有竞争力的 BOM/制造报价，并计入 NRE、认证和售后。

**“能让屏幕亮起来”只是电子原型；“客户装上能放心用”才是产品。**

<a id="status"></a>
## 17. 证据表、已知问题与项目时间线

### 17.1 目前已经真正验证的事实

| 命题 | 证据 | 状态 |
|---|---|---|
| CarPlay 约占 62% 屏宽 | 180/290 mm 尺量 | **实测** |
| 车机向 iPhone 声明 800×480 / 183×109 | sysdiagnose | **日志证实** |
| Zoom ON 截图 972×584，不改变物理窗口 | iPhone 截图与车上观察 | **实测** |
| ON 会话 H.264 编码器 800×480 | iPhone 统一日志 | **日志证实** |
| CarPlay ViewAreas Disabled | iPhone 统一日志 | **日志证实** |
| 原厂导航、Audio、Settings 与 CarPlay 都不能全屏 | 多个原厂界面测试 | **实测** |
| Trip 可以全屏 | 原厂界面测试 | **实测** |
| DCU / MEU 的两个 P/N | Service Menu Product Information 照片 | **实测** |
| Service Menu 两页完整、三个遮挡项已辨认 | S0 车内照片 | **实测** |
| ICS 参与画面合成、GVIF 拓扑 | Toyota TIS NCF/EWD Doc ID 摘要 | **文档支持，工程接线前仍须核原图** |
| HVXT06 接在屏幕前的 GVIF、支持 24:9 显示档 | Beat-Sonic 官方三页 PDF | **厂商手册证实，尚未本车安装** |

### 17.2 还没有答案的问题

| 未解决项 | 最合适的证据 |
|---|---|
| DCU 是否具有对主内容全宽输出的正常控制能力 | GTS 参数、控制日志、GVIF 帧测量 |
| Trip 全屏像素究竟在哪里合成 | DCU→ICS 与 ICS→DISP 两段视频对照 |
| LCD 原生分辨率是不是 1920×720 | 面板型号/规格书或实际像素测量 |
| 触控从 DISP 回传的物理路径 | 台架被动监听、芯片/协议验证 |
| HVXT06 是否接受真正的 8:3 HDMI 输入 | EDID、定制时序测试与网格图 |
| 现成 ODM 方案能否做本车全屏 CarPlay | 实物板卡、实车演示、触控/音频/回退测试 |
| 故障或断电时怎样保证重要原厂显示 | 专门的硬件故障注入与车辆场景验收 |

### 17.3 时间线：包括那些后来被修正的判断

| 日期/阶段 | 发生了什么 |
|---|---|
| 2026-10，早期调研 | 研究 Toyota 固定分屏、USB 代理、DCU 软件与外部显示路线；许多结论只是猜测 |
| 2026-11-24，E0 | 左右交换、Trip 全屏、iPhone sysdiagnose |
| 2026-11-24，Zoom A/B | 972×584 vs 800×480，但物理 CarPlay 窗口不变 |
| 2026-11-24，E0b-1/2 | 测得 290×110 / 180×110 mm；原厂界面全屏能力矩阵 |
| 2026-11-24，v1.0/v1.1 | 整理首次系统研究档案与 H1a/H1b/H1c |
| 2026-11-24，R-EXT v2 | ICS 合成证据推动外部视频路线；DCU-only 曾被过早归档 |
| 2026-11-24，X0-0 | 取得 HVXT06 官方手册，确认 GVIF 串入与 24:9、AUX 播放方式 |
| 2026-11-24，产品化讨论 | 明确国内供应链、ODM、HVXT06 只作研究参考 |
| 2026-11-25，D0-1 | TIS 离线审计发现 DCU Customize 缺页，Trip 与 MEU 的归属更复杂 |
| 2026-11-25，S0 | 实车 Service Menu 全部可见页检查，补出 Camera Setting、Repro info、Version；并纠正照片透视造成的“窗口大小不同”误判 |
| 2026-11-25，采购决策 | 选定 Mongoose-Plus Toyota3/MFC3（$575）用于下一次 GTS/Techstream 只读扫描；DCU-only 暂停等待硬件 |

另外，原计划的 E0b-3“拍照比较文字清晰度”因为证据价值低被主动跳过；E0b-4 的 T1–T9 加第二份 sysdiagnose、E0b-5 MMR、E0b-6 重启观察都**没有执行**，不能因为写进实验清单就记为完成。

<a id="references"></a>
## 18. 资料来源、术语与下一步

### 18.1 我把哪些材料当成什么等级的证据

- **第一手实测**：CarPlay 左右切换、Trip、原厂界面矩阵、发光区域尺量、S0 Service Menu 照片。
- **iPhone 端原始诊断**：2026-11-24 的 sysdiagnose，以及 Mac 本地 `carplay_logs.txt` 的时间戳和行号。原始包可能包含个人标识，不公开。
- **Toyota 维修资料（已保存原件的内部索引）**：`NM100000001MTZI`、`NM100000001MHH8`、`RM100000001M537`、`RM100000001M5AW`、`RM100000001LQC3`、`RM100000001M50V`、`RM100000001M50X`、`EM36Q0U`、`T-SB-0055-22` 等；这里使用的是研究项目汇总的审计摘录，电气实操前必须返回原始图纸核验。
- **厂商手册**：[Beat-Sonic 官网手册目录](https://www.beatsonicusa.com/pages/manuals)、[Manual-HVXT06.pdf](https://drive.google.com/file/d/1zQh9EW9msPQ6yVCuKz2qJVeekw1Eeo1g/view)。
- **原厂诊断与认证资料入口**：[Toyota TIS / Techstream](https://techinfo.toyota.com/)、[Apple MFi Program](https://mfi.apple.com/)。
- **公开逆向参考**：[AVCLAN-Mockingboard](https://github.com/halleysfifthinc/AVCLAN-Mockingboard)。其年代和使用场景与本车不同。

这篇不是设备选型广告，也不是兼容性承诺。有很多变量依然未知，特别是 **真正的 GVIF 视频时序、触控返回通道、LCD 真实像素和合法 CarPlay 超宽接收模块**。

### 18.2 最后的项目状态

**截至 2026-11-25：**

- 车已买，能继续做可重复的实验。
- S0 菜单审计已完成，未找到一键全屏开关。
- D0-1 TIS 离线审计已完成，**DCU Customize 的实车数据仍缺失**。
- Mongoose-Plus Toyota3/MFC3 型号已选，**等设备到货后**完成 GTS/Techstream 只读扫描。
- 在读到真实配置以前，DCU-only **hold**，不是宣告失败。
- R-EXT-A（HVXT06）与 R-EXT-B（自研 GVIF/ODM）保持候选；目前没有把 HVXT06 买进来，也没有完成外部视频改装。

这轮研究给我最大的教训，其实挺朴素：**看到一块全屏 Trip，并不能推出每一个 ECU 的视频能力；看到 iPhone 截图分辨率变化，也不能推出物理显示区域变化；看到一个有 Full 模式的 HDMI 盒子，更不能推出它支持原生超宽 CarPlay。**

接下来只有一个明确的动作：等 MFC3 到货，读 DCU 和 ICS 的真实配置。这会决定我们能不能继续走那条便宜、优雅的 DCU-only 路线。

*本文是持续更新的个人研究记录；实验与产品方案并不构成适用于其他车辆的安全改装说明。*
