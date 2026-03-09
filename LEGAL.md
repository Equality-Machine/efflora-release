# Legal and Ethics

## Purpose

This project is educational research. We aim to:

1. Understand how Google's VirtualDisplay-based automation works
2. Demonstrate that the architecture can be opened to other models
3. Explore what's possible when mobile AI agents don't conflict with user interaction
4. Contribute to the discussion about open vs. closed AI infrastructure

## What We Did

### Reverse Engineering

We analyzed publicly available Android 17 system components through:

- Static analysis of system binaries (services.jar, framework.jar)
- Runtime inspection via Frida
- Network protocol analysis
- AIDL interface reconstruction from binary artifacts

This is **reverse engineering for interoperability**, protected under:
- US Copyright Act Section 1201(f) - Reverse engineering for interoperability
- EU Software Directive Article 6 - Decompilation for interoperability
- Similar provisions in other jurisdictions

### System Modification

We created patches that modify Android system components to:

- Remove package whitelist restrictions
- Bypass proprietary protocol requirements
- Enable use of alternative AI model backends

These modifications are applied to **your own device** that you have root access to. This is similar to custom ROMs, root apps, and other modifications that have existed in the Android ecosystem since its beginning.

## What We Didn't Do

### No Proprietary Code Distribution

- We do **not** distribute Google's compiled binaries
- We do **not** distribute AOSP source code (it's freely available)
- We **only** distribute our patch files that modify open-source AOSP code

### No Service Abuse

- We do **not** provide tools to abuse Google's services
- We **do not** crack or bypass Google account authentication
- Our system works with **your own API keys** and **your own AI model backends**

### No Circumvention of Access Controls

The security gates we bypass are not designed to:
- Protect copyrighted content
- Prevent piracy
- Secure payment systems
- Protect user data from unauthorized access

They are **artificial restrictions** that limit which apps can use ComputerControl API and which AI backends can be used. These restrictions serve business interests, not security.

## IP and Copyright

### Google's Rights

Google owns:
- Android trademark
- Original AOSP code (Apache 2.0 licensed)
- Proprietary additions in Pixel system images
- Gemini and Robin protocol implementation

We respect these rights. We:
- Don't use Google trademarks except for factual reference
- Don't distribute their proprietary code
- Don't claim their technology as ours

### Our Rights

We own:
- Our patch files
- Our openGemi agent implementation
- Our documentation and research findings
- Our OpenRouter integration code

These are released under MIT License (see LICENSE file).

### Fair Use

Our documentation includes:
- Screenshots of Android UI for educational purposes
- Architecture diagrams based on our reverse engineering
- Code snippets showing how the system works

This is **transformative fair use** for educational and commentary purposes.

## Responsible Use

### What's Acceptable

- Using on your own device with your own API keys
- Research and education
- Building agents for your personal productivity
- Contributing improvements to openGemi

### What's Not Acceptable

- Using to abuse or attack services
- Violating terms of service of apps you automate
- Automating actions that would be illegal if done manually
- Distributing as a way to bypass app protections
- Commercial use without understanding legal implications

### App Terms of Service

When you use openGemi to automate apps:

- You are responsible for complying with each app's terms of service
- Some apps prohibit automation - respect that
- Some apps have rate limits - respect that
- Don't use openGemi to spam, scrape, or abuse services

This is the same as if you were using the app manually. Automation doesn't give you permission to violate ToS.

## Disclaimer

### No Warranty

This software is provided "AS IS" without warranty of any kind. See LICENSE file.

### Device Risk

System modifications can:
- Brick your device
- Void your warranty
- Expose security vulnerabilities
- Cause data loss

**Only use on devices you can afford to lose or restore.**

### Account Risk

Using openGemi may:
- Violate app terms of service
- Result in account bans
- Trigger security alerts

**Only automate apps where you accept these risks.**

### Legal Risk

Laws vary by jurisdiction. Consult a lawyer if you:
- Plan commercial use
- Plan to distribute modified versions
- Are subject to specific regulations
- Have questions about your jurisdiction

We are not lawyers. This is not legal advice.

## Transparency

### Why We're Doing This

We believe AI infrastructure should be open. Google built an amazing architecture for mobile AI agents, then locked it to Gemini only. This creates:

- **Vendor lock-in**: Can't use Claude, GPT-4, or local models
- **Privacy concerns**: All automation goes through Google servers
- **Innovation barrier**: Can't build specialized agents

By opening the architecture, we enable:

- **Model choice**: Use any vision-capable AI model
- **Privacy options**: Self-hosted or local models
- **Innovation**: Build agents for specific workflows

### What We Hope Happens

**Best case**: Google sees value in opening the API officially
- Announces public ComputerControl API in Android 18
- Allows third-party AI backends with proper security model
- Supports the ecosystem of AI agent developers

**Realistic case**: Community builds on our research
- Custom ROMs include openGemi patches
- Developers create specialized agents
- Standards emerge for mobile AI agent interfaces

**Worst case**: Nothing changes
- At least we documented how it works
- Researchers understand the architecture
- Maybe it influences future decisions

## Contact for Legal Concerns

For legitimate legal or security concerns:

1. Open a GitHub issue (preferred for transparency)
2. Email: [to be added]

We will respond promptly to:
- Valid DMCA notices (we'll explain why we believe fair use applies)
- Security vulnerabilities (responsible disclosure)
- Trademark concerns (we're happy to clarify fair use)

We will ignore:
- Threats
- Demands without legal basis
- Fishing expeditions for user data (we don't collect any)

---

Built with respect for intellectual property, user rights, and the tradition of tinkering that made computing what it is today.
