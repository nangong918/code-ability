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
| 1 | 底部 Tab + 横向分页 | `BottomNavigation` + `ViewPager2` | `Scaffold` + `HorizontalPager` + `rememberPagerState` |
| 2 | 自定义列表项 | 自定义 `View` + `RecyclerView.Adapter` | 独立 `@Composable`（`ContactMessageItem`）+ `LazyColumn` |
| 3 | 下拉刷新（假数据） | `SwipeRefreshLayout` | `PullToRefreshBox` + ViewModel `delay` + Toast |
| 4 | 消息项缩放进/chat 页 | `Activity` 转场 / `sharedElement` | 根导航 `AnimatedContent` + `WeChatTransitionStyle.MESSAGE_ZOOM` |
| 5 | 聊天顶部的「加载更早消息」 | `RecyclerView` 顶部 + 手势 / `OnScrollListener` | `LazyListState.firstVisibleItemIndex` + `detectVerticalDragGestures` |
| 6 | 左右两种气泡 | `getItemViewType` + 两种 ViewHolder | `ChatSender` 枚举 + `Row` 的 `Arrangement` 分支 |
| 7 | 头像放大进详情 | Shared Element / 自定义 Transition | `AVATAR_ZOOM` 的 `scaleIn` / `scaleOut` |
| 8 | 详情多图左右滑 | `ViewPager` | `HorizontalPager`（头像色块轮播） |
| 9 | 朋友圈点赞评论实时刷新 | `ViewModel` + `LiveData` / `Observable` | `MutableStateFlow` + `copy` 更新 `momentsByUser` |
| 10 | 语音通话叠层布局 | `ConstraintLayout` | `Box` + `Modifier.align(Alignment.*)` |
| 11 | 页面栈复用防 OOM | `FragmentManager` back stack / `singleTop` | `pageStack` + `pushOrReuse` 按 `page.key` 去重 |
| 12 | **（补充）** 单向数据流 | MVP / 手动 setText | `WeChatIntent` → `processIntent` → `WeChatUiState` |
| 13 | **（补充）** 列表不在底部提示 | 监听 `RecyclerView` 滚动 | `derivedStateOf` + `AnimatedVisibility`「回到最新消息」 |
| 14 | **（补充）** UI 本地预览 | Layout Preview | `@Preview` + `VectorDemoTheme`（同文件底部） |



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


#### 9. 页面栈复用（防循环创建 OOM）

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



### Flutter实现

[FlutterUI.md](FlutterUI.md)

















