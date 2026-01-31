---
name: android-flow-architecture
description: AndroidプロジェクトでのViewModel、Flow、Repository、UIの実装パターンガイド。新規機能実装時やリファクタリング時に参照。Jetpack Compose + Koin + Ktor環境向け。
---

# ViewModel/Flow/Repository 実装ガイド

このスキルは、AndroidプロジェクトでのViewModel、Flow、Repository、UIの実装パターンを提供します。
Jetpack Compose + Koin（DI）+ Ktor（HTTP Client）環境を想定しています。

## 1. ViewModel実装パターン

### 基本構造

```kotlin
class MyViewModel : ViewModel(), KoinComponent {
    // 依存性注入
    private val repository: MyRepository by inject()

    // プライベートMutableStateFlow（書き込み）
    private val _uiState = MutableStateFlow(UiState())
    // パブリックStateFlow（読み取り専用）
    val uiState: StateFlow<UiState> = _uiState
        .stateIn(viewModelScope, SharingStarted.Lazily, UiState())

    // 初期化処理
    init {
        loadData()
    }

    private fun loadData() {
        viewModelScope.launch {
            // 非同期処理
        }
    }
}
```

### リロードトリガーパターン

データの再読み込みに使用：

```kotlin
class EditorViewModel(
    private val dateString: String,
) : ViewModel(), KoinComponent {
    private val repository: PostRepository by inject()

    // リロードトリガー
    private val reloadTrigger = MutableStateFlow(0L)

    @OptIn(ExperimentalCoroutinesApi::class)
    private val resource = user.flatMapLatest { user ->
        reloadTrigger.flatMapLatest {
            repository.getPostsFlow(date, user.userId)
        }
    }.stateIn(viewModelScope, SharingStarted.Lazily, Resource.none())

    fun reload() {
        reloadTrigger.update {
            Clock.System.now().toEpochMilliseconds()
        }
    }
}
```

### データ範囲管理パターン

日付範囲などのパラメータ管理に使用：

```kotlin
class StatsViewModel(val userId: Long): ViewModel(), KoinComponent {
    private data class ReloadTrigger(
        val start: LocalDate,
        val end: LocalDate,
        val trigger: Long
    )

    private val reloadTrigger = MutableStateFlow(
        ReloadTrigger(getDefaultStartDate(), getDefaultEndDate(), 0L)
    )

    @OptIn(ExperimentalCoroutinesApi::class)
    private val resource = reloadTrigger.flatMapLatest { (start, end, _) ->
        repository.getUserStatsWithRange(userId, start, end)
    }.map { Resource.success(it) }
     .onStart { emit(Resource.loading()) }
     .catch { emit(Resource.error(it)) }
     .stateIn(viewModelScope, SharingStarted.Lazily, Resource.none())

    fun setDateRange(start: LocalDate, end: LocalDate) {
        reloadTrigger.update {
            ReloadTrigger(start = start, end = end, trigger = it.trigger)
        }
    }
}
```

### フォーム管理パターン

入力フォームの状態管理に使用：

```kotlin
class LoginViewModel : ViewModel(), KoinComponent {
    private val loginRepository: LoginRepository by inject()

    // 入力フィールド
    private val _username = MutableStateFlow("")
    val username = _username.stateIn(viewModelScope, SharingStarted.Lazily, "")

    private val _password = MutableStateFlow("")
    val password = _password.stateIn(viewModelScope, SharingStarted.Lazily, "")

    // バリデーション
    val isPostEnabled = combine(_username, _password) { name, password ->
        validateInput(name, password)
    }.stateIn(viewModelScope, SharingStarted.Lazily, false)

    // ローディング状態
    private val postMetadata = MutableStateFlow(Resource.none<Unit>())
    val isLoading = isLoading(postMetadata)
        .stateIn(viewModelScope, SharingStarted.Lazily, false)

    // 更新メソッド
    fun updateName(name: String) {
        _username.update { name }
    }

    fun updatePassword(password: String) {
        _password.update { password }
    }

    // 送信処理
    fun onLogin() {
        postMetadata.update { Resource.loading() }
        viewModelScope.launch {
            runCatching {
                loginRepository.login(username.value, password.value)
            }.onSuccess {
                postMetadata.update { Resource.success(Unit) }
            }.onFailure { throwable ->
                postMetadata.update { Resource.error(throwable) }
            }
        }
    }
}
```

### Resourceパターンでの統一的な状態管理

```kotlin
// Resource定義
sealed class Resource<T>(
    val data: T? = null,
    val throwable: Throwable? = null,
    val status: Status
) {
    class Success<T>(data: T) : Resource<T>(data = data, status = Status.SUCCESS)
    class Loading<T>(data: T? = null) : Resource<T>(data = data, status = Status.LOADING)
    class Error<T>(throwable: Throwable, data: T? = null) : Resource<T>(
        data = data,
        throwable = throwable,
        status = Status.ERROR
    )
    class None<T> : Resource<T>(status = Status.NONE)

    companion object {
        fun <T> success(data: T) = Success(data)
        fun <T> loading(data: T? = null) = Loading(data)
        fun <T> error(throwable: Throwable, data: T? = null) = Error(throwable, data)
        fun <T> none() = None<T>()
    }

    fun isLoading() = status == Status.LOADING
    fun isSuccess() = status == Status.SUCCESS
    fun isError() = status == Status.ERROR
}

enum class Status {
    SUCCESS, ERROR, LOADING, NONE
}

// Resourceを使った状態管理
private val postMetadata = MutableStateFlow(Resource.none<Unit>())

val isLoading: StateFlow<Boolean> = isLoading(resource, postMetadata, statsResource)
    .stateIn(viewModelScope, SharingStarted.Lazily, false)

val errorMessage = getErrorMessage(resource, postMetadata, statsResource)
    .stateIn(viewModelScope, SharingStarted.Lazily, null)

fun removeItem(itemId: Long) {
    viewModelScope.launch {
        if (postMetadata.value.isLoading()) return@launch
        runCatching {
            postMetadata.value = Resource.loading()
            repository.removeItem(itemId)
        }.onSuccess {
            postMetadata.value = Resource.success(Unit)
        }.onFailure { throwable ->
            postMetadata.value = Resource.error(throwable)
        }
    }
}

// isLoading ヘルパー関数
fun isLoading(vararg resources: Flow<Resource<*>?>): Flow<Boolean> {
    return combine(resources.toList()) { values ->
        values.any { it?.isLoading() == true }
    }
}

// getErrorMessage ヘルパー関数
fun getErrorMessage(vararg resources: Flow<Resource<*>?>): Flow<ErrorMessage?> {
    return flow {
        val values = arrayOfNulls<Throwable?>(resources.size)
        resources.mapIndexed { index, flow ->
            flow.map { index to it?.throwable }
        }.merge().collect { (index, e) ->
            values[index] = e
            emit(values.firstOrNull { it != null })
        }
    }.map { e -> e?.let { ErrorMessage.of(it) } }
}
```

## 2. Flow使用パターン

### StateFlowとSharingStarted

```kotlin
// SharingStarted.Lazily - 最初のサブスクライバが現れたときに開始
val user = currentUser
    .mapNotNull { it.toEntity().toUiState() }
    .stateIn(viewModelScope, SharingStarted.Lazily, null)

// SharingStarted.WhileSubscribed() - アクティブなサブスクライバがいる間のみ
val items = itemList
    .mapNotNull { list -> list?.map { it.toItemState() }?.toImmutableList() }
    .stateIn(viewModelScope, SharingStarted.WhileSubscribed(), persistentListOf())

// SharingStarted.Eagerly - 即座に開始（非推奨、特別な理由がない限り使用しない）
```

### Flow変換オペレーター

```kotlin
// map - 値の変換
val userId = user.mapNotNull { it?.userId }
    .stateIn(viewModelScope, SharingStarted.Lazily, -1L)

// flatMapLatest - 最新の値に基づいて新しいFlowを生成
@OptIn(ExperimentalCoroutinesApi::class)
val resource = user.flatMapLatest { u ->
    repository.getItemsFlow(date, u.userId)
}.stateIn(viewModelScope, SharingStarted.Lazily, Resource.none())

// combine - 複数のFlowを結合
val isPostEnabled = combine(_username, _password) { name, password ->
    validateInput(name, password)
}.stateIn(viewModelScope, SharingStarted.Lazily, false)

// filterNotNull - null値を除外
private val currentUser = userPreference.currentUser
    .stateIn(viewModelScope, SharingStarted.WhileSubscribed(), null)
    .filterNotNull()
```

### SharedFlow - イベント管理

```kotlin
// DataSourceでのイベント通知
internal class ItemDataSource: KoinComponent {
    sealed interface Action {
        val id: Long
        data class Remove(override val id: Long): Action
        data class Update(override val id: Long): Action
        data class Add(override val id: Long): Action
    }

    val action = MutableSharedFlow<Action>()
}

// イベントの発行
dataSource.action.emit(ItemDataSource.Action.Remove(id))

// イベントの購読
dataSource.action.collect { action ->
    when(action) {
        is ItemDataSource.Action.Remove -> handleRemove(action)
        is ItemDataSource.Action.Update -> handleUpdate(action)
        is ItemDataSource.Action.Add -> handleAdd(action)
    }
}
```

## 3. Repository実装パターン

### 基本構造

```kotlin
// インターフェース定義
interface ItemRepository {
    suspend fun removeItem(itemId: Long): Unit
    suspend fun addItem(item: RequestItemEntity): Unit
    suspend fun updateItem(itemId: Long, item: RequestItemEntity): Unit
    fun getItemsFlow(date: LocalDate, userId: Long): Flow<Resource<List<Item>>>
}

// 実装
class ItemRepositoryImpl : ItemRepository, KoinComponent {
    private val dataSource: ItemDataSource by inject()
    private val api: ItemsApi by inject()

    override suspend fun removeItem(itemId: Long) {
        withContext(Dispatchers.IO) {
            api.deleteItem(itemId).tryBody()
        }.also {
            // 変更を通知
            dataSource.action.emit(ItemDataSource.Action.Remove(itemId))
        }
    }
}
```

### リアクティブなデータストリーム

```kotlin
override fun getItemsFlow(date: LocalDate, userId: Long) = channelFlow {
    val range = date.getFirstAndLastDayOfMonth()
    val res = getItems(userId, range.first, range.second)
    send(res)

    // 変更を監視して自動更新
    launch {
        dataSource.action.collect { action ->
            when(action) {
                is ItemDataSource.Action.Remove,
                is ItemDataSource.Action.Update,
                is ItemDataSource.Action.Add -> {
                    if (action.id in range) {
                        val newList = getItems(userId, range.first, range.second)
                        send(newList)
                    }
                }
            }
        }
    }
}.map { Resource.success(it) }
 .onStart { emit(Resource.loading()) }
 .catch { emit(Resource.error(it)) }
 .flowOn(Dispatchers.IO)
```

### ページネーションパターン

```kotlin
interface UserRepository {
    data class UserInfoResponse(
        val users: List<UserEntity>,
        val nextCursor: Long?
    )

    suspend fun getUsers(cursor: Long? = null): UserInfoResponse
    fun getUsersFlow(cursor: Flow<Long?>): Flow<UserInfoResponse>
}

class UserRepositoryImpl : UserRepository, KoinComponent {
    private val api: UsersApi by inject()
    private val LIMIT = 20L

    override fun getUsersFlow(cursor: Flow<Long?>) = channelFlow {
        var mutableList: MutableList<UserEntity> = mutableListOf()
        cursor.collect { c ->
            val res = getUsers(c)
            if (c == null) {
                // リロード時はリストをクリア
                mutableList.clear()
            }
            mutableList.addAll(res.users)
            send(UserInfoResponse(
                users = mutableList.toList(),
                nextCursor = res.nextCursor
            ))
        }
    }.flowOn(Dispatchers.IO)
}
```

## 4. UI実装パターン（Compose）

### Screen - 状態管理層

```kotlin
@Composable
internal fun LoginScreen(
    navigateToHome: () -> Unit,
    viewModel: LoginViewModel = koinInject()
) {
    // StateFlowを監視
    val isLoading by viewModel.isLoading.collectAsStateWithLifecycle()
    val errorMessage by viewModel.dialogErrorMessage.collectAsStateWithLifecycle()
    val username by viewModel.username.collectAsStateWithLifecycle()
    val password by viewModel.password.collectAsStateWithLifecycle()
    val isPostEnabled by viewModel.isPostEnabled.collectAsStateWithLifecycle()

    // Content層へ委譲（プレビュー可能）
    LoginContent(
        username = { username },
        onUsernameChange = viewModel::updateName,
        password = { password },
        onPasswordChange = viewModel::updatePassword,
        onClickLogin = viewModel::onLogin,
        loginEnabled = { isPostEnabled },
        isLoading = isLoading
    )

    // 条件付きUI
    if (errorMessage != null) {
        ErrorDialog(
            message = errorMessage?.getErrorMessage(),
            onDismiss = viewModel::clearEvent,
            onRetry = viewModel::clearEvent
        )
    }
}
```

### Content - 表示層（プレビュー可能）

```kotlin
@Composable
fun EditorTabContent(
    items: ImmutableList<ItemState>,
    monthlyStats: MonthWorkStats?,
    isLoading: Boolean,
    isRefreshing: Boolean,
    onClickAddNewItem: () -> Unit,
    onClickRemoveItem: (id: Long) -> Unit,
    onClickItem: (ItemState) -> Unit,
    onRefresh: () -> Unit,
    modifier: Modifier = Modifier,
    contentPadding: PaddingValues = PaddingValues(0.dp),
    pullToRefreshState: PullToRefreshState = rememberPullToRefreshState()
) {
    PullToRefreshBox(
        modifier = modifier,
        isRefreshing = isRefreshing,
        state = pullToRefreshState,
        onRefresh = onRefresh,
    ) {
        LazyColumn(modifier = Modifier.fillMaxSize()) {
            items(items = items, key = { i -> i.id }) { state ->
                ItemRow(
                    state = state,
                    onClick = { onClickItem(state) },
                    onClickRemove = { onClickRemoveItem(state.id) }
                )
            }
        }
    }
}
```

### リフレッシュとローディング状態管理

```kotlin
@Composable
fun MyScreen(viewModel: MyViewModel = koinInject()) {
    val isLoading by viewModel.isLoading.collectAsStateWithLifecycle()

    var isRefreshing by rememberSaveable { mutableStateOf(false) }

    // ローディングが終わったらリフレッシュ状態をリセット
    if (isRefreshing) {
        LaunchedEffect(isLoading) {
            isRefreshing = isLoading
        }
    }

    MyContent(
        isLoading = isLoading,
        isRefreshing = isRefreshing,
        onRefresh = {
            viewModel.reload()
            isRefreshing = true
        }
    )
}
```

### Koin Scopeでの動的ViewModel管理

タブごとに独立したViewModelインスタンスが必要な場合：

```kotlin
@Composable
fun EditorTabScreen(date: LocalDate) {
    val scope: Scope = getKoin().getScopeOrNull(date.toString())
        ?: getKoin().createScope(date.toString(), named(EditViewScope))
    val viewModel: EditorViewModel = scope.get(
        parameters = { parametersOf(date.toString()) }
    )

    val user by viewModel.user.collectAsStateWithLifecycle(null)
    // ...
}
```

## 5. データモデル実装パターン

### UIState定義

```kotlin
@Immutable
data class ItemState(
    val id: Long,
    val userId: Long,
    val date: LocalDate,
    val inTime: LocalTime,
    val outTime: LocalTime,
    val workRate: Int,
    val expenses: Int,
    val description: String,
    val isError: Boolean
) {
    // 計算プロパティ
    val visibleList: List<String>
        get() = listOf(
            date.toDateString,
            inTime.toTimeString,
            outTime.toTimeString,
            workRate.toString(),
            expenses.toString()
        )

    val isNewItem: Boolean
        get() = id == 0L

    companion object {
        // デフォルト値
        val empty = ItemState(
            id = 0L,
            userId = 0L,
            date = LocalDate(2023, 1, 1),
            inTime = LocalTime(9, 0),
            outTime = LocalTime(18, 0),
            workRate = 100,
            expenses = 0,
            description = "",
            isError = false,
        )

        // ファクトリーメソッド
        fun createNew(date: LocalDate, userId: Long): ItemState {
            return empty.copy(id = 0L, userId = userId, date = date)
        }
    }
}
```

### Enumでの型安全な状態管理

```kotlin
data class StatsComparison(
    val weeklyAverage: Comparison?,
    val totalHours: Comparison?,
) {
    enum class Comparison {
        UP, DOWN, SAME;

        fun image() = when (this) {
            UP -> LottieImage.Increase
            DOWN -> LottieImage.Decrease
            SAME -> LottieImage.FlatArrow
        }

        val color @Composable get() = when (this) {
            UP ->  Color(0xFF47D747)
            DOWN -> Color(0xFFCB1251)
            SAME -> Color(0xFF888888)
        }
    }
}
```

### エンティティ変換パターン

```kotlin
// 拡張関数で変換ロジックを記述
fun MonthlyStatsEntity.toUiState(): MonthWorkStats {
    return MonthWorkStats(
        year = year.toInt(),
        month = month.toInt(),
        totalHours = totalHours,
        averageHours = averageHours,
        lestHours = lestHours.toFloat(),
        comparison = comparison.toUiState()
    )
}

fun ComparisonEntity.toUiState(): StatsComparison {
    return StatsComparison(
        weeklyAverage = weeklyAverageComparison?.toUiState(),
        totalHours = totalHoursComparison?.toUiState()
    )
}
```

## ベストプラクティス

### 1. ViewModelでのDO/DON'T

**DO:**
- `ViewModel()` + `KoinComponent` を実装
- `viewModelScope` でコルーチンを管理
- プライベート `MutableStateFlow` とパブリック `StateFlow` を分離
- `Resource` パターンで統一的な状態管理
- `runCatching` でエラーハンドリング

**DON'T:**
- ViewModelでUI関連のクラス（Context、View等）を保持しない
- GlobalScopeを使用しない
- StateFlowを直接公開しない（MutableStateFlowを外部に公開しない）

### 2. Flowでのベストプラクティス

**DO:**
- `stateIn` でFlowをStateFlowに変換
- 適切な `SharingStarted` を選択
- `flowOn(Dispatchers.IO)` でスレッドを指定
- `distinctUntilChanged()` で重複排除

**DON'T:**
- UIスレッドでブロッキング操作を行わない
- Flowのコレクションを忘れない（デッドコードになる）

### 3. RepositoryでのDO/DON'T

**DO:**
- インターフェースと実装を分離
- `withContext(Dispatchers.IO)` でIO操作
- `channelFlow` でリアクティブなストリーム
- DataSourceで変更通知を管理

**DON'T:**
- Repository内でViewModelに依存しない
- 直接UIに関わるロジックを含めない

### 4. UIでのDO/DON'T

**DO:**
- Screen（状態管理）とContent（表示）を分離
- `collectAsStateWithLifecycle()` を使用
- `@Immutable` データクラスで最適化
- メソッド参照でコールバックを渡す

**DON'T:**
- ContentレベルでViewModelに依存しない
- `collectAsState()` は使わず `collectAsStateWithLifecycle()` を使う

## 実装時の確認事項

新規機能実装時は以下を確認：

1. ViewModelは `ViewModel()` + `KoinComponent` を実装しているか
2. StateFlowは適切な `SharingStarted` を使用しているか
3. Repositoryはインターフェースと実装が分離されているか
4. UIStateは `@Immutable` アノテーションが付いているか
5. Screen/Content層が適切に分離されているか
6. エラーハンドリングは `Resource` パターンで統一されているか
7. 非同期処理は `viewModelScope.launch` または `withContext` を使用しているか
8. Flowは適切な Dispatcher（主に `Dispatchers.IO`）で実行されているか

## トラブルシューティング

### StateFlowが更新されない

```kotlin
// 悪い例
_uiState.value.copy(name = newName)  // 参照が変わらないので更新されない

// 良い例
_uiState.update { it.copy(name = newName) }
// または
_uiState.value = _uiState.value.copy(name = newName)
```

### flatMapLatestが期待通りに動かない

```kotlin
// ExperimentalCoroutinesApiアノテーションを追加
@OptIn(ExperimentalCoroutinesApi::class)
val resource = trigger.flatMapLatest { ... }
```

### collectAsStateWithLifecycleが使えない

```kotlin
// build.gradle.ktsに追加されているか確認
// implementation("androidx.lifecycle:lifecycle-runtime-compose:x.x.x")
```

## 関連ドキュメント

詳細な情報は以下のドキュメントを参照してください：

- [architecture.md](./architecture.md) - UI/Logic分離アーキテクチャの詳細
- [examples.md](./examples.md) - 完全な実装例集
