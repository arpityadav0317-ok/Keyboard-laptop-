# Universal Keyboard & Mouse

A cross-platform Flutter mobile application with native Android Bluetooth HID emulation (Kotlin), honest iOS platform handling (Swift), and an encrypted C# .NET 8 Windows Companion Receiver.

## Included Modules
- `mobile_app/` — Complete Flutter app (5 screens, dark charcoal laptop keyboard, separate numeric keypad, multi-touch touchpad, Emoji/Text/Media/Presentation/Gaming/Shortcut modes, multi-language & theme settings).
- `mobile_app/android/` — Native Kotlin `BluetoothHidDevice` composite Keyboard + Mouse + Media HID implementation (Android 9+ / API 28+).
- `mobile_app/ios/` — Native Swift configuration with `Info.plist` local-network permissions and honest iOS HID capability reporting.
- `windows_companion/` — Lightweight C# .NET 8 Windows receiver using Win32 [`SendInput`](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-sendinput), UDP LAN auto-discovery, 6-digit PIN / QR pairing, and AES-256-GCM encryption.
- `.github/workflows/build.yml` — Automated GitHub Actions workflow to build both `app-debug.apk` and `UniversalKbReceiver.exe` in the cloud without needing Android Studio locally.

## Quick Setup Instructions

### Option A: Cloud Build via GitHub Actions (No PC / Android Studio Required)
1. Upload the contents of this archive to a GitHub repository.
2. Open the **Actions** tab and run **Build Mobile APK & Windows Companion**.
3. Download `UniversalKeyboardMouse-Android-APK` (`app-debug.apk`) for Android and `UniversalKbReceiver-Windows-x64` (`UniversalKbReceiver.exe`) for Windows.

### Option B: Local Build
**Mobile App (Flutter 3.24+):**
```bash
cd mobile_app
flutter pub get
flutter build apk --debug        # Android
flutter build ios --no-codesign  # iOS (macOS + Xcode required)
```

**Windows Companion Receiver (.NET 8 SDK):**
```powershell
cd windows_companion
dotnet run
```

## Pairing & Usage
- **Bluetooth HID (Android -> Windows / macOS / Smart TV / Google TV):**
  1. Open **Devices -> Bluetooth HID** in the app and tap **Discoverable**.
  2. Pair your phone from your PC or TV Bluetooth settings.
  3. Tap the paired host in the app to connect and start typing or using the touchpad.
- **Encrypted Wi-Fi Mode (Android & iOS -> Windows PC):**
  1. Ensure phone and PC are on the same Wi-Fi network and run `UniversalKbReceiver.exe` on Windows.
  2. Open **Devices -> Wi-Fi Companion** and tap **Discover PCs** (or **Scan QR** / **Enter IP & PIN Manually**).
  3. Enter the 6-digit PIN shown on your PC to establish an AES-256-GCM encrypted session.

See `ARCHITECTURE.md` for full technical details and platform Bluetooth limitations.
