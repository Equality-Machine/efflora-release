# openGemi

Open-source implementation of mobile AI agents using Android's VirtualDisplay architecture.

## The Problem

Every mobile automation tool today has to take over your screen. ADB, Accessibility Service, cloud emulators - they all have the same flaw: when the AI agent is running, you can't use your phone.

Google solved this with Android 17's Screen Automation (internally called "Bonobo"). They created a parallel virtual display where AI agents work independently, never touching your real screen.

This project reverse-engineers that system and replaces Google's proprietary Gemini backend with open model support.

## What Google Built

Android 17 introduces VirtualDisplay-based automation infrastructure:

- **ComputerControl Session**: Low-level system service for creating isolated virtual displays
- **Security Gates**: Six layers including feature flags, permissions, app whitelist, SSL certificates, runtime consent, and user authorization
- **Robin Protocol**: Proprietary gRPC connection to Gemini Pro
- **Bonobo Agent**: Google's internal codename for their AI automation agent

Technical details in [docs/01-what-google-built.md](docs/01-what-google-built.md)

## What openGemi Built

We reverse-engineered the ComputerControl system and replaced the Gemini-locked backend with a model-agnostic gateway:

- **System Patches**: Modified services.jar and CC Extensions stub library to bypass security gates
- **OpenRouter Gateway**: Replaces Robin protocol with standard API supporting Claude, GPT-4o, Llama, etc.
- **Virtual Display Agent**: Runs in isolated environment with ImageReader screenshot capture
- **AIDL Interface**: Reverse-engineered Binder IPC for ComputerControl Session

The agent captures screenshots from the virtual display, sends them to your chosen model, and executes actions without touching your main screen.

Technical implementation in [docs/03-how-opengemi-works.md](docs/03-how-opengemi-works.md)

## Why This Matters

Traditional mobile automation approaches:

| Approach | Screen Conflict | Device Requirements | Limitations |
|----------|----------------|---------------------|-------------|
| ADB | Blocks real screen | Desktop computer | Requires USB/WiFi debug |
| Accessibility Service | Blocks real screen | None | Limited to UI interactions |
| Cloud Emulator | No conflict | High-end server | Expensive, complex setup |
| openGemi | No conflict | Android 17+ device | Requires system modification |

VirtualDisplay solves the fundamental problem: your phone becomes a parallel universe where AI agents work independently.

See [docs/02-why-it-matters.md](docs/02-why-it-matters.md) for detailed comparison.

## Current Status

**Research prototype.** This demonstrates that Google's architecture can be opened up, but it requires:

- Rooted Android 17 device or custom ROM
- System-level modifications (not a simple APK install)
- Technical knowledge of Android internals

We're exploring cleaner integration paths that might work without root on future Android versions.

## Installation

See [INSTALL.md](INSTALL.md) for detailed setup instructions.

**Warning:** This requires modifying system files. Only proceed if you understand the risks.

## Model Support

Currently tested:

- Anthropic Claude 3.7 Sonnet
- OpenAI GPT-4o
- Google Gemini Pro (via OpenRouter, ironically)
- Meta Llama 3.3 70B

Add your OpenRouter API key to use any supported model.

## Legal

This is educational research. We reverse-engineered publicly available Android system components for compatibility purposes. See [LEGAL.md](LEGAL.md).

## License

MIT License - see [LICENSE](LICENSE)

## Contact

For serious collaboration or security concerns: [Open an issue]

---

Built with curiosity about what's possible when you open the closed parts.
