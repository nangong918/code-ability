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

```xml

```
```kotlin

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



#### 2. 自定义列表项：`ContactMessageItem` + `LazyColumn`

**XML：** 继承 `RecyclerView.ViewHolder`，在 `item_contact_message.xml` 里摆头像、昵称、预览、时间、红点。

**Compose：** 把一整行拆成可复用 `@Composable`，列表只负责 `items` 循环。

```32:94:magic-vector/demo/kmp/shared/src/commonMain/kotlin/com/vectordemo/ui/view/wechat/components/ContactMessageItem.kt
fun ContactMessageItem(
    preview: WeChatMessagePreview,
    contact: WeChatContact?,
    onClick: () -> Unit,
    onAvatarClick: () -> Unit,
) {
    Row(
        modifier = Modifier.fillMaxWidth().clickable(onClick = onClick).padding(...),
        verticalAlignment = Alignment.CenterVertically,
    ) {
        WeChatAvatar(..., onClick = onAvatarClick)
        Column(modifier = Modifier.weight(1f)) { /* 名称 + 预览 */ }
        Column(horizontalAlignment = Alignment.End) { /* 时间 + 未读角标 */ }
    }
}
```

消息页挂载（≈ `RecyclerView.setAdapter`）：

```220:238:magic-vector/demo/kmp/shared/src/commonMain/kotlin/com/vectordemo/ui/view/wechat/WeChatDemoScreen.kt
    PullToRefreshBox(isRefreshing = state.refreshingMessages, onRefresh = { ... }) {
        LazyColumn {
            items(state.messagePreviews, key = { it.contactId }) { preview ->
                ContactMessageItem(
                    preview = preview,
                    contact = contact,
                    onClick = { processIntent(WeChatIntent.OpenChatFromMessage(preview.contactId)) },
                    onAvatarClick = { processIntent(WeChatIntent.OpenProfileFromAvatar(preview.contactId)) },
                )
                Divider(...)
            }
        }
    }
```

`LazyColumn` ≈ `RecyclerView`；`items(..., key = {})` ≈ stable id，利于复用与动画。

---

#### 3. 下拉刷新（假刷新）

**XML：** 包一层 `SwipeRefreshLayout`，`setOnRefreshListener` 里请求接口。

**Compose：** Material3 `PullToRefreshBox`，刷新状态来自 `state.refreshingMessages`。

ViewModel 侧（2 秒延迟 + Toast，无真实网络）：

```220:227:magic-vector/demo/kmp/shared/src/commonMain/kotlin/com/vectordemo/viewModel/wechat/WeChatDemoVm.kt
    private fun refreshMessages() {
        if (_uiState.value.refreshingMessages) return
        viewModelScope.launch {
            _uiState.update { it.copy(refreshingMessages = true) }
            delay(2000)
            _uiState.update { it.copy(refreshingMessages = false) }
            emitEffect(WeChatEffect.ShowToast("刷新成功"))
        }
    }
```

Toast 通过 `Channel` + `WeChatEffect`，在 `App.kt` 的 `LaunchedEffect(weChatDemoVm)` 里接到全局 `toastMessage`——等价于 XML 里 `Activity.runOnUiThread { Toast }`。

---

#### 4. 消息项缩放进入聊天页

**XML：** `ActivityOptions.makeScaleUpAnimation` 或共享元素 Transition。

**Compose：** 打开聊天时 VM 设 `WeChatTransitionStyle.MESSAGE_ZOOM` + `NavAction.PUSH`；`AnimatedContent` 的 `transitionSpec` 用 `scaleIn` / `scaleOut` + `spring`。

```186:187:magic-vector/demo/kmp/shared/src/commonMain/kotlin/com/vectordemo/viewModel/wechat/WeChatDemoVm.kt
            is WeChatIntent.OpenChatFromMessage -> openChat(intent.userId, WeChatTransitionStyle.MESSAGE_ZOOM)
```

```851:866:magic-vector/demo/kmp/shared/src/commonMain/kotlin/com/vectordemo/ui/view/wechat/WeChatDemoScreen.kt
    WeChatTransitionStyle.MESSAGE_ZOOM -> {
        if (action == WeChatNavAction.PUSH) {
            (fadeIn(...) + scaleIn(initialScale = 0.85f, ...)) togetherWith
                (fadeOut(...) + scaleOut(targetScale = 1.03f, ...))
        } else { /* POP 反向缩放 */ }
    }
```

返回时 `navigateBack()` 对 `WeChatPage.Chat` 同样用 `MESSAGE_ZOOM` 的 POP 分支。

---

#### 5. 聊天列表：顶部下拉加载历史

**XML：** `RecyclerView` 滑到 `position == 0` 且继续下拉时触发加载；或反向 `LinearLayoutManager` + 顶部 footer。

**Compose：** `rememberLazyListState()`；`firstVisibleItemIndex == 0` 时用 `pointerInput` + `detectVerticalDragGestures` 累计 `topDragOffset > 80` 触发 `LoadOlderHistory`。

```444:448:magic-vector/demo/kmp/shared/src/commonMain/kotlin/com/vectordemo/ui/view/wechat/WeChatDemoScreen.kt
    LaunchedEffect(listState.firstVisibleItemIndex, topDragOffset, canLoadMoreHistory) {
        if (listState.firstVisibleItemIndex == 0 && topDragOffset > 80f && canLoadMoreHistory) {
            onLoadMoreHistory()
            topDragOffset = 0f
        }
    }
```

```335:350:magic-vector/demo/kmp/shared/src/commonMain/kotlin/com/vectordemo/viewModel/wechat/WeChatDemoVm.kt
    private fun loadOlderHistory() {
        val userId = _uiState.value.activeChatUserId ?: return
        val all = chatHistoryByUser[userId].orEmpty()
        val loaded = loadedHistoryCountByUser[userId] ?: INITIAL_VISIBLE_HISTORY
        if (loaded >= all.size || _uiState.value.loadingHistory) return
        viewModelScope.launch {
            _uiState.update { it.copy(loadingHistory = true) }
            delay(900)
            loadedHistoryCountByUser[userId] = min(all.size, loaded + HISTORY_PAGE_SIZE)
            refreshVisibleHistory(userId = userId, shouldScrollToLatest = false)
            ...
        }
    }
```

全量历史在 `chatHistoryByUser`，UI 只显示 `visibleChatMessages` 尾部窗口（初始 12 条，每次 +8）——对应 ListView 只 bind 部分数据。

---

#### 6. 两种聊天气泡（itemType）

**XML：** `getItemViewType` 返回 `TYPE_LEFT` / `TYPE_RIGHT`，`onCreateViewHolder` inflate 不同 layout。

**Compose：** 同一 `itemsIndexed`，用 `message.sender == ChatSender.ME` 决定 `Arrangement.End/Start` 和气泡颜色。

```485:527:magic-vector/demo/kmp/shared/src/commonMain/kotlin/com/vectordemo/ui/view/wechat/WeChatDemoScreen.kt
            val isMe = message.sender == ChatSender.ME
            Row(
                horizontalArrangement = if (isMe) Arrangement.End else Arrangement.Start,
                ...
            ) {
                if (!isMe) { WeChatAvatar(...); ... }
                Column(...) {
                    Box(
                        modifier = Modifier
                            .clip(RoundedCornerShape(10.dp))
                            .background(if (isMe) Color(0xFF95EC69) else Color.White)
                            ...
                    ) { Text(message.text) }
                }
                if (isMe) { ... WeChatAvatar(WeChatContact("me", ...)) }
            }
```

发送消息：输入框 `mutableStateOf` 本地持有草稿，点发送发 `WeChatIntent.SendMessage`：

```395:400:magic-vector/demo/kmp/shared/src/commonMain/kotlin/com/vectordemo/ui/view/wechat/WeChatDemoScreen.kt
                Button(onClick = {
                    val msg = input.trim()
                    if (msg.isNotBlank()) {
                        input = ""
                        processIntent(WeChatIntent.SendMessage(msg))
                    }
                }) { Text("发送") }
```

---

#### 7. 头像放大进入用户详情

与目标 4 相同机制，样式为 `AVATAR_ZOOM`（更小 initialScale，模拟头像放大）：

```187:188:magic-vector/demo/kmp/shared/src/commonMain/kotlin/com/vectordemo/viewModel/wechat/WeChatDemoVm.kt
            is WeChatIntent.OpenProfileFromAvatar -> openProfile(intent.userId, WeChatTransitionStyle.AVATAR_ZOOM)
```

通讯录点整行进入详情用 `NORMAL`（无缩放强调）：

```188:188:magic-vector/demo/kmp/shared/src/commonMain/kotlin/com/vectordemo/viewModel/wechat/WeChatDemoVm.kt
            is WeChatIntent.OpenProfileFromContact -> openProfile(intent.userId, WeChatTransitionStyle.NORMAL)
```

---

#### 8. 详情页头像 ViewPager

**XML：** `ViewPager2` + `FragmentStateAdapter`。

**Compose：** 详情顶部 `HorizontalPager(pageCount = contact.avatarPalette.size)`，每页一个色块 + 居中名字（Demo 用调色板代替多张照片）。

```572:590:magic-vector/demo/kmp/shared/src/commonMain/kotlin/com/vectordemo/ui/view/wechat/WeChatDemoScreen.kt
            HorizontalPager(
                state = avatarPagerState,
                modifier = Modifier.fillMaxWidth().aspectRatio(1f),
            ) { page ->
                Box(
                    modifier = Modifier.fillMaxSize()
                        .background(Color(contact.avatarPalette[page].toLong())),
                    contentAlignment = Alignment.Center,
                ) { Text(contact.name, ...) }
            }
```

朋友圈整体在 **`LazyColumn` 里向下滚**（不是 NestedScrollView 包 WebView，而是 Compose 嵌套：外层 `LazyColumn` + 内层 `LazyVerticalGrid` 九宫格）。

---

#### 9. 朋友圈：九宫格 + ViewModel 驱动 UI 更新

**XML：** `GridView` / `RecyclerView GridLayoutManager`；点赞后 `notifyItemChanged`；评论 `EditText` + 提交。

**Compose：**

- 九宫格：`LazyVerticalGrid(GridCells.Fixed(3))` + `BoxWithConstraints` 算 cell 高度（避免网格在 `LazyColumn` 里高度未知）。
- 点赞 / 评论：Intent 进 VM，改 `momentsByUser` immutable map。

```369:381:magic-vector/demo/kmp/shared/src/commonMain/kotlin/com/vectordemo/viewModel/wechat/WeChatDemoVm.kt
    private fun toggleMomentLike(userId: String, momentId: String) {
        val updated = _uiState.value.momentsByUser.toMutableMap()
        val rows = updated[userId].orEmpty().map { moment ->
            if (moment.id != momentId) moment
            else if (moment.liked) moment.copy(liked = false, likeCount = ...)
            else moment.copy(liked = true, likeCount = moment.likeCount + 1)
        }
        updated[userId] = rows
        _uiState.update { it.copy(momentsByUser = updated) }
    }
```

`MomentCard` 内 `moment.comments.forEach` 会自动随 state 重组，等价于 `ChangeNotifier` / `LiveData` 通知 View 刷新。

---

#### 10. 语音通话页：`Box` 对齐（ConstraintLayout）

**XML：** `ConstraintLayout` 约束头像居中、按钮贴底左右。

**Compose：** 单层 `Box(fillMaxSize)` + 子项 `Modifier.align`：

```787:846:magic-vector/demo/kmp/shared/src/commonMain/kotlin/com/vectordemo/ui/view/wechat/WeChatDemoScreen.kt
    Box(modifier = Modifier.fillMaxSize().background(Color(0xFF121212))) {
        Box(modifier = Modifier.align(Alignment.TopCenter).padding(top = 24.dp), ...) { Text("返回") }
        WeChatAvatar(modifier = Modifier.align(Alignment.Center).size(180.dp), ...)
        Column(modifier = Modifier.align(Alignment.TopCenter).padding(top = 90.dp), ...) { /* 姓名 + 时长 */ }
        Box(modifier = Modifier.align(Alignment.BottomCenter).fillMaxWidth().padding(...)) {
            Button(modifier = Modifier.align(Alignment.CenterStart).width(120.dp), ...) { Text("静音") }
            Button(modifier = Modifier.align(Alignment.CenterEnd).width(120.dp), ...) { Text("挂断") }
        }
    }
```

通话计时在 VM：`startCallTimer()` 每秒 `callDurationSeconds++`（≈ `Handler.postDelayed` 循环）。

---

#### 11. 页面栈复用（防循环创建 OOM）

**XML：** `launchMode="singleTop"`、`FragmentTransaction` 带 tag 复用、或 Navigation `popUpTo` + `launchSingleTop`。

**Compose Demo：** 内存栈 `pageStack`，`pushOrReuse` 若 `nextPage.key` 已存在则 **pop 到该页**，不重复 new：

```295:309:magic-vector/demo/kmp/shared/src/commonMain/kotlin/com/vectordemo/viewModel/wechat/WeChatDemoVm.kt
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

---

#### 12. （补充）单向数据流 MVI

| 概念 | 本 Demo |
|------|---------|
| Intent | `sealed class WeChatIntent`（用户操作） |
| State | `data class WeChatUiState` + `StateFlow` |
| Effect | `WeChatEffect.ShowToast`（一次性副作用） |

```181:199:magic-vector/demo/kmp/shared/src/commonMain/kotlin/com/vectordemo/viewModel/wechat/WeChatDemoVm.kt
    fun processIntent(intent: WeChatIntent) {
        when (intent) {
            is WeChatIntent.SelectTab -> _uiState.update { it.copy(activeTab = intent.tab) }
            WeChatIntent.RefreshMessages -> refreshMessages()
            is WeChatIntent.OpenChatFromMessage -> openChat(...)
            ...
            WeChatIntent.NavigateBack -> navigateBack()
        }
    }
```

UI **不**直接改 `pageStack`，只 `processIntent`——和 XML 里 Presenter 收事件再改 Model 一样。

---

#### 13. （补充）「回到最新消息」气泡

**XML：** 监听 `RecyclerView` 是否滑到底部，显示 `FloatingActionButton` 或 Snackbar 样式条。

**Compose：** `derivedStateOf` 读 `listState.layoutInfo`，最后可见 index `< total - 1` 时显示；点击 `animateScrollToItem(lastIndex)`。

```321:428:magic-vector/demo/kmp/shared/src/commonMain/kotlin/com/vectordemo/ui/view/wechat/WeChatDemoScreen.kt
    val showBackToLatest by remember {
        derivedStateOf {
            val visibleInfo = listState.layoutInfo.visibleItemsInfo
            ...
            total > 0 && lastVisible < total - 1
        }
    }
    AnimatedVisibility(visible = showBackToLatest, ...) {
        Text("回到最新消息", modifier = Modifier.clickable { listState.animateScrollToItem(...) })
    }
```

新消息应滚到底：`scrollToLatestToken` 在 VM 递增，`LaunchedEffect` 触发 `animateScrollToItem`。

---

#### 14. （补充）通讯录搜索

**XML：** `EditText` + `TextWatcher` 过滤 Adapter 列表。

**Compose：** `OutlinedTextField` → `UpdateContactQuery`；`WeChatUiState.filteredContacts` 计算属性过滤。

```130:138:magic-vector/demo/kmp/shared/src/commonMain/kotlin/com/vectordemo/viewModel/wechat/WeChatDemoVm.kt
    val filteredContacts: List<WeChatContact>
        get() = if (contactQuery.isBlank()) contacts
        else contacts.filter { it.name.contains(...) || it.subtitle.contains(...) }
```

---

#### 15. （补充）`@Preview` 不跑真机也能看 UI

`WeChatDemoScreen.kt` / `ContactMessageItem.kt` 底部多个 `@Preview`，传入假 `WeChatUiState`，Android Studio Compose Preview 即可对照 XML 的 Layout Editor。

---

#### 常用 Modifier 与 XML 对照速查

| Compose | 近似 XML |
|---------|----------|
| `Modifier.fillMaxWidth()` | `layout_width="match_parent"` |
| `Modifier.padding(12.dp)` | `android:padding="12dp"` |
| `Modifier.weight(1f)`（在 Row/Column 内） | `layout_weight="1"` |
| `Modifier.clickable { }` | `android:onClick` / `setOnClickListener` |
| `Modifier.clip(RoundedCornerShape)` | `shape` / `ViewOutlineProvider` |
| `Arrangement.SpaceEvenly` | `LinearLayout` 等分间距 |
| `LazyColumn` + `items` | `RecyclerView` |
| `AnimatedContent` / `AnimatedVisibility` | `TransitionManager` / 属性动画 |
| `Box` + `align` | `ConstraintLayout` 约束 |
| `remember` / `mutableStateOf` | 成员变量 + `invalidate` |
| `collectAsState()` | 观察 `LiveData` |

---

#### 学习建议（从 XML 转 Compose）

1. 先画 **状态树**：本 Demo 是 `WeChatPage`（全屏）× `WeChatTab`（Home 内分页），不要和 `AppRoute` 混在一层。
2. 列表一律想 **「数据 + LazyColumn」**，不要找 `Adapter`；item 类型用 `when`/枚举分支，不是 `viewType` 整数。
3. 动画优先挂在 **导航容器**（`AnimatedContent`）上，而不是每个 item。
4. 业务逻辑放 **ViewModel + Intent**，Compose 函数保持纯 UI，方便 Preview 和 KMP 共享。
5. 在 IDE 里打开 `WeChatDemoScreen.kt` 的 Preview 面板，对照本文各节代码逐块改参数观察效果。

更细的 Compose-only 笔记可继续写在 [JetpackComposeUI.md](JetpackComposeUI.md)。



### Flutter实现

[FlutterUI.md](FlutterUI.md)

















