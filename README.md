# openGemi

## Google 在 Android 17 里悄悄造了什么

你大概从来没听说过这个功能。

Google 在移动操作系统里埋下了一个野心极大的 AI 功能，藏在 Android 17 深处。有一套完整的系统，能让 AI 代理替你操作手机——启动应用、点击按钮、读取屏幕、完成任务——而且**全程不会抢占你正在看的那块屏幕**。

Google 对外称之为 **Screen Automation**，内部代号 **Bonobo**。

但你大概从来没听说过它。因为 Google 给它加了六层安全门，并且只允许自家的 AI（Gemini）在自家的手机（Pixel 10 及以后机型）上调用。这不是什么 Beta 功能，也不在设置菜单里。它深埋在系统底层，等着被激活。

**这个项目就是要把它打开。**

---

## 核心思路：一块只有 AI 能看到的"第二屏幕"

假设你正常地用着手机——刷消息、看视频、随便逛逛。现在想象手机内部还有一块你看不见的屏幕，AI 助手正在那块隐形屏幕上打开应用、填表单、查价格、等它忙完了，再把结果交给你。

**Google 做的就是这件事。**

在技术层面叫 **VirtualDisplay**，但概念很简单：一块完全由软件实现的手机屏幕。应用可以在上面运行，按钮可以被点击，截图可以被获取——但它完全不会出现在你面前的物理屏幕上。它是 AI 的有专属工作区。

这一设计解决了手机自动化最大的难题：**谁来控制屏幕？**之前所有让 AI 操作手机的方案都建立在同一个矛盾——AI 需要看到屏幕才能判断下一步，也需要控制屏幕才能执行操作；但如果你在用这块屏幕，一切就乱了。AI 点了个按钮，你被强制跳走；你滑了一下页面，AI 就懵了。

**Google 的解法简洁可怕：直接给 AI 一块专属屏幕，你保留你的。互不干扰。**

---

## 为什么你从来没见过这个功能

既然这套系统如此精密，为什么没人在用？因为 Google 给它套了六层安全门，关死以后只给自家用了。八层吻别，实际上比任何 Google 体系之外的人都无法触及：

1. **Feature Flag 默认关闭** - 整套系统统一个配置开关控制，而这个开关被硬编码为 `false`。就算其他条件全部满足，这一个开关就能让它保持休眠。

2. **权限校验** - 系统需要特定的权限，而这些权限对普通应用不开放——只有系统级组件才能申请。

3. **设备白名单** - 即便拿到了正确的权限，系统还会检查你的手机是否在批准名单上。目前这个名单只包含 Pixel 10 及更新的设备。

4. **证书校验** - 试图使用该系统的应用必须由特定的加密证书签名。实际上，这意味着它必须是 Google 自己的软件。

5. **用户同意流程** - 在 AI 启动会话之前，用户必须明确同意。流程中置入了授权页面和确认弹窗。

6. **运行时拦截** - 就算你想尽办法通过了以上所有关卡，仍然有额外的运行时检查会在系统不满足时直接终止流程。

这些都不是 Bug，也不是疏忽。它们是有意为之的分层安全措施。Google 显然希望这套系统存在，但前提是它完全在自己的掌控之下——自己的 AI、自己的手机、自己的规则。

---

## 但问题来了：这一切只能配合 Gemini 在 Pixel 上使用

一套强大的、工程质量过硬的、平台级的 AI 自动化系统——被锁死在一个 AI 供应商的一个手机品牌上。

虚拟屏幕不关心是哪个 AI 在分析截图、控制会话不关心是哪个模型在决定下一步点哪里。基础设施本身是通用的，限制是人为的。

**这就是 openGemi 要做的事：**

把这套系统打开。移除人为限制。让任何人都能用自己选择的 AI 模型——Claude、GPT-4、本地运行的 Llama，whatever——来控制自己的 Android 手机，使用 Google 已经造好但藏起来的这套基础设施。

---

## 我们做了什么

### 1. 逆向工程 Google 的系统

通过静态分析（反编译 `services.jar`）、运行时检查（Frida hook）、网络协议分析（抓包 Robin gRPC），我们摸清了整套架构：

- **ComputerControl Manager Service** - 系统核心，管理虚拟屏幕生命周期
- **六层安全门的实现细节** - 每一道门在哪检查、如何绕过
- **Robin Protocol** - Google 专有的 gRPC 协议，连接 Gemini Pro
- **AIDL 接口** - 应用层如何向系统请求 Session

所有这些都没开源，没文档。是我们一点点抠出来的。

### 2. 打补丁：移除限制

我们修改了系统组件，绕过安全门：

```java
// 原始代码
private static final Set<String> ALLOWED_PACKAGES = ImmutableSet.of(
    "com.google.android.apps.bard",
    "com.google.android.googlequicksearchbox"
);

// 补丁后
private static final Set<String> ALLOWED_PACKAGES = ImmutableSet.of(
    "*"  // 允许任何应用
);
```

类似的修改应用到 Feature Flag、权限检查、证书校验等所有六层门。结果：任何应用都能创建 ComputerControl Session。

### 3. 替换 AI 后端：从 Gemini 到 OpenRouter

Google 的 Bonobo agent 硬编码连接到 Robin 协议 → Gemini Pro。我们换成：

```
openGemi Agent → OpenRouter Gateway → 你选择的任何模型
```

OpenRouter 是一个 AI 模型聚合器，支持：
- Anthropic Claude（Sonnet、Opus）
- OpenAI GPT-4o/o1
- Google Gemini Pro（讽刺的是，你可以通过 OpenRouter 用它）
- Meta Llama 3.3
- 以及几十个其他开源/商业模型

你只需要提供 API key，选择模型，其余的交给 openGemi。

### 4. 开源整个实现

- 系统补丁（diff 文件，应用到 AOSP）
- Agent 实现（Kotlin，开源 APK）
- 技术文档（[docs/](docs/)）
- 安装指南（[INSTALL.md](INSTALL.md)）

---

## 这意味着什么

**你的手机变成两个手机。**

一个给你用，刷微博、看视频、回消息。
一个给 AI 用，订餐厅、比价格、填表单。

同时运行。零干扰。

这不是科幻，这是 Google 已经造出来的技术。只是他们把它锁在 Gemini + Pixel 的围墙里。

**openGemi 把墙拆了。**

---

## 当前状态

**这是研究原型，不是产品。**

需要：
- Rooted Android 17 设备或自定义 ROM
- 修改系统文件（不是简单装个 APK）
- 懂 Android 底层（知道 ADB、fastboot、刷机是什么）

如果你不确定这些是什么意思，现在还不是你用的时候。等后续版本可能会有更简单的安装方式（虽然不保证）。

但如果你是开发者、研究者、或者就是好奇心重的 geek——这是目前唯一能让你在非 Pixel 设备上、用非 Gemini 模型、体验 Google 级别移动 AI 自动化的方式。

---

## 快速开始

1. **阅读文档**，了解 Google 造了什么、为什么重要、怎么实现的：
   - [Google 造了什么](docs/01-what-google-built.md)
   - [为什么这个很重要](docs/02-why-it-matters.md)
   - [openGemi 是怎么做的](docs/03-how-opengemi-works.md)

2. **安装系统** - 两条路：
   - [完整构建](INSTALL.md#method-1-system-build)：编译打补丁的 Android 系统镜像（干净但慢，2-8 小时）
   - [运行时补丁](INSTALL.md#method-2-runtime-patching)：用 Frida 在运行时绕过检查（快但每次重启失效）

3. **配置 OpenRouter**，选择你想用的 AI 模型

4. **运行第一个 Agent** - 试试简单的："打开设置"，"查天气"，"订明天的餐厅"

5. **反馈问题** - [GitHub Issues](https://github.com/Equality-Machine/efflora-release/issues)

---

## 支持的模型

实测可用：

- **Anthropic Claude** 3.7 Sonnet / Opus（推荐，理解力最强）
- **OpenAI GPT-4o**（快，便宜）
- **Google Gemini Pro**（via OpenRouter，讽刺吧）
- **Meta Llama 3.3** 70B（开源选项）

理论上 OpenRouter 支持的所有视觉模型都能用。只要模型能看图、输出结构化 action，就行。

---

## 法律与道德

这是**教育研究项目**。我们：

- 逆向的是公开可获取的 Android 系统组件（interoperability，受法律保护）
- 没分发 Google 的专有代码（只分发我们的补丁）
- 没破解加密或绕过内容保护（只是移除人为的 API 限制）
- 明确这是 for 自己的设备，自担风险

详见 [LEGAL.md](LEGAL.md)。

如果你是 Google 的律师，来聊聊吧。我们相信这是合法的，也愿意讨论。

---

## 许可

MIT License - 见 [LICENSE](LICENSE)

---

## 联系

- **Issues**：[GitHub Issues](https://github.com/Equality-Machine/efflora-release/issues)
- **讨论**：[GitHub Discussions](https://github.com/Equality-Machine/efflora-release/discussions)
- **安全问题**：开 issue，标注 security

---

造这个项目是因为好奇：当你把封闭的东西打开，会发生什么。

结果是：**Google 已经解决了移动 AI 自动化的核心难题。只是他们把解决方案锁起来了。**

现在它开放了。看看世界会怎么用它。
