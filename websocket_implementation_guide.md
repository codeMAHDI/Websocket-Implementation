# Flutter WebSocket Implementation Guide

This guide provides a standardized template for implementing real-time messaging using WebSockets in Flutter, based on the implementation used in this project.

## 1. Prerequisites (Setup)

Add the necessary dependencies to your [pubspec.yaml](file:///c:/flutter_project/property_management/pubspec.yaml):
```yaml
dependencies:
  web_socket_channel: ^3.0.1
  jwt_decoder: ^2.0.1  # Highly recommended for robust ID extraction
  flutter_spinkit: ^5.2.1 # For typing indicators
```

## 2. Global Identity (The most critical step)

WebSocket usually requires the `senderId`. Make sure you save the `userId` during the **Login** process.

**In your Auth Controller:**
```dart
// Inside login success logic
var user = response.data['user'];
await SharePrefsHelper.setString('userId', (user['id'] ?? user['_id']) ?? "");
```

---

## 3. Controller Template ([IndividualChatController](file:///c:/flutter_project/property_management/lib/view/screen/common/message/controller/individual_chat_controller.dart#17-320))

### A. Variables & Lifecycle
```dart
class ChatController extends GetxController {
  WebSocketChannel? _channel;
  StreamSubscription? _socketSubscription;
  RxBool isOtherTyping = false.obs;
  Timer? _typingTimer;
  String? myId;
  String? socketChannelName; // Usually passed from the previous screen

  @override
  void onClose() {
    _socketSubscription?.cancel();
    _channel?.sink.close();
    _typingTimer?.cancel();
    super.onClose();
  }
}
```

### B. Initialization
```dart
void initChat(String channelId, String cName) async {
  socketChannelName = cName;
  
  // 1. Get My ID (Robust way)
  myId = await SharePrefsHelper.getString('userId');
  if (myId == null || myId!.isEmpty) {
    String token = await SharePrefsHelper.getString('token');
    if (token.isNotEmpty) {
      myId = JwtDecoder.decode(token)['id']; // Fallback
    }
  }

  // 2. Connect
  _connectWebSocket();
}
```

### C. Connection & Event Handling
```dart
void _connectWebSocket() {
  final String url = "wss://your-socket-url.com";
  _channel = IOWebSocketChannel.connect(Uri.parse(url));

  // 1. Subscribe to Channel
  _channel?.sink.add(jsonEncode({
    "type": "subscribe",
    "channelName": socketChannelName
  }));

  // 2. Listen for Stream
  _socketSubscription = _channel?.stream.listen((event) {
    final data = jsonDecode(event);

    switch (data['type']) {
      case 'message':
        var newMessage = Model.fromJson(data['data']);
        if (!messages.any((m) => m.id == newMessage.id)) {
          messages.insert(0, newMessage);
        }
        break;
      case 'typing':
        if (data['senderId'] != myId) {
          isOtherTyping.value = data['isTyping'] ?? false;
        }
        break;
    }
  });
}
```

### D. Hybrid Messaging (WebSocket vs API)
Always send **Just Text** via WebSocket for speed, but use **API** for Images.

```dart
Future<void> sendMessage() async {
  String text = controller.text.trim();
  
  if (selectedImages.isEmpty) {
    // TEXT ONLY -> WebSocket
    _channel?.sink.add(jsonEncode({
      "type": "message",
      "channelName": socketChannelName,
      "senderId": myId,
      "message": text
    }));
    controller.clear();
  } else {
    // IMAGES -> REST API
    await sendViaApi(text, selectedImages);
  }
}
```

---

## 4. UI Implementation ([IndividualChatScreen](file:///c:/flutter_project/property_management/lib/view/screen/common/message/view/individual_chat_screen.dart#16-402))

### Typing Indicator
Place this above your Input Field:
```dart
Obx(() {
  if (!controller.isOtherTyping.value) return SizedBox();
  return Row(
    children: [
      SpinKitThreeBounce(color: Colors.blue, size: 12),
      Text(" User is typing..."),
    ],
  );
})
```

### Real-time Trigger
Update the `TextField` to notify the other user:
```dart
TextField(
  onChanged: (value) => controller.sendTyping(true),
  // ...
)
```

## Summary Checklist
- [ ] `userId` saved in SharedPrefs?
- [ ] `web_socket_channel` connected?
- [ ] `subscribe` event sent?
- [ ] [onClose](file:///c:/flutter_project/property_management/lib/view/screen/common/message/controller/individual_chat_controller.dart#53-61) cancels subscription?
- [ ] `reverse: true` set in ListView? (Messages flow from bottom)
