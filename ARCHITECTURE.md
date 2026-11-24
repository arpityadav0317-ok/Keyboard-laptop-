# Universal Keyboard & Mouse — System Architecture & Platform Bluetooth Analysis

## 1. Platform-Specific Bluetooth HID Limitations

| Platform | Bluetooth HID Emulation | Official API / Constraint | Recommended Connection Mode |
| :--- | :--- | :--- | :--- |
| **Android 9+ (API 28+)** | **Supported natively** (Composite Keyboard + Mouse + Consumer Media) | Uses [`BluetoothHidDevice`](https://developer.android.com/reference/android/bluetooth/BluetoothHidDevice) and [`BluetoothHidDeviceAppSdpSettings`](https://developer.android.com/reference/android/bluetooth/BluetoothHidDeviceAppSdpSettings) to register a real USB HID Report Descriptor over Classic Bluetooth HID. | **Bluetooth HID (Primary)** — works directly with Windows, macOS, Android TV, Google TV, and tablets with zero companion software; **Encrypted Wi-Fi** available as alternative. |
| **iOS (iPhone / iPad)** | **Blocked by OS for third-party apps** | Apple's [`CBPeripheralManager`](https://developer.apple.com/documentation/corebluetooth/cbperipheralmanager) restricts third-party apps from publishing the standard BLE HID over GATT service (`0x1812`) and Classic Bluetooth HID is unavailable to public apps ([Apple Developer Forums](https://developer.apple.com/forums/tags/core-bluetooth?page=6)). | **Encrypted Wi-Fi Companion (Primary)** — connects over local LAN using AES-256-GCM to the lightweight Companion Receiver. |
| **Target Hosts (Windows / macOS / Smart TVs)** | Standard Bluetooth HID hosts accept Android HID natively | Windows, macOS, Android TV, and Google TV natively recognize standard USB HID Report IDs 1 (Keyboard), 2 (Mouse), and 3 (Consumer Control). | Pair once in host Bluetooth settings (Android) or run the Companion Receiver (Wi-Fi). |

## 2. Dual-Transport Clean Architecture

```
+-------------------------------------------------------------------+
|                   Flutter UI Layer (Cross-Platform)               |
|  Home | Device Discovery | Laptop Keyboard + Numpad | Touchpad    |
|  Special Modes (Text, Emoji, Media, Slides, Gaming, Shortcuts)    |
+---------------------------------+---------------------------------+
                                  |
                     [ ConnectionManager (Security Gate) ]
                     Blocks all input unless paired & active
                                  |
          +-----------------------+-----------------------+
          |                                               |
+---------v-----------------------+   +-------------------v-------------------+
|  BluetoothHidService (Primary)  |   |  WifiClientService (Universal / iOS)  |
|  Flutter MethodChannel Bridge   |   |  UDP Broadcast Discovery (48291)      |
|  -> Android Kotlin              |   |  PBKDF2-HMAC-SHA256 + AES-256-GCM     |
|     BluetoothHidDevice (API 28+)|   |  Encrypted TCP Stream (48290)         |
|  -> iOS Swift (Reports Honest   |   +-------------------+-------------------+
|     HID Limitation)             |                       |
+---------------------------------+                       v
                                      +---------------------------------------+
                                      |   Windows Companion Receiver (C# .NET)|
                                      |   EncryptedReceiverServer + PIN/QR    |
                                      |   IInputInjector -> Win32 SendInput   |
                                      +---------------------------------------+
```

## 3. Composite USB HID Report Descriptor (Android)
Defined in `mobile_app/android/app/src/main/kotlin/com/universalkb/app/HidDescriptors.kt`:
- **Report ID 1 (Boot Keyboard, 8 bytes):** `[ModifierMask, 0x00, Key1, Key2, Key3, Key4, Key5, Key6]`
- **Report ID 2 (Precision Mouse, 5 bytes):** `[ButtonMask (L/R/M), dX (-127..127), dY (-127..127), VerticalWheel, HorizontalPan]`
- **Report ID 3 (Consumer Control, 2 bytes):** 16-bit Consumer Usage ID (`0x00CD` Play/Pause, `0x00E9` Vol+, `0x00EA` Vol-, `0x00E2` Mute, `0x00B5` Next, `0x00B6` Prev).

## 4. Encrypted Wi-Fi Protocol Specification
1. **Discovery:** Mobile broadcasts `{"type":"UKM_DISCOVER_V1"}` on UDP `48291`; receiver replies with `{"type":"UKM_RECEIVER_ANNOUNCE","name":"...","port":48290}`.
2. **Pairing & Key Derivation:** Receiver generates a 6-digit PIN and per-session 16-byte random salt + 12-byte nonce. Both sides derive a 256-bit key via `PBKDF2-HMAC-SHA256` (100,000 iterations).
3. **Authenticated Transport:** Every input frame is encrypted with **AES-256-GCM** (`12B IV`, ciphertext, `16B` authentication tag). Tampered or unauthenticated packets are immediately rejected.
4. **Windows Input Injection:** `WindowsSendInputInjector.cs` calls the Win32 [`SendInput`](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-sendinput) API via P/Invoke. The `IInputInjector` interface allows dropping in macOS (`CGEvent`) or Linux (`uinput`) implementations without changing networking or crypto code.
