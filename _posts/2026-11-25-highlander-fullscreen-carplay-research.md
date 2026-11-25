---
title: "Highlander 12.3 英寸全屏 CarPlay｜研究日志（持续更新）"
date: 2026-11-25
permalink: /blogs/highlander-fullscreen-carplay-research/
lang: zh-CN
hidden_from_blog: true
extra_css: /assets/css/highlander-research.css
read_time: true
read_time_minutes: 30
excerpt: "2022 Highlander 12.3 英寸 CarPlay 全屏项目内部研究日志：实车数据、iPhone 日志、TIS/GVIF 拓扑、路线决策和待执行实验。"
tags: [carplay, toyota, automotive-research, gvif, embedded-systems]
---

*内部研究日志 · 研究对象：2022 Highlander Limited / 12.3″ JBL Premium Audio · 截止日期：2026-11-25 · 状态：进行中 · **Unlisted**（不出现在博客列表，但 URL 可公开访问）*

<div class="hl-tldr" markdown="1">

## TL;DR / 当前状态

**问题：** 原厂 12.3″ 屏幕的 CarPlay 只能使用约 **62% 的屏幕宽度**，希望实现 **原生全宽 UI**，不是把画面横向拉伸。后续目标是开发可量产的低成本产品，不仅改好这一辆车。

**已实测：** LCD 发光区 **290×110 mm**，CarPlay **180×110 mm**；iPhone 日志显示车机协商 **800×480 px / 183×109 mm**，H.264 也以 **800×480** 编码；`ViewAreas: Disabled`。原厂导航/Audio/Settings/CarPlay 只能分屏和左右换位，Trip 可以全屏。

**已确认的链路：** Toyota TIS/EWD 中存在 `MEU→DCU→ICS→DISP` 的 GVIF 视频路径；ICS 参与画面合成。**尚未确定的是 DCU 是否能产生全宽帧，或者是否能正常请求 ICS 将主多媒体内容全屏。**

**最新实验：** S0 车机 Service Menu 两页全部核对，找出 `Camera Setting`、`Repro info`、`Version` 三个在资料截图里看不清的项目；未发现显式 CarPlay 全屏开关。此前根据照片透视猜测菜单宽度变化的结论**已撤回**。

**下一动作：** DCU-only **hold**，等待 **Opus IVS Mongoose-Plus Toyota3/MFC3（$575，已下单）** 到货，用 Techstream/GTS 做一次**只读** ECU 配置审计，重点补齐 DCU `Customize Parameters` 缺页。在拿到结果前，不买 HVXT06，也不执行 GVIF 跳线旁路。

**备选产品路线：** R-EXT（外部 CarPlay 接收器 + 原厂 GVIF 链路视频接入），优先调研国内 ODM。Beat-Sonic HVXT06 已从**官方安装手册**核实其 GVIF 串接方式和 Full 24:9 显示模式，但它**不是 CarPlay 接收器，不能运行我们的代码，也不进入最终 BOM**。

</div>

**导航 / 目录（固定锚点）**

1. [目标、边界与验收定义](#goal)
2. [实车硬件基线：DCU / MEU / ICS / DISP](#hardware)
3. [E0b 实测：显示模式矩阵](#field)
4. [E0 日志：CarPlay 协商与 H.264](#ioslog)
5. [E0-A/B：Smart Display Zoom](#zoom)
6. [E0b-1：物理尺寸与 62.07%](#geometry)
7. [TIS/EWD：视频、控制与音频链路](#topology)
8. [H1a/H1b/H1c：待验证的三种架构](#hypotheses)
9. [决策记录：DCU-only 与 R-EXT](#route-shift)
10. [X0-0：HVXT06 官方手册拆解](#hvxt06)
11. [R-EXT：视频、触控、音频的技术任务](#rext)
12. [固件路线：可研究，不作为首版默认方案](#firmware)
13. [D0-1：TIS 离线诊断项审计](#audit)
14. [S0：Service Menu 实车补证](#service-menu)
15. [D0-2：MFC3 / Techstream 待执行计划](#diagnostics)
16. [产品路线：国内 ODM 与交付要求](#product)
17. [事实、未决项与时间线](#status)
18. [参考资料和续研入口](#references)

**记录约定：** `[实测]` = 本车或 iPhone 原始数据；`[TIS]` = 已保存的 Toyota 原件索引、部分摘录由 Claude 审计；`[手册]` = 已核对的第三方厂商 PDF；`[推断]` = 合乎现有数据但尚未证明；`[待测]` = 设计中的实验。所有示意图均不作为未经验证的电气接线指令。

---
<a id="goal"></a>
## 1. 目标、边界与验收定义

### Scope

| 项目 | 当前定义 |
|---|---|
| 目标车辆 | 2022 Highlander Limited，原厂 12.3″ Premium Audio / JBL |
| 症状 | CarPlay 始终局限在 180 mm 主区，左右可以互换但不能真正全屏 |
| 产品目标 | 保留原厂屏幕，iPhone 为全宽显示区域直接生成 CarPlay UI |
| 排除项 | 横向拉伸 16:9、裁切填满、单纯更换 iPhone 截图尺寸 |
| 原厂功能约束 | 触摸、音频、Siri/电话、倒车、空调、方向盘交互、掉电恢复必须经过验收 |
| 改装边界 | 不主动向动力/ADAS CAN 注入未知消息；不将 Toyota 专有固件分发作为产品方案 |

**成功判据 S1–S4：** S1 画面覆盖完整 LCD 有效区；S2 CarPlay 接收器协商的逻辑画布本身为宽幅比例（例如 8:3）；S3 原厂触摸输入能正确驱动 CarPlay 且不误触后台；S4 音频、通话、语音、重要原厂提示和故障回退正常。

**未证明的前提：** LCD 原生像素可能是 1920×720，但我们从未直接测量。后续所有使用 1920×720 的地方均是示例，而不是屏幕已确认规格。

<a id="hardware"></a>
## 2. 实车硬件基线：DCU / MEU / ICS / DISP

### Vehicle / ECU inventory

| 单元 | 实物或资料记录 | 职责 / 备注 |
|---|---|---|
| DCU / Radio Receiver | Panasonic；P/N `86120-0E420`；Model ID `17TDTENA-DA30`；Audio `1225` | CarPlay 端点、主多媒体、原车音频 |
| MEU / Navigation ECU | P/N `86840-0E075`；Model ID `17TMTENA-DA50`；Nav `1225` | 导航扩展模块 |
| ICS / Integration Control | DENSO TMOT；TSB 中更换件号 `840C2-48010` | 图像合成；该件号尚未通过本车 ICS 标签独立确认 |
| DISP / Multi-display | 原厂 12.3″；发光区 290×110 mm | LCD 原生分辨率未知 |
| JBL 放大器 | 由 DCU 的 MOST 音频链路管理 | R-EXT 音频对接不能简单假定有 AUX 引脚 |

**证据：** DCU、MEU 的 P/N 后来由实车 `Product Information` 菜单照片再次核对；ICS 件号仅记录为适用 TSB 的替换件，不是实车拆机读标。

**命名陷阱：** iPhone 枚举的 `CV-RT29N04J` 是 CarPlay 设备标识，不能当 Toyota 零件号。HID 字段 `20.2.14` 也不能冒充车机完整固件版本。服务菜单中的 Audio/Nav 版本记录为 `1225`。

**平台边界：** 研究的是旧款 12.3″ TMOT 架构；不能把 2023+ Toyota Audio Multimedia 的界面、更新方式和改装设备直接套用。

<a id="field"></a>
## 3. E0b 实测：显示模式矩阵

### Input / Steps

停在 P 挡，分别打开 CarPlay、原厂导航、Audio、Settings、Trip。记录“左/右互换”“是否存在全屏动作”“全屏时是否保留其它 UI”。E0b-3 的文字清晰度微距对比因判别力低而**主动跳过**。

| 源界面 | 左右互换 | 全屏 | 结果 |
|---|---|---|---|
| CarPlay | 是 | **否** | 固定主显示区域 |
| Toyota 原厂导航 | 是 | **否** | 与 CarPlay 同类布局限制 |
| Audio / Radio | 是 | **否** | 同上 |
| Settings | 是 | **否** | 同上 |
| Trip Information | 是 | **是** | 唯一已观察到的此类全屏状态 |

<figure class="hl-figure">
<img src="{{ '/assets/images/highlander/01-window-modes.svg' | relative_url }}" alt="实测 CarPlay 左右两种布局和 Trip 全屏，按 290 毫米发光宽度绘制" loading="lazy">
<figcaption>图 1 · 按实测 290/180/110 mm 绘制；不是屏幕实际像素图。</figcaption>
</figure>

### Interpretation

- `[实测]` Trip 全屏证明这台车存在至少一种完整 LCD 有效区的视频显示状态。
- `[实测]` 原厂导航等不能全屏，因此“只是 iPhone / CarPlay 限制”的解释过于简单。
- `[未知]` Trip 是哪块 ECU 最终绘制像素，DCU 送出的 GVIF 是否为全宽帧。
- 在 Trip 遮住 CarPlay 的过程中，iPhone 侧截图仍反映音乐状态变化；**未抓到该时段连续的视频帧统计**，不应扩大解释。

<a id="ioslog"></a>
## 4. E0 日志：CarPlay 协商与 H.264

**采集方法：** 我在 **2026-11-24 19:35 PST** 左右生成了一份 iPhone sysdiagnose，在 Mac 上用统一日志工具解码，得到约 7,581 行的 CarPlay 相关日志。第一份解码文本保存在本地 `~/Desktop/carplay_logs.txt`。

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

**结论：** `972×584` 是 Zoom ON 时的 UI 画布，而本次采集的实际 H.264 编码器初始化仍为 `800×480`。`[未知]` LCD 原生像素、最终放大倍率与谁执行合成。

<a id="zoom"></a>
## 5. E0-A/B：Smart Display Zoom

### A/B 实验

| 操作 | iPhone 保存的 CarPlay 截图 | 车上 CarPlay 有效区域 |
|---|---:|---|
| Smart Display Zoom ON | 972×584 px | 180×110 mm，不变 |
| Smart Display Zoom OFF | 800×480 px | 180×110 mm，不变 |

**观察：** 关闭 Zoom 后图标大小与 UI 排版密度发生变化，车屏窗口边界不变。原 sysdiagnose 的 ON 状态记录里，H.264 编码器是 800×480；OFF 状态**没有第二份单独的编码初始化日志**。

**判断：** `[实测]` Smart Display Zoom 不是本车全屏开关。此分支已停止。

<a id="geometry"></a>
## 6. E0b-1：物理尺寸与 62.07%

### 几何核对

| 来源 | 宽 × 高 | 状态 |
|---|---|---|
| 实车发光区域 | **290 × 110 mm** | `[实测]` |
| 实车 CarPlay 区域 | **180 × 110 mm** | `[实测]` |
| iPhone 收到的 `ScreenPhysicalSize` | **183 × 109 mm** | `[日志]` |
| iPhone 收到的 `ScreenPixelSize` | **800 × 480** | `[日志]` |

$$
\text{CarPlay 占宽}=\frac{180}{290}=62.07\%,\qquad \frac{290}{110}\approx2.636
$$

物理宽高比接近 8:3；尺量与 iPhone 收到的 183×109 mm 也基本一致。这解释了为什么原厂 CarPlay 的界限如此稳定。

<figure class="hl-figure">
<img src="{{ '/assets/images/highlander/02-carplay-resolution.svg' | relative_url }}" alt="Smart Display Zoom UI 画布、H264 编码器和车屏物理窗口的不同层级" loading="lazy">
<figcaption>图 2 · 三种尺寸分别来自 UI、编码和尺量，不能互相替代。</figcaption>
</figure>

**待验证的缩放模型（绝不作为事实）：** 如果屏幕原生为 1920×720，并且将 800×480 放大 1.5 倍，则主视频会成为 1200×720，占全屏 62.5%。它与 62.07% 的实测很接近，但**不能证明 LCD 是 1920×720，更不能证明缩放发生在 ICS 还是 DCU**。

<a id="topology"></a>
## 7. TIS/EWD：视频、控制与音频链路

**输入：** 已合法保存的 Toyota RM、EWD、NCF、TSB 资料包（并非完整订阅镜像）。这一节记录拓扑与对应 Doc ID，所有端子接线前必须返回原始 EWD 核对。

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

<figure class="hl-figure">
<img src="{{ '/assets/images/highlander/03-ecu-topology.svg' | relative_url }}" alt="MEU、DCU、ICS、DISP 与 MOST、摄像头及 AVC-LAN 的连接关系" loading="lazy">
<figcaption>图 3 · 根据 TIS/EWD 摘要绘制的逻辑拓扑。虚线是控制概念，不是已解析报文。</figcaption>
</figure>

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
## 8. H1a/H1b/H1c：待验证的三种架构

### Working hypotheses

| ID | 机制 | 能解释什么 | 仍缺什么 |
|---|---|---|---|
| **H1a** | DCU 输出完整宽幅画面，ICS 主要承接/叠加 | Trip 可能源于 MEU→DCU；CarPlay 窗口限制也可能由 DCU UI 决定 | DCU→ICS 的真实视频帧 |
| **H1b** | DCU 主要输出较窄主内容，ICS 最终布局、合成并铺到 LCD | NCF 明确写 ICS 合成 DCU 画面和空调等页面 | Trip 全宽时视频路径如何变化 |
| **H1c** | DCU 和 ICS 协作，存在模式请求或按源切换不同视频配置 | 既保留 ICS 合成职责，也允许 DCU 参与全屏控制 | AVC-LAN/其他控制消息与视频同步证据 |

<figure class="hl-figure">
<img src="{{ '/assets/images/highlander/04-architecture-hypotheses.svg' | relative_url }}" alt="DCU 主导、ICS 主导和混合架构三种工作假设" loading="lazy">
<figcaption>图 4 · 三种假设都未排除；画面中的块长不是已测量的帧宽。</figcaption>
</figure>

### 重要的两次纠错

**第一次（2026-11-24）：** 看到 CarPlay / 原厂导航 / Audio 只在窄窗口、Trip 却能全屏，又读到 ICS 的图像合成说明，于是把 H1b 当成强支持、提前归档 DCU-only。问题是，**“ICS 参与合成”不等于“DCU 不能输出全宽或请求全屏”**。

**第二次（2026-11-25）：** TIS 故障诊断将 Trip / Energy Monitor 的功能问题主要归到 MEU，而 MEU 通过 DCU 发送视频。于是 H1a/H1c 重新成为合理候选。但“MEU 被列为故障责任件”**仍不能严格推出**“Trip 最终每个像素都由 MEU 经 DCU 输出”，因为数据归属与 UI 合成归属可能不同。

**当前判定：** `[文档]` ICS 参与合成；`[实测]` Trip 可全屏；`[未知]` 画面尺寸和控制权在 DCU/ICS 之间如何分配。暂停凭 UI 截图给 H1a/H1b 打概率分数。

<a id="route-shift"></a>
## 9. 决策记录：DCU-only 与 R-EXT

### 2026-11-24：第一次路线决策

| 路线 | 核心改动 | 成功前提 | 当前 |
|---|---|---|---|
| DCU-only / 原厂配置解锁 | 只改 DCU 可见配置、受支持的模式请求或软件行为 | DCU 有办法产生/请求全宽 CarPlay，且能有可部署入口 | **hold，待 D0-2** |
| R-EXT | 在最终 GVIF 视频链路接入外部超宽 CarPlay | 搞定 GVIF、触控、双向音频、安全回退 | **保留研发主线** |

此前因 H1b 被高估，DCU-only 曾被过早归档。D0-1 审计后撤回这一判断。两条路线不是互斥产品理念：**先用一次有界诊断验证原厂解锁，再决定是否投入外部硬件**。

<a id="hvxt06"></a>
## 10. X0-0：HVXT06 官方手册拆解

**X0-0 的目标：** 不买硬件，先通过官方安装手册判断视频到底插在 DCU→ICS 还是 ICS→DISP 一侧，确定 HVXT06 是否能作为 R-EXT 的视频验证工具。

我们查到 [Beat-Sonic HVXT06](https://www.beatsonicusa.com/pages/manuals)，面向特定 Toyota/Venza/Highlander 12.3 英寸原厂显示屏，官方标价当时约 **649 美元**。这个价格对做一个大众化产品并不理想，但作为研发参照物非常有价值。

后来最关键的一步，是实际取得并阅读它的[官方三页安装手册](https://drive.google.com/file/d/1zQh9EW9msPQ6yVCuKz2qJVeekw1Eeo1g/view)。

<figure class="hl-figure">
<img src="{{ '/assets/images/highlander/05-beatsonic-insertion.svg' | relative_url }}" alt="HVXT06 通过两条 GVIF 线串在屏幕前，并通过 Cable A 连接 DCU" loading="lazy">
<figcaption>图 5 · HVXT06 官方手册第 1–3 页的连接与 24:9 模式摘要；并非实车安装照片。</figcaption>
</figure>

### 10.1 官方手册确认项

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
## 11. R-EXT：视频、触控、音频的技术任务

<figure class="hl-figure">
<img src="{{ '/assets/images/highlander/06-rext-branches.svg' | relative_url }}" alt="使用 HVXT06 的研发路线和自研或 ODM GVIF 的产品路线" loading="lazy">
<figcaption>图 6 · R-EXT-A 是验证路线，R-EXT-B/ODM 才可能成为低成本产品；均未实车完成。</figcaption>
</figure>

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
## 12. 固件路线：可研究，不作为首版默认方案

### 路线决策记录

本阶段不把“给客户刷 Panasonic / DENSO 固件”作为产品基本路径，**并不是因为已证明技术上做不到**。主要是交付成本、车型兼容、更新/回滚、失误风险与授权责任在产品场景里不可控。

| 事实或假设 | 当前状态 |
|---|---|
| DCU/MEU 存在经 Toyota TIS 提供 `.kwi` 软件包的官方更新历史 | `[TIS/TSB]` |
| ICS 故障 TSB 使用更换件处理，未看到同等现场升级流程 | `[TIS/TSB]`；**不等于硬件绝无升级机制** |
| 本车 Secure Boot、镜像签名实现及可否自行刷写 | `[未知]`；不以“车规通常有签名”充当实测 |
| 即使重写 DCU，仍可能需要适配 ICS 的显示控制 | `[推断]` |
| 产品销售涉及固件许可、版权和反规避合规 | 上市前法律审查项，不能仅凭个人研究例外判断 |

**保留：** 自有备用件上的只读固件分析、正常诊断接口探索；**不做：** 将分发第三方专有固件或绕过原厂安全机制设为默认商业路线。若 D0-2 意外找到正常的布局配置，这条“低 BOM”路线优先级会重新提高。

<a id="audit"></a>
## 13. D0-1：TIS 离线诊断项审计

**目的：** 完成 DCU-only 的低成本最终验证第一段：离线审计现有 TIS 原件是否列有全屏布局选项、Data List 状态、可用的控制接口。

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
## 14. S0：Service Menu 实车补证

**实验目的：** TIS 服务菜单截图有三项被遮挡。实车确认其名称，审计全部可见页，不写入任何配置。

<figure class="hl-figure">
<img src="{{ '/assets/images/highlander/08-service-menu.svg' | relative_url }}" alt="依据车内照片重绘的 Service Menu 和 Function Check Setting 两页菜单" loading="lazy">
<figcaption>图 7 · 用户实车照片的菜单结构重绘：Camera Setting、Repro info、Version 均已识别。</figcaption>
</figure>

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
## 15. D0-2：MFC3 / Techstream 待执行计划

**采购/执行状态：** 设备型号已选定，尚未收到 GTS 实车读取结果。执行窗口是 MFC3 到货之后，**当前 DCU-only 处于 hold**。

<figure class="hl-figure">
<img src="{{ '/assets/images/highlander/07-d0-decision.svg' | relative_url }}" alt="D0-1 TIS、S0 菜单、D0-2 GTS 和最终决定的流程" loading="lazy">
<figcaption>图 8 · D0 是有停止条件的诊断研究，不是无限期追查隐藏固件代码。</figcaption>
</figure>

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
## 16. 产品路线：国内 ODM 与交付要求

**产品原则：** 不把 HVXT06 的 649 美元零售价当产品 BOM。优先寻找掌握 Toyota/Lexus GVIF 接入的国内 ODM，必要时自主开发触控和控制模块。当前仍处于供应商验证之前；没有“已找到可量产方案商”的证据。

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
## 17. 事实、未决项与时间线

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
## 18. 参考资料和续研入口

### 18.1 我把哪些材料当成什么等级的证据

- **第一手实测**：CarPlay 左右切换、Trip、原厂界面矩阵、发光区域尺量、S0 Service Menu 照片。
- **iPhone 端原始诊断**：2026-11-24 的 sysdiagnose，以及 Mac 本地 `carplay_logs.txt` 的时间戳和行号。原始包可能包含个人标识，不公开。
- **Toyota 维修资料（已保存原件的内部索引）**：`NM100000001MTZI`、`NM100000001MHH8`、`RM100000001M537`、`RM100000001M5AW`、`RM100000001LQC3`、`RM100000001M50V`、`RM100000001M50X`、`EM36Q0U`、`T-SB-0055-22` 等；这里使用的是研究项目汇总的审计摘录，电气实操前必须返回原始图纸核验。
- **厂商手册**：[Beat-Sonic 官网手册目录](https://www.beatsonicusa.com/pages/manuals)、[Manual-HVXT06.pdf](https://drive.google.com/file/d/1zQh9EW9msPQ6yVCuKz2qJVeekw1Eeo1g/view)。
- **原厂诊断与认证资料入口**：[Toyota TIS / Techstream](https://techinfo.toyota.com/)、[Apple MFi Program](https://mfi.apple.com/)。
- **公开逆向参考**：[AVCLAN-Mockingboard](https://github.com/halleysfifthinc/AVCLAN-Mockingboard)。其年代和使用场景与本车不同。

**内部原件索引（不公开个人诊断原始包）：** 本地 `~/Desktop/carplay_logs.txt`；Toyota TIS 的 RM/NCF/EWD/TSB 原件；实车 S0 照片；Beat-Sonic `Manual-HVXT06.pdf`。这些原件可能带有个人或设备标识，博客仅保留无敏感信息的摘录。

这篇不是兼容性承诺。有很多变量依然未知，特别是 **真正的 GVIF 视频时序、触控返回通道、LCD 真实像素和合法 CarPlay 超宽接收模块**。

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
