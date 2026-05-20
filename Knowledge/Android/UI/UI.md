# UI


## View的核心理论

### WMS（WindowManagerService）
1.  渲染与窗口管理原理
    - 解释 Flutter / KMP Compose 渲染引擎的工作机制
    - 视图窗口绘制流程（对应 Android WMS 窗口管理）
    - 视图层级管理（View 叠加逻辑，对应 XML FrameLayout 层级，说明 Z-index 实现）
    - 屏幕适配方案（对应 Android dp/px 转换、像素密度适配）

2.  触摸事件分发机制
    - 点击、长按、滑动事件的派发流程（DOWN/MOVE/UP）
    - 事件拦截与上层消费逻辑（对应 Android onInterceptTouchEvent / onTouchEvent）
    - 多视图重叠时的事件优先级处理

3.  视图叠加机制
    - 多视图重叠时的绘制顺序、遮挡规则
    - 对应 Android FrameLayout 的层级管理逻辑


## UI Demo
仿制Wechat， UI包括：UI 渲染、布局、事件、动画、列表、页面栈。
所有 UI 需与 Android XML 原理一一对应，适配 KMP (Compose) / Flutter 开发学习。


## 项目目标


这个demo点开之后是三个page：消息，通讯录。**（1. Navigation和ViewPager，LinearLayout，ScrollerView）**

消息页面的消息Item要跟微信一样，包括头像，名称，消息预览，右侧要有消息时间。消息属于自定义view，
要单独创建文件或者组合函数叫做Contact Message **（2. 自定义view和RecyclerView）**

消息页面要能下拉刷新 **（3. 下拉刷新）** 这个下拉刷新是假的，根本不用去读取数据，直接延迟两秒toast弹出刷新成功就行。

点击消息页面的消息会进入消息页面可以发送消息，这个进入聊天页面是消息item动态变大进入页面，
然后点击右上角的返回也是动态变小返回消息list **（4. view动画）**

聊天页面要能下拉刷新 **（5. list感知下拉到底下拉加载更多）** 因为聊天页面其实最开始只能展示一个屏幕的消息，但是总的历史消息不止这么多，要下拉从历史的list中加载，也就是listview要能感知到下拉到底了，然后从list中加载历史聊天记录

消息要分为对方发的和我发的，我也要能真实的发消息 **（6. List中根据itemType展示两种不同的view）**

点击对方或者自己的头像要进入用户详情页面，这个过程也是头像动态变大然后在详情顶部填充满的正方形 **（7. view动效）**

这个其实每个人的头像都是2张以上，所进入详情页面之后用户的头像是又可以跟viewPager一样右滑和左滑切换的 **（8. viewPager）**

详情里面是可以下拉看这个人的朋友圈的，朋友圈是九宫格照片和文字的组合，还有日期，可以点赞和评论并更新view **（9. viewModel、changeNotifier实时更新view）**
然后刚刚说了聊天页面往上拉会加载历史记录，然后只要list判断现在不是在最下方就会在发送框的上方弹出一个小气泡“回到最新消息”

然后可以跟用户进行语音通话，其实是假的，主要是绘制view，这个view要显示对方头像，然后要显示通话时间以及静音和挂断按钮 **（10. 这个页面明显不能使用LinearLayout，要使用ConstraintLayout）**

然后回到主页面，主页面的通讯录显示的是用户列表，顶部有所搜view，可以搜索筛选联系人，点击联系人按钮也是跳转到用户详情页面 
**（11. 页面复用AMS中重复的Page不应该重复创建而是从栈中取出，因为这时候用户可以循环点击头像进入用户详情，再点击发送消息进入聊天页面，这样循环点击如果不做栈处理就会循环创建导致内存OOM，应该采用页面复用）**





## 实现


### Jetpack Compose（KMP）实现

[JetpackComposeUI.md](JetpackComposeUI.md)

### Flutter实现

[FlutterUI.md](FlutterUI.md)

















