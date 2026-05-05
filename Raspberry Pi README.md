# Building MeshChat: A Complete Guide
### A Bluetooth Mesh Chat App for Android, iPhone & Raspberry Pi — Explained from Zero

---

## Before We Write Any Code: Understanding What We're Building

### What is this app?

You're building a system where phones **and** Raspberry Pi nodes can send text messages to each other using only Bluetooth — no Wi-Fi, no mobile data, no internet, no server. Think of it like walkie-talkies, but for text, and with dedicated relay stations you can place around a building or outdoor space.

### The two parts of this project

**Part A — The Flutter App** runs on Android and iPhone. It's the human interface: you type messages, read messages, and see who's nearby.

**Part B — The Raspberry Pi Node** runs a Python relay service. It has no screen of its own (though you can add one). Its job is to sit in a fixed location, stay always-on, and relay messages between phones that are too far apart to reach each other directly.

### Why is this impressive?

Most apps work like this:

```
Your Phone → Internet → Company Server → Internet → Their Phone
```

What you're building works like this:

```
Your Phone → Bluetooth → Pi Node → Bluetooth → Their Phone
```

No internet. No server. No middleman. The phones and Pi nodes form a self-organizing wireless network through the air.

---

## The Full System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                       MeshChat Network                          │
│                                                                 │
│   📱 Phone A          🖥 Pi Node 1          📱 Phone B          │
│   (Flutter App)  ←BLE→  (Python)  ←BLE→  (Flutter App)        │
│                              ↕ BLE                              │
│                         🖥 Pi Node 2                            │
│                          (Python)                               │
│                              ↕ BLE                              │
│                          📱 Phone C                             │
│                         (Flutter App)                           │
└─────────────────────────────────────────────────────────────────┘
```

Pi nodes solve a fundamental problem: phones — especially iPhones — can't reliably advertise BLE and scan at the same time, and they stop doing both when backgrounded. Pi nodes are always on, always advertising, always scanning, and always relaying. They're the backbone of the mesh.

---

## Part 1: Understanding the Technology

### What is Bluetooth Low Energy (BLE)?

Regular Bluetooth (the kind that connects headphones) is designed for continuous audio streaming — it needs a lot of power and a permanent connection.

BLE (Bluetooth Low Energy) was invented for devices that need to send small bursts of data occasionally, like fitness trackers reporting your heart rate every few seconds. It uses much less battery.

BLE has two roles that matter to us:

**Peripheral role:** A device advertises a signal into the air saying "I exist, connect to me." Like a person standing in a crowd with a sign showing their name. Anyone nearby scanning for BLE devices can see it.

**Central role:** A device scans for peripherals and connects to them. Once connected, it can read data from the peripheral and subscribe to notifications (automatic pushes of new data).

**The key challenge:** phones — particularly iPhones — cannot reliably play both roles at the same time. Raspberry Pi nodes running BlueZ (Linux's Bluetooth stack) can, which is why they make ideal mesh relay nodes.

### What is GATT?

GATT stands for Generic Attribute Profile. It's the standardized way for two BLE devices to share data once connected.

Think of it like a filing cabinet:

- The cabinet is called a **Service** (identified by a UUID — a unique ID number)
- Each drawer is called a **Characteristic**
- One drawer holds messages you receive
- Another drawer holds the device's display name

When Phone A connects to a Pi Node, Phone A can read the Pi's "name" drawer and subscribe to the Pi's "messages" drawer to get notified when new messages arrive.

### What is a UUID?

UUID stands for Universally Unique Identifier. It looks like this:

```
12345678-1234-1234-1234-123456789abc
```

Your app uses a specific UUID as its "handshake." When devices scan for BLE peers, they filter by this UUID — so they find only other MeshChat nodes (phones and Pi nodes), ignoring headphones, fitness trackers, and everything else.

### What is a Mesh Network?

In a normal Bluetooth connection, A talks to B directly:

```
A ←→ B
```

In a mesh network, every device can relay messages for others:

```
Phone A ←→ Pi Node ←→ Phone B
```

If Phone A can't reach Phone B directly (too far away, or walls in the way), the Pi Node in the middle relays the message. This extends range dramatically.

**TTL (Time To Live)** prevents messages from bouncing forever. Each message starts with TTL = 7. Every time a device relays it, TTL drops by 1. When TTL hits 0, the device discards it instead of relaying.

**Message IDs** prevent loops. Every message has a unique ID. Devices keep a list of IDs they've already relayed — if they see the same ID again, they silently ignore it.

### What is Flutter?

Flutter is a framework made by Google that lets you write one codebase and deploy it to both Android and iPhone. You write in a language called Dart, and Flutter compiles it to native code for both platforms.

### What is BlueZ?

BlueZ is the official Bluetooth stack for Linux, and it's built into Raspberry Pi OS. Python can access it through two libraries: `bleak` (central/scanning role) and `bless` (peripheral/advertising role). Together they allow the Pi to do what phones can't: run both roles simultaneously.

---

## Part 2: Project Setup — Flutter App

### Step 1 — Install Flutter

**On Windows:**

1. Go to https://flutter.dev/docs/get-started/install/windows
2. Download and extract the Flutter SDK to `C:\flutter`
3. Add `C:\flutter\bin` to your system PATH (search "environment variables" in Windows Start)
4. Open a new terminal and run:

```bash
flutter doctor
```

Fix anything with an ✗. The important ones are Flutter itself and the Android toolchain.

**Install Android Studio:**
- Download from https://developer.android.com/studio
- During installation, make sure to install the Android SDK

**Install VS Code (recommended editor):**
- Download from https://code.visualstudio.com
- Install the "Flutter" and "Dart" extensions from the Extensions panel

### Step 2 — Create the Flutter Project

```bash
flutter create meshchat
cd meshchat
code .
```

Your folder structure:

```
meshchat/
├── android/          ← Android-specific files
├── ios/              ← iOS-specific files
├── lib/              ← Your app code lives here
│   └── main.dart
├── pubspec.yaml      ← Dependency list
└── test/
```

### Step 3 — Add Dependencies

Open `pubspec.yaml` and replace the `dependencies:` section:

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

- `flutter_blue_plus` — lets your app talk to the phone's Bluetooth hardware
- `provider` — connects the BLE data layer to the UI so the screen refreshes automatically when a new message arrives
- `uuid` — generates unique IDs for messages (used for deduplication)
- `shared_preferences` — saves the username to local storage so it persists between app restarts
- `intl` — formats timestamps like "14:35"

Then run:

```bash
flutter pub get
```

---

## Part 3: Platform Permissions — Flutter App

### Step 4 — Android Permissions

Open `android/app/src/main/AndroidManifest.xml` and add inside the `<manifest>` tag, before `<application>`:

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

In `android/app/build.gradle`, set:

```gradle
defaultConfig {
    minSdkVersion 21
}
```

Android 12+ requires separate `BLUETOOTH_SCAN`, `BLUETOOTH_ADVERTISE`, and `BLUETOOTH_CONNECT` permissions. Location permission is required because BLE scanning can theoretically reveal your position via nearby Bluetooth beacons.

### Step 5 — iOS Permissions

Open `ios/Runner/Info.plist` and add inside the `<dict>` tag:

```xml
<key>NSBluetoothAlwaysUsageDescription</key>
<string>Used to find and chat with nearby devices over Bluetooth mesh</string>
<key>NSBluetoothPeripheralUsageDescription</key>
<string>Used to advertise your presence to nearby devices</string>
```

iOS requires a human-readable explanation for every permission. Without these, the app crashes the moment it tries to use Bluetooth.

---

## Part 4: Flutter Data Models

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

**Key design decisions:**

- JSON keys are abbreviated (`'c'` not `'content'`) to stay within BLE's ~512-byte packet limit
- `withDecrementedTTL()` returns `null` at TTL=1 — a clean Dart signal meaning "drop this message, don't relay"
- `senderId` is a UUID generated at app start; `senderName` is the human-readable name. Never use name for identity checks — two users might pick the same name

### Step 7 — The Peer Model

Create `lib/models/peer.dart`:

```dart
import 'package:flutter_blue_plus/flutter_blue_plus.dart';

enum PeerType { phone, piNode, unknown }

class Peer {
  final String deviceId;
  String name;
  PeerType type;
  BluetoothDevice? device;
  int rssi;
  DateTime lastSeen;
  bool isConnected;

  Peer({
    required this.deviceId,
    required this.name,
    this.type = PeerType.unknown,
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

  // Pi nodes advertise names starting with "MeshNode-"
  static PeerType inferType(String name) {
    if (name.startsWith('MeshNode-')) return PeerType.piNode;
    return PeerType.phone;
  }
}
```

The `PeerType` enum lets the UI show a router icon for Pi nodes and a person icon for phones. The naming convention `MeshNode-*` is how the Pi identifies itself — the Flutter app reads the peer's name from `NAME_CHAR_UUID` and checks if it starts with `"MeshNode-"`.

---

## Part 5: The Flutter BLE Service

Create `lib/services/ble_service.dart`:

### Section A — UUIDs and Setup

```dart
import 'dart:async';
import 'dart:convert';
import 'package:flutter_blue_plus/flutter_blue_plus.dart';
import 'package:uuid/uuid.dart';
import '../models/message.dart';
import '../models/peer.dart';

class BleService {
  // These UUIDs must match exactly across the Flutter app and the Pi node.
  // Generate your own at https://www.uuidgenerator.net for production use.
  static const String SERVICE_UUID       = '12345678-1234-1234-1234-123456789abc';
  static const String MESSAGE_CHAR_UUID  = '12345678-1234-1234-1234-123456789abd';
  static const String NAME_CHAR_UUID     = '12345678-1234-1234-1234-123456789abe';

  final String localUserId   = const Uuid().v4();
  String       localUserName = 'User';

  final Map<String, Peer>     _peers          = {};
  final Set<String>           _seenMessageIds = {};
  final List<ChatMessage>     _messages       = [];

  // Cache characteristics so we don't call discoverServices() on every send
  final Map<String, BluetoothCharacteristic> _msgCharCache = {};

  final _messageController = StreamController<ChatMessage>.broadcast();
  final _peerController    = StreamController<Map<String, Peer>>.broadcast();

  Stream<ChatMessage>     get messageStream => _messageController.stream;
  Stream<Map<String, Peer>> get peerStream  => _peerController.stream;
  List<ChatMessage>       get messages      => List.unmodifiable(_messages);
  Map<String, Peer>       get peers         => Map.unmodifiable(_peers);
```

### Section B — Scanning

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

    // Restart scanning every 35 seconds so new devices are discovered continuously
    Future.delayed(const Duration(seconds: 35), startScanning);
  }
```

`withServices: [Guid(SERVICE_UUID)]` tells the Bluetooth hardware to filter scan results — only devices advertising your UUID get reported. This filters out every headphone, fitness tracker, and other BLE device in the vicinity.

### Section C — Connecting to a Discovered Device

```dart
  Future<void> _handleDiscoveredDevice(BluetoothDevice device, int rssi) async {
    final id = device.remoteId.str;

    if (_peers.containsKey(id) && _peers[id]!.isConnected) {
      _peers[id]!.rssi      = rssi;
      _peers[id]!.lastSeen  = DateTime.now();
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
            if (c.characteristicUuid.toString() == MESSAGE_CHAR_UUID) msgChar  = c;
            if (c.characteristicUuid.toString() == NAME_CHAR_UUID)    nameChar = c;
          }

          String peerName = 'Unknown';
          if (nameChar != null) {
            peerName = utf8.decode(await nameChar.read());
          }

          if (msgChar != null) {
            await msgChar.setNotifyValue(true);
            msgChar.lastValueStream.listen((data) {
              if (data.isNotEmpty) _handleIncomingData(data, id);
            });
            // Cache the characteristic — avoids re-discovering on every send
            _msgCharCache[id] = msgChar;
          }

          _peers[id] = Peer(
            deviceId:    id,
            name:        peerName,
            type:        Peer.inferType(peerName),
            device:      device,
            rssi:        rssi,
            isConnected: true,
          );

          _peerController.add(_peers);
          break;
        }
      }
    } catch (e) {
      debugPrint('Connection failed to $id: $e');
    }
  }
```

### Section D — Processing an Incoming Message

```dart
  void _handleIncomingData(List<int> data, String fromDeviceId) {
    try {
      final message = ChatMessage.fromJson(
        jsonDecode(utf8.decode(data)) as Map<String, dynamic>,
      );

      if (_seenMessageIds.contains(message.id)) return;
      _seenMessageIds.add(message.id);

      // Only show messages from others in the UI
      if (message.senderId != localUserId) {
        _messages.add(message);
        _messageController.add(message);
      }

      // Relay with decremented TTL to all other peers (mesh hop)
      final relayed = message.withDecrementedTTL();
      if (relayed != null) {
        _relayMessage(relayed, excludeDeviceId: fromDeviceId);
      }
    } catch (e) {
      debugPrint('Failed to parse incoming message: $e');
    }
  }

  Future<void> _relayMessage(ChatMessage msg, {required String excludeDeviceId}) async {
    final data = utf8.encode(jsonEncode(msg.toJson()));
    for (var entry in _peers.entries) {
      if (entry.key == excludeDeviceId) continue;
      if (!entry.value.isConnected) continue;
      await _writeToDevice(entry.key, data);
    }
  }
```

### Section E — Sending a Message

```dart
  Future<void> sendMessage(String content) async {
    final message = ChatMessage(
      senderId:   localUserId,
      senderName: localUserName,
      content:    content,
    );

    // Pre-mark our own ID so relayed echoes are silently ignored
    _seenMessageIds.add(message.id);
    _messages.add(message);
    _messageController.add(message);

    final data = utf8.encode(jsonEncode(message.toJson()));
    for (var peer in _peers.values) {
      if (!peer.isConnected) continue;
      await _writeToDevice(peer.deviceId, data);
    }
  }

  Future<void> _writeToDevice(String deviceId, List<int> data) async {
    try {
      // Use cached characteristic if available
      final char = _msgCharCache[deviceId];
      if (char == null) return;

      // BLE MTU is typically 512 bytes after negotiation — chunk anything larger
      if (data.length <= 512) {
        await char.write(data, withoutResponse: true);
      } else {
        for (int i = 0; i < data.length; i += 512) {
          final end   = (i + 512 > data.length) ? data.length : i + 512;
          final chunk = data.sublist(i, end);
          await char.write(chunk, withoutResponse: true);
        }
      }
    } catch (e) {
      debugPrint('Write failed to $deviceId: $e');
      _peers[deviceId]?.isConnected = false;
      _msgCharCache.remove(deviceId);
    }
  }

  void dispose() {
    _messageController.close();
    _peerController.close();
  }
}
```

---

## Part 6: Flutter State Management

### Step 8 — Chat Provider

Create `lib/providers/chat_provider.dart`:

```dart
import 'package:flutter/foundation.dart';
import 'package:shared_preferences/shared_preferences.dart';
import '../services/ble_service.dart';
import '../models/message.dart';
import '../models/peer.dart';

class ChatProvider extends ChangeNotifier {
  final BleService _ble = BleService();

  List<ChatMessage>   get messages => _ble.messages;
  Map<String, Peer>   get peers    => _ble.peers;
  String              get userName => _ble.localUserName;

  ChatProvider() { _init(); }

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

`ChangeNotifier` is the bridge between the background BLE layer and the UI. When a message arrives via Bluetooth, `messageStream` fires → `notifyListeners()` is called → every widget watching this provider rebuilds with the new message visible.

---

## Part 7: Flutter UI

### Step 9 — Chat Screen

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
  final TextEditingController _inputController  = TextEditingController();
  final ScrollController      _scrollController = ScrollController();

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

    final phoneCount = chat.peers.values.where((p) => p.type == PeerType.phone).length;
    final nodeCount  = chat.peers.values.where((p) => p.type == PeerType.piNode).length;

    return Scaffold(
      backgroundColor: const Color(0xFF121212),
      appBar: AppBar(
        backgroundColor: const Color(0xFF1E1E1E),
        title: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            const Text('#mesh',
                style: TextStyle(color: Colors.white, fontWeight: FontWeight.bold)),
            Text('$phoneCount phone(s) · $nodeCount node(s)',
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
                      isMe: chat.messages[i].senderId ==
                          context.read<ChatProvider>().messages[i].senderId,
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
              style: TextStyle(color: Colors.white, fontSize: 18, fontWeight: FontWeight.bold)),
          const SizedBox(height: 12),
          if (peers.isEmpty)
            const Text('No peers found yet', style: TextStyle(color: Colors.grey)),
          ...peers.values.map((p) {
            final icon = p.type == PeerType.piNode ? Icons.router : Icons.phone_android;
            final color = p.type == PeerType.piNode ? Colors.blueAccent : Colors.greenAccent;
            return ListTile(
              leading: CircleAvatar(
                  backgroundColor: color,
                  child: Icon(icon, color: Colors.black)),
              title: Text(p.name, style: const TextStyle(color: Colors.white)),
              subtitle: Text(p.proximityLabel, style: TextStyle(color: color)),
              trailing: Text('${p.rssi} dBm',
                  style: const TextStyle(color: Colors.grey, fontSize: 12)),
            );
          }),
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
        title: const Text('Set Username', style: TextStyle(color: Colors.white)),
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
            child: const Text('Save', style: TextStyle(color: Colors.greenAccent)),
          ),
        ],
      ),
    );
  }
}

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
        constraints: BoxConstraints(maxWidth: MediaQuery.of(context).size.width * 0.75),
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
                    color: isMe ? Colors.black : Colors.white, fontSize: 15)),
            const SizedBox(height: 2),
            Text(
              DateFormat('HH:mm').format(message.timestamp),
              style: TextStyle(
                  color: isMe ? Colors.black54 : Colors.grey, fontSize: 10),
            ),
          ],
        ),
      ),
    );
  }
}
```

### Step 10 — main.dart

Replace `lib/main.dart`:

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

---

## Part 8: Flutter File Structure

```
meshchat/lib/
├── main.dart                    ← App entry point
├── models/
│   ├── message.dart             ← ChatMessage data model + serialization
│   └── peer.dart                ← Peer model with PeerType (phone vs Pi node)
├── services/
│   └── ble_service.dart         ← All Bluetooth logic (scan, connect, relay)
├── providers/
│   └── chat_provider.dart       ← ChangeNotifier state bridge
└── screens/
    └── chat_screen.dart         ← Chat UI + MessageBubble widget
```

---

## Part 9: Raspberry Pi Node Setup

The Pi acts as a **dedicated mesh relay node**. It has no chat UI — it simply stays on, scans for nearby devices advertising your `SERVICE_UUID`, relays messages between them, and optionally logs everything to a database and serves a web dashboard.

### Which Pi Model to Use

| Model | Best For | Notes |
|---|---|---|
| **Pi Zero 2W** | Portable/battery node | Tiny (~credit card), ~$15, built-in BLE, runs the relay fine |
| **Pi 4 Model B** | Fixed base station | More power, run relay + Flask dashboard + SQLite simultaneously |
| **Pi 3 Model B+** | Middle ground | Solid choice if you already have one |

For a portable node you can drop anywhere with a USB power bank, the **Zero 2W** is the sweet spot.

### Step 11 — Prepare the Pi

Flash **Raspberry Pi OS Lite** (headless, no desktop needed) using the Raspberry Pi Imager. During flashing, use the gear icon to:
- Set hostname: `meshnode`
- Enable SSH
- Configure Wi-Fi (so you can SSH in wirelessly)

Boot the Pi, SSH in, then update and install dependencies:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y python3-pip python3-venv bluetooth bluez

python3 -m venv ~/meshenv
source ~/meshenv/bin/activate

pip install bleak bless aiohttp
```

`bleak` handles the central role (scanning + connecting to peers).
`bless` handles the peripheral role (advertising + hosting a GATT server).
`aiohttp` is for the optional web dashboard.

### Step 12 — Configure BlueZ

BlueZ sometimes needs coaxing to allow Python to run a GATT server. Run these once:

```bash
sudo systemctl stop bluetooth
sudo hciconfig hci0 up
sudo bluetoothctl power on
```

To make Bluetooth come up automatically at boot, add to `/etc/rc.local` (before `exit 0`):

```bash
hciconfig hci0 up
```

### Step 13 — Create the Project Structure

```bash
mkdir ~/meshchat-node && cd ~/meshchat-node
touch relay.py db.py dashboard.py
```

### Step 14 — The Relay Script

Create `relay.py`:

```python
import asyncio
import json
import logging
from datetime import datetime
from bleak import BleakScanner, BleakClient
from bless import BlessServer, BlessGATTCharacteristic, GATTCharacteristicProperties, GATTAttributePermissions

# ── Configuration ──────────────────────────────────────────────────────────────
# These UUIDs must match the Flutter app exactly
SERVICE_UUID      = "12345678-1234-1234-1234-123456789abc"
MESSAGE_CHAR_UUID = "12345678-1234-1234-1234-123456789abd"
NAME_CHAR_UUID    = "12345678-1234-1234-1234-123456789abe"

NODE_NAME         = "MeshNode-Pi"   # Must start with "MeshNode-" so Flutter recognises it
SCAN_INTERVAL_S   = 15              # Re-scan every N seconds
TTL_DEFAULT       = 7

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s  %(levelname)s  %(message)s",
    datefmt="%H:%M:%S",
)
log = logging.getLogger("meshnode")

# ── Shared state ───────────────────────────────────────────────────────────────
seen_ids:          set[str]              = set()
connected_clients: dict[str, BleakClient] = {}
message_log:       list[dict]            = []          # in-memory; see db.py for persistence
gatt_server:       BlessServer | None    = None

# ── Message handling ───────────────────────────────────────────────────────────

def handle_incoming(data: bytes, from_device_id: str | None = None) -> None:
    """Parse, deduplicate, log, and relay a received message."""
    try:
        msg = json.loads(data.decode("utf-8"))
    except (json.JSONDecodeError, UnicodeDecodeError):
        log.warning("Received unreadable data — ignoring")
        return

    msg_id = msg.get("id")
    if not msg_id or msg_id in seen_ids:
        return

    seen_ids.add(msg_id)

    sender = msg.get("sn", "?")
    content = msg.get("c", "")
    log.info(f"[{sender}]: {content}")

    # Store in memory (swap out for db.save_message(msg) if using SQLite)
    message_log.append({**msg, "received_at": datetime.utcnow().isoformat()})

    # Relay with decremented TTL
    ttl = msg.get("ttl", 1)
    if ttl > 1:
        msg["ttl"] = ttl - 1
        relayed_bytes = json.dumps(msg).encode("utf-8")
        asyncio.create_task(relay_to_all(relayed_bytes, exclude=from_device_id))

async def relay_to_all(data: bytes, exclude: str | None) -> None:
    """Write data to every connected peer except the one it came from."""
    for device_id, client in list(connected_clients.items()):
        if device_id == exclude:
            continue
        try:
            await client.write_gatt_char(MESSAGE_CHAR_UUID, data, response=False)
        except Exception as e:
            log.warning(f"Relay to {device_id} failed: {e}")
            connected_clients.pop(device_id, None)

# ── GATT Server (peripheral role) ─────────────────────────────────────────────

async def start_gatt_server() -> BlessServer:
    """Advertise as a MeshChat node so phones can connect to us."""
    server = BlessServer(name=NODE_NAME)
    await server.add_new_service(SERVICE_UUID)

    # Message characteristic — phones write here to send us messages,
    # and we notify phones here when we have a message for them
    await server.add_new_characteristic(
        SERVICE_UUID,
        MESSAGE_CHAR_UUID,
        properties=(
            GATTCharacteristicProperties.write_without_response |
            GATTCharacteristicProperties.notify
        ),
        permissions=(
            GATTAttributePermissions.readable |
            GATTAttributePermissions.writeable
        ),
        value=None,
    )

    # Name characteristic — phones read this to learn our display name
    await server.add_new_characteristic(
        SERVICE_UUID,
        NAME_CHAR_UUID,
        properties=GATTCharacteristicProperties.read,
        permissions=GATTAttributePermissions.readable,
        value=NODE_NAME.encode("utf-8"),
    )

    def on_write(characteristic: BlessGATTCharacteristic, value: bytearray, **kwargs):
        handle_incoming(bytes(value), from_device_id=None)

    server.write_request_func = on_write
    await server.start()
    log.info(f"GATT server running — advertising as '{NODE_NAME}'")
    return server

# ── BLE Scanner (central role) ─────────────────────────────────────────────────

async def scanning_loop() -> None:
    """Continuously scan for new MeshChat peers and connect to them."""
    log.info("Starting BLE scan loop...")
    while True:
        try:
            devices = await BleakScanner.discover(
                timeout=10.0,
                service_uuids=[SERVICE_UUID],
            )
            for device in devices:
                if device.address not in connected_clients:
                    asyncio.create_task(connect_to_peer(device))
        except Exception as e:
            log.error(f"Scan error: {e}")

        await asyncio.sleep(SCAN_INTERVAL_S)

async def connect_to_peer(device) -> None:
    """Connect to a discovered peer, read its name, and subscribe to messages."""
    device_id = device.address
    try:
        client = BleakClient(device_id, timeout=10.0)
        await client.connect()

        if not client.is_connected:
            return

        # Read the peer's display name
        try:
            name_bytes = await client.read_gatt_char(NAME_CHAR_UUID)
            peer_name  = name_bytes.decode("utf-8")
        except Exception:
            peer_name = device.name or device_id

        log.info(f"Connected to: {peer_name} ({device_id})")
        connected_clients[device_id] = client

        # Subscribe to message notifications from this peer
        await client.start_notify(
            MESSAGE_CHAR_UUID,
            lambda _, data: handle_incoming(data, from_device_id=device_id),
        )

        # Handle disconnect — remove from connected set
        def on_disconnect(_):
            log.info(f"Disconnected from {peer_name} ({device_id})")
            connected_clients.pop(device_id, None)

        client.set_disconnected_callback(on_disconnect)

    except Exception as e:
        log.warning(f"Could not connect to {device_id}: {e}")
        connected_clients.pop(device_id, None)

# ── Entry point ────────────────────────────────────────────────────────────────

async def main() -> None:
    global gatt_server
    gatt_server = await start_gatt_server()
    await scanning_loop()   # runs forever

if __name__ == "__main__":
    asyncio.run(main())
```

Run it with:

```bash
source ~/meshenv/bin/activate
sudo python3 relay.py
```

You should see output like:

```
10:24:01  INFO  GATT server running — advertising as 'MeshNode-Pi'
10:24:01  INFO  Starting BLE scan loop...
10:24:13  INFO  Connected to: Harsh (AA:BB:CC:DD:EE:FF)
10:24:28  INFO  [Harsh]: Hello from across the room!
```

### Step 15 — Optional: SQLite Message Persistence

Create `db.py`:

```python
import sqlite3
import json
from pathlib import Path

DB_PATH = Path.home() / "meshchat-node" / "messages.db"

def init_db() -> None:
    with sqlite3.connect(DB_PATH) as conn:
        conn.execute("""
            CREATE TABLE IF NOT EXISTS messages (
                id          TEXT PRIMARY KEY,
                sender_id   TEXT,
                sender_name TEXT,
                content     TEXT,
                timestamp   INTEGER,
                received_at TEXT
            )
        """)

def save_message(msg: dict) -> None:
    with sqlite3.connect(DB_PATH) as conn:
        conn.execute(
            "INSERT OR IGNORE INTO messages VALUES (?,?,?,?,?,datetime('now'))",
            (msg["id"], msg["sid"], msg["sn"], msg["c"], msg["ts"]),
        )

def get_recent(limit: int = 50) -> list[dict]:
    with sqlite3.connect(DB_PATH) as conn:
        rows = conn.execute(
            "SELECT sender_name, content, timestamp FROM messages "
            "ORDER BY timestamp DESC LIMIT ?", (limit,)
        ).fetchall()
    return [{"sn": r[0], "c": r[1], "ts": r[2]} for r in rows]
```

In `relay.py`, replace the `message_log.append(...)` line with:

```python
from db import init_db, save_message
# at top of main(): init_db()
# in handle_incoming(): save_message(msg)
```

### Step 16 — Optional: Web Dashboard

Create `dashboard.py`:

```python
from aiohttp import web
from db import get_recent
import json

async def handle_index(request):
    messages = get_recent(50)
    rows = "".join(
        f"<tr><td>{m['sn']}</td><td>{m['c']}</td></tr>"
        for m in reversed(messages)
    )
    html = f"""
    <!DOCTYPE html>
    <html>
    <head>
      <title>MeshChat Node Dashboard</title>
      <meta http-equiv="refresh" content="5">
      <style>
        body  {{ font-family: monospace; background: #121212; color: #eee; padding: 2em; }}
        h1    {{ color: #00ff88; }}
        table {{ border-collapse: collapse; width: 100%; }}
        th, td {{ border: 1px solid #333; padding: 8px 12px; text-align: left; }}
        th    {{ background: #1e1e1e; color: #00ff88; }}
        tr:nth-child(even) {{ background: #1a1a1a; }}
      </style>
    </head>
    <body>
      <h1>🔗 MeshNode-Pi — Live Feed</h1>
      <table>
        <tr><th>Sender</th><th>Message</th></tr>
        {rows}
      </table>
      <p style="color:#555">Auto-refreshes every 5 seconds</p>
    </body>
    </html>
    """
    return web.Response(text=html, content_type="text/html")

async def start_dashboard(host="0.0.0.0", port=5000):
    app = web.Application()
    app.router.add_get("/", handle_index)
    runner = web.AppRunner(app)
    await runner.setup()
    site = web.TCPSite(runner, host, port)
    await site.start()
    print(f"Dashboard available at http://meshnode.local:{port}")
```

In `relay.py`'s `main()`, add:

```python
from dashboard import start_dashboard
# inside async def main():
await start_dashboard()
```

Access it from any browser on the same network: `http://meshnode.local:5000`

### Step 17 — Auto-Start on Boot (systemd)

So the relay starts automatically every time the Pi powers on:

```bash
sudo nano /etc/systemd/system/meshchat.service
```

Paste:

```ini
[Unit]
Description=MeshChat Relay Node
After=bluetooth.target network.target

[Service]
Type=simple
User=pi
WorkingDirectory=/home/pi/meshchat-node
ExecStartPre=/usr/bin/hciconfig hci0 up
ExecStart=/home/pi/meshenv/bin/python3 /home/pi/meshchat-node/relay.py
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Enable and start it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable meshchat
sudo systemctl start meshchat

# Check it's running
sudo systemctl status meshchat

# Watch live logs
journalctl -u meshchat -f
```

---

## Part 10: Pi Node File Structure

```
meshchat-node/
├── relay.py        ← Main relay loop (GATT server + BLE scanner + message handling)
├── db.py           ← SQLite persistence (optional)
├── dashboard.py    ← Web dashboard via aiohttp (optional)
└── messages.db     ← Auto-created by db.py
```

---

## Part 11: Running and Testing

### Step 18 — Run the Flutter App on Android

1. Enable Developer Options: Settings → About Phone → tap "Build Number" 7 times
2. Enable USB Debugging in Developer Options
3. Connect phone to computer via USB
4. Run:

```bash
flutter devices   # confirm your phone is listed
flutter run
```

First build takes 3–5 minutes. Subsequent builds are faster.

### Step 19 — Test the Full System

You need: 1× Raspberry Pi running `relay.py` + 2× phones running the Flutter app.

1. Power on the Pi and confirm the relay is running (`sudo systemctl status meshchat`)
2. Open the Flutter app on both phones — wait 15–30 seconds for discovery
3. Check the app bar — it should show "1 node(s)" once the Pi is discovered
4. Move Phone B far enough away that it can't see Phone A in the peers list
5. Send a message from Phone A — it should arrive on Phone B, relayed through the Pi

To verify the relay is working, watch the Pi's logs:

```bash
journalctl -u meshchat -f
```

You should see the message appear in the log as it passes through.

### Common Problems and Fixes

**Pi doesn't appear in the phone's peer list:**
- Run `sudo hciconfig hci0 up` on the Pi and restart the service
- Make sure `SERVICE_UUID` in `relay.py` matches `ble_service.dart` exactly — even one character off breaks discovery

**"bleak not found" error:**
- Make sure you activated the virtual environment: `source ~/meshenv/bin/activate`

**Messages arrive on Pi logs but not on the second phone:**
- Check that the second phone has the app in the foreground (iOS restricts background BLE)
- Verify both phones show the Pi as a connected peer in the app's peer sheet

**Build fails on Flutter:**
- Run `flutter clean && flutter pub get && flutter run`
- Confirm `minSdkVersion 21` is set in `android/app/build.gradle`

---

## Part 12: Extending the Project

### Extension A: Physical Button on the Pi (GPIO)

Connect a tactile button to GPIO pin 17 and GND. When pressed, the Pi broadcasts a preset message (e.g., "🔴 Emergency — converge on this node") into the mesh:

```python
from gpiozero import Button
from signal import pause

sos_button = Button(17)

def send_sos():
    msg = ChatMessage(sender_id="pi-node", sender_name=NODE_NAME, content="🔴 SOS — Emergency")
    asyncio.create_task(relay_to_all(json.dumps(msg.to_dict()).encode(), exclude=None))

sos_button.when_pressed = send_sos
```

### Extension B: OLED Display (I2C)

Attach an SSD1306 OLED (128×64, I2C) and show the last received message in real time. Install `luma.oled` and call `display_message(sender, content)` inside `handle_incoming()`:

```python
from luma.core.interface.serial import i2c
from luma.oled.device import ssd1306
from luma.core.render import canvas
from PIL import ImageFont

serial = i2c(port=1, address=0x3C)
device = ssd1306(serial)

def display_message(sender: str, content: str):
    with canvas(device) as draw:
        draw.text((0, 0),  sender,  fill="white")
        draw.text((0, 20), content, fill="white")
```

### Extension C: AI Message Summarizer

Add a `!summary` command. When the Pi sees it in the mesh, it sends the last 20 messages to the Claude API and broadcasts the summary back:

```python
import anthropic

client = anthropic.Anthropic()  # uses ANTHROPIC_API_KEY env var

async def ai_summary():
    recent = "\n".join(f"{m['sn']}: {m['c']}" for m in message_log[-20:])
    response = client.messages.create(
        model="claude-opus-4-6",
        max_tokens=200,
        messages=[{"role": "user", "content": f"Summarize this chat in 2 sentences:\n{recent}"}],
    )
    summary = response.content[0].text
    summary_msg = build_message(sender_name=NODE_NAME, content=f"📋 Summary: {summary}")
    await relay_to_all(json.dumps(summary_msg).encode(), exclude=None)
```

This ties directly into your PostPilot and LangGraph work — AI integration across platforms.

---

## Part 13: How All the Pieces Connect

```
User types a message on Phone A
            ↓
    ChatScreen calls chat.sendMessage()
            ↓
    ChatProvider calls _ble.sendMessage()
            ↓
    BleService serializes to JSON bytes
    → marks ID as seen (prevents echo)
    → writes bytes to all connected peers via GATT
            ↓
┌───────────────────────────────────────┐
│  Raspberry Pi Node receives bytes     │
│  → deserializes JSON                  │
│  → deduplication check                │
│  → logs message (terminal + SQLite)   │
│  → decrements TTL                     │
│  → relays to all other peers          │
└───────────────────────────────────────┘
            ↓
    Phone B's BleService receives bytes
    → deserializes to ChatMessage
    → deduplication check
    → adds to messages list
    → emits on messageStream
            ↓
    ChatProvider calls notifyListeners()
            ↓
    ChatScreen rebuilds — new message visible
```

The Pi node is transparent to the Flutter app — phones treat it like any other peer. The only visible difference is the router icon in the peers sheet and the `MeshNode-*` name prefix.

---

## Final Project Structure

```
project-root/
│
├── meshchat/                   ← Flutter app (Android + iPhone)
│   ├── android/
│   ├── ios/
│   └── lib/
│       ├── main.dart
│       ├── models/
│       │   ├── message.dart
│       │   └── peer.dart
│       ├── services/
│       │   └── ble_service.dart
│       ├── providers/
│       │   └── chat_provider.dart
│       └── screens/
│           └── chat_screen.dart
│
└── meshchat-node/              ← Raspberry Pi relay node (Python)
    ├── relay.py                ← Core relay loop
    ├── db.py                   ← SQLite persistence (optional)
    ├── dashboard.py            ← Web dashboard (optional)
    └── messages.db             ← Auto-created
```
