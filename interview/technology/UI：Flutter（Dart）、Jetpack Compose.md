# UI：Flutter（Dart）、Jetpack Compose

## flutter

### flutter的StatefulWidget和StatelessWidget区别是什么？怎么选择？
StatelessWidget（无状态组件）：界面一旦渲染，就再也不会变
StatefulWidget（有状态组件）：界面会根据数据变化自动刷新

需要自己思考什么是不可变的，其实StatelessWidget有时候拓展性太弱了，
比如说我开发微信demo，现在如果发送消息气泡，不会变这里是合理的。
但是过两天产品说消息气泡要读了之后显示已读，并且消息要可让用户编辑，
那就只能无奈重构StatefulWidget

虽然是否已读能根据父级传递`isReed`来改变，但是要让用户编辑消息内容的话，
就会出现需要重新绘制气泡大小，这个是`StatelessWidget`无法实现的。
它只能切换，不能重绘。

真的是牛魔的这个玩意

### 怎么实现一个flutter自定义view？自定义列表呢？列表的item不同呢？

自定义view
```dart
import 'package:flutter/material.dart';

// 你要的：可编辑 + 已读状态 → 只有 Stateful
class MessageBubble extends StatefulWidget {
  final String initialText;
  final bool isMe;
  final bool isRead; // 已读状态

  const MessageBubble({
    super.key,
    required this.initialText,
    required this.isMe,
    required this.isRead,
  });

  @override
  State<MessageBubble> createState() => _MessageBubbleState();
}

class _MessageBubbleState extends State<MessageBubble> {
  late TextEditingController _controller;

  @override
  void initState() {
    super.initState();
    // 初始化编辑框文字
    _controller = TextEditingController(text: widget.initialText);
  }

  @override
  Widget build(BuildContext context) {
    return Align(
      alignment: widget.isMe ? Alignment.centerRight : Alignment.centerLeft,
      child: Container(
        margin: const EdgeInsets.symmetric(vertical: 6, horizontal: 12),
        padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 12),
        decoration: BoxDecoration(
          color: widget.isMe ? Colors.green[100] : Colors.grey[200],
          borderRadius: BorderRadius.circular(18),
        ),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.end,
          children: [
            // 可编辑文本（自动换行、自动调整气泡大小）
            TextField(
              controller: _controller,
              maxLines: null,
              style: const TextStyle(fontSize: 16),
              decoration: const InputDecoration(
                isDense: true,
                border: InputBorder.none,
              ),
              onChanged: (value) {
                setState(() {}); // 实时刷新气泡大小
              },
            ),

            const SizedBox(height: 4),

            // 已读 / 未读 状态（只有自己发的才显示）
            if (widget.isMe)
              Text(
                widget.isRead ? "已读" : "未读",
                style: const TextStyle(
                  fontSize: 11,
                  color: Colors.grey,
                ),
              ),
          ],
        ),
      ),
    );
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }
}
```
列表
```dart
ListView.builder(
  itemCount: 10,
  itemBuilder: (context, index) {
    return MessageBubble(
      initialText: "这是可编辑消息 $index",
      isMe: index % 2 == 0,
      isRead: true, // 控制已读未读
    );
  },
)
```

### Dart 的单线程模型是如何运行的？

* 主线程（一个人干活）
* MicroTask 队列（微任务队列，超级优先）
* Event 队列（事件队列，普通优先）

微队列（MicroTask）
 Future.microtask(() {})
 优先级最高，会插队

事件队列（Event）
 Future(() {})
 网络请求
 定时器 Timer
 点击、滑动等 UI 事件
 图片 IO
 所有 “异步” 操作

Dart其实就是类似Android的`HandlerThread`

#### 那 Dart 异步（async/await）是什么？

就是 Handler.post(runnable)
完全一样：把任务放进队列，等主线程空闲再执行。

```dart
// Dart
Future(() {
  print("异步任务");
});

// 等价 Android
handler.post(new Runnable() {
  @Override
  public void run() {
    System.out.println("异步任务");
  }
});
```

### 讲讲await for 与 stream流？Stream 与 Future是什么关系？Stream 有哪两种订阅模式？分别是怎么调用的？

#### Stream 与 Future 是什么关系？

```text
Future = 只会返回 1 次数据（异步单值）
1 个	网络请求、延时、一次结果	    async/await
Stream = 会返回 N 次数据（异步多值）
多个	    聊天消息、进度条、位置、事件流	await for、listen()
```

Stream = 一条管道，源源不断地流出数据

#### 什么是 await for？

监听 Stream，每来一条数据就执行一次

while + 等待 + 接收数据

```dart
// 创建一个流：每秒发射一个数字
Stream<int> counterStream() async* {
  for (int i = 1; i <= 5; i++) {
    await Future.delayed(Duration(seconds: 1));
    yield i; // 流出一个数据
  }
}

// await for 监听流
void listenStream() async {
  await for (int value in counterStream()) {
    print("收到：$value"); // 每秒执行一次
  }
  print("流关闭");
}
```

#### Stream 两种订阅模式

1) Single-subscription（单订阅）
   默认模式，一个 Stream 只能被监听 1 次
   ```dart
    // 1. 创建单订阅流
    final stream = Stream.fromIterable([1,2,3]);
    
    // 2. 监听一次 ✅
    stream.listen((data) => print(data));
    
    // 3. 再次监听 ❌ 报错！
    // stream.listen(...) → 崩溃
   ```
2) Broadcast（广播模式）
   可以被多人监听，N 个 listen () 都可以
   ```dart
    // 1. 创建广播流
    final broadcastStream = StreamController<int>.broadcast().stream;
    
    // 或者把普通流转成广播
    // final broadcast = stream.asBroadcastStream();
    
    // 2. 监听 1 ✅
    broadcastStream.listen((data) => print("监听1：$data"));
    
    // 3. 监听 2 ✅
    broadcastStream.listen((data) => print("监听2：$data"));
   ```    

#### AI开发中的使用

先定义消息模型：
```dart
class ChatMessage {
  final String content;
  final bool isMe;
  final bool isLoading; // 是否正在流式输出

  ChatMessage({
    required this.content,
    required this.isMe,
    this.isLoading = false,
  });
}
```

网络服务层（核心：Future + Stream + SSE）

```dart
import 'dart:async';
import 'dart:convert';
import 'package:http/http.dart' as http;

class ChatService {
  // ------------------------------
  // 1. Future：一次性加载历史消息
  // ------------------------------
  Future<List<ChatMessage>> getHistoryMessages() async {
    await Future.delayed(const Duration(seconds: 1)); // 模拟网络请求

    return [
      ChatMessage(content: "你好！", isMe: true),
      ChatMessage(content: "我是AI助手", isMe: false),
    ];
  }

  // ------------------------------
  // 2. Stream + SSE：流式获取AI回复
  // ------------------------------
  Stream<String> sendMessageStream(String question) async* {
    final uri = Uri.parse("http://你的SSE接口地址:8080/ai/chat");
    
    final request = http.Request("GET", uri)
      ..headers["Accept"] = "text/event-stream";

    final response = await request.send();

    // 监听SSE流（逐字返回）
    await for (final chunk in response.stream.transform(utf8.decoder)) {
      yield chunk; // 不断吐出文字
    }
  }
}
```

### Flutter中的Widget、State、Context 的核心概念？是为了解决什么问题？

* Widget = 界面配置（蓝图）
* State = 界面数据（状态）
* Context = 组件位置 / 环境（上下文）

### Widget 唯一标识Key是做什么用的？

用来确定viewTree的位置，如果没有 Key → 消息头像、文字、已读状态可能错乱

```dart
ListView.builder(
  itemCount: messages.length,
  itemBuilder: (context, index) {
    final msg = messages[index];
    return MessageBubble(
      // 👇 这就是给消息一个唯一ID
      key: ValueKey(msg.id), // 消息ID 作为唯一标识
      text: msg.text,
      isMe: msg.isMe,
    );
  },
)
```









