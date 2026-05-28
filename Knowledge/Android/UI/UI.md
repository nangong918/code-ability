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


#### 补充后的学习目标



| # | 目标 | XML / 传统 Android 对应 | Compose / KMP 实现要点 |
|---|------|------------------------|------------------------|
| 1 | 底部 Tab + 横向分页 | BottomNavigation + ViewPager2 | Scaffold + HorizontalPager + ememberPagerState |
| 2 | 自定义列表项与会话列表 | 自定义 View + RecyclerView | ContactMessageItem + LazyColumn |
| 3 | 下拉刷新（假数据） | SwipeRefreshLayout | PullToRefreshBox + ViewModel delay + Toast |
| 4 | 子页面转场动效（消息/头像缩放） | 转场动画 / sharedElement | AnimatedContent + MESSAGE_ZOOM / AVATAR_ZOOM |
| 5 | 聊天页列表（双气泡·历史·回到最新） | getItemViewType + 顶部加载 + 滚动监听 | ChatMessageList + ChatSender + AnimatedVisibility |
| 6 | 详情头像轮播 + 朋友圈 | ViewPager + 网格列表 | HorizontalPager + LazyVerticalGrid + StateFlow |
| 7 | 语音通话约束布局 | ConstraintLayout | ConstraintLayout（Android）/ Box.align（iOS） |
| 8 | 通讯录搜索 + 页面栈复用 | 搜索框 + singleTop | ilteredContacts + pushOrReuse |
| 9 | 页面路由 + MVI | 多 Activity/Fragment | WeChatIntent + pageStack + when(page) |



#### 页面管理

##### 路由和栈管理

原先的XML开发，页面切换是用Activity和Fragment实现，而在Jetpack Compose中只有一个Activity。

原先的XML开发是`AMS` 管 Activity 栈，`FragmentManager` 管 Fragment 栈。

**Activity 栈（AMS）** 

- `startActivity` → 任务栈 **push**；系统返回键 → **pop**。
- `launchMode`（`singleTop`、`singleTask`）解决「栈里是否重复同一 Activity」——对应本 Demo 内层 `pushOrReuse` 按 `page.key` 去重。

**Fragment 栈**

- 同一窗口里 `replace` + `addToBackStack` 压入 Fragment；返回先 pop Fragment，栈空才 finish Activity。
- 底部 Tab 常用 `ViewPager2 + Fragment`（不压全局返回栈）；全屏子页才压栈。

**本 Demo 的 WeChat 子导航**

Activity + Fragment 返回栈 + 自定义转场；实现体换成 `@Composable`，栈由 ViewModel 维护。

```kotlin
object AppNavigator {
    private val routeStack = MutableStateFlow(listOf(AppRoute.START))
    fun navigate(route: AppRoute) {
        val stack = routeStack.value
        val next = if (stack.lastOrNull() == route) stack else stack + route
        publishStack(next)
    }
    fun goBack(): Boolean { /* dropLast */ }
}
```

**Navigation Compose** 指依赖 `androidx.navigation:navigation-compose` 的 `NavHost` / `NavController`——官方路由表、返回栈、Deep Link、与 `NavBackStackEntry` 生命周期绑定。

**本 Demo 外层** 为 KMP 共享，用的是自研轻量栈 + `when`，**不是** `NavHost`：


##### Activity

原先的Android开发页面依托于Activity，页面直接的跳转逻辑顺序强依赖于AMS的Activity栈。

而现在使用Jetpack Compose之后，整个App只有一个全局的MainActivity，页面的栈逻辑需要自己进行管理。

Jetpack中页面的切换逻辑基于App，App中会根据route的不同进行Screen切换，直接进行数据据切换而不是页面跳转，取消了冗余的Intent和数据传递。

参考代码：
```kotlin
@Composable
fun App() {
    VectorDemoTheme {
        when (route) {
            AppRoute.START -> StartScreen()
            AppRoute.LOGIN -> ComposeLoginScreen(
                state = loginState,
                savedAccounts = loginDataState.savedUserSessions,
                processIntent = { loginVm.processIntent(it) },
            )
            AppRoute.REGISTER -> ComposeRegisterScreen(
                state = registerState,
                processIntent = { registerVm.processIntent(it) },
            )
            AppRoute.MAIN -> MainScreen(
                state = mainState,
                processIntent = { mainVm.processIntent(it) },
            )
            AppRoute.HELLO -> HelloScreen()
            AppRoute.CHAT -> ChatListScreen(
                state = chatState,
                onBack = { AppNavigator.goBack() },
                onSend = { chatListVm.sendMessage(it) },
            )
        }
    }
}
```
上面代码实现了以`AppNavigator`存储page栈，根据不同的route进行不同的Screen页面跳转（切换）

##### Fragment

**Fragment → Composable（本项目中）**

App的碎片再XML中使用Fragment，而在Jetpack Compose中则使用@Composable组合函数实现。

在Demo中使用的是`AnimatedContent`切换，AnimatedContent 不是一个普通 View，它是「带自动过渡动画的内容切换器」

没有 `FragmentTransaction`；**`currentPage` 一变，`AnimatedContent` 重组出对应 Composable**。
路由参数用密封类（等同 Fragment `arguments`）：

源码如下：
```kotlin
@OptIn(ExperimentalAnimationApi::class)
@Composable
fun WeChatDemoScreen(
    state: WeChatUiState,
    processIntent: (WeChatIntent) -> Unit,
    onBackToCatalog: () -> Unit,
) {
    AnimatedContent(
        targetState = state.currentPage,
        transitionSpec = { weChatTransitionSpec(state.navAction, state.transitionStyle) },
        label = "wechat-root-nav",
    ) { page ->
        when (page) {
            WeChatPage.Home -> WeChatHomePage(
                state = state,
                processIntent = processIntent,
                onBackToCatalog = onBackToCatalog,
            )
            is WeChatPage.Chat -> WeChatChatPage(
                state = state,
                userId = page.userId,
                processIntent = processIntent,
            )
            is WeChatPage.Profile -> WeChatProfilePage(
                state = state,
                userId = page.userId,
                processIntent = processIntent,
            )
            is WeChatPage.VoiceCall -> WeChatVoiceCallPage(
                state = state,
                userId = page.userId,
                processIntent = processIntent,
            )
        }
    }
}
```



#### 底部 Tab + 横向分页

**XML：** `LinearLayout` 垂直放 `ViewPager2` + 底部 `RadioGroup` / `BottomNavigationView`，`TabLayoutMediator` 同步页码。

**Compose：** 顶栏 + 底栏在 `Scaffold` 的 `topBar` / `bottomBar`；中间 `HorizontalPager` 三页（消息 / 通讯录 / 发现）。Tab 点击与滑动双向同步用两个 `LaunchedEffect`。

```kotlin
/**
 * 微信主页（单Activity架构下的主页面）
 * 包含：顶部标题栏 + ViewPager页面 + 底部Navigation
 *
 * @param state 页面UI状态（当前选中的Tab）
 * @param processIntent 发送意图事件（切换Tab）
 * @param onBackToCatalog 返回上一级页面
 */
@OptIn(ExperimentalFoundationApi::class, ExperimentalMaterial3Api::class)
@Composable
private fun WeChatHomePage(
    state: WeChatUiState,
    processIntent: (WeChatIntent) -> Unit,
    onBackToCatalog: () -> Unit,
) {
    // 1. 创建ViewPager状态管理器（对应Android ViewPager2）
    // 初始页 = 当前选中的Tab
    // 总页数 = Tab数量
    val pagerState = rememberPagerState(
        initialPage = state.activeTab.ordinal,
        pageCount = { WeChatTab.entries.size },
    )

    // 2. 监听外部Tab变化 → 自动滚动ViewPager
    // 作用：点击底部导航 → 让Pager同步切换页面
    LaunchedEffect(state.activeTab) {
        if (pagerState.currentPage != state.activeTab.ordinal) {
            pagerState.animateScrollToPage(state.activeTab.ordinal)
        }
    }
    // 3. 监听ViewPager滑动 → 同步更新底部Tab选中状态
    // 作用：左右滑动页面 → 让底部导航跟着变
    LaunchedEffect(pagerState.currentPage) {
        val tab = WeChatTab.entries[pagerState.currentPage]
        if (tab != state.activeTab) {
            processIntent(WeChatIntent.SelectTab(tab))
        }
    }

    // Scaffold = 官方标准页面骨架（对应XML里的根布局）
    // 自带：topBar / bottomBar / content 区域
    Scaffold(
        // ---------------------
        // 顶部标题栏（Toolbar）
        // ---------------------
        topBar = {
            // 水平布局
            Row(
                modifier = Modifier
                    // 填充满宽度
                    .fillMaxWidth()
                    .background(Color(0xFF1F1F1F))
                    // 添加内边距
                    .padding(horizontal = 12.dp, vertical = 12.dp),
                // 水平居中
                verticalAlignment = Alignment.CenterVertically,
            ) {
                // 返回按钮
                TextButton(onClick = onBackToCatalog) { Text("返回", color = Color.White) }
                Text(
                    text = "WeChat UI Demo",
                    color = Color.White,
                    style = MaterialTheme.typography.titleMedium,
                    modifier = Modifier.weight(1f),
                    textAlign = TextAlign.Center,
                )
                // 占位，让标题完全居中（平衡左边返回按钮）
                Spacer(modifier = Modifier.width(60.dp))
            }
        },
        // ---------------------
        // 底部导航栏（BottomNav）
        // ---------------------
        bottomBar = {
            Row(
                modifier = Modifier
                    .fillMaxWidth()
                    .background(Color(0xFFF6F6F6))
                    .padding(vertical = 8.dp),
                // 水平居中
                horizontalArrangement = Arrangement.SpaceEvenly,
            ) {
                // 遍历所有Tab，生成底部导航项
                WeChatTab.entries.forEach { tab ->
                    val selected = tab == state.activeTab
                    Text(
                        text = tab.toTitle(),
                        modifier = Modifier
                            .clip(RoundedCornerShape(10.dp))
                            .clickable { processIntent(WeChatIntent.SelectTab(tab)) }
                            .padding(horizontal = 14.dp, vertical = 8.dp),

                        // 选中：绿色加粗 / 未选中：灰色普通
                        color = if (selected) Color(0xFF1AAD19) else Color(0xFF777777),
                        fontWeight = if (selected) FontWeight.Bold else FontWeight.Normal,
                    )
                }
            }
        },
    ) { padding ->
        // ---------------------
        // 页面主体：HorizontalPager = ViewPager2
        // 左右滑动切换页面
        // ---------------------
        HorizontalPager(
            state = pagerState,
            modifier = Modifier
                .fillMaxSize()
                // 添加内边距
                .padding(padding),
        ) { page ->
            // 根据Tab生成页面
            when (WeChatTab.entries[page]) {
                WeChatTab.MESSAGES -> WeChatMessagesPage(state = state, processIntent = processIntent)
                WeChatTab.CONTACTS -> WeChatContactsPage(state = state, processIntent = processIntent)
                WeChatTab.DISCOVER -> WeChatDiscoverPage()
            }
        }
    }
}
```


* 布局

`Column` ≈ 垂直 `LinearLayout`；`Row` + `Modifier.weight(1f)` ≈ 水平 `layout_weight`。

```xml
<!-- 垂直 -->
<LinearLayout
    android:orientation="vertical"></LinearLayout>
<!-- 水平 -->
<LinearLayout
    android:orientation="horizontal"></LinearLayout>
<!--权重 layout_weight-->
<TextView
    android:layout_weight="1"/>
<!--帧布局（层叠、叠加）-->
<FrameLayout>
<!-- 叠加 View -->
</FrameLayout>
<!--约束布局-->
<ConstraintLayout>
</ConstraintLayout>
```
```kotlin
Column()  // 垂直 → 从上到下
Row()     // 水平 → 从左到右
Text(
    modifier = Modifier.weight(1f)
)
// 帧布局（层叠、叠加）
Box() {
    // 叠加组件
}
// 约束布局
ConstraintLayout {
}
```

* 页面骨架

Scaffold ≈ 整套 XML 页面根布局

```xml
<LinearLayout>
    <Toolbar     />  <!-- 顶部 -->
    <Content     />  <!-- 中间 -->
    <BottomNav   />  <!-- 底部 -->
</LinearLayout>
```
```kotlin
Scaffold(
    topBar = {  },     // 顶部标题栏
    bottomBar = {  },   // 底部导航
    floatingActionButton = {  }, // 悬浮按钮
) { innerPadding ->
    // 页面内容（自动避开 topBar + bottomBar）
}
```

`Scaffold` 就是：自带顶部 + 底部 + 悬浮按钮的页面壳子

* 常用控件

XML → Compose

```text
TextView        → Text
Button          → Button
ImageView       → Image
EditText        → TextField
RecyclerView    → LazyColumn
ScrollView      → Column(Modifier.verticalScroll())
ViewPager2      → HorizontalPager
CardView        → Card
CheckBox        → Checkbox
Switch          → Switch
ProgressBar     → CircularProgressIndicator / LinearProgressIndicator
```

* 宽高匹配 + 边距

```text
match_parent     → Modifier.fillMaxSize()
wrap_content     → 什么都不写（默认就是 wrap）
match_parent 宽  → Modifier.fillMaxWidth()
match_parent 高  → Modifier.fillMaxHeight()
```

```xml
<LinearLayout
        android:orientation="vertical"
        android:padding="10dp"
        android:layout_margin="10dp"></LinearLayout>
```
```kotlin
Modifier.padding(10.dp)   // 内边距
Modifier.margin(10.dp)    // 外边距
```

* 圆角、背景

```xml
<shape>
    <solid android:color="#fff" />
    <corners android:radius="10dp" />
</shape>
```
```kotlin
Modifier
    .background(Color.White)
    .clip(RoundedCornerShape(10.dp))
```

* 列表
```xml
<androidx.recyclerview.widget.RecyclerView
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    app:layoutManager="androidx.recyclerview.widget.LinearLayoutManager"/>
```
```kotlin
LazyColumn {
    items(list) { item ->
        MessageItem(item)
    }
}
```

* 页面切换
```kotlin
// XML
startActivity(intent)

// Compose
// 现在只用：
AnimatedContent(page)
// 或
Navigation
```



#### 自定义列表

**XML：** 聊天页 = `RecyclerView` + `Adapter`；每一行 = `item_chat_left.xml` / `item_chat_right.xml`（`getItemViewType`）；
顶部 footer 提示「下拉加载更早」；`OnScrollListener` 判断是否在底部。

**Compose：** `WeChatChatPage` + `ChatMessageList`；`LazyColumn` + `itemsIndexed`；`message.sender` 分支左右气泡；
`LazyListState` + `pointerInput` 感知顶部下拉；`derivedStateOf` 控制「回到最新消息」。

* RecyclerView → LazyColumn 对照

```text
RecyclerView                    → LazyColumn + rememberLazyListState()
Adapter + ViewHolder            → itemsIndexed { } 内 @Composable 行 UI
getItemViewType                 → message.sender == ChatSender.ME 分支
顶部 footer / 下拉加载            → item { 提示文案 } + detectVerticalDragGestures
OnScrollListener 是否在底部       → listState.layoutInfo + derivedStateOf
notifyDataSetChanged            → _uiState.update { visibleChatMessages = ... }
```


##### 自定义列表项Item

XML里用 ViewHolder 根据不同的 viewType 绑定不同的 viewBinding
Compose中 直接使用 @Composable 组合函数 + itemsIndexed 

自定义view
```kotlin
@Composable
fun ChatMessageItem(
    message: WeChatChatMessage,
    contact: WeChatContact?,
    onAvatarClick: (String) -> Unit,
) {
    val isMe = message.sender == ChatSender.ME

    Row(
        modifier = Modifier
            .fillMaxWidth()
            .padding(horizontal = 10.dp),
        horizontalArrangement = if (isMe) Arrangement.End else Arrangement.Start,
        verticalAlignment = Alignment.Top,
    ) {
        // 对方发送，先显示头像
        if (!isMe) {
            WeChatAvatar(
                modifier = Modifier.size(36.dp),
                contact = contact,
                paletteIndex = 0,
                onClick = { onAvatarClick(contact?.id.orEmpty()) },
            )
            Spacer(modifier = Modifier.width(8.dp))
        }
        // 消息体
        Column(horizontalAlignment = if (isMe) Alignment.End else Alignment.Start) {
            Text(
                text = message.timeLabel,
                style = MaterialTheme.typography.labelSmall,
                color = Color(0xFF8A8A8A),
            )
            Spacer(modifier = Modifier.height(2.dp))
            Box(
                modifier = Modifier
                    .clip(RoundedCornerShape(10.dp))
                    .background(if (isMe) Color(0xFF95EC69) else Color.White)
                    .padding(horizontal = 10.dp, vertical = 8.dp),
            ) {
                Text(message.text, style = MaterialTheme.typography.bodyMedium)
            }
        }
        // 我发送，后显示头像
        if (isMe) {
            Spacer(modifier = Modifier.width(8.dp))
            WeChatAvatar(
                modifier = Modifier.size(36.dp),
                contact = meContactForChat,
                paletteIndex = 0,
                onClick = { onAvatarClick("me") },
            )
        }
    }
}
```

##### 自定义列表

在XML中使用的是RecyclerView
Compose中使用的是LazyColumn

```kotlin
    // ==============================================
    // 聊天消息列表 = LazyColumn（对应 XML RecyclerView）
    // 带自动复用、滑动状态、手势监听
    // ==============================================
    LazyColumn(
        state = listState,
        modifier = modifier
            .fillMaxWidth()
            .background(Color(0xFFEFEFEF))
            // 监听垂直下拉手势，实现“顶部下拉加载历史”
            .pointerInput(canLoadMoreHistory) {
                detectVerticalDragGestures { _, dragAmount ->
                    // 条件：已经滑到顶部 + 向下拖拽 + 允许加载历史
                    if (listState.firstVisibleItemIndex == 0 && dragAmount > 0 && canLoadMoreHistory) {
                        topDragOffset += dragAmount
                    } else {
                        // 其他情况重置偏移，避免误触发
                        topDragOffset = 0f
                    }
                }
            },
        // Item 之间的间距（对应 XML ItemDecoration）
        verticalArrangement = Arrangement.spacedBy(8.dp),
    ) {
        // ==============================================
        // 列表 Header（头部）
        // 显示：下拉加载更早消息
        // ==============================================
        item {
            Spacer(modifier = Modifier.height(8.dp))
            if (canLoadMoreHistory) {
                Text(
                    text = "下拉加载更早消息",
                    modifier = Modifier
                        .fillMaxWidth()
                        .padding(vertical = 4.dp),
                    textAlign = TextAlign.Center,
                    style = MaterialTheme.typography.labelMedium,
                    color = Color(0xFF808080),
                )
            }
        }
        // ==============================================
        // 消息列表主体（多条气泡 Item），相当于 XML Adapter + ViewHolder
        // key：保证列表刷新稳定，避免闪烁、错乱
        // ==============================================
        itemsIndexed(
            items = messages,
            // 通过index + message.id 生成 view中item的唯一id
            key = { index, message -> "${message.id}-$index" },
        ) { _, message ->   // 每一项要显示的 UI
            ChatMessageItem(
                message = message,
                contact = contact,
                onAvatarClick = onAvatarClick,
            )
        }
        // ==============================================
        // 列表 Footer（底部）
        // 仅作底部间距，让最后一条消息不贴底
        // ==============================================
        item { Spacer(modifier = Modifier.height(8.dp)) }
    }
```


#### 下拉刷新

**XML：** 包一层 `SwipeRefreshLayout`，`setOnRefreshListener` 里请求接口。

**Compose：** Material3 `PullToRefreshBox`，刷新状态来自 `state.refreshingMessages`。

```kotlin
// 下拉刷新
PullToRefreshBox(
    isRefreshing = state.refreshingMessages,
    onRefresh = { processIntent(WeChatIntent.RefreshMessages) },
    modifier = Modifier.fillMaxSize(),
) {
    LazyColumn(
        modifier = Modifier.fillMaxSize(),
    ) {
        items(state.messagePreviews, key = { it.contactId }) { preview ->
            val contact = state.contacts.firstOrNull { it.id == preview.contactId }
            ContactMessageItem(
                preview = preview,
                contact = contact,
                onClick = { processIntent(WeChatIntent.OpenChatFromMessage(preview.contactId)) },
                onAvatarClick = { processIntent(WeChatIntent.OpenProfileFromAvatar(preview.contactId)) },
            )
            Divider(color = Color(0xFFEDEDED))
        }
    }
}
```

#### 消息项缩放进入聊天页

**Compose：** `AnimatedContent` 的 `transitionSpec` 用 `scaleIn` / `scaleOut` + `spring`。

```kotlin
// ==============================
// AnimatedContent = 页面切换动画载体
// 作用：根据 targetState 切换页面，并自动播放动画
// ==============================
AnimatedContent(
    targetState = state.currentPage,       // 目标页面（切换它就会播放动画）
    transitionSpec = {                     // 动画规格（上面定义的缩放/淡入淡出）
        weChatTransitionSpec(state.navAction, state.transitionStyle)
    },
    label = "wechat-root-nav",             // 动画标签（调试用）
) { page ->
}
```

`transitionSpec` 的参数选用

```kotlin
/**
 * 页面切换动画配置（核心：自定义进/出栈动画）
 * 对应 XML：overridePendingTransition 跳转动画
 *
 * @param action 页面动作：PUSH 进栈 / POP 出栈（返回）
 * @param style 动画类型：消息放大 / 头像放大 / 普通淡入淡出
 */
@OptIn(ExperimentalAnimationApi::class)
private fun weChatTransitionSpec(
    action: WeChatNavAction,
    style: WeChatTransitionStyle,
) = when (style) {
    // ==============================
    // 1. 消息列表 → 聊天页面：缩放动画（中心放大效果）
    // ==============================
    WeChatTransitionStyle.MESSAGE_ZOOM -> {
        if (action == WeChatNavAction.PUSH) {
            // 【进栈】新页面从小放大（0.85 → 1）+ 淡入
            // 旧页面稍微放大（1→1.03）+ 淡出
            (fadeIn(animationSpec = spring(stiffness = Spring.StiffnessLow)) +
                    scaleIn(initialScale = 0.85f, animationSpec = spring(stiffness = Spring.StiffnessMediumLow))) togetherWith
                    (fadeOut(animationSpec = spring(stiffness = Spring.StiffnessMedium)) +
                            scaleOut(targetScale = 1.03f, animationSpec = spring(stiffness = Spring.StiffnessLow)))
        } else {
            // 【出栈】返回：页面从大缩小回去 + 淡入
            // 前一个页面恢复原状
            (fadeIn(animationSpec = spring(stiffness = Spring.StiffnessLow)) +
                    scaleIn(initialScale = 1.04f, animationSpec = spring(stiffness = Spring.StiffnessLow))) togetherWith
                    (fadeOut(animationSpec = spring(stiffness = Spring.StiffnessMedium)) +
                            scaleOut(targetScale = 0.86f, animationSpec = spring(stiffness = Spring.StiffnessMediumLow)))
        }
    }

    // ==============================
    // 2. 头像 → 用户详情页：头像放大动画
    // ==============================
    WeChatTransitionStyle.AVATAR_ZOOM -> {
        if (action == WeChatNavAction.PUSH) {
            // 新页面从头像大小(0.7)放大进入
            // 旧页面轻微放大退出
            (fadeIn() + scaleIn(initialScale = 0.7f)) togetherWith (fadeOut() + scaleOut(targetScale = 1.06f))
        } else {
            // 返回时缩小回到头像位置
            (fadeIn() + scaleIn(initialScale = 1.05f)) togetherWith (fadeOut() + scaleOut(targetScale = 0.75f))
        }
    }
    // ==============================
    // 3. 普通页面：淡入淡出动画
    // ==============================
    WeChatTransitionStyle.NORMAL -> {
        if (action == WeChatNavAction.PUSH) {
            fadeIn() togetherWith fadeOut()
        } else {
            fadeIn() togetherWith fadeOut()
        }
    }
}
```



#### 详情页头像 ViewPager

**XML：** `ViewPager2` + `FragmentStateAdapter`。

**Compose：** 详情顶部 `HorizontalPager(pageCount = contact.avatarPalette.size)`，每页一个色块 + 居中名字（Demo 用调色板代替多张照片）。

```kotlin
// 横向滑动的 ViewPager（= XML ViewPager2）
HorizontalPager(
    state = avatarPagerState,          // 滑动状态、当前页码
    modifier = Modifier
        .fillMaxWidth()                 // 宽度铺满
        .aspectRatio(1f),               // 宽高比 1:1 → 正方形（头像区域）
) { page ->                             // page = 当前滑动到的索引

    // 每一页的 UI（这里用颜色块+名字模拟头像）
    Box(
        modifier = Modifier
            .fillMaxSize()
            .background(Color(contact.avatarPalette[page].toLong())),
        contentAlignment = Alignment.Center,
    ) {
        Text(
            text = contact.name,
            color = Color.White,
            style = MaterialTheme.typography.headlineSmall,
        )
    }
}
```

朋友圈整体在 **`LazyColumn` 里向下滚**（不是 NestedScrollView 包 WebView，而是 Compose 嵌套：外层 `LazyColumn` + 内层 `LazyVerticalGrid` 九宫格）。

---

#### 朋友圈：九宫格 + ViewModel 驱动 UI 更新

##### 网格布局

**XML：** `GridView` / `RecyclerView GridLayoutManager`；点赞后 `notifyItemChanged`；评论 `EditText` + 提交。

**Compose：**

- 九宫格：`LazyVerticalGrid(GridCells.Fixed(3))` + `BoxWithConstraints` 算 cell 高度（避免网格在 `LazyColumn` 里高度未知）。
- 点赞 / 评论：Intent 进 VM，改 `momentsByUser` immutable map。

```kotlin
// BoxWithConstraints = 可以获取父容器宽高的布局（用来计算九宫格大小）
BoxWithConstraints(modifier = Modifier.fillMaxWidth()) {
    // 1. 计算每一个格子的大小：屏幕宽度 - 间距 ÷ 3（九宫格每一张图的尺寸）
    val cellSize = (maxWidth - 12.dp) / 3

    // 2. 计算需要多少行：(图片总数 + 2) ÷ 3 = 向上取整（1-3图1行，4-6图2行，7-9图3行）
    val rows = (photos.size + 2) / 3

    // 3. 计算整个九宫格总高度：行高 + 行间距
    val gridHeight = (cellSize * rows) + (6.dp * (rows - 1).coerceAtLeast(0))

    // ==============================
    // LazyVerticalGrid = 网格列表（对应 XML：GridView / GridLayoutManager）
    // 固定3列 = 九宫格
    // ==============================
    LazyVerticalGrid(
        columns = GridCells.Fixed(3), // 固定3列（核心：九宫格）
        userScrollEnabled = false,    // 禁止列表滚动（朋友圈只展示，不单独滚动）
        modifier = Modifier
            .fillMaxWidth()
            .height(gridHeight),      // 固定高度，根据图片数量计算
        horizontalArrangement = Arrangement.spacedBy(6.dp), // 图片水平间距
        verticalArrangement = Arrangement.spacedBy(6.dp),   // 图片垂直间距
    ) {
        // 遍历图片列表，渲染每一项
        items(photos, key = { it.label }) { photo ->
            // 图片加载（对应 XML：ImageView + Glide）
            AsyncImage(
                model = Res.getUri(photo.resourcePath), // 图片地址
                contentDescription = photo.label,
                contentScale = ContentScale.Crop,        // 图片居中裁剪（仿朋友圈）
                modifier = Modifier
                    .size(cellSize)          // 每个格子都是正方形
                    .clip(RoundedCornerShape(8.dp)) // 圆角
                    .clickable { onPhotoClick(photo) }, // 点击查看大图
            )
        }
    }
}
```


##### ViewModel数据联动

**基础原理：Compose 自动刷新 vs XML LiveData**
XML 传统模式（必须手动订阅）
XML + LiveData = 必须通过 `.observe()` 手动监听并更新 UI
```kotlin
// 必须观察
viewModel.liveData.observe(this) { data ->
    // 手动更新 UI
    textView.setText(data.text)
    commentAdapter.setList(data.comments)
}
```

**Compose 原生模式（参数驱动自动刷新）**

Compose 是参数驱动响应式：上层传入的状态参数变化，UI 自动重组刷新，无需手动监听、findViewById、Adapter 通知。

```kotlin
// 不用观察
// 不用手动更新
// 不用 findView
// 不用 adapter.notify

// 只要参数变了 → UI 自动刷新
// 仅接收状态参数，参数变化 → UI 自动刷新
fun MomentCard(moment: WeChatMoment) {
    Text(moment.text)
    moment.comments.forEach { comment ->
        Text(text = "${comment.authorName}: ${comment.content}")
    }
}
```

外部业务状态（点赞、新增评论、消息更新）变化，UI 自动同步更新
Compose 内置响应式能力，原生替代 `ChangeNotifier` / `LiveData` 的通知刷新逻辑


**状态分层：ViewModel 业务状态 vs UI 局部状态**

* Screen 级页面：必须使用 ViewModel + StateFlow（KMP 标准）

状态必须放在 ViewModel，不能放在 remember：避免业务逻辑与 UI 耦合，难测试、难复用、生命周期不稳定
StateFlow 是 KMP 最优方案：纯 Kotlin 实现，无平台依赖，Android /iOS 全平台兼容，替代 LiveData

MVI标准实现
```kotlin
// ViewModel 层（commonMain 跨平台）
private val _uiState = MutableStateFlow(LoginState())
val uiState: StateFlow<LoginState> = _uiState.asStateFlow()

// UI 层订阅
val loginState by loginVm.uiState.collectAsState()
ComposeLoginScreen(state = loginState, processIntent = { loginVm.processIntent(it) })
```

* 自定义组件（Component）：仅用 remember { mutableStateOf } 存局部 UI 状态


#### Jetpack Compose 约束布局 ConstraintLayout


##### 核心 API

- createRefs()：创建多个引用 ID（对应 XML 组件 id）
- constrainAs(xxxRef)：给组件绑定引用
- top / bottom / start / end：上下左右约束
- linkTo：建立与父布局 / 其他组件的相对关系
- margin：边距
- Dimension.value：固定宽高
- Dimension.fillToConstraints：填充约束范围内的空间

**1.组件引用ID**

作用：定义唯一标识，用于组件之间互相约束依赖

Compose：`createRefs()` 批量创建组件引用

```kotlin
val (backRef, avatarRef, nameRef) = createRefs()
```

**XML**：通过 `android:id` 定义控件唯一ID

```xml
<ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
                  xmlns:app="http://schemas.android.com/apk/res-auto">
    <!-- id -->
    <ImageView
            android:id="@+id/back"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            app:layout_constraintTop_toTopOf="parent"
            app:layout_constraintStart_toStartOf="parent"/>
</ConstraintLayout>
```

**2.绑定组件约束**

作用：将UI组件与创建的引用绑定，配置约束规则
Compose：constrainAs(xxxRef) 绑定对应引用，内部编写约束逻辑
```kotlin
Modifier.constrainAs(backRef) { }
```


**3.约束方向标识**

作用：指定上下左右四个约束方向

Compose：原生方向关键字

- `top` / `bottom` / `start` / `end`

XML：对应约束属性前缀

- `layout_constraintTop_to`

- `layout_constraintBottom_to`

- `layout_constraintStart_to`

- `layout_constraintEnd_to`


**4.建立相对约束关系**

作用：让当前组件 关联父布局/其他组件，实现相对定位

Compose：`linkTo()` 绑定约束目标

```kotlin
// 关联父布局顶部
top.linkTo(parent.top)
// 关联其他组件底部
top.linkTo(nameRef.bottom)
```

**XML**：通过属性指定约束目标

```xml
<ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
                  xmlns:app="http://schemas.android.com/apk/res-auto">
    <!-- 关联父布局顶部 -->
    <!-- 关联其他组件底部 -->
    <ImageView
            android:id="@+id/back"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            app:layout_constraintTop_toTopOf="parent"
            app:layout_constraintTop_toBottomOf="@id/name"/>
</ConstraintLayout>

```


**5. 约束边距 margin**

作用：约束定位时的外边距，仅对约束方向生效
Compose：`linkTo`方法传参设置 margin

```kotlin
top.linkTo(parent.top, margin = 24.dp)
```

**XML**：独立 margin 属性

```xml
<ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
                  xmlns:app="http://schemas.android.com/apk/res-auto">
    <!-- 距离顶部 -->
    <ImageView
            android:id="@+id/back"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_marginTop="24dp"/>
</ConstraintLayout>
```


**固定宽高尺寸**

作用：给约束布局内组件设置固定宽高

Compose：`Dimension.value()`

```kotlin
width = Dimension.value(180.dp)
```

XML：固定 dp 尺寸

```text
android:layout_width="180dp"
```

** 7.约束内填充尺寸**

作用：在左右/上下约束范围内自动填满剩余空间（最常用）

Compose：`Dimension.fillToConstraints`

```kotlin
width = Dimension.fillToConstraints
```

XML：经典 `0dp` 填充写法

```text
android:layout_width="0dp"
app:layout_constraintStart_toStartOf="parent"
app:layout_constraintEnd_toEndOf="parent"
```

##### Compose 代码实现
```kotlin
@Composable
internal actual fun WeChatVoiceCallLayoutPlatform(
    contact: WeChatContact?,
    displayName: String,
    duration: String,
    callMuted: Boolean,
    onBack: () -> Unit,
    onToggleMute: () -> Unit,
    onEndCall: () -> Unit,
) {
    // 约束布局根容器
    ConstraintLayout(
        modifier = Modifier
            .fillMaxSize()
            .background(Color(0xFF121212)),
    ) {
        // 定义所有组件的引用（相当于 XML 里的 id）
        val (backRef, avatarRef, nameRef, durationRef, muteRef, hangupRef) = createRefs()

        // 顶部返回按钮
        Box(
            modifier = Modifier
                .constrainAs(backRef) {
                    top.linkTo(parent.top, margin = 24.dp)
                    start.linkTo(parent.start)
                    end.linkTo(parent.end)
                    width = Dimension.value(48.dp)
                    height = Dimension.value(48.dp)
                }
                .clip(CircleShape)
                .clickable(onClick = onBack)
                .background(Color(0x33FFFFFF)),
            contentAlignment = Alignment.Center,
        ) {
            Text("返回", color = Color.White)
        }

        // 中间大头像（完全居中）
        WeChatAvatar(
            modifier = Modifier
                .constrainAs(avatarRef) {
                    top.linkTo(parent.top)
                    bottom.linkTo(parent.bottom)
                    start.linkTo(parent.start)
                    end.linkTo(parent.end)
                    width = Dimension.value(180.dp)
                    height = Dimension.value(180.dp)
                },
            contact = contact,
            paletteIndex = 2,
        )

        // 顶部居中用户名
        Text(
            text = displayName,
            color = Color.White,
            style = MaterialTheme.typography.headlineSmall,
            textAlign = TextAlign.Center,
            modifier = Modifier.constrainAs(nameRef) {
                top.linkTo(parent.top, margin = 90.dp)
                start.linkTo(parent.start)
                end.linkTo(parent.end)
                width = Dimension.fillToConstraints
            },
        )

        // 通话时长（在名字下方居中）
        Text(
            text = duration,
            color = Color(0xFFDADADA),
            textAlign = TextAlign.Center,
            modifier = Modifier.constrainAs(durationRef) {
                top.linkTo(nameRef.bottom, margin = 8.dp)
                start.linkTo(parent.start)
                end.linkTo(parent.end)
                width = Dimension.fillToConstraints
            },
        )

        // 底部左侧：静音按钮
        Button(
            onClick = onToggleMute,
            modifier = Modifier.constrainAs(muteRef) {
                bottom.linkTo(parent.bottom, margin = 34.dp)
                start.linkTo(parent.start, margin = 24.dp)
                width = Dimension.value(120.dp)
            },
        ) {
            Text(if (callMuted) "取消静音" else "静音")
        }

        // 底部右侧：挂断按钮
        Button(
            onClick = onEndCall,
            modifier = Modifier.constrainAs(hangupRef) {
                bottom.linkTo(parent.bottom, margin = 34.dp)
                end.linkTo(parent.end, margin = 24.dp)
                width = Dimension.value(120.dp)
            },
        ) {
            Text("挂断")
        }
    }
}
```


#### 页面栈复用（防循环创建 OOM）

**XML：** `launchMode="singleTop"`、`FragmentTransaction` 带 tag 复用、或 Navigation `popUpTo` + `launchSingleTop`。

**Compose Demo：** 内存栈 `pageStack`，`pushOrReuse` 若 `nextPage.key` 已存在则 **pop 到该页**，不重复 new：

```kotlin
    private fun pushOrReuse(nextPage: WeChatPage, transitionStyle: WeChatTransitionStyle) {
        val existingIndex = pageStack.indexOfFirst { it.key == nextPage.key }
        if (existingIndex == pageStack.lastIndex) return
        if (existingIndex >= 0) {
            while (pageStack.size > existingIndex + 1) {
                val removed = pageStack.removeAt(pageStack.lastIndex)
                if (removed is WeChatPage.VoiceCall) stopCallTimer()
            }
            publishNavState(WeChatNavAction.POP, transitionStyle)
        } else {
            pageStack += nextPage
            publishNavState(WeChatNavAction.PUSH, transitionStyle)
        }
    }
```

`WeChatPage` 的 `key`（如 `chat:u_lina`、`profile:u_lina`）≈ Activity/Fragment 的 canonical name。

典型路径：消息 → 聊天 → 点头像 → 详情 → 发消息 → 聊天 → … 栈上同 userId 的 Chat/Profile 只保留一份实例。





更细的 Compose-only 笔记可继续写在 [JetpackComposeUI.md](JetpackComposeUI.md)。



### Flutter 实现

Demo 源码：`demo/flutter/flutternew/lib/`（目录分级与 KMP `ui/view/wechat` 对齐）。

| 层级 | 路径 |
|------|------|
| 入口 Page | `page/wechat_demo_page.dart` |
| 根 Screen | `ui/view/wechat/we_chat_demo_screen.dart` |
| 页面转场 | `ui/view/wechat/we_chat_page_transition.dart` |
| 自定义组件 | `ui/view/wechat/components/*.dart` |
| ViewModel | `viewmodel/wechat_demo_vm.dart` |
| 模型 / Intent | `domain/model/wechat/wechat_models.dart` |

#### 补充后的学习目标

| # | 目标 | XML / 传统 Android 对应 | Flutter 实现要点 |
|---|------|------------------------|------------------|
| 1 | 底部 Tab + 横向分页 | `BottomNavigation` + `ViewPager2` | `Scaffold` + `PageView` + `PageController` |
| 2 | 自定义列表项与会话列表 | 自定义 `View` + `RecyclerView` | `ContactMessageItem` + `ListView` |
| 3 | 下拉刷新（假数据） | `SwipeRefreshLayout` | `RefreshIndicator` + `delay` + `SnackBar` |
| 4 | 子页面转场动效（消息/头像缩放） | 转场动画 / `sharedElement` | `WeChatAnimatedContent` 双层 `Stack` + `SpringSimulation` |
| 5 | 聊天页列表（双气泡·历史·回到最新） | `getItemViewType` + 顶部加载 + 滚动监听 | `ChatMessageList` + `ChatMessageItem` + 滚动监听气泡 |
| 6 | 详情头像轮播 + 朋友圈 | `ViewPager` + 网格列表 | `PageView` + `GridView` + `ChangeNotifier` |
| 7 | 语音通话约束布局 | `ConstraintLayout` | `Stack` + `Positioned`（等效约束） |
| 8 | 通讯录搜索 + 页面栈复用 | 搜索框 + `singleTop` | `TextField` + `filteredContacts` + `pushOrReuse` |
| 9 | 页面路由 + MVI | 多 Activity/Fragment | `WeChatIntent` + `_pageStack` + `AnimatedBuilder` |

#### 页面管理


* MaterialApp

MaterialApp 是 Flutter 应用的**根入口 Widget**，相当于：
- Android：MainApplication + 根 MainActivity
- Compose：整个 App 的根容器
- iOS：UIApplication + UIWindow

它负责：
- 应用主题（Theme）
- 国际化（多语言）
- 路由/页面跳转管理（最核心）
- 全局配置（标题、Debug 条、主题）

* 根组件 MyApp

```dart
class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(/*...*/);
  }
}
```

`MyApp` 是整个 App 的起点
`StatelessWidget` 无状态，只负责配置全局环境
`BuildContext` 是 Widget 树中的当前上下文

##### 路由和栈管理

**MyApp中配置**

在 `MyApp` 中，`MaterialApp.routes` 配置路由表，`MaterialApp.home` 指定默认页，`MaterialApp.initialRoute` 制定启动页面。

```dart
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      // 核心：使用routes映射表（替代onGenerateRoute）
      routes: appRoutes,
      // 初始路由：先走启动页鉴权，再决定去主页面或登录页
      initialRoute: AppRoutes.start,
      // 可选：兜底处理未知路由（如果需要）
      onUnknownRoute: unknownRoute,
    );
  }
```

**路由表**

appRoutes 路由表 把「字符串路由名称」 → 映射到「具体页面 Widget」
相当于 AndroidManifest.xml 里注册所有 Activity

```dart
// config/app_route.dart
final Map<String, WidgetBuilder> appRoutes = {
  AppRoutes.start: (BuildContext context) {
    return const StartPage();
  },
  AppRoutes.login: (BuildContext context) {
    return const LoginPage();
  },
};
```

* WidgetBuilder

```dart
typedef WidgetBuilder = Widget Function(BuildContext context);
```

WidgetBuilder = 一个接收 context，返回 Widget 的函数

* 为什么路由要用 WidgetBuilder，而不是直接写 Widget？

原因 1：路由是 “懒加载” 的
- 不是启动 App 就把所有页面都创建好
- 只有跳转到这个页面时，才执行这个函数，才创建页面
- 节省性能、节省内存（和 Android 只在启动时创建 Activity 一样）

原因 2：需要传入 BuildContext
- 页面需要 context 才能：
  - 跳转下一个页面
  - 弹出页面
  - 获取主题
  - 获取语言
  - 获取上层状态

* 页面跳转

页面跳转分别使用 `Navigator.pushNamed` 和 `Navigator.pushNamedAndRemoveUntil`，含义分别是：
- `Navigator.pushNamed`：跳转到指定页面，并入栈
- `Navigator.pushNamedAndRemoveUntil`：跳转到指定页面，并出栈所有指定页面之前的页面

```dart
// 出栈
Future<void> _confirmLogout() async {
  final shouldLogout = await showDialog<bool>(
    context: context,
  );
  // 跳转并出栈
  Navigator.pushNamedAndRemoveUntil(
    context,
    AppRoutes.login,
        (route) => false,
  );
}
  // 跳转
class _LoginPageState extends State<LoginPage> {
  late final LoginVm _vm;
  StreamSubscription<LoginEffect>? _effectSub;

  @override
  void initState() {
    super.initState();
    _vm = LoginVm();
    _effectSub = _vm.effects.listen(_consumeEffect);
  }

  // 副作用流
  void _consumeEffect(LoginEffect effect) {
    if (effect is LoginNavigateToMain) {
      // 跳转并出栈
      Navigator.pushNamedAndRemoveUntil(context, AppRoutes.main, (route) => false);
    } else if (effect is LoginNavigateToRegister) {
      // 跳转
      Navigator.pushNamed(context, AppRoutes.register);
    }
  }
}

```

外层 App 用 `MaterialApp.routes`（`config/app_route.dart`）进入 Demo；
**WeChat 内层** 在 `WeChatDemoVm` 内维护 `_pageStack`（对应 Compose 内存栈）。

**Activity 栈（AMS）** → Flutter 单 `Activity` / 单引擎；跨 Demo 用 `Navigator.pushNamed`，返回 `Navigator.pop`。

**Fragment 栈** → `WeChatDemoScreen` + `WeChatAnimatedContent`：`currentPage` 变化时切换子 Widget（等同 `AnimatedContent` + `when(page)`）。

```dart
/// 微信 UI Demo 根 Screen（对应 KMP WeChatDemoScreen）
class WeChatDemoScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return WeChatAnimatedContent(
      page: state.currentPage,
      navAction: state.navAction,
      transitionStyle: state.transitionStyle,
      child: _buildPage(context),
    );
  }

  // 构建页面
  Widget _buildPage(BuildContext context) {
    final page = state.currentPage;
    // 主页
    if (page is WeChatPageHome) {
      return _WeChatHomePage(
        state: state,
        processIntent: processIntent,
        onBackToCatalog: onBackToCatalog,
      );
    }
    // 聊天
    if (page is WeChatPageChat) {
      return _WeChatChatPage(
        state: state,
        userId: page.userId,
        processIntent: processIntent,
      );
    }
    return const SizedBox.shrink();
  }
}

```

#### 底部 Tab + 横向分页

**XML：** 垂直 `LinearLayout`：`ViewPager2` + 底部 `BottomNavigationView`。

**Flutter：** `Scaffold` 的 `appBar` / `bottomNavigationBar` + `body: PageView`；Tab 与页码双向同步（对应 KMP 两个 `LaunchedEffect`）。

```dart
// ==============================
// 底部导航三个页面：消息、联系人、发现
// 对应 Android BottomNavigationView 的三个Item
// ==============================
enum WeChatTab { messages, contacts, discover }

// ==============================
// 全局UI状态（来自ViewModel/Store）
// 单一数据源 MVI 模式
// ==============================
class WeChatUiState {
  // 当前选中的底部Tab
  final WeChatTab activeTab;
}

// ==============================
// 主页（有状态组件）
// 接收外部状态 + 回调事件
// ==============================
class _WeChatHomePage extends StatefulWidget {
  // 页面状态（来自VM）
  final WeChatUiState state;
  // 页面事件回调（发送意图给VM）
  final WeChatProcessIntent processIntent;

  const _WeChatHomePage({
    required this.state,
    required this.processIntent,
  });

  @override
  State<_WeChatHomePage> createState() => _WeChatHomePageState();
}

// ==============================
// 主页状态（控制PageView滑动、动画、生命周期）
// ==============================
class _WeChatHomePageState extends State<_WeChatHomePage> {
  // PageView控制器：管理滑动、页码、动画
  // 对应 Android ViewPager2.ViewPager2()
  late PageController _pageController;

  // ==============================
  // 初始化：只走一次
  // ==============================
  @override
  void initState() {
    super.initState();
    // 根据当前选中的tab索引，初始化PageView位置
    _pageController = PageController(initialPage: widget.state.activeTab.index);
  }

  // ==============================
  // 组件更新（外部state变化时触发）
  // 作用：VM状态变化 → 同步PageView页码
  // 对应 Android StateFlow 观察刷新
  // ==============================
  @override
  void didUpdateWidget(covariant _WeChatHomePage oldWidget) {
    super.didUpdateWidget(oldWidget);

    // 如果【新选中的Tab】≠【旧Tab】
    // 并且PageView已绑定上下文
    // 并且当前PageView页码≠目标页码
    if (oldWidget.state.activeTab != widget.state.activeTab &&
        _pageController.hasClients &&
        _pageController.page?.round() != widget.state.activeTab.index) {
      // 执行PageView翻页动画（平滑切换）
      _pageController.animateToPage(
        widget.state.activeTab.index, // 目标页码
        duration: const Duration(milliseconds: 280), // 动画时长
        curve: Curves.easeOut, // 动画插值器
      );
    }
  }

  // ==============================
  // 销毁：释放控制器，防止内存泄漏
  // ==============================
  @override
  void dispose() {
    _pageController.dispose();
    super.dispose();
  }

  // ==============================
  // 页面构建UI
  // ==============================
  @override
  Widget build(BuildContext context) {
    // Scaffold = 页面骨架（AppBar + 内容 + 底部导航）
    return Scaffold(
      // 顶部标题栏
      appBar: AppBar(
        backgroundColor: const Color(0xFF1F1F1F),
        foregroundColor: Colors.white,
        elevation: 0,
        leading: TextButton(
          onPressed: widget.onBackToCatalog,
          child: const Text('返回', style: TextStyle(color: Colors.white)),
        ),
        title: const Text('WeChat UI Demo'),
        centerTitle: true,
      ),

      // ==============================
      // 底部导航栏（自定义，非系统BottomNav）
      // 三个可点击文字：消息、联系人、发现
      // ==============================
      bottomNavigationBar: ColoredBox(
        color: const Color(0xFFF6F6F6),
        child: Padding(
          padding: const EdgeInsets.symmetric(vertical: 8),
          child: Row(
            mainAxisAlignment: MainAxisAlignment.spaceEvenly,
            children: WeChatTab.values.map((tab) {
              // 判断当前tab是否选中
              final selected = tab == widget.state.activeTab;
              return GestureDetector(
                // 点击 → 发送切换Tab意图
                onTap: () => widget.processIntent(WeChatSelectTab(tab)),
                child: Padding(
                  padding: const EdgeInsets.symmetric(horizontal: 14, vertical: 8),
                  child: Text(
                    weChatTabTitle(tab),
                    style: TextStyle(
                      color: selected
                          ? const Color(0xFF1AAD19) // 选中绿色
                          : const Color(0xFF777777), // 未选中灰色
                      fontWeight:
                      selected ? FontWeight.bold : FontWeight.normal,
                    ),
                  ),
                ),
              );
            }).toList(),
          ),
        ),
      ),
      
      // ==============================
      // PageView 可滑动页面（三页切换）
      // 对应 Android ViewPager2
      // ==============================
      body: PageView(
        // pageController: 页面管理
        controller: _pageController,
        onPageChanged: (index) {
          final tab = WeChatTab.values[index];
          if (tab != widget.state.activeTab) {
            // 手动滑动 → intent 通知VM切换选中Tab
            widget.processIntent(WeChatSelectTab(tab));
          }
        },
        // 三个子页面
        children: [
          _WeChatMessagesPage(state: widget.state, processIntent: widget.processIntent),
          _WeChatContactsPage(state: widget.state, processIntent: widget.processIntent),
          const _WeChatDiscoverPage(),
        ],
      ),
    );
  }
}
```


_WeChatHomePage = Activity 类
- 它是 “页面的壳”
- 持有页面需要的参数（state、processIntent）
- 自己不做 UI，只创建状态


_WeChatHomePageState = 真正的 Activity 本体
StatefulWidget + State 组合 = Android Activity
```dart
class _WeChatHomePageState extends State<_WeChatHomePage> { /* ... */ }
```

```text
onCreate → initState()
onDestroy → dispose()
setContentView → build()
```

* 布局

| XML / Compose | Flutter |
|---------------|---------|
| `LinearLayout` vertical | `Column` |
| `LinearLayout` horizontal | `Row` |
| `layout_weight` | `Expanded` / `Flexible` |
| `FrameLayout` | `Stack` |
| `ConstraintLayout` | `Stack` + `Positioned` 或第三方 `flutter_constraintlayout` |

```dart
Widget build(BuildContext context) {
  return Scaffold(
      Column(children: [/* ... */]),                  // 垂直
      Row(children: [Expanded(child: [/* ... */])]),  // 水平 + weight
      Stack(children: [/* ... */]),                   // 层叠
      Positioned(                                     // 约束
        left: 0,
        right: 0,
        bottom: 72,
        child: Center(),
      ),
  );
}
```

* 页面骨架

`Scaffold` ≈ 带 `AppBar` + `body` + `bottomNavigationBar` 的整页骨架（对应 Compose `Scaffold`）。
Scaffold = `R.layout.main_activity`

* 常用控件

```text
TextView        → Text
Button          → ElevatedButton / FilledButton / TextButton
ImageView       → Image / Image.asset
EditText        → TextField
RecyclerView    → ListView / ListView.builder
ScrollView      → SingleChildScrollView / ListView
ViewPager2      → PageView
CardView        → Card
```

* 宽高与边距

```text
match_parent         → SizedBox.expand() / width: double.infinity / height: double.infinity
wrap_content         → 不写宽高（Flutter 自动包裹内容）
layout_margin         → 外层 Padding（Flutter 无 margin，用 Padding 替代）
padding               → Padding(padding: EdgeInsets.all(10))
```

```text
# 1. match_parent 宽高铺满
layout_width="match_parent"
layout_height="match_parent"

Container(
  width: double.infinity,
  height: double.infinity,
)

# 2. padding 内边距
android:padding="10dp"

Padding(
  padding: EdgeInsets.all(10),
  child: Text("带内边距"),
)
```

#### 自定义列表

**XML：** 消息 Tab = `RecyclerView` + 自定义行；聊天 = 左右两种 `ViewHolder`。

**Flutter：** 列表容器 `ListView.separated`（对应 `LazyColumn` / `RecyclerView`）。

```text
RecyclerView                    → ListView + ScrollController
Adapter + ViewHolder            → itemBuilder 返回独立 Widget
getItemViewType                 → ChatSender.me 分支布局
notifyDataSetChanged            → ChangeNotifier.notifyListeners()
```



##### StatelessWidget 与 StatefulWidget

在页面设计中，Screen级别（Page页面）需要`StatefulWidget`，view的状态要存储在内部。
自定义view（Item）需要`StatelessWidget`，Item 自己没有状态，所有数据都靠外部传进来。

核心规则：Stateless vs Stateful 变量限制
* StatelessWidget（组件/Item）
  - 内部**只能放 final 常量**
    - 注意：Flutter 里的 final ≠ 数据不能变，这个变量指针被锁定，不能再指向别的对象。
    - dart：final int count;
    - c++：int* const count; // 指针本身不能变
  - 不能放可变变量（不能放普通int、String、bool）
  - 一旦创建，**不可改变**
  - 作用：纯展示、纯渲染、无状态

* StatefulWidget（页面）
  - 放在 State 里的变量**可以随便改**
  - 可以是普通变量、控制器、动态值
  - 改变后调用 setState() 就能刷新 UI
  - 作用：管理 UI 状态、生命周期、控制器


所以对于这两者的理解可以总结为：
1. Android里面的那种简单的`fragment`可以直接用`StatefulWidget`然后就不使用`viewmodel`了
2. 页面是可以实现全部view变量控制交给`viewmodel`管理的，也就意味着完全可以也用`StatelessWidget`

所以引申出三种场景：
* `局部状态页面`
Compose = remember（无 ViewModel）
```kotlin
@Composable
fun SimpleCounterScreen() {
    // 局部 UI 状态（像 Flutter State）
    val count = remember { mutableStateOf(0) }

    Column(
        modifier = Modifier.fillMaxSize(),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text("计数：${count.value}")
        Button(onClick = { count.value++ }) {
            Text("加一")
        }
    }
}
```
Flutter = StatefulWidget（无 ViewModel）
```dart
class SimpleCounterPage extends StatefulWidget {
  const SimpleCounterPage({super.key});

  @override
  State<SimpleCounterPage> createState() => _SimpleCounterPageState();
}

class _SimpleCounterPageState extends State<SimpleCounterPage> {
  // 局部 UI 状态（对应 Compose remember）
  int count = 0;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Text("计数：$count"),
            ElevatedButton(
              onPressed: () {
                setState(() {
                  count++;
                });
              },
              child: const Text("加一"),
            ),
          ],
        ),
      ),
    );
  }
}
```

* 带 ViewModel 的标准页面（MVI/UDF）
Compose + ViewModel（MutableState）
```kotlin
// VM
class CounterVm : ViewModel() {
    val count = mutableStateOf(0)
    fun increment() {
        count.value++
    }
}

// Screen（Compose 天然就是 Stateless 组合函数）
@Composable
fun CounterVmScreen() {
    val vm: CounterVm = viewModel()

    Column(
        modifier = Modifier.fillMaxSize(),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text("计数：${vm.count.value}")
        Button(onClick = { vm.increment() }) {
            Text("加一")
        }
    }
}
```
Flutter + ChangeNotifier VM + Stateless 页面
```dart
// VM
class CounterVm extends ChangeNotifier {
  int count = 0;

  void increment() {
    count++;
    notifyListeners();
  }
}

// 页面完全 Stateless（全靠 VM）
class CounterVmPage extends StatelessWidget {
  const CounterVmPage({super.key});

  @override
  Widget build(BuildContext context) {
    final vm = Provider.of<CounterVm>(context);

    return Scaffold(
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Text("计数：${vm.count}"),
            ElevatedButton(
              onPressed: vm.increment,
              child: const Text("加一"),
            ),
          ],
        ),
      ),
    );
  }
}
```

* 自定义组件（纯展示、无状态）
Compose = 无状态组件
```kotlin
@Composable
fun CounterItem(
    count: Int,
    onClick: () -> Unit
) {
    Column(
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text("当前：$count")
        Button(onClick = onClick) {
            Text("点击")
        }
    }
}
```
Flutter = StatelessWidget
```dart
class CounterItem extends StatelessWidget {
  // final变量
  final int count;
  final VoidCallback onClick;

  const CounterItem({
    super.key,
    required this.count,
    required this.onClick,
  });

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Text("当前：$count"),
        ElevatedButton(
          onPressed: onClick,
          child: const Text("点击"),
        ),
      ],
    );
  }
}
```

##### 自定义列表项 Item

```dart
// contact_message_item.dart — 消息 Tab 会话行
class ContactMessageItem extends StatelessWidget {
  // WeChatAvatar + 名称 + 预览 + 时间 + 未读红点
}

// chat_message_item.dart — 聊天气泡
final isMe = message.sender == ChatSender.me;
Row(
  mainAxisAlignment: isMe ? MainAxisAlignment.end : MainAxisAlignment.start,
  children: [/* 头像 + 气泡 */],
);
```

##### 自定义列表

```dart
// chat_message_list.dart
ListView.separated(
  controller: scrollController,
  itemCount: messages.length + 2, // header「下拉加载更早」+ footer
  ...
);

// 顶部下拉加载历史：NotificationListener<ScrollNotification>
// 条件：pixels <= 0 且向下拖 + canLoadMoreHistory → onLoadMoreHistory
```

`WeChatChatPage` 中 `ScrollController` 监听实现「回到最新消息」悬浮气泡（对应 KMP `derivedStateOf` + `AnimatedVisibility`）。

#### 下拉刷新

**XML：** `SwipeRefreshLayout` 包裹列表。

**Flutter：** `RefreshIndicator` + ViewModel 延迟；Effect 在 Page 层展示 `SnackBar`。

```dart
RefreshIndicator(
  onRefresh: () async {
    processIntent(const WeChatRefreshMessages());
    await Future.delayed(const Duration(seconds: 2));
  },
  child: ListView.separated(...),
)
```

```dart
// wechat_demo_vm.dart
_emitEffect(const WeChatShowToast('刷新成功'));

// wechat_demo_page.dart
_vm.effects.listen((e) {
  if (e is WeChatShowToast) ScaffoldMessenger.of(context).showSnackBar(...);
});
```

#### 消息项缩放进入聊天页

**说明：** Flutter **可以**做与 KMP 相同的进/出叠层缩放，不是 Compose 独有；关键是**同时**渲染离场页与进场页并分别计算 opacity / scale（对应 `fadeIn+scaleIn togetherWith fadeOut+scaleOut`）。

**Compose：** `AnimatedContent` + `weChatTransitionSpec`。

**Flutter：** `WeChatAnimatedContent`（`we_chat_page_transition.dart`）缓存 `oldWidget.child`，`Stack` 两层 + `SpringSimulation`。

| 样式 | PUSH 进场 | PUSH 离场 |
|------|-----------|-----------|
| `messageZoom` | 0.85→1 淡入 | 1→1.03 淡出 |
| `avatarZoom` | 0.7→1 淡入 | 1→1.06 淡出 |

```dart
// wechat_demo_vm.dart
_openChat(userId, WeChatTransitionStyle.messageZoom);
_openProfile(userId, WeChatTransitionStyle.avatarZoom);

// we_chat_page_transition.dart — 与 KMP weChatTransitionSpec 数值对齐
final spec = _zoomSpec(widget.navAction, widget.transitionStyle);
// 进场 opacity: t, scale: lerp(enterScaleBegin, 1, t)
// 离场 opacity: 1-t, scale: lerp(1, exitScaleEnd, t)
```

#### 详情页头像 ViewPager

**XML：** 顶部 `ViewPager2` 正方形头像区。

**Flutter：** `PageView.builder` + `AspectRatio(aspectRatio: 1)`（`we_chat_demo_screen.dart` · `_WeChatProfilePage`）。

```dart
PageView.builder(
  controller: _avatarPageController,
  itemCount: palette.length,
  itemBuilder: (_, page) => ColoredBox(
    color: Color(palette[page]),
    child: Center(child: Text(contact.name)),
  ),
)
```

#### 朋友圈：九宫格 + ViewModel 驱动 UI 更新

##### 网格布局

**XML：** `GridView` / `RecyclerView` + `GridLayoutManager`。

**Flutter：** `moment_photo_grid.dart` — `LayoutBuilder` 算 cell 宽高，`GridView.builder` 固定 3 列、`physics: NeverScrollableScrollPhysics`（对应 `LazyVerticalGrid` + 固定高度）。

```dart
final cellSize = (constraints.maxWidth - spacing * 2) / 3;
final rows = (photos.length + 2) ~/ 3;
final gridHeight = cellSize * rows + spacing * (rows - 1).clamp(0, rows);
```

##### ViewModel 数据联动

**XML + LiveData：** 必须 `observe()` 后手动改 View。

**Flutter：** `ChangeNotifier` + `AnimatedBuilder`；业务状态在 `WeChatDemoVm`，组件局部状态用 `StatefulWidget`（如 `MomentCard` 内评论输入框）。

```dart
// 点赞 — wechat_demo_vm.dart
void _toggleMomentLike(...) {
  _update(_state.copyWith(momentsByUser: updated));
}

// UI 自动重建 — wechat_demo_page.dart
AnimatedBuilder(
  animation: _vm,
  builder: (_, __) => WeChatDemoScreen(state: _vm.state, ...),
);
```

| Compose | Flutter |
|---------|---------|
| `MutableStateFlow` | `ChangeNotifier` + `notifyListeners` |
| `collectAsState()` | `AnimatedBuilder` / `ListenableBuilder` |
| 参数变化触发重组 | `state` 传入子 Widget，父 `notifyListeners` 后重建 |

* Screen 级：状态必须在 `WeChatDemoVm`，不要写在页面 `setState` 里做业务逻辑。

* 组件级：`MomentCard` 内 `TextEditingController` 仅存输入框草稿。

#### Flutter 约束布局（Stack + Positioned）

**XML：** `ConstraintLayout` 锚定返回键、居中头像、底栏双按钮。

**Flutter：** `we_chat_voice_call_layout.dart` 用 `Stack` + `Positioned` + `LayoutBuilder` 表达同等约束（本 Demo 不额外引包，学习阶段与 KMP iOS 的 `Box.align` 思路一致）。

```dart
Stack(
  children: [
    Positioned(top: 24, left: (w - 48) / 2, child: /* 返回 */),
    Positioned(left: (w - 180) / 2, top: (h - 180) / 2, child: WeChatAvatar(size: 180, ...)),
    Positioned(left: 24, bottom: 34, child: FilledButton(/* 静音 */)),
    Positioned(right: 24, bottom: 34, child: FilledButton(/* 挂断 */)),
  ],
)
```

| Compose ConstraintLayout | Flutter |
|------------------------|---------|
| `createRefs()` | `Stack` 子节点 |
| `constrainAs` + `linkTo` | `Positioned` / `Align` |
| `Dimension.fillToConstraints` | `left` + `right` 同时约束 |

#### 页面栈复用（防循环创建 OOM）

**XML：** `launchMode="singleTop"`、Fragment 回栈去重。

**Flutter：** 与 KMP 相同逻辑的 `pushOrReuse`：

```dart
void _pushOrReuse(WeChatPage nextPage, WeChatTransitionStyle style) {
  final existingIndex = _pageStack.indexWhere((p) => p.key == nextPage.key);
  if (existingIndex >= 0) {
    while (_pageStack.length > existingIndex + 1) {
      _pageStack.removeLast();
    }
    _publishNavState(WeChatNavAction.pop, style);
  } else {
    _pageStack.add(nextPage);
    _publishNavState(WeChatNavAction.push, style);
  }
}
```

`WeChatPageChat(userId).key` → `"chat:$userId"`。典型循环：消息 → 聊天 → 头像 → 详情 → 发消息 → 聊天，栈内同 userId 的 Chat/Profile 只保留一份。

通讯录：`WeChatUpdateContactQuery` + `filteredContacts` getter（`wechat_models.dart`）。


更细的 Flutter 笔记可继续写在 [FlutterUI.md](FlutterUI.md)。
