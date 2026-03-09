# Installation Guide

## Prerequisites

Before you begin, understand that openGemi requires **system-level modifications**. This is not a regular app install.

### Required

- Android 17+ device (Pixel 9, or custom ROM based on Android 17)
- Unlocked bootloader
- Root access (via Magisk or similar)
- Basic understanding of ADB and Android system architecture
- OpenRouter API key (or direct API keys for Claude/GPT-4)

### Knowledge Prerequisites

You should be comfortable with:
- Using ADB command line
- Flashing system images
- Modifying system partitions
- Debugging Android issues
- Reversing changes if something breaks

**If you're not comfortable with these, this project isn't for you yet.** Wait for future versions that may work without root.

## Installation Methods

Two paths: **System Build** (clean but slow) or **Runtime Patching** (quick but fragile).

---

## Method 1: System Build (Recommended)

Build a modified Android system image with openGemi patches baked in.

### 1.1 Setup AOSP Build Environment

```bash
# Install dependencies (Ubuntu/Debian)
sudo apt-get install git-core gnupg flex bison build-essential \
  zip curl zlib1g-dev gcc-multilib g++-multilib libc6-dev-i386 \
  libncurses5 lib32ncurses5-dev x11proto-core-dev libx11-dev \
  lib32z1-dev libgl1-mesa-dev libxml2-utils xsltproc unzip \
  fontconfig python3

# Install repo tool
mkdir ~/bin
PATH=~/bin:$PATH
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+x ~/bin/repo
```

### 1.2 Download Android 17 Source

```bash
mkdir aosp-android17
cd aosp-android17

# Initialize repo for Android 17
repo init -u https://android.googlesource.com/platform/manifest \
  -b android-17.0.0_r1 --depth=1

# Sync source (warning: ~100GB download)
repo sync -c -j8
```

This will take 1-4 hours depending on internet speed.

### 1.3 Apply openGemi Patches

```bash
# Clone openGemi patches
cd ~/aosp-android17
git clone https://github.com/Equality-Machine/openGemi.git openGemi-patches

# Apply patches to frameworks/base
cd frameworks/base
git am ~/aosp-android17/openGemi-patches/system-patches/*.patch

# Verify patches applied
git log --oneline -5
# Should see commits like:
# - CCManagerService: Remove package whitelist
# - CCManagerService: Bypass security gates
# - CC Extensions: Remove SSL pinning
```

### 1.4 Build System Image

```bash
cd ~/aosp-android17

# Setup build environment
source build/envsetup.sh

# Choose target device
# For Pixel 9: lunch aosp_komodo-userdebug
# For emulator: lunch aosp_arm64-eng
lunch aosp_komodo-userdebug

# Build (warning: takes 2-8 hours, needs 32GB RAM + 300GB disk)
make -j$(nproc)
```

### 1.5 Flash Device

```bash
# Boot device into fastboot mode
adb reboot bootloader

# Flash system partition
fastboot flash system out/target/product/komodo/system.img
fastboot flash boot out/target/product/komodo/boot.img
fastboot flash vendor out/target/product/komodo/vendor.img

# Reboot
fastboot reboot

# Verify Android 17 is running
adb shell getprop ro.build.version.release
# Should show: 17
```

### 1.6 Install openGemi Agent App

```bash
# Build agent APK
cd ~/aosp-android17/openGemi-patches/agent
./gradlew assembleDebug

# Install
adb install -r app/build/outputs/apk/debug/opengemi-agent-debug.apk
```

### 1.7 Configure OpenRouter

```bash
# Set API key
adb shell pm grant com.opengemi.agent android.permission.WRITE_SECURE_SETTINGS
adb shell content insert --uri content://com.opengemi.agent/config \
  --bind key:s:openrouter_api_key \
  --bind value:s:YOUR_API_KEY_HERE

# Verify
adb shell content query --uri content://com.opengemi.agent/config
```

### 1.8 Test Installation

```bash
# Launch agent
adb shell am start -n com.opengemi.agent/.MainActivity

# Check logs
adb logcat | grep openGemi

# Should see:
# [openGemi] CC session created: session_id=1
# [openGemi] VirtualDisplay initialized: 1080x2400
# [openGemi] OpenRouter connection: OK
```

---

## Method 2: Runtime Patching (Quick Test)

Use Frida to patch system_server at runtime. Changes lost on reboot.

### 2.1 Install Frida

```bash
# Install frida-tools
pip3 install frida-tools

# Download frida-server for Android
wget https://github.com/frida/frida/releases/download/16.1.4/frida-server-16.1.4-android-arm64.xz
unxz frida-server-16.1.4-android-arm64.xz
mv frida-server-16.1.4-android-arm64 frida-server

# Push to device
adb root
adb push frida-server /data/local/tmp/
adb shell chmod 755 /data/local/tmp/frida-server

# Run frida-server
adb shell /data/local/tmp/frida-server &
```

### 2.2 Apply Runtime Patches

```bash
# Clone openGemi
git clone https://github.com/Equality-Machine/openGemi.git
cd openGemi/frida-patches

# Run patch script
python3 patch_cc_manager.py

# Should output:
# [*] Attaching to system_server...
# [*] Patching CCManagerService.isAllowedPackage...
# [*] Patching CCManagerService.hasPermission...
# [*] Patching CCExtensions.verifyCertificate...
# [*] Patches applied successfully
```

### 2.3 Install Agent and Test

Same as Method 1 steps 1.6-1.8.

**Important**: Patches are lost on reboot. Re-run `patch_cc_manager.py` after each boot.

---

## Configuration

### OpenRouter Setup

1. Sign up at https://openrouter.ai
2. Add credits to your account ($5 minimum)
3. Generate API key
4. Configure in app:

```bash
adb shell am start -n com.opengemi.agent/.SettingsActivity
# Or use content provider method from 1.7
```

### Model Selection

Edit agent config:

```bash
# Claude 3.7 Sonnet (recommended, best balance)
adb shell content update --uri content://com.opengemi.agent/config \
  --bind model:s:anthropic/claude-3.7-sonnet

# GPT-4o (faster, cheaper)
adb shell content update --uri content://com.opengemi.agent/config \
  --bind model:s:openai/gpt-4o

# Llama 3.3 70B (cheapest, local option)
adb shell content update --uri content://com.opengemi.agent/config \
  --bind model:s:meta-llama/llama-3.3-70b
```

### Performance Tuning

```bash
# Reduce screenshot quality to save bandwidth
adb shell content update --uri content://com.opengemi.agent/config \
  --bind screenshot_quality:i:70

# Lower resolution (saves cost, faster inference)
adb shell content update --uri content://com.opengemi.agent/config \
  --bind display_width:i:720 \
  --bind display_height:i:1560

# Increase action delay (more reliable for slow apps)
adb shell content update --uri content://com.opengemi.agent/config \
  --bind action_delay_ms:i:1500
```

---

## Verification

### Check System Patches

```bash
# Verify CC service is available
adb shell service list | grep computercontrol
# Should show: 149  computercontrol: [android.computercontrol.IComputerControlManager]

# Check feature flag
adb shell getprop persist.cc.enabled
# Should show: true

# Test session creation
adb shell am instrument -w -e class \
  com.opengemi.agent.test.CCSessionTest \
  com.opengemi.agent.test/androidx.test.runner.AndroidJUnitRunner
# Should show: testCreateSession: PASSED
```

### Check Agent Status

```bash
# Open agent and look for status indicators
adb shell am start -n com.opengemi.agent/.StatusActivity

# Should show:
# ✓ System patches: OK
# ✓ CC service: Available
# ✓ VirtualDisplay: Created (1080x2400)
# ✓ OpenRouter: Connected
# ✓ Model: anthropic/claude-3.7-sonnet
```

---

## Troubleshooting

### Issue: "CC service not found"

**Cause**: System patches not applied or feature flag disabled

**Fix**:
```bash
# Enable feature flag
adb root
adb shell setprop persist.cc.enabled true
adb reboot

# Verify service after reboot
adb shell service list | grep computercontrol
```

### Issue: "SecurityException: Package not whitelisted"

**Cause**: Runtime patches not applied or lost on reboot

**Fix**:
```bash
# Re-apply Frida patches (Method 2)
python3 frida-patches/patch_cc_manager.py

# Or rebuild system image with patches (Method 1)
```

### Issue: "SSL certificate pinning failure"

**Cause**: CC Extensions library still has certificate pinning

**Fix**:
```bash
# Check if CC Extensions patch applied
adb shell dumpsys package com.google.android.ext.computercontrol | grep versionCode

# If not patched, need to rebuild or use Frida patch
python3 frida-patches/patch_ssl_pinning.py
```

### Issue: "Agent crashes on session creation"

**Cause**: Permission issues or missing AIDL stubs

**Fix**:
```bash
# Grant required permissions
adb root
adb shell pm grant com.opengemi.agent android.permission.CREATE_CC_SESSION
adb shell pm grant com.opengemi.agent android.permission.INJECT_EVENTS

# Check logs for specific error
adb logcat -s openGemi:V
```

### Issue: "OpenRouter API errors"

**Cause**: Invalid API key or insufficient credits

**Fix**:
1. Check API key: https://openrouter.ai/keys
2. Check credits: https://openrouter.ai/credits
3. Verify configuration:
```bash
adb shell content query --uri content://com.opengemi.agent/config
```

---

## Uninstallation

### Remove Agent App

```bash
adb uninstall com.opengemi.agent
```

### Revert System Patches (Method 1)

```bash
# Flash stock system image
fastboot flash system stock_system.img
fastboot reboot
```

### Stop Frida Patches (Method 2)

```bash
# Just reboot (patches are runtime-only)
adb reboot
```

---

## Security Considerations

### Risks

- **Root access required**: Exposes device to security risks
- **System modification**: May brick device if done incorrectly
- **API key storage**: OpenRouter key stored on device
- **Screen capture**: Agent captures all UI content in virtual display

### Mitigation

- Use a dedicated test device, not your daily driver
- Don't install sensitive apps (banking, etc.) on test device
- Rotate API keys regularly
- Monitor API usage for unexpected charges
- Keep device updated with security patches

---

## Getting Help

### Resources

- **Documentation**: [/docs](/docs)
- **Issues**: [GitHub Issues](https://github.com/Equality-Machine/openGemi/issues)
- **Discussions**: [GitHub Discussions](https://github.com/Equality-Machine/openGemi/discussions)

### What to Include in Bug Reports

```
**Device**: Pixel 9 / Android 17 build XYZ
**Method**: System Build / Runtime Patching
**Error**: [exact error message from logcat]
**Logs**: [attach output of `adb logcat -s openGemi:V`]
**Steps**: [what you did before the error]
```

---

## Next Steps

Once installed:

1. **Read the docs**: Understand what openGemi can and can't do
2. **Start simple**: Try basic tasks like "open Settings"
3. **Monitor behavior**: Watch what the agent does in VirtualDisplay
4. **Iterate**: Adjust prompts and configuration based on results
5. **Share findings**: Report issues and contribute improvements

This is research software. Expect rough edges.
