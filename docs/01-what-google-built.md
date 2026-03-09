# What Google Built: Android 17 Screen Automation

## Overview

Android 17 introduced a new system-level automation infrastructure internally codenamed "Bonobo". Unlike previous automation methods, it's built on VirtualDisplay architecture, allowing AI agents to control a phone without taking over the user's screen.

## Core Architecture

### VirtualDisplay Foundation

Google's implementation uses Android's VirtualDisplay API in a way that was never publicly documented:

- Creates an isolated virtual display (typically 1080x2400 or device native resolution)
- Runs in a separate window manager context
- Has its own input event stream
- Captures screenshots via ImageReader without affecting the real display

The key innovation: the virtual display is completely independent of the physical screen. The user can browse, play games, or watch videos while the AI agent works in its own sandbox.

## ComputerControl Session System

The heart of the implementation is a new system service called ComputerControl (CC). This isn't documented in public AOSP or Android SDK.

### Service Components

**CC Manager Service** (`services.jar`)
- Singleton system service running in system_server process
- Manages virtual display lifecycle
- Handles permission and security gates
- Coordinates with WindowManager and ActivityManager

**CC Extensions Library** (stub library)
- AIDL interface for apps to request CC sessions
- Provides SecurityGate verification methods
- Handles SSL certificate pinning for Robin protocol

**CC Agent** (Bonobo)
- Google's AI automation agent
- Runs inside the virtual display environment
- Communicates with Gemini Pro via Robin protocol

### Security Gates

Google implemented six security layers to restrict access:

1. **Feature Flag**: `com.android.flags.ENABLE_CC_SESSIONS`
   - System property check, enabled only in specific Android builds
   - Pixel 9 and certain developer preview builds

2. **Permission**: `android.permission.CREATE_CC_SESSION`
   - System-level permission (protectionLevel: signature|privileged)
   - Only platform-signed or pre-installed system apps

3. **App Whitelist**: Hardcoded package names
   - `com.google.android.apps.bard` (Google Gemini app)
   - `com.google.android.googlequicksearchbox` (Google Search)
   - Checked against calling UID in CCManagerService

4. **SSL Certificate Pinning**: Robin protocol endpoints
   - Enforces Google's SSL certificates
   - Prevents man-in-the-middle or custom backends

5. **Runtime Consent**: User authorization dialog
   - "Allow Gemini to control your device in the background?"
   - One-time approval stored in system settings

6. **User Authorization**: Gemini account linkage
   - Requires active Google account
   - Validates OAuth token with Google servers

## Robin Protocol

Google created a proprietary gRPC protocol called "Robin" for Gemini communication:

### Protocol Structure

```
Client (Android) → SSL/TLS → Robin gRPC Endpoint → Gemini Pro
```

**Endpoints** (discovered via Frida runtime analysis):
- `robin.googleapis.com:443`
- Primary endpoint for screenshot upload and action commands

**Message Format**:
- Protobuf-serialized messages
- Screenshot encoded as WebP (quality 80-90)
- Screen metadata (resolution, density, orientation)
- Accessibility tree snapshot (optional, for context)
- Action responses (tap coordinates, swipe vectors, text input)

**Authentication**:
- OAuth 2.0 bearer token from user's Google account
- Refresh token cycle every 30 minutes
- Token scoped to `https://www.googleapis.com/auth/gemini.mobile.control`

## Technical Details Discovered

### AIDL Interface (Reverse-Engineered)

```java
interface IComputerControlManager {
    // Create a new CC session
    CCSession createSession(in CCSessionRequest request);

    // Check if calling app has permission
    boolean hasPermission(String packageName);

    // Verify all security gates
    SecurityGateResult verifySecurityGates(in SecurityContext context);

    // Destroy session
    void destroySession(int sessionId);
}
```

### Virtual Display Parameters

Discovered via runtime inspection:

- **Display Size**: Matches physical device (e.g., 1080x2400)
- **Density**: Same as physical display (e.g., 420dpi)
- **Flags**: `VIRTUAL_DISPLAY_FLAG_OWN_CONTENT_ONLY | VIRTUAL_DISPLAY_FLAG_SECURE`
- **Surface**: Backed by ImageReader with RGBA_8888 format

### Screenshot Capture Method

Google uses ImageReader instead of traditional screenshot methods:

```java
ImageReader reader = ImageReader.newInstance(
    width, height,
    PixelFormat.RGBA_8888,
    2  // max images
);
virtualDisplay.setSurface(reader.getSurface());
Image image = reader.acquireLatestImage();
// Convert to WebP and send to Robin
```

This approach:
- Zero overhead on physical display
- No SurfaceFlinger involvement
- Bypasses screenshot security restrictions
- Frame rate limited only by VirtualDisplay refresh rate

## Why This Architecture Matters

Traditional automation methods (ADB, Accessibility Service) have fundamental limitations:

- **Screen Conflict**: Can't use phone while automation runs
- **Permissions**: Require dangerous permissions that expose user data
- **Performance**: Screenshot capture affects real UI rendering
- **Detection**: Apps can detect automation and block functionality

VirtualDisplay solves all of these:
- Parallel execution: user and agent work independently
- Isolated environment: agent can't access real user data
- Zero UI overhead: physical screen rendering unaffected
- Undetectable: apps running in virtual display can't tell they're automated

## What We Learned From Reverse Engineering

Key findings from our analysis:

1. **services.jar modifications**: Google added ~15KB of new code to CCManagerService
2. **SecurityGate.verify()**: The critical bottleneck - all six gates checked serially
3. **Robin protocol**: Standard gRPC with protobuf, not a custom protocol
4. **Whitelist storage**: Lives in `/data/system/cc_whitelist.xml`
5. **Feature flag location**: System property `persist.cc.enabled`

These discoveries made it possible to patch the system and open it up.

## Comparison to Public APIs

Google didn't create new Android APIs - they repurposed existing ones:

| Component | Public API | Google's Usage |
|-----------|-----------|----------------|
| Virtual Display | `DisplayManager.createVirtualDisplay()` | Same API, restricted by security gates |
| Screenshot | `ImageReader` | Same API, connected to virtual display |
| Input Events | `InputManager.injectInputEvent()` | Same API, injected into virtual display |
| Window Management | `WindowManager` | Standard API, manages virtual windows |

The innovation isn't new APIs - it's the security gate architecture that restricts who can use existing APIs.

## Source Code Availability

Google hasn't open-sourced this:
- Not in AOSP master branch
- Not in Android 17 release tags
- Pixel system images contain compiled code only

Our reverse engineering used:
- Frida for runtime hooking
- jadx for DEX decompilation
- Wireshark for Robin protocol analysis
- Custom AOSP builds for testing patches

---

**Next**: [Why It Matters](02-why-it-matters.md) - Why VirtualDisplay is a paradigm shift for mobile automation.
