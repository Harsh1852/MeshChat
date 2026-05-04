# Building MeshChat: A Beginner's Complete Guide
### A Bluetooth Mesh Chat App for Android & iPhone — Explained from Zero

---

## Before We Write Any Code: Understanding What We're Building

### What is this app?

You're building an app where two (or more) phones can send text messages to each other using only Bluetooth — no Wi-Fi, no mobile data, no internet, no server. Think of it like walkie-talkies, but for text.

### Why is this impressive?

Most apps you've used (WhatsApp, Instagram, iMessage) work like this:

```
Your Phone → Internet → Company Server → Internet → Their Phone
```

Your message travels to a company's server in some data center, and then back out to the other person. If there's no internet, nothing works.

What you're building works like this:

```
Your Phone → Bluetooth signal (through the air) → Their Phone
```

No internet. No server. No middleman. The phones talk directly to each other through the air, the same way a TV remote talks to your television.

---

## Part 1: Understanding the Technology

### What is Bluetooth Low Energy (BLE)?

Regular Bluetooth (the kind that connects headphones) is designed for continuous audio streaming — it needs a lot of power and a permanent connection.

BLE (Bluetooth Low Energy) was invented for devices that need to send small bursts of data occasionally, like fitness trackers reporting your heart rate every few seconds. It uses much less battery.

BLE has two modes that matter to us:

**Advertising mode:** A device broadcasts a signal into the air saying "I exist, here's my name." Like a person standing in a crowd shouting their name. Anyone nearby with BLE scanning turned on can hear it.

**Connected mode (GATT):** Two devices establish a direct connection and can exchange data. Like two people sitting down and having a conversation after one of them heard the other shouting their name.

### What is GATT?

GATT stands for Generic Attribute Profile. It's just a standardized way for two BLE devices to share data once they're connected.

Think of it like a filing cabinet:

- The cabinet itself is called a **Service** (identified by a UUID, which is just a unique ID number)
- Each drawer in the cabinet is called a **Characteristic**
- One drawer might hold the messages you receive
- Another drawer might hold your display name

When phone A connects to phone B, phone A can read phone B's "name" drawer and subscribe to phone B's "messages" drawer so it gets notified any time a new message appears.

### What is Flutter?

Flutter is a framework made by Google that lets you write one codebase and deploy it to both Android and iPhone. Without Flutter, you'd need to write two separate apps — one in Kotlin/Java for Android, one in Swift for iOS. With Flutter, you write in a language called Dart and it compiles to native code for both platforms.

Think of Flutter as a universal translator — you write instructions once, and it translates them into iPhone language AND Android language automatically.

### What is a UUID?

UUID stands for Universally Unique Identifier. It's a 128-bit number written like this:

```
12345678-1234-1234-1234-123456789abc
```

It's used to identify things uniquely. When your app scans for BLE devices, it looks for devices that are advertising a specific UUID — your app's UUID. This is how it filters out headphones, fitness trackers, and other random Bluetooth devices and only finds other phones running your app.

### What is a Mesh Network?

In a normal connection, A talks to B directly:

```
A ←→ B
```

In a mesh network, every device can relay messages for other devices:

```
A ←→ B ←→ C ←→ D
```

So if A is too far from D, A sends to B, B sends to C, C sends to D. The message "hops" through the network. This extends range dramatically.

To prevent a message from bouncing forever (A→B→C→B→C forever...), each message carries a TTL (Time To Live) counter. Every time a device relays the message, it decrements the TTL by 1. When TTL hits 0, the device discards the message instead of relaying it.

To prevent a device from relaying the same message twice (creating loops), every message has a unique ID. Devices keep a list of IDs they've already seen and skip any message whose ID is in that list.

---

## Part 2: Project Setup

### Step 1 — Install Flutter

Flutter is the tool that builds your app. You need to install it on your computer first.

**On Windows:**

1. Go to https://flutter.dev/docs/get-started/install/windows
2. Download the Flutter SDK zip file
3. Extract it to `C:\flutter` (don't put it in `Program Files` — Flutter doesn't like spaces in the path)
4. Add `C:\flutter\bin` to your system PATH:
   - Search "environment variables" in Windows Start
   - Click "Edit the system environment variables"
   - Click "Environment Variables"
   - Under "System variables", find "Path", click Edit
   - Click New and add `C:\flutter\bin`
   - Click OK on everything

5. Open a new terminal (PowerShell or CMD) and run:
```bash
flutter doctor
```

This command checks if everything is installed correctly. It will show you a checklist. The important ones are Flutter itself, Android toolchain, and Android Studio. Fix anything with an ✗ next to it.

**Install Android Studio:**
- Download from https://developer.android.com/studio
- During installation, make sure to install the Android SDK
- After installing, open Android Studio and let it finish installing components

**Why do we need Android Studio even if we're using Flutter?**
Flutter needs Android Studio's build tools (the compilers that turn your Dart code into an Android app). You don't need to use Android Studio as your code editor — you can use VS Code — but the tools it installs behind the scenes are essential.

**Install VS Code (recommended code editor):**
- Download from https://code.visualstudio.com
- Install the "Flutter" extension and the "Dart" extension from the Extensions panel

### Step 2 — Create the Project

Open a terminal, navigate to wherever you want to store your projects, and run:

```bash
flutter create meshchat
cd meshchat
```

**What does this do?**
`flutter create meshchat` generates a complete starter Flutter project in a folder called `meshchat`. It creates all the boilerplate files you need — the Android project, the iOS project, and the Dart source files that sit on top of both.

After running this, open the folder in VS Code:
```bash
code .
```

You'll see a folder structure like this:

```
meshchat/
├── android/          ← Android-specific files (you'll edit one file here)
├── ios/              ← iOS-specific files (you'll edit one file here)
├── lib/              ← THIS is where you write your app code
│   └── main.dart     ← The entry point of your app
├── pubspec.yaml      ← Your app's dependency list (like package.json in Node)
└── test/             ← Test files (ignore for now)
```

You'll spend almost all your time inside `lib/`.

### Step 3 — Add Dependencies

Open `pubspec.yaml`. This file lists all the external packages your app needs — like a shopping list for your project.

Find the `dependencies:` section and replace it with:

```yaml
dependencies:
  flutter:
    sdk: flutter
  flutter_blue_plus: ^1.31.0
  provider: ^6.1.1
  uuid: ^4.3.3
  shared_preferences: ^2.2.2
  intl: ^0.19.0
```

**What each package does:**

- `flutter_blue_plus` — This is the most important one. It's a library that lets your Flutter app talk to the phone's Bluetooth hardware. Without this, your Dart code has no way to access BLE.

- `provider` — State management. When a new message arrives via Bluetooth, you need the chat screen to automatically refresh and show the new message. Provider is a system for connecting your data layer (the BLE service) to your UI layer (the chat screen) so the UI updates automatically when data changes.

- `uuid` — Generates universally unique IDs. Used to give each message a unique identity for deduplication.

- `shared_preferences` — Lets you save small pieces of data (like the user's chosen username) to the phone's local storage, so it persists even after the app is closed and reopened.

- `intl` — Provides date/time formatting utilities. Used to display timestamps like "14:35" on messages.

After editing `pubspec.yaml`, run:

```bash
flutter pub get
```

This downloads all the packages to your machine.

---

## Part 3: Platform Permissions

### Why do we need permissions?

Both Android and iOS protect sensitive hardware (like Bluetooth, camera, microphone) behind a permission system. Your app must explicitly declare what hardware it needs to access, and in some cases, ask the user at runtime. If you don't declare permissions, the app will crash the moment it tries to use Bluetooth.

### Step 4 — Android Permissions

Open `android/app/src/main/AndroidManifest.xml`.

This is an XML file that describes your Android app to the operating system. It's like a job application — you tell Android what your app is called, what it needs, and what it can do.

Find the `<manifest>` opening tag and add these lines directly inside it, before the `<application>` tag:

```xml
<uses-permission android:name="android.permission.BLUETOOTH" />
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />
<uses-permission android:name="android.permission.BLUETOOTH_SCAN"
    android:usesPermissionFlags="neverForLocation" />
<uses-permission android:name="android.permission.BLUETOOTH_ADVERTISE" />
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
```

**Why are there so many Bluetooth permissions?**
Android 12 split the old single `BLUETOOTH` permission into several specific ones:
- `BLUETOOTH_SCAN` — to scan/discover nearby devices
- `BLUETOOTH_ADVERTISE` — to broadcast your presence
- `BLUETOOTH_CONNECT` — to actually connect to a device

The location permissions are required because BLE scanning could theoretically be used to determine your location (via Bluetooth beacons in stores, etc.), so Android requires location permission for BLE scanning.

Also in `android/app/build.gradle`, find `minSdkVersion` and set it to `21`:

```gradle
defaultConfig {
    minSdkVersion 21
    ...
}
```

**Why?** `flutter_blue_plus` requires Android API level 21 (Android 5.0) or higher. This line ensures your app won't be installed on older devices that can't support it.

### Step 5 — iOS Permissions

Open `ios/Runner/Info.plist`.

This is the iOS equivalent of AndroidManifest.xml. It's also XML.

Add these lines inside the `<dict>` tag:

```xml
<key>NSBluetoothAlwaysUsageDescription</key>
<string>Used to find and chat with nearby devices over Bluetooth mesh</string>
<key>NSBluetoothPeripheralUsageDescription</key>
<string>Used to advertise your presence to nearby devices</string>
```

**Why?** iOS requires that every permission usage has a human-readable explanation. When the app first requests Bluetooth access, iOS shows a popup with your explanation text so the user knows why the app needs it. If you don't include these keys, iOS will crash your app the moment it tries to use Bluetooth.

---

## Part 4: Building the Data Models

### What is a data model?

A data model is a Dart class that represents a piece of information in your app. Instead of passing around raw maps and strings, you define structured objects — a `ChatMessage` object has a `content` field, a `senderId` field, a `timestamp` field, etc. This makes your code readable and safe.

Create a folder: `lib/models/`

### Step 6 — The Message Model

Create `lib/models/message.dart`:

```dart
import 'package:uuid/uuid.dart';

class ChatMessage {
  final String id;
  final String senderId;
  final String senderName;
  final String content;
  final DateTime timestamp;
  final int ttl;
  bool delivered;

  ChatMessage({
    String? id,
    required this.senderId,
    required this.senderName,
    required this.content,
    DateTime? timestamp,
    this.ttl = 7,
    this.delivered = false,
  })  : id = id ?? const Uuid().v4(),
        timestamp = timestamp ?? DateTime.now();

  Map<String, dynamic> toJson() => {
        'id': id,
        'sid': senderId,
        'sn': senderName,
        'c': content,
        'ts': timestamp.millisecondsSinceEpoch,
        'ttl': ttl,
      };

  factory ChatMessage.fromJson(Map<String, dynamic> json) => ChatMessage(
        id: json['id'],
        senderId: json['sid'],
        senderName: json['sn'],
        content: json['c'],
        timestamp: DateTime.fromMillisecondsSinceEpoch(json['ts']),
        ttl: json['ttl'] ?? 7,
      );

  ChatMessage? withDecrementedTTL() {
    if (ttl <= 1) return null;
    return ChatMessage(
      id: id,
      senderId: senderId,
      senderName: senderName,
      content: content,
      timestamp: timestamp,
      ttl: ttl - 1,
    );
  }
}
```

**Understanding every piece:**

`final String id;`
Every message gets a unique ID (generated by the `uuid` package). This is how devices avoid relaying the same message twice. When device B receives a message, it checks: "Have I seen this ID before?" If yes, it ignores it. If no, it processes it and adds the ID to its seen-list.

`final String senderId;`
A unique identifier for the person who sent the message. This is generated once when the app starts and never changes (until the app is reinstalled). It's used to distinguish "my own messages" from others' messages.

`final String senderName;`
The human-readable display name the sender chose (like "Harsh"). This is what appears in the chat UI.

`final int ttl;` (default: 7)
Time To Live. Starts at 7 and decrements by 1 each time a device relays it. When it hits 0, the message is dropped instead of relayed. 7 hops × ~30 meters per hop = up to ~210 meters of range in a dense crowd.

`Map<String, dynamic> toJson()`
Before sending a message over Bluetooth, you need to convert the `ChatMessage` object into bytes (raw binary data). The path is: ChatMessage → JSON map → JSON string → bytes.

Notice the keys are abbreviated (`'sid'` instead of `'senderId'`, `'c'` instead of `'content'`). BLE has a maximum packet size of around 512 bytes. Using short keys saves space.

`factory ChatMessage.fromJson(...)`
The reverse: when you receive bytes from Bluetooth, you convert them back into a `ChatMessage` object. Bytes → JSON string → JSON map → ChatMessage.

`withDecrementedTTL()`
Returns a new ChatMessage with TTL reduced by 1, or `null` if TTL is already at 1 (meaning this is the last hop and the message should be dropped, not relayed).

### Step 7 — The Peer Model

Create `lib/models/peer.dart`:

```dart
import 'package:flutter_blue_plus/flutter_blue_plus.dart';

class Peer {
  final String deviceId;
  String name;
  BluetoothDevice? device;
  int rssi;
  DateTime lastSeen;
  bool isConnected;

  Peer({
    required this.deviceId,
    required this.name,
    this.device,
    this.rssi = -100,
    DateTime? lastSeen,
    this.isConnected = false,
  }) : lastSeen = lastSeen ?? DateTime.now();

  String get proximityLabel {
    if (rssi > -60) return 'Very Close';
    if (rssi > -75) return 'Nearby';
    if (rssi > -90) return 'Far';
    return 'Very Far';
  }
}
```

**Understanding every piece:**

`final String deviceId`
Every Bluetooth device has a hardware address (like `AA:BB:CC:DD:EE:FF`) that uniquely identifies it. On Android you get the raw MAC address. On iOS, Apple randomizes this for privacy, but `flutter_blue_plus` handles this and gives you a consistent ID per device per app session.

`BluetoothDevice? device`
The actual `flutter_blue_plus` object representing the connected device. You need this to send data to the device. The `?` means it can be null — when you first discover a peer you haven't connected to yet, you have their ID but not yet the device object.

`int rssi`
Received Signal Strength Indicator. This is a negative number (in dBm) representing how strong the Bluetooth signal is. -40 means very close, -90 means very far. We use this to show proximity labels in the UI.

`proximityLabel` getter
This is a computed property — it doesn't store a value, it calculates one on the fly based on `rssi` every time you ask for it. RSSI thresholds are approximate and vary by device hardware, but these ranges are reasonable.

---

## Part 5: The BLE Service — The Heart of the App

This is the most complex file, but it's where all the interesting things happen. Take your time with each section.

Create `lib/services/ble_service.dart`:

### Section A — Setup and UUIDs

```dart
import 'dart:async';
import 'dart:convert';
import 'package:flutter_blue_plus/flutter_blue_plus.dart';
import 'package:uuid/uuid.dart';
import '../models/message.dart';
import '../models/peer.dart';

class BleService {
  static const String SERVICE_UUID =
      '12345678-1234-1234-1234-123456789abc';
  static const String MESSAGE_CHAR_UUID =
      '12345678-1234-1234-1234-123456789abd';
  static const String NAME_CHAR_UUID =
      '12345678-1234-1234-1234-123456789abe';

  final String localUserId = const Uuid().v4();
  String localUserName = 'User';
```

**Why three UUIDs?**

`SERVICE_UUID` identifies your app's BLE service. When your phone scans for nearby BLE devices, it specifically looks for devices advertising this UUID. This filters out every Bluetooth device in the world except other phones running your app.

`MESSAGE_CHAR_UUID` identifies the "messages" characteristic — the channel for receiving text messages.

`NAME_CHAR_UUID` identifies the "name" characteristic — when you connect to someone, you read this to get their display name.

**Important:** These are placeholder UUIDs. For your real project, generate your own at https://www.uuidgenerator.net to avoid collisions with other apps.

`localUserId` is generated fresh when the app starts. It's your unique identity on the network. This is different from your display name — the ID never changes (within a session), but the display name can be changed anytime.

### Section B — Streams

```dart
  final Map<String, Peer> _peers = {};
  final Set<String> _seenMessageIds = {};
  final List<ChatMessage> _messages = [];

  final _messageController = StreamController<ChatMessage>.broadcast();
  final _peerController = StreamController<Map<String, Peer>>.broadcast();

  Stream<ChatMessage> get messageStream => _messageController.stream;
  Stream<Map<String, Peer>> get peerStream => _peerController.stream;
```

**What is a Stream?**

A Stream is like a pipe for events. You can push data into one end and listen at the other end. Whenever new data arrives, any listener gets notified automatically.

This is crucial for a chat app. When a message arrives via Bluetooth (which happens on a background thread at an unpredictable time), you need the UI to instantly refresh. Streams are how you bridge the gap between "something happened in the background" and "update the UI now."

`StreamController.broadcast()` creates a stream that can have multiple listeners — both the chat screen and a notification system could listen to the same message stream simultaneously.

`_seenMessageIds` is a `Set` (not a List) because Sets automatically ignore duplicates and have very fast lookup. This is the deduplication mechanism — every time we process a message, we add its ID here. Next time we see the same ID, we check this set and know to ignore it.

### Section C — Scanning

```dart
  void startScanning() {
    FlutterBluePlus.startScan(
      withServices: [Guid(SERVICE_UUID)],
      timeout: const Duration(seconds: 30),
      continuousUpdates: true,
    );

    FlutterBluePlus.scanResults.listen((results) {
      for (ScanResult r in results) {
        _handleDiscoveredDevice(r.device, r.rssi);
      }
    });
  }
```

**What's happening here?**

`startScan(withServices: [Guid(SERVICE_UUID)])` tells the phone's Bluetooth hardware to start scanning the airwaves and report any device that is advertising your `SERVICE_UUID`. The hardware does the filtering — only matching devices get reported to your app.

`continuousUpdates: true` means the scan results stream will emit every time the RSSI updates for already-discovered devices, not just when a new device is found. This is how you get live proximity updates.

`timeout: 30 seconds` — after 30 seconds, scanning stops automatically. In a real app, you'd restart the scan periodically.

`FlutterBluePlus.scanResults.listen(...)` subscribes to the stream of scan results. Every time a matching device is discovered (or its RSSI updates), this callback fires.

### Section D — Connecting to a Discovered Device

```dart
  Future<void> _handleDiscoveredDevice(
      BluetoothDevice device, int rssi) async {
    final id = device.remoteId.str;

    if (_peers.containsKey(id) && _peers[id]!.isConnected) {
      _peers[id]!.rssi = rssi;
      _peers[id]!.lastSeen = DateTime.now();
      _peerController.add(_peers);
      return;
    }

    try {
      await device.connect(timeout: const Duration(seconds: 10));
      final services = await device.discoverServices();

      for (BluetoothService service in services) {
        if (service.serviceUuid.toString() == SERVICE_UUID) {
          BluetoothCharacteristic? msgChar;
          BluetoothCharacteristic? nameChar;

          for (var c in service.characteristics) {
            if (c.characteristicUuid.toString() == MESSAGE_CHAR_UUID) {
              msgChar = c;
            }
            if (c.characteristicUuid.toString() == NAME_CHAR_UUID) {
              nameChar = c;
            }
          }

          String peerName = 'Unknown';
          if (nameChar != null) {
            final nameBytes = await nameChar.read();
            peerName = utf8.decode(nameBytes);
          }

          if (msgChar != null) {
            await msgChar.setNotifyValue(true);
            msgChar.lastValueStream.listen((data) {
              if (data.isNotEmpty) {
                _handleIncomingData(data, id);
              }
            });
          }

          _peers[id] = Peer(
            deviceId: id,
            name: peerName,
            device: device,
            rssi: rssi,
            isConnected: true,
          );

          _peerController.add(_peers);
          break;
        }
      }
    } catch (e) {
      print('Connection failed to $id: $e');
    }
  }
```

**Step by step:**

1. `if (_peers.containsKey(id) && _peers[id]!.isConnected)` — If we're already connected to this device, don't try to connect again. Just update their signal strength (RSSI) and timestamp.

2. `await device.connect(...)` — Establish a GATT connection with the discovered device. This is like "picking up the phone" after seeing someone in the scan.

3. `await device.discoverServices()` — Once connected, ask the device: "What services do you offer?" The other device responds with a list of services and their characteristics.

4. `service.serviceUuid.toString() == SERVICE_UUID` — Find the specific service that belongs to your app.

5. Reading the name characteristic: `nameChar.read()` returns bytes. `utf8.decode(...)` converts bytes to a string. So you get the other person's display name.

6. `msgChar.setNotifyValue(true)` — This is key. You're telling the other device: "Subscribe me to notifications on this characteristic." From this moment on, whenever the other device writes a new message to this characteristic, your phone gets a notification with the new data automatically. You don't have to keep asking "any new messages?" — they get pushed to you.

7. `msgChar.lastValueStream.listen(...)` — This is the stream that fires when a notification arrives with new data.

### Section E — Processing an Incoming Message

```dart
  void _handleIncomingData(List<int> data, String fromDeviceId) {
    try {
      final jsonStr = utf8.decode(data);
      final json = jsonDecode(jsonStr);
      final message = ChatMessage.fromJson(json);

      if (_seenMessageIds.contains(message.id)) return;
      _seenMessageIds.add(message.id);

      if (message.senderId != localUserId) {
        _messages.add(message);
        _messageController.add(message);
      }

      final relayed = message.withDecrementedTTL();
      if (relayed != null) {
        _relayMessage(relayed, excludeDeviceId: fromDeviceId);
      }
    } catch (e) {
      print('Failed to parse incoming message: $e');
    }
  }
```

**The full journey of a received message:**

1. `data` arrives as `List<int>` — raw bytes
2. `utf8.decode(data)` turns bytes into a JSON string like `{"id":"abc","sn":"Harsh","c":"Hello",...}`
3. `jsonDecode(jsonStr)` parses the JSON string into a Dart map
4. `ChatMessage.fromJson(json)` wraps the map in a proper ChatMessage object
5. **Deduplication check:** If we've seen this message ID before, stop here and return
6. Add the ID to `_seenMessageIds` so we never process it again
7. **Display check:** If the message is NOT from ourselves (`senderId != localUserId`), add it to the messages list and push it to the UI stream
8. **Relay check:** Decrement TTL. If TTL > 0, relay the message to all other connected peers (except the one it came from)

Step 8 is what makes this a mesh network, not just point-to-point chat.

### Section F — Sending a Message

```dart
  Future<void> sendMessage(String content) async {
    final message = ChatMessage(
      senderId: localUserId,
      senderName: localUserName,
      content: content,
    );

    _seenMessageIds.add(message.id);
    _messages.add(message);
    _messageController.add(message);

    final data = utf8.encode(jsonEncode(message.toJson()));
    for (var peer in _peers.values) {
      if (!peer.isConnected || peer.device == null) continue;
      await _writeToDevice(peer.device!, data);
    }
  }
```

**Why do we add our own message to `_seenMessageIds` immediately?**

Because when we send a message to peer B, peer B might try to relay it back to peer A (us). If we haven't marked our own message ID as "seen", we'd display our own message twice when it bounces back. By pre-marking the ID, we silently ignore any relayed copies of our own messages.

### Section G — Writing to a Device

```dart
  Future<void> _writeToDevice(
      BluetoothDevice device, List<int> data) async {
    try {
      final services = await device.discoverServices();
      for (var service in services) {
        if (service.serviceUuid.toString() == SERVICE_UUID) {
          for (var char in service.characteristics) {
            if (char.characteristicUuid.toString() == MESSAGE_CHAR_UUID) {
              if (data.length <= 512) {
                await char.write(data, withoutResponse: true);
              } else {
                for (int i = 0; i < data.length; i += 512) {
                  final chunk = data.sublist(
                      i, i + 512 > data.length ? data.length : i + 512);
                  await char.write(chunk, withoutResponse: true);
                }
              }
            }
          }
        }
      }
    } catch (e) {
      print('Write failed: $e');
    }
  }
```

**Why the 512-byte chunking?**

BLE has an MTU (Maximum Transmission Unit) — the maximum number of bytes that can be sent in one packet. The default MTU is 20 bytes, but after negotiation it can go up to 512 bytes. Long messages need to be split into chunks.

`withoutResponse: true` means we send the data without waiting for the receiver to acknowledge receipt. This is faster (like UDP vs TCP in networking terms). For a chat app this is fine — if a message is lost, it's lost, which is acceptable behavior.

---

## Part 6: State Management with Provider

### Why do we need state management?

Your BLE service runs in the background. Your chat screen is the UI. They're separate classes. When a message arrives via Bluetooth (in the BLE service), how does the chat screen know to refresh?

You could make the chat screen directly reference the BLE service. But as your app grows, this creates a mess of circular dependencies and hard-to-follow code.

Provider solves this cleanly: it sits between the BLE service and the UI, and when data changes, the UI automatically rebuilds.

### Step 9 — Chat Provider

Create `lib/providers/chat_provider.dart`:

```dart
import 'package:flutter/foundation.dart';
import 'package:shared_preferences/shared_preferences.dart';
import '../services/ble_service.dart';
import '../models/message.dart';
import '../models/peer.dart';

class ChatProvider extends ChangeNotifier {
  final BleService _ble = BleService();

  List<ChatMessage> get messages => _ble.messages;
  Map<String, Peer> get peers => _ble.peers;
  String get userName => _ble.localUserName;

  ChatProvider() {
    _init();
  }

  Future<void> _init() async {
    final prefs = await SharedPreferences.getInstance();
    _ble.localUserName = prefs.getString('username') ?? 'User';

    _ble.messageStream.listen((_) => notifyListeners());
    _ble.peerStream.listen((_) => notifyListeners());

    _ble.startScanning();
    notifyListeners();
  }

  Future<void> setUserName(String name) async {
    _ble.localUserName = name;
    final prefs = await SharedPreferences.getInstance();
    await prefs.setString('username', name);
    notifyListeners();
  }

  Future<void> sendMessage(String content) async {
    await _ble.sendMessage(content);
    notifyListeners();
  }

  @override
  void dispose() {
    _ble.dispose();
    super.dispose();
  }
}
```

**Understanding ChangeNotifier:**

`ChangeNotifier` is a Flutter class. When you extend it, you get the ability to call `notifyListeners()`. Any widget in the UI that is "watching" this provider will automatically rebuild when `notifyListeners()` is called.

So the flow is:
1. BLE service receives message → emits on `messageStream`
2. ChatProvider is listening to `messageStream` → calls `notifyListeners()`
3. Chat screen widget is watching ChatProvider → rebuilds with the new message visible

`SharedPreferences.getInstance()` opens a small key-value store on the device. It's like a tiny database for simple values. Here we use it to save and retrieve the username. `prefs.getString('username') ?? 'User'` means: get the stored username, or if there is none (first launch), use 'User' as the default.

---

## Part 7: Building the UI

The UI has one main screen: the chat screen. It shows the list of messages, a text input at the bottom, and some buttons in the top bar.

### Step 10 — Chat Screen

Create `lib/screens/chat_screen.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import 'package:intl/intl.dart';
import '../providers/chat_provider.dart';
import '../models/message.dart';
import '../models/peer.dart';

class ChatScreen extends StatefulWidget {
  const ChatScreen({super.key});

  @override
  State<ChatScreen> createState() => _ChatScreenState();
}

class _ChatScreenState extends State<ChatScreen> {
  final TextEditingController _inputController = TextEditingController();
  final ScrollController _scrollController = ScrollController();

  void _scrollToBottom() {
    WidgetsBinding.instance.addPostFrameCallback((_) {
      if (_scrollController.hasClients) {
        _scrollController.animateTo(
          _scrollController.position.maxScrollExtent,
          duration: const Duration(milliseconds: 300),
          curve: Curves.easeOut,
        );
      }
    });
  }

  @override
  Widget build(BuildContext context) {
    final chat = context.watch<ChatProvider>();
    _scrollToBottom();

    return Scaffold(
      backgroundColor: const Color(0xFF121212),
      appBar: AppBar(
        backgroundColor: const Color(0xFF1E1E1E),
        title: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            const Text('#mesh',
                style: TextStyle(color: Colors.white, fontWeight: FontWeight.bold)),
            Text('${chat.peers.length} peer(s) nearby',
                style: const TextStyle(color: Colors.greenAccent, fontSize: 12)),
          ],
        ),
        actions: [
          IconButton(
            icon: const Icon(Icons.people, color: Colors.white),
            onPressed: () => _showPeersSheet(context, chat.peers),
          ),
          IconButton(
            icon: const Icon(Icons.person, color: Colors.white),
            onPressed: () => _showUsernameDialog(context, chat),
          ),
        ],
      ),
      body: Column(
        children: [
          Expanded(
            child: chat.messages.isEmpty
                ? const Center(
                    child: Text('No messages yet.\nSend one!',
                        textAlign: TextAlign.center,
                        style: TextStyle(color: Colors.grey)))
                : ListView.builder(
                    controller: _scrollController,
                    padding: const EdgeInsets.all(12),
                    itemCount: chat.messages.length,
                    itemBuilder: (_, i) => _MessageBubble(
                      message: chat.messages[i],
                      isMe: chat.messages[i].senderName == chat.userName,
                    ),
                  ),
          ),
          _buildInput(chat),
        ],
      ),
    );
  }

  Widget _buildInput(ChatProvider chat) {
    return Container(
      color: const Color(0xFF1E1E1E),
      padding: const EdgeInsets.symmetric(horizontal: 12, vertical: 8),
      child: Row(
        children: [
          Expanded(
            child: TextField(
              controller: _inputController,
              style: const TextStyle(color: Colors.white),
              decoration: InputDecoration(
                hintText: 'Message...',
                hintStyle: const TextStyle(color: Colors.grey),
                filled: true,
                fillColor: const Color(0xFF2C2C2C),
                border: OutlineInputBorder(
                  borderRadius: BorderRadius.circular(24),
                  borderSide: BorderSide.none,
                ),
                contentPadding:
                    const EdgeInsets.symmetric(horizontal: 16, vertical: 10),
              ),
            ),
          ),
          const SizedBox(width: 8),
          CircleAvatar(
            backgroundColor: Colors.greenAccent,
            child: IconButton(
              icon: const Icon(Icons.send, color: Colors.black, size: 18),
              onPressed: () {
                final text = _inputController.text.trim();
                if (text.isNotEmpty) {
                  chat.sendMessage(text);
                  _inputController.clear();
                }
              },
            ),
          ),
        ],
      ),
    );
  }

  void _showPeersSheet(BuildContext context, Map<String, Peer> peers) {
    showModalBottomSheet(
      context: context,
      backgroundColor: const Color(0xFF1E1E1E),
      builder: (_) => ListView(
        padding: const EdgeInsets.all(16),
        children: [
          const Text('Nearby Peers',
              style: TextStyle(
                  color: Colors.white,
                  fontSize: 18,
                  fontWeight: FontWeight.bold)),
          const SizedBox(height: 12),
          if (peers.isEmpty)
            const Text('No peers found yet',
                style: TextStyle(color: Colors.grey)),
          ...peers.values.map((p) => ListTile(
                leading: const CircleAvatar(
                    backgroundColor: Colors.greenAccent,
                    child: Icon(Icons.person, color: Colors.black)),
                title: Text(p.name,
                    style: const TextStyle(color: Colors.white)),
                subtitle: Text(p.proximityLabel,
                    style: const TextStyle(color: Colors.greenAccent)),
                trailing: Text('${p.rssi} dBm',
                    style: const TextStyle(color: Colors.grey)),
              )),
        ],
      ),
    );
  }

  void _showUsernameDialog(BuildContext context, ChatProvider chat) {
    final controller = TextEditingController(text: chat.userName);
    showDialog(
      context: context,
      builder: (_) => AlertDialog(
        backgroundColor: const Color(0xFF1E1E1E),
        title: const Text('Set Username',
            style: TextStyle(color: Colors.white)),
        content: TextField(
          controller: controller,
          style: const TextStyle(color: Colors.white),
          decoration: const InputDecoration(
            hintText: 'Enter your name',
            hintStyle: TextStyle(color: Colors.grey),
            enabledBorder: UnderlineInputBorder(
                borderSide: BorderSide(color: Colors.greenAccent)),
          ),
        ),
        actions: [
          TextButton(
            onPressed: () {
              chat.setUserName(controller.text.trim());
              Navigator.pop(context);
            },
            child: const Text('Save',
                style: TextStyle(color: Colors.greenAccent)),
          ),
        ],
      ),
    );
  }
}
```

**Key UI concepts:**

`StatefulWidget` vs `StatelessWidget`: A StatelessWidget is for UI that never changes after being built. A StatefulWidget has internal state that can change (like the scroll position, or whether a loading spinner is showing). We use StatefulWidget here because of the scroll controller.

`context.watch<ChatProvider>()` — this is Provider in action. It reads the ChatProvider from the widget tree AND subscribes to it. Any time ChatProvider calls `notifyListeners()`, this widget rebuilds automatically.

`TextEditingController` — tracks what the user has typed in the text field. You use it to read the current value (`_inputController.text`) and to clear it after sending (`_inputController.clear()`).

`ListView.builder` — instead of building all message bubbles at once (which would be slow for 1000 messages), this builds only the ones currently visible on screen, plus a few above and below for smooth scrolling.

`_scrollToBottom()` — after each rebuild (which happens when a new message arrives), we animate the scroll position to the bottom so the newest message is always visible. `addPostFrameCallback` waits until the current frame finishes rendering before scrolling, ensuring the new message is already in the list before we try to scroll to it.

### Step 11 — The Message Bubble Widget

Add this class to the same `chat_screen.dart` file:

```dart
class _MessageBubble extends StatelessWidget {
  final ChatMessage message;
  final bool isMe;

  const _MessageBubble({required this.message, required this.isMe});

  @override
  Widget build(BuildContext context) {
    return Align(
      alignment: isMe ? Alignment.centerRight : Alignment.centerLeft,
      child: Container(
        margin: const EdgeInsets.symmetric(vertical: 4),
        padding: const EdgeInsets.symmetric(horizontal: 14, vertical: 10),
        constraints: BoxConstraints(
            maxWidth: MediaQuery.of(context).size.width * 0.75),
        decoration: BoxDecoration(
          color: isMe ? Colors.greenAccent : const Color(0xFF2C2C2C),
          borderRadius: BorderRadius.circular(16),
        ),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            if (!isMe)
              Text(message.senderName,
                  style: const TextStyle(
                      color: Colors.greenAccent,
                      fontWeight: FontWeight.bold,
                      fontSize: 12)),
            Text(message.content,
                style: TextStyle(
                    color: isMe ? Colors.black : Colors.white,
                    fontSize: 15)),
            const SizedBox(height: 2),
            Text(
              DateFormat('HH:mm').format(message.timestamp),
              style: TextStyle(
                  color: isMe ? Colors.black54 : Colors.grey,
                  fontSize: 10),
            ),
          ],
        ),
      ),
    );
  }
}
```

**The design logic:**
- `isMe` bubbles align right (like iMessage) and are green
- Others' bubbles align left and are dark grey
- `maxWidth: 75%` prevents bubbles from stretching edge-to-edge
- `if (!isMe)` only shows the sender's name for other people's messages (you know it's you, so no need to label your own)
- `DateFormat('HH:mm')` formats timestamps as 24-hour time like "14:35"

---

## Part 8: Wiring Everything Together

### Step 12 — main.dart

Replace the entire contents of `lib/main.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import 'providers/chat_provider.dart';
import 'screens/chat_screen.dart';

void main() {
  runApp(
    ChangeNotifierProvider(
      create: (_) => ChatProvider(),
      child: const MyApp(),
    ),
  );
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'MeshChat',
      debugShowCheckedModeBanner: false,
      theme: ThemeData.dark(),
      home: const ChatScreen(),
    );
  }
}
```

**What is `main()`?**
In Dart, `main()` is the entry point of your program — the first thing that runs when the app launches.

**What is `ChangeNotifierProvider`?**
This wraps your entire app and makes `ChatProvider` available to every widget in the tree. Think of it like a global context. Any widget anywhere in the app can call `context.watch<ChatProvider>()` to access it.

**`debugShowCheckedModeBanner: false`** removes the red "DEBUG" banner in the top-right corner of the screen during development.

---

## Part 9: Your Final File Structure

At this point, your `lib/` folder should look like:

```
lib/
├── main.dart                    ← App entry point
├── models/
│   ├── message.dart             ← ChatMessage data model
│   └── peer.dart                ← Peer data model
├── services/
│   └── ble_service.dart         ← All Bluetooth logic
├── providers/
│   └── chat_provider.dart       ← State management bridge
└── screens/
    └── chat_screen.dart         ← The UI
```

---

## Part 10: Running and Testing

### Step 13 — Run on Android

1. Enable Developer Options on your Android phone:
   - Go to Settings → About Phone
   - Tap "Build Number" 7 times until it says "You are now a developer"
   - Go back to Settings → Developer Options
   - Enable "USB Debugging"

2. Connect your phone to your computer with a USB cable

3. When prompted on your phone, tap "Allow USB Debugging"

4. In your project terminal, run:
```bash
flutter devices
```
You should see your phone listed. Then:
```bash
flutter run
```

Flutter will compile the app and install it on your phone. First build takes 3-5 minutes. Subsequent builds are faster.

### Step 14 — Test Between Two Phones

To see the BLE messaging actually work, you need two devices:

1. Install and run the app on your Android phone as above
2. For your iPhone, either:
   - **Option A (Recommended):** Use [Codemagic](https://codemagic.io) — free CI/CD for Flutter. Connect your GitHub repo, it builds the iOS IPA for you, and you install it via TestFlight
   - **Option B:** If you have access to a Mac, run `flutter run` connected to the iPhone via USB (requires Xcode)

3. Open the app on both phones, make sure Bluetooth is enabled
4. Wait 10-30 seconds for discovery to complete (watch the peer count in the app bar)
5. Set different usernames on each phone (tap the person icon in the top right)
6. Send a message from one phone — it should appear on the other within 1-2 seconds

### Common Problems and Fixes

**"No devices found" after scanning:**
- Make sure Bluetooth is ON on both phones
- Make sure location permission is granted (Android requires it for BLE scanning)
- The scan timeout is 30 seconds — restart the app to re-trigger scanning

**"Build failed" errors:**
- Run `flutter clean` then `flutter pub get` then `flutter run` again
- Check that `minSdkVersion` is set to 21 in `build.gradle`

**Messages not appearing:**
- Check that both phones have the same `SERVICE_UUID` in `ble_service.dart`
- Make sure the app is in the foreground on both phones (background BLE is restricted on iOS)

---

## Part 11: What to Add Next (Portfolio Boosters)

Once the basic app works, here are three features you can add — each one teaches you something new:

### Feature A: End-to-End Encryption
Add the `pointycastle` package to encrypt message content with a shared passphrase before sending. You'd add a "room password" field to the app, and only phones with the matching password can read messages. This demonstrates security knowledge.

### Feature B: Live Proximity Radar
Build a visual that shows nearby peers as dots on a circle, closer to the center = stronger signal (higher RSSI). This demonstrates real-time data visualization skills.

### Feature C: AI Chat Summarizer
Add a "Summarize" button that takes the last 20 messages and sends them to the Claude API, returning a brief TL;DR. This is a direct callback to your PostPilot and LangGraph work, showing AI integration across platforms.

---

## Summary: How All the Pieces Connect

```
User types a message
        ↓
ChatScreen calls chat.sendMessage()
        ↓
ChatProvider calls _ble.sendMessage()
        ↓
BleService creates ChatMessage object
  → adds to local messages list
  → marks ID as seen
  → serializes to JSON bytes
  → writes bytes to all connected peer devices via GATT
        ↓
Other phone's BleService receives notification on MESSAGE_CHAR_UUID
  → deserializes bytes to ChatMessage
  → checks seenMessageIds (dedup)
  → adds to messages list
  → emits on messageStream
  → decrements TTL and relays to other peers
        ↓
ChatProvider hears messageStream
  → calls notifyListeners()
        ↓
ChatScreen rebuilds with new message visible
```

That's the full loop. Every message you send goes through this exact path.
