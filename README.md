# OpenGemi
### Your Phone, Any AI

**Open-source mobile AI agents using the VirtualDisplay infrastructure Google built but locked away.**

**把 Google 限制在 Pixel + Gemini 的 AI 自动化能力，解放给所有 Android 设备和所有 AI 模型。**

---

### TL;DR

Google built a complete system in Android 17 that lets AI agents control your phone without taking over your screen. They locked it to Gemini on Pixel devices only. We reverse-engineered it, reimplemented the functionality based on open-source AOSP, and made it work with any AI model (Claude, GPT-4, Llama, etc.) on any Android 17+ device.

**一句话总结**：Google 在 Android 17 里造了一套 AI 控制手机的系统，但只给自己的 Gemini + Pixel 用。我们基于开源 AOSP 重新实现了这个功能，现在任何 Android 17+ 设备都能用任何 AI 模型。

---

### Why This Matters

- **Google solved mobile AI automation** - VirtualDisplay lets AI and human use the phone simultaneously, zero conflict
- **But kept it closed** - Six security gates lock it to Gemini + Pixel only
- **We opened it** - Any model, any device (that runs Android 17)

**为什么重要**：
- **Google 解决了移动 AI 自动化的核心问题** - AI 和人可以同时用手机，互不干扰
- **但只给自己用** - 六道门锁死，只能 Gemini + Pixel
- **我们打开了** - 任何 AI，任何设备（只要跑 Android 17）

---

### What You Get

```
Your Phone = Two Phones

┌─────────────────────┐     ┌─────────────────────┐
│   Physical Screen   │     │   Virtual Screen    │
│                     │     │                     │
│   You:              │     │   AI Agent:         │
│   - Browse          │     │   - Book restaurant │
│   - Watch videos    │     │   - Compare prices  │
│   - Reply messages  │     │   - Fill forms      │
│                     │     │                     │
│   No interference   │ ←→  │   No interference   │
└─────────────────────┘     └─────────────────────┘
```

**你得到的是**：一个手机变两个，一个给你用，一个给 AI 用，谁都不耽误谁。

---

## What Google Secretly Built in Android 17

You've probably never heard of this feature.

Google buried a wildly ambitious AI capability deep in the mobile operating system. Hidden in Android 17, there's a complete system that lets AI agents operate your phone—launch apps, tap buttons, read screens, complete tasks—all **without ever taking over the screen you're looking at**.

Google calls it **Screen Automation** externally. Internal codename: **Bonobo**.

But you've probably never heard of it. Because Google locked it behind six layers of security gates and restricted it to their own AI (Gemini) on their own phones (Pixel 10 and newer). This isn't a beta feature. It's not in the settings menu. It's buried deep in the system layer, waiting to be activated.

**This project removes those restrictions through system modifications.**

---

**Google 在 Android 17 里偷偷藏了什么**

你大概从来没听说过。

Google 在 Android 17 里埋了个野心极大的功能。一整套系统，能让 AI 代理帮你操作手机——开应用、点按钮、读屏幕、完成任务——**而且完全不会抢你正在看的那块屏幕**。

对外叫 **Screen Automation**，内部代号 **Bonobo**。

但你从没听说过，因为 Google 给它加了六道锁，只让自己的 AI（Gemini）在自己的手机（Pixel 10+）上用。这不是 Beta 功能，设置里也找不到。它就躺在系统底层，等着被激活。

**这个项目通过系统修改移除了这些限制。**

---

## Core Concept: A Second Screen Only the AI Can See

Imagine you're using your phone normally—scrolling messages, watching videos, browsing around. Now imagine there's a second screen inside your phone that you can't see, where an AI assistant is opening apps, filling forms, checking prices. When it's done, it hands you the result.

**That's exactly what Google built.**

At the technical level it's called **VirtualDisplay**, but the concept is simple: a completely software-implemented phone screen. Apps can run on it, buttons can be tapped, screenshots can be captured—but it never appears on the physical display in front of you. It's the AI's dedicated workspace.

This design solves the biggest problem in mobile automation: **who controls the screen?** Every previous approach to letting AI operate phones had the same fundamental conflict—AI needs to see the screen to decide what to do next, and needs to control the screen to execute actions; but if you're using that screen, everything breaks. AI taps a button, you get force-jumped away; you swipe a page, AI gets confused.

**Google's solution is elegantly brutal: give the AI its own dedicated screen. You keep yours. Zero interference.**

---

**核心原理：一块只有 AI 能看见的第二屏幕**

假设你正常用手机——刷消息、看视频、瞎逛。同时手机里还有一块你看不见的屏幕，AI 在那上面开应用、填表格、查价格。搞定了就把结果给你。

**Google 造的就是这个。**

技术上叫 **VirtualDisplay**，概念很简单：纯软件虚拟出来的手机屏幕。应用能跑，按钮能点，截图能抓——但你物理屏幕上完全看不见。这是 AI 的专属工作台。

这设计解决了手机自动化的最大难题：**屏幕到底归谁管？** 以前所有让 AI 操作手机的方案都有同一个矛盾——AI 要看屏幕才知道下一步干啥，要控制屏幕才能执行操作；但你在用屏幕的话，一切就乱套了。AI 点个按钮，你被强制跳走；你滑一下页面，AI 就懵了。

**Google 的解法简洁粗暴：给 AI 一块专属屏幕，你的还是你的。谁也不耽误谁。**

---

## Why You've Never Seen This

If this system is so sophisticated, why isn't anyone using it? Because Google locked it down with six layers of security gates, then kept it exclusively for themselves. Beyond the great wall, no one outside Google's ecosystem can touch it:

1. **Feature Flag Disabled by Default** - The entire system is controlled by a single configuration switch, hardcoded to `false`. Even if all other conditions are met, this one switch keeps it dormant.

2. **Permission Checks** - The system requires specific permissions that ordinary apps can't request—only system-level components can apply.

3. **Device Whitelist** - Even with the right permissions, the system checks if your phone is on an approved list. Currently, that list only includes Pixel 10 and newer devices.

4. **Certificate Validation** - Apps attempting to use the system must be signed with specific cryptographic certificates. In practice, this means they must be Google's own software.

5. **User Consent Flow** - Before the AI can start a session, the user must explicitly consent. The flow includes authorization pages and confirmation dialogs.

6. **Runtime Interception** - Even if you somehow pass all the above gates, there are additional runtime checks that will terminate the process if the system isn't satisfied.

These aren't bugs or oversights. They're deliberate, layered security measures. Google clearly wants this system to exist, but only under their complete control—their AI, their phones, their rules.

---

**为什么你从没见过**

这么牛的系统，怎么没人用？因为 Google 上了六道锁，然后只给自己用。长城之外，Google 生态之外的人碰都碰不到：

1. **Feature Flag 默认关闭** - 整个系统一个开关控制，硬编码成 `false`。其他条件全满足也没用，这开关一关，系统就休眠。

2. **权限校验** - 需要特殊权限，普通应用申请不了——只有系统级组件才行。

3. **设备白名单** - 就算拿到权限，系统还要检查你手机在不在批准名单上。目前名单只有 Pixel 10 及以后的型号。

4. **证书校验** - 想用这系统的应用必须用特定的加密证书签名。实际上就是说，必须是 Google 自己的软件。

5. **用户同意流程** - AI 启动之前，用户必须明确同意。流程里有授权页、确认弹窗。

6. **运行时拦截** - 就算你千辛万苦过了前面五关，还有额外的运行时检查，系统不爽就直接掐断。

这些不是 Bug，不是疏忽。是故意设计的多层防护。Google 显然想让这系统存在，但前提是完全在自己控制下——自己的 AI、自己的手机、自己的规矩。

---

## The Catch: It Only Works with Gemini on Pixel

A powerful, production-quality, platform-level AI automation system—locked to one AI vendor on one phone brand.

VirtualDisplay doesn't care which AI is analyzing screenshots. Session control doesn't care which model decides where to tap next. The infrastructure itself is model-agnostic. The restrictions are artificial.

**This is what OpenGemi does:**

Remove those restrictions through system modifications. Let anyone use the AI model of their choice—Claude, GPT-4, locally-run Llama, whatever—to control their own Android phone, using the open-source AOSP infrastructure we reimplemented.

---

**问题在这：只能 Gemini + Pixel**

一套强大的、生产级的、平台级 AI 自动化系统——锁死在一个 AI 厂商的一个手机品牌上。

VirtualDisplay 不关心是哪个 AI 在分析截图。Session 控制不关心是哪个模型在决定下一步点哪。基础设施本身是模型无关的。限制是人为加上去的。

**OpenGemi 就是干这个的：**

通过系统修改移除这些限制。让任何人都能用自己选的 AI 模型——Claude、GPT-4、本地跑的 Llama，随便什么——来控制自己的 Android 手机，基于我们重新实现的开源 AOSP 基础设施。

---

## What We Built

### 1. Reverse-Engineered Google's System

Through static analysis (decompiling `services.jar`), runtime inspection (Frida hooks), and network protocol analysis (packet capturing Robin gRPC), we mapped out the entire architecture:

- **ComputerControl Manager Service** - Core system component managing virtual display lifecycle
- **Six Security Gate Implementation Details** - How the system checks work
- **Robin Protocol** - Google's proprietary gRPC protocol connecting to Gemini Pro
- **AIDL Interface** - How apps request Sessions from the system

None of this was open-sourced. No documentation. We analyzed it piece by piece.

### 2. System Patches: Remove the Restrictions

We modified system components based on open-source AOSP to remove the restrictions:

```java
// Original code
private static final Set<String> ALLOWED_PACKAGES = ImmutableSet.of(
    "com.google.android.apps.bard",
    "com.google.android.googlequicksearchbox"
);

// Patched
private static final Set<String> ALLOWED_PACKAGES = ImmutableSet.of(
    "*"  // Allow any package
);
```

Similar modifications applied to Feature Flags, permission checks, certificate validation—all six gates. Result: any app can create a ComputerControl Session.

### 3. Replace AI Backend: From Gemini to OpenRouter

Google's Bonobo agent is hardcoded to connect: Robin protocol → Gemini Pro. We replaced it:

```
OpenGemi Agent → OpenRouter Gateway → Any model you choose
```

OpenRouter is an AI model aggregator supporting:
- Anthropic Claude (Sonnet, Opus)
- OpenAI GPT-4o/o1
- Google Gemini Pro (ironically, you can use it via OpenRouter)
- Meta Llama 3.3
- Plus dozens of other open-source/commercial models

Just provide your API key, select a model, OpenGemi handles the rest.

### 4. Open-Source Everything

- System patches (diff files, apply to AOSP)
- Agent implementation (Kotlin, open-source APK)
- Technical documentation ([docs/](docs/))
- Installation guide ([INSTALL.md](INSTALL.md))

---

**我们做了什么**

### 1. 逆向 Google 的系统

通过静态分析（反编译 `services.jar`）、运行时检查（Frida hook）、网络协议分析（抓包 Robin gRPC），把整套架构摸清了：

- **ComputerControl Manager Service** - 系统核心，管虚拟屏幕生命周期
- **六道安全门的实现细节** - 系统如何进行检查
- **Robin Protocol** - Google 专有的 gRPC 协议，连 Gemini Pro
- **AIDL 接口** - 应用怎么向系统请求 Session

这些都没开源，没文档。一点点分析出来的。

### 2. 打补丁：去掉限制

基于开源 AOSP 修改系统组件，移除这些限制：

```java
// 原始代码
private static final Set<String> ALLOWED_PACKAGES = ImmutableSet.of(
    "com.google.android.apps.bard",
    "com.google.android.googlequicksearchbox"
);

// 打补丁后
private static final Set<String> ALLOWED_PACKAGES = ImmutableSet.of(
    "*"  // 允许任何应用
);
```

Feature Flag、权限检查、证书校验——六道门全改。结果：任何应用都能创建 ComputerControl Session。

### 3. 换 AI 后端：从 Gemini 换成 OpenRouter

Google 的 Bonobo agent 硬编码连接：Robin 协议 → Gemini Pro。我们换成：

```
OpenGemi Agent → OpenRouter Gateway → 你选的任何模型
```

OpenRouter 是个 AI 模型聚合器，支持：
- Anthropic Claude（Sonnet、Opus）
- OpenAI GPT-4o/o1
- Google Gemini Pro（讽刺吧，通过 OpenRouter 还能用它）
- Meta Llama 3.3
- 以及几十个其他开源/商业模型

提供 API key，选模型，剩下的 OpenGemi 搞定。

### 4. 全部开源

- 系统补丁（diff 文件，打到 AOSP 上）
- Agent 实现（Kotlin，开源 APK）
- 技术文档（[docs/](docs/)）
- 安装指南（[INSTALL.md](INSTALL.md)）

---

## What This Means

**Your phone becomes two phones.**

One for you—scroll social media, watch videos, reply to messages.
One for the AI—book restaurants, compare prices, fill forms.

Running simultaneously. Zero interference.

This isn't science fiction. This is technology Google already built. They just locked it inside the Gemini + Pixel walled garden.

**OpenGemi removes those restrictions.**

---

**这意味着什么**

**你的手机变成两个。**

一个给你——刷社交、看视频、回消息。
一个给 AI——订餐厅、比价格、填表单。

同时跑。谁也不影响谁。

这不是科幻。这是 Google 已经造出来的技术。只是他们把它锁在 Gemini + Pixel 的围墙里了。

**OpenGemi 移除了这些限制。**

---

## Current Status

**This is a research prototype, not a product.**

Requirements:
- Rooted Android 17 device or custom ROM
- System-level modifications (not a simple APK install)
- Understanding of Android internals (know what ADB, fastboot, flashing means)

If you're uncertain what these mean, now isn't your time. Wait for future versions that might have simpler installation (though no promises).

But if you're a developer, researcher, or just a deeply curious geek—this is currently the only way to experience Google-level mobile AI automation on non-Pixel devices, with non-Gemini models.

---

**当前状态**

**这是研究原型，不是产品。**

需要：
- Root 过的 Android 17 设备或自定义 ROM
- 系统级修改（不是装个 APK 那么简单）
- 懂 Android 底层（知道 ADB、fastboot、刷机是啥）

不确定这些是啥意思的话，现在还不是你的时候。等将来可能会有更简单的安装方式（不保证）。

但如果你是开发者、研究者、或者就是好奇心爆棚——这是目前唯一能在非 Pixel 设备上、用非 Gemini 模型、体验 Google 级移动 AI 自动化的方法。

---

## Quick Start

1. **Read the documentation** to understand what Google built, why it matters, how it's implemented:
   - [What Google Built](docs/01-what-google-built.md)
   - [Why This Matters](docs/02-why-it-matters.md)
   - [How OpenGemi Works](docs/03-how-opengemi-works.md)

2. **Install the system** - Two paths:
   - [Full Build](INSTALL.md#method-1-system-build): Compile patched Android system image (clean but slow, 2-8 hours)
   - [Runtime Patching](INSTALL.md#method-2-runtime-patching): Use Frida to modify system behavior at runtime (fast but lost on reboot)

3. **Configure OpenRouter**, select your preferred AI model

4. **Run your first agent** - Try something simple: "Open Settings", "Check weather", "Book restaurant for tomorrow"

5. **Report issues** - [GitHub Issues](https://github.com/Equality-Machine/efflora-release/issues)

---

**快速开始**

1. **读文档**，了解 Google 造了啥、为啥重要、怎么实现的：
   - [Google 造了什么](docs/01-what-google-built.md)
   - [为什么重要](docs/02-why-it-matters.md)
   - [OpenGemi 怎么做的](docs/03-how-opengemi-works.md)

2. **装系统** - 两条路：
   - [完整构建](INSTALL.md#method-1-system-build)：编译打过补丁的 Android 系统镜像（干净但慢，2-8 小时）
   - [运行时补丁](INSTALL.md#method-2-runtime-patching)：用 Frida 运行时修改系统行为（快但重启就没了）

3. **配置 OpenRouter**，选你想用的 AI 模型

4. **跑第一个 agent** - 试点简单的："打开设置"、"查天气"、"订明天的餐厅"

5. **反馈问题** - [GitHub Issues](https://github.com/Equality-Machine/efflora-release/issues)

---

## Supported Models

Tested and working:

- **Anthropic Claude** 3.5 Sonnet / Opus (recommended, best comprehension)
- **OpenAI GPT-4o** (fast, cost-effective)
- **Google Gemini Pro** (via OpenRouter—ironic, right?)
- **Meta Llama 3.3** 70B (open-source option)

In theory, any vision-capable model OpenRouter supports will work. As long as the model can see images and output structured actions, it's good.

---

**支持的模型**

测过能用的：

- **Anthropic Claude** 3.5 Sonnet / Opus（推荐，理解力最强）
- **OpenAI GPT-4o**（快，便宜）
- **Google Gemini Pro**（通过 OpenRouter——讽刺吧）
- **Meta Llama 3.3** 70B（开源选项）

理论上 OpenRouter 支持的所有视觉模型都能用。只要模型能看图、能输出结构化操作指令，就行。

---

Built this because we were curious: what happens when you open up the closed parts?

Turns out: **Google already solved the core problem of mobile AI automation. They just locked the solution away.**

Now it's open. Let's see what the world does with it.

---

造这个是因为好奇：把封闭的东西打开，会发生什么？

结果发现：**Google 早就解决了移动 AI 自动化的核心问题。只是他们把答案锁起来了。**

现在锁开了。看世界会拿它干什么吧。
