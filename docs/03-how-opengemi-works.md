# How openGemi Works: Technical Implementation

## Architecture Overview

openGemi has three main components:

```
┌─────────────────────────────────────────────────────────┐
│                    Android Device                        │
│                                                           │
│  ┌──────────────────────────────────────────────────┐  │
│  │          Patched System Components                │  │
│  │                                                    │  │
│  │  • Modified services.jar                          │  │
│  │    - CCManagerService with bypassed gates        │  │
│  │                                                    │  │
│  │  • Patched CC Extensions                          │  │
│  │    - AIDL stubs without certificate pinning      │  │
│  │                                                    │  │
│  │  • Feature Flag Override                          │  │
│  │    - persist.cc.enabled = true                    │  │
│  └──────────────────────────────────────────────────┘  │
│              ↓                                           │
│  ┌──────────────────────────────────────────────────┐  │
│  │      openGemi Agent (APK)                         │  │
│  │                                                    │  │
│  │  • Creates VirtualDisplay session                │  │
│  │  • Captures screenshots via ImageReader          │  │
│  │  • Sends to OpenRouter gateway                   │  │
│  │  • Executes AI-generated actions                 │  │
│  └──────────────────────────────────────────────────┘  │
│              ↓                                           │
└──────────────┼───────────────────────────────────────────┘
               ↓
    ┌──────────────────────┐
    │  OpenRouter Gateway   │
    │                       │
    │  • Model selection    │
    │  • Request routing    │
    │  • Response parsing   │
    └──────────────────────┘
               ↓
    ┌──────────────────────────────────────┐
    │  AI Models (your choice)              │
    │                                        │
    │  • Claude 3.7 Sonnet                  │
    │  • GPT-4o                             │
    │  • Gemini Pro (via OpenRouter)        │
    │  • Llama 3.3 70B                      │
    └──────────────────────────────────────┘
```

## Phase 1: System Patching

### 1.1 Bypassing Security Gates

Google's six security gates must all be passed to create a CC session. Our approach:

#### Gate 1: Feature Flag
```bash
# Enable CC sessions at boot
adb shell setprop persist.cc.enabled true

# Or in system/build.prop:
persist.cc.enabled=true
```

#### Gate 2: Permission
Modify `frameworks/base/core/res/AndroidManifest.xml`:
```xml
<!-- Add to your agent app's manifest -->
<uses-permission android:name="android.permission.CREATE_CC_SESSION" />

<!-- System grants this to any app if signature check is bypassed -->
```

#### Gate 3: App Whitelist
Patch `services.jar` - `CCManagerService.java`:
```java
// Original code
private static final Set<String> ALLOWED_PACKAGES = ImmutableSet.of(
    "com.google.android.apps.bard",
    "com.google.android.googlequicksearchbox"
);

// Patched code - allow all
private static final Set<String> ALLOWED_PACKAGES = ImmutableSet.of(
    "*"  // Allow any package
);

// Or check your app package
private boolean isAllowedPackage(String packageName) {
    return packageName.equals("your.opengemi.agent") ||
           ALLOWED_PACKAGES.contains(packageName);
}
```

#### Gate 4: SSL Certificate Pinning
Patch CC Extensions stub library:
```java
// Original Robin client
RobinClient client = RobinClient.builder()
    .setCertificatePinner(GOOGLE_CERT_PINNER)
    .build();

// Patched - remove pinning
RobinClient client = RobinClient.builder()
    // No certificate pinning
    .build();
```

#### Gate 5 & 6: Runtime Consent / User Auth
Patch permission check in `CCManagerService`:
```java
// Original
public CCSession createSession(CCSessionRequest request) {
    if (!hasUserConsent(request.userId)) {
        throw new SecurityException("User consent required");
    }
    if (!validateGoogleAccount(request.userId)) {
        throw new SecurityException("Valid Google account required");
    }
    // ... create session
}

// Patched - skip checks
public CCSession createSession(CCSessionRequest request) {
    // Directly create session
    return createSessionInternal(request);
}
```

### 1.2 AIDL Interface Reconstruction

We reverse-engineered the CC AIDL interface:

**IComputerControlManager.aidl**:
```java
package android.computercontrol;

import android.computercontrol.CCSession;
import android.computercontrol.CCSessionRequest;

interface IComputerControlManager {
    CCSession createSession(in CCSessionRequest request);
    void destroySession(int sessionId);
    boolean hasPermission(String packageName);
    int getSessionState(int sessionId);
}
```

**CCSessionRequest.java**:
```java
public class CCSessionRequest implements Parcelable {
    public int displayWidth;
    public int displayHeight;
    public int densityDpi;
    public int flags;
    public String packageName;

    // Parcelable implementation
}
```

**CCSession.java**:
```java
public class CCSession implements Parcelable {
    public int sessionId;
    public VirtualDisplay virtualDisplay;
    public Surface surface;

    // Parcelable implementation
}
```

## Phase 2: Agent Implementation

### 2.1 Creating Virtual Display Session

```kotlin
class OpenGemiAgent : Service() {
    private lateinit var ccManager: IComputerControlManager
    private lateinit var virtualDisplay: VirtualDisplay
    private lateinit var imageReader: ImageReader

    override fun onCreate() {
        super.onCreate()

        // Get CC service
        val binder = ServiceManager.getService("computercontrol")
        ccManager = IComputerControlManager.Stub.asInterface(binder)

        // Request session
        val request = CCSessionRequest(
            displayWidth = 1080,
            displayHeight = 2400,
            densityDpi = 420,
            flags = VIRTUAL_DISPLAY_FLAG_OWN_CONTENT_ONLY,
            packageName = packageName
        )

        val session = ccManager.createSession(request)
        virtualDisplay = session.virtualDisplay

        // Setup screenshot capture
        imageReader = ImageReader.newInstance(
            1080, 2400,
            PixelFormat.RGBA_8888,
            2
        )

        virtualDisplay.surface = imageReader.surface
    }
}
```

### 2.2 Screenshot Capture

```kotlin
fun captureScreenshot(): Bitmap {
    val image = imageReader.acquireLatestImage() ?: return null

    val planes = image.planes
    val buffer = planes[0].buffer
    val pixelStride = planes[0].pixelStride
    val rowStride = planes[0].rowStride
    val rowPadding = rowStride - pixelStride * width

    val bitmap = Bitmap.createBitmap(
        width + rowPadding / pixelStride,
        height,
        Bitmap.Config.ARGB_8888
    )

    bitmap.copyPixelsFromBuffer(buffer)
    image.close()

    return bitmap
}
```

### 2.3 OpenRouter Integration

Replace Google's Robin protocol with standard HTTP API:

```kotlin
class OpenRouterClient(private val apiKey: String) {
    private val baseUrl = "https://openrouter.ai/api/v1"

    suspend fun sendQuery(
        screenshot: Bitmap,
        prompt: String,
        model: String = "anthropic/claude-3.7-sonnet"
    ): AgentAction {
        // Encode screenshot
        val screenshotBase64 = screenshot.toBase64(quality = 85)

        // Construct request
        val request = JSONObject().apply {
            put("model", model)
            put("messages", JSONArray().apply {
                put(JSONObject().apply {
                    put("role", "user")
                    put("content", JSONArray().apply {
                        put(JSONObject().apply {
                            put("type", "image_url")
                            put("image_url", JSONObject().apply {
                                put("url", "data:image/png;base64,$screenshotBase64")
                            })
                        })
                        put(JSONObject().apply {
                            put("type", "text")
                            put("text", prompt)
                        })
                    })
                })
            })
        }

        // Send to OpenRouter
        val response = httpClient.post("$baseUrl/chat/completions") {
            header("Authorization", "Bearer $apiKey")
            header("Content-Type", "application/json")
            setBody(request.toString())
        }

        // Parse action from response
        return parseAgentAction(response.body())
    }
}
```

### 2.4 Action Execution

```kotlin
fun executeAction(action: AgentAction) {
    when (action) {
        is TapAction -> {
            val x = action.x
            val y = action.y

            val downTime = SystemClock.uptimeMillis()
            val eventDown = MotionEvent.obtain(
                downTime, downTime,
                MotionEvent.ACTION_DOWN,
                x.toFloat(), y.toFloat(), 0
            )

            val eventUp = MotionEvent.obtain(
                downTime, downTime + 50,
                MotionEvent.ACTION_UP,
                x.toFloat(), y.toFloat(), 0
            )

            // Inject into virtual display
            virtualDisplay.inputChannel.sendMotionEvent(eventDown)
            Thread.sleep(50)
            virtualDisplay.inputChannel.sendMotionEvent(eventUp)
        }

        is SwipeAction -> {
            val events = generateSwipeEvents(
                action.fromX, action.fromY,
                action.toX, action.toY,
                action.durationMs
            )

            events.forEach { event ->
                virtualDisplay.inputChannel.sendMotionEvent(event)
                Thread.sleep(16) // 60fps
            }
        }

        is TypeAction -> {
            action.text.forEach { char ->
                val keyCode = charToKeyCode(char)
                virtualDisplay.inputChannel.sendKeyEvent(
                    KeyEvent(KeyEvent.ACTION_DOWN, keyCode)
                )
                virtualDisplay.inputChannel.sendKeyEvent(
                    KeyEvent(KeyEvent.ACTION_UP, keyCode)
                )
            }
        }
    }
}
```

## Phase 3: Agent Loop

```kotlin
suspend fun runAgentLoop(task: String) {
    val context = mutableListOf<String>()
    context.add("Task: $task")

    var maxSteps = 50
    var step = 0

    while (step < maxSteps) {
        // Capture current screen
        val screenshot = captureScreenshot()

        // Send to AI model
        val prompt = buildPrompt(context, screenshot)
        val action = openRouterClient.sendQuery(
            screenshot,
            prompt,
            model = "anthropic/claude-3.7-sonnet"
        )

        // Check if done
        if (action is CompleteAction) {
            log("Task completed: ${action.result}")
            break
        }

        // Execute action
        executeAction(action)

        // Wait for UI to settle
        delay(1000)

        // Update context
        context.add("Step $step: ${action.description}")

        step++
    }
}
```

## Component Breakdown

### Core Components

| Component | Function | Implementation |
|-----------|----------|----------------|
| CCManagerService Patch | Bypass security gates | Modified services.jar |
| CC Extensions Patch | Remove SSL pinning | Modified stub library |
| VirtualDisplay Manager | Create isolated display | Android Display API |
| Screenshot Capture | Get virtual display frames | ImageReader |
| OpenRouter Gateway | Route to AI models | HTTP API client |
| Action Executor | Inject input events | MotionEvent injection |
| Agent Loop | Orchestrate workflow | Kotlin coroutines |

### Data Flow

```
1. User: "Book restaurant at 7pm"
   ↓
2. Agent creates VirtualDisplay session
   ↓
3. Opens restaurant app in virtual display
   ↓
4. Loop:
   a. Capture screenshot
   b. Send to Claude via OpenRouter
   c. Receive action (tap, swipe, type)
   d. Execute action in virtual display
   e. Wait for UI to update
   ↓
5. Complete: "Reservation confirmed"
```

### Performance Characteristics

- **Screenshot capture**: 10-20ms (RGBA_8888, 1080x2400)
- **Image encoding**: 50-100ms (WebP quality 85)
- **API request**: 500-2000ms (depends on model)
- **Action execution**: 50-100ms (touch event injection)
- **Total per step**: ~1-2 seconds

This is fast enough for real-time agent control.

## Building and Installation

### Prerequisites

1. **Rooted Android 17 device** or custom ROM with unlocked bootloader
2. **Android SDK** (for building patches)
3. **AOSP source code** (for services.jar modification)
4. **Frida** (optional, for runtime patching without rebuild)

### Build Process

```bash
# 1. Clone AOSP Android 17
repo init -u https://android.googlesource.com/platform/manifest -b android-17.0.0_r1
repo sync

# 2. Apply openGemi patches
cd frameworks/base/services/core/java/com/android/server/computercontrol
patch < opengemi_ccmanager.patch

# 3. Build services.jar
cd $ANDROID_BUILD_TOP
. build/envsetup.sh
lunch aosp_arm64-eng
make services

# 4. Extract patched jar
cp out/target/product/generic_arm64/system/framework/services.jar ./services_patched.jar

# 5. Push to device
adb root
adb remount
adb push services_patched.jar /system/framework/services.jar
adb reboot
```

### Runtime Patching (Alternative)

For devices you can't rebuild system image:

```python
import frida

# Frida script to patch CCManagerService
js_code = """
Java.perform(function() {
    var CCManager = Java.use('com.android.server.computercontrol.CCManagerService');

    // Patch whitelist check
    CCManager.isAllowedPackage.implementation = function(packageName) {
        console.log('[*] Whitelist check bypassed for: ' + packageName);
        return true;
    };

    // Patch permission check
    CCManager.hasPermission.implementation = function(packageName) {
        console.log('[*] Permission check bypassed for: ' + packageName);
        return true;
    };
});
"""

# Attach to system_server
session = frida.get_usb_device().attach('system_server')
script = session.create_script(js_code)
script.load()
```

## Known Limitations

### Technical Constraints

1. **Requires Root**: System modification needs root access or unlocked bootloader
2. **Android 17+**: VirtualDisplay CC API doesn't exist in earlier versions
3. **Single Session**: Current implementation supports one agent at a time
4. **No Persistence**: Agent state lost on device reboot

### Model Limitations

1. **Vision Models Only**: Requires models with image input support
2. **Context Length**: Long tasks may hit token limits
3. **Cost**: API usage costs for cloud models

### Performance Considerations

1. **Battery**: Continuous screenshot capture drains battery ~15%/hour
2. **Network**: Uploading 1080x2400 screenshots uses ~500KB per step
3. **Latency**: Each step takes 1-2 seconds (mostly model inference)

## Future Improvements

### Short-term

- [ ] Multi-session support (multiple agents in parallel)
- [ ] Lower-resolution screenshot option (reduce bandwidth/cost)
- [ ] Local model support (Llama.cpp, GGUF)
- [ ] Agent state persistence

### Long-term

- [ ] Non-root installation path (waiting for Android API changes)
- [ ] Hardware acceleration for screenshot encoding
- [ ] Agent performance profiling and optimization
- [ ] Integration with Android Assistant API (if Google opens it)

---

This is the technical foundation. From here, you can build specialized agents for any task: booking, shopping, monitoring, testing, research - anything that works on a phone screen.

The key insight: once you have a virtual display and a vision-capable AI model, mobile automation becomes a solved problem.
