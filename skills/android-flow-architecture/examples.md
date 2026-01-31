# 実装例集

このドキュメントには、実際のプロジェクトで使用できる詳細な実装例が含まれています。

## 完全なViewModel実装例

### 1. EditorViewModel - 複雑な状態管理

```kotlin
class EditorViewModel(
    private val dateString: String,
) : ViewModel(), KoinComponent {
    val date = LocalDate.parse(dateString)

    // 依存性注入
    private val itemRepository: ItemRepository by inject()
    private val userRepository: UserRepository by inject()
    private val userPreference: UserPreference by inject()

    // リロードトリガー
    val reloadTrigger = MutableStateFlow(0L)

    // ユーザー情報の取得
    private val currentUser = userPreference.currentUser
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(), null)
        .filterNotNull()

    val user = currentUser
        .mapNotNull { it.toEntity().toUiState() }
        .stateIn(viewModelScope, SharingStarted.Lazily, null)

    private val userId = user.mapNotNull { it?.userId }
        .stateIn(viewModelScope, SharingStarted.Lazily, -1L)

    // データのフェッチング（リアクティブ）
    @OptIn(ExperimentalCoroutinesApi::class)
    private val resource = user.flatMapLatest { user ->
        reloadTrigger.flatMapLatest {
            itemRepository.getItemsFlow(date, user.userId)
        }
    }.stateIn(viewModelScope, SharingStarted.Lazily, Resource.none())

    // アイテム一覧
    private val itemList = resource.mapNotNull { it.data }
        .stateIn(viewModelScope, SharingStarted.Lazily, null)

    val items = itemList
        .mapNotNull { list -> list?.map { it.toItemState() }?.toImmutableList() }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(), persistentListOf())

    // 統計情報のフェッチング
    @OptIn(ExperimentalCoroutinesApi::class)
    private val statsResource = userId.flatMapLatest { id ->
        reloadTrigger.flatMapLatest {
            userRepository.getUserStats(id, date.year, date.monthNumber)
        }
    }.map { Resource.success(it) }
        .onStart { emit(Resource.loading()) }
        .catch { emit(Resource.error(it)) }
        .stateIn(viewModelScope, SharingStarted.Lazily, Resource.none())

    val stats = statsResource.mapNotNull { it.data?.toUiState() }
        .stateIn(viewModelScope, SharingStarted.Lazily, null)

    // POST操作のメタデータ
    private val postMetadata = MutableStateFlow(Resource.none<Unit>())

    // ローディング状態の統合（複数のResourceを監視）
    val isLoading: StateFlow<Boolean> = isLoading(resource, postMetadata, statsResource)
        .stateIn(viewModelScope, SharingStarted.Lazily, false)

    // エラーメッセージの統合
    val errorMessage = getErrorMessage(resource, postMetadata, statsResource)
        .stateIn(viewModelScope, SharingStarted.Lazily, null)

    // アイテム削除
    fun removeItem(itemId: Long) {
        viewModelScope.launch {
            if (postMetadata.value.isLoading()) return@launch
            val item = itemList.value?.find { it.id == itemId } ?: return@launch

            runCatching {
                postMetadata.value = Resource.loading()
                itemRepository.removeItem(item.id)
            }.onSuccess {
                postMetadata.value = Resource.success(Unit)
            }.onFailure { throwable ->
                postMetadata.value = Resource.error(throwable)
            }
        }
    }

    // 同期（強制リロード）
    @OptIn(ExperimentalTime::class)
    fun sync() {
        reloadTrigger.update { Clock.System.now().toEpochMilliseconds() }
    }

    // エラーのクリア
    fun clearEvent() {
        postMetadata.value = Resource.none()
    }
}
```

**ポイント**:
- 複数のリポジトリからデータを取得
- `flatMapLatest` でリアクティブなデータフェッチング
- `reloadTrigger` で強制リロード
- 複数の `Resource` を統合して `isLoading` と `errorMessage` を生成
- アイテム削除時のローディング状態管理

### 2. LoginViewModel - フォーム管理

```kotlin
internal class LoginViewModel : ViewModel(), DefaultLifecycleObserver, KoinComponent {
    private val loginRepository: LoginRepository by inject()
    private val userPreference: UserPreference by inject()

    // 入力フィールド
    private val _username = MutableStateFlow("")
    val username = _username.stateIn(viewModelScope, SharingStarted.Lazily, "")

    private val _password = MutableStateFlow("")
    val password = _password.stateIn(viewModelScope, SharingStarted.Lazily, "")

    // バリデーション（複数のFlowを結合）
    val isPostEnabled = combine(_username, _password) { name, password ->
        validateInput(name, password)
    }.stateIn(viewModelScope, SharingStarted.Lazily, false)

    private fun validateInput(username: String, password: String): Boolean {
        return username.isNotEmpty() && password.isNotEmpty()
    }

    // POST操作のメタデータ
    private val postMetadata = MutableStateFlow(Resource.none<Unit>())

    // ローディング状態
    val isLoading = isLoading(postMetadata)
        .stateIn(viewModelScope, SharingStarted.Lazily, false)

    // ダイアログ用のエラーメッセージ
    val dialogErrorMessage = getErrorMessage(postMetadata)
        .stateIn(viewModelScope, SharingStarted.Lazily, null)

    // ログイン成功イベント
    private val _loginSuccess = MutableSharedFlow<Unit>()
    val loginSuccess = _loginSuccess.asSharedFlow()

    // 入力更新
    fun updateName(name: String) {
        _username.update { name }
    }

    fun updatePassword(password: String) {
        _password.update { password }
    }

    // ログイン処理
    fun onLogin() {
        postMetadata.update { Resource.loading() }
        viewModelScope.launch {
            runCatching {
                loginRepository.login(username.value, password.value)
            }.onSuccess {
                postMetadata.update { Resource.success(Unit) }
                _loginSuccess.emit(Unit)
            }.onFailure { throwable ->
                postMetadata.update { Resource.error(throwable) }
            }
        }
    }

    fun clearEvent() {
        postMetadata.update { Resource.none() }
    }

    // ライフサイクル対応（初期化時にキャッシュから復元）
    override fun onResume(owner: LifecycleOwner) {
        super.onResume(owner)
        viewModelScope.launch {
            val name = userPreference.getCachedUsername()
            if (name.isNotEmpty()) {
                _username.value = name
            }
        }
    }
}
```

**ポイント**:
- `combine` で複数のフィールドをバリデーション
- `MutableSharedFlow` で一回限りのイベント（ログイン成功）を管理
- `DefaultLifecycleObserver` でライフサイクルに対応
- エラー時もローディング状態を適切に管理

### 3. UsersViewModel - ページネーション

```kotlin
class UsersViewModel : ViewModel(), KoinComponent {
    private val userRepository: UserRepository by inject()

    // ページネーションのカーソル
    private val cursorTrigger = MutableStateFlow<Long?>(null)

    // ユーザー一覧（累積）
    @OptIn(ExperimentalCoroutinesApi::class)
    private val resource = cursorTrigger
        .flatMapLatest { cursor ->
            userRepository.getUsersFlow(flowOf(cursor))
        }
        .map { Resource.success(it) }
        .onStart { emit(Resource.loading()) }
        .catch { emit(Resource.error(it)) }
        .stateIn(viewModelScope, SharingStarted.Lazily, Resource.none())

    // 表示用のアイテムリスト
    val items = resource.mapNotNull { res ->
        res.data?.users?.map { it.toUiState() }?.toImmutableList()
    }.stateIn(viewModelScope, SharingStarted.Lazily, persistentListOf())

    // 次のカーソル
    val nextCursor = resource.mapNotNull { it.data?.nextCursor }
        .stateIn(viewModelScope, SharingStarted.Lazily, null)

    // ローディング状態
    val isLoading = isLoading(resource)
        .stateIn(viewModelScope, SharingStarted.Lazily, false)

    // エラーメッセージ
    private val _errorMessage = MutableStateFlow<ErrorMessage?>(null)
    val errorMessage = _errorMessage.asStateFlow()

    init {
        // 初期ロード
        loadNext(null)

        // エラーの監視
        viewModelScope.launch {
            resource.collect { res ->
                if (res.status == Status.ERROR) {
                    _errorMessage.value = res.throwable?.let { ErrorMessage.of(it) }
                }
            }
        }
    }

    // 次のページをロード
    fun loadNext(cursor: Long?) {
        cursorTrigger.value = cursor
    }

    // リロード（最初から）
    fun reload() {
        cursorTrigger.value = null
    }

    fun clearErrorMessage() {
        _errorMessage.value = null
    }
}
```

**ポイント**:
- カーソルベースのページネーション
- `flatMapLatest` で最新のカーソルに基づいてデータをフェッチ
- リロード時はカーソルを `null` にリセット

### 4. StatsViewModel - 日付範囲管理

```kotlin
class StatsViewModel(val userId: Long): KoinComponent, ViewModel() {
    private val userRepository: UserRepository by inject()

    // 日付範囲とトリガーを一つのデータクラスで管理
    private data class ReloadTrigger(
        val start: LocalDate,
        val end: LocalDate,
        val trigger: Long
    )

    private val reloadTrigger = MutableStateFlow(
        ReloadTrigger(getDefaultStartDate(), getDefaultEndDate(), 0L)
    )

    // 統計データのフェッチング
    @OptIn(ExperimentalCoroutinesApi::class)
    private val resource = reloadTrigger.flatMapLatest { (start, end, _) ->
        userRepository.getUserStatsWithRange(userId, start, end)
    }.map { Resource.success(it) }
     .onStart { emit(Resource.loading()) }
     .catch { emit(Resource.error(it)) }
     .stateIn(viewModelScope, SharingStarted.Lazily, Resource.none())

    // UIState
    val statsState = resource.mapNotNull { res ->
        res.data?.let { entity ->
            StatsState(
                userId = userId,
                startDate = reloadTrigger.value.start,
                endDate = reloadTrigger.value.end,
                monthlyStats = entity.monthlyStats.map { it.toUiState() },
                totalHours = entity.totalHours,
                totalAverageHours = entity.totalAverageHours,
                totalLestHours = entity.totalLestHours.toFloat()
            )
        }
    }.stateIn(viewModelScope, SharingStarted.Lazily, null)

    // ローディング状態
    val isLoading = isLoading(resource)
        .stateIn(viewModelScope, SharingStarted.Lazily, false)

    // エラーメッセージ
    val errorMessage = getErrorMessage(resource)
        .stateIn(viewModelScope, SharingStarted.Lazily, null)

    // 日付範囲の設定
    fun setDateRange(start: LocalDate, end: LocalDate) {
        reloadTrigger.update {
            ReloadTrigger(start = start, end = end, trigger = it.trigger)
        }
    }

    // リロード
    @OptIn(ExperimentalTime::class)
    fun reload() {
        reloadTrigger.update {
            ReloadTrigger(it.start, it.end, Clock.System.now().toEpochMilliseconds())
        }
    }

    private fun getDefaultStartDate(): LocalDate {
        val now = Clock.System.now().toLocalDateTime(TimeZone.currentSystemDefault())
        return LocalDate(now.year, now.monthNumber, 1)
    }

    private fun getDefaultEndDate(): LocalDate {
        val now = Clock.System.now().toLocalDateTime(TimeZone.currentSystemDefault())
        return LocalDate(now.year, now.monthNumber, now.date)
    }
}
```

**ポイント**:
- データクラスで複数のパラメータを管理
- 日付範囲が変わると自動的に再フェッチ
- `trigger` フィールドで強制リロードを実現

## 完全なRepository実装例

### 1. ItemRepositoryImpl - リアクティブなデータストリーム

```kotlin
interface ItemRepository {
    suspend fun removeItem(itemId: Long): Unit
    suspend fun addItem(item: RequestItemEntity): Unit
    suspend fun updateItem(itemId: Long, item: RequestItemEntity): Unit
    fun getItemsFlow(date: LocalDate, userId: Long): Flow<Resource<List<Item>>>
    fun getItem(itemId: Long): Flow<Item?>
}

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

    override suspend fun addItem(item: RequestItemEntity) {
        withContext(Dispatchers.IO) {
            api.addItem(item).tryBody()
        }.also {
            dataSource.action.emit(ItemDataSource.Action.Add(item.id))
        }
    }

    override suspend fun updateItem(itemId: Long, item: RequestItemEntity) {
        withContext(Dispatchers.IO) {
            api.updateItem(itemId, item).tryBody()
        }.also {
            dataSource.action.emit(ItemDataSource.Action.Update(itemId))
        }
    }

    override fun getItemsFlow(date: LocalDate, userId: Long) = channelFlow {
        val range = date.getFirstAndLastDayOfMonth()
        val res = getItems(userId, range.first, range.second)
        send(res)

        // DataSourceの変更イベントを監視
        launch {
            dataSource.action.collect { action ->
                when(action) {
                    is ItemDataSource.Action.Remove,
                    is ItemDataSource.Action.Update,
                    is ItemDataSource.Action.Add -> {
                        // 該当する日付範囲の場合のみ再フェッチ
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

    override fun getItem(itemId: Long) = channelFlow {
        val item = withContext(Dispatchers.IO) {
            api.getItem(itemId).tryBody()
        }
        send(item)

        // 更新を監視
        launch {
            dataSource.action.collect { action ->
                when(action) {
                    is ItemDataSource.Action.Update -> {
                        val updatedItem = withContext(Dispatchers.IO) {
                            api.getItem(itemId).tryBody()
                        }
                        send(updatedItem)
                    }
                    is ItemDataSource.Action.Remove -> {
                        send(null)
                    }
                    else -> {}
                }
            }
        }
    }.flowOn(Dispatchers.IO)

    private suspend fun getItems(
        userId: Long,
        start: LocalDate,
        end: LocalDate
    ): List<Item> {
        return api.getItems(
            userId = userId,
            startDate = start.toString(),
            endDate = end.toString()
        ).tryBody()
    }
}
```

**ポイント**:
- `channelFlow` で変更通知に応答するリアクティブストリーム
- `DataSource` パターンで変更イベントを管理
- 該当する日付範囲の場合のみ再フェッチして効率化
- `tryBody()` でエラーハンドリング（拡張関数）

### 2. UserRepositoryImpl - ページネーションとキャッシュ

```kotlin
interface UserRepository {
    data class UserInfoResponse(
        val users: List<UserEntity>,
        val nextCursor: Long?
    )

    suspend fun getUsers(cursor: Long? = null): UserInfoResponse
    fun getUsersFlow(cursor: Flow<Long?>): Flow<UserInfoResponse>
    fun getUserStats(userId: Long, year: Int, month: Int): Flow<UserWorkStatsEntity>
    fun getUserStatsWithRange(
        userId: Long,
        start: LocalDate,
        end: LocalDate
    ): Flow<UserStatsWithRangeEntity>
}

class UserRepositoryImpl : UserRepository, KoinComponent {
    private val api: UsersApi by inject()
    private val LIMIT = 20L

    override suspend fun getUsers(cursor: Long?): UserRepository.UserInfoResponse {
        return withContext(Dispatchers.IO) {
            val response = api.getUsers(
                cursor = cursor,
                limit = LIMIT
            ).tryBody()

            UserRepository.UserInfoResponse(
                users = response.users,
                nextCursor = response.nextCursor
            )
        }
    }

    override fun getUsersFlow(cursor: Flow<Long?>) = channelFlow {
        var mutableList: MutableList<UserEntity> = mutableListOf()
        cursor.collect { c ->
            val res = getUsers(c)
            if (c == null) {
                // リロード時はリストをクリア
                mutableList.clear()
            }
            mutableList.addAll(res.users)
            send(UserRepository.UserInfoResponse(
                users = mutableList.toList(),
                nextCursor = res.nextCursor
            ))
        }
    }.flowOn(Dispatchers.IO)

    override fun getUserStats(userId: Long, year: Int, month: Int) = channelFlow {
        val res = api.getUserStats(
            userId = userId,
            year = year.toLong(),
            month = month.toLong()
        ).tryBody()
        send(res)
    }.flowOn(Dispatchers.IO)

    override fun getUserStatsWithRange(
        userId: Long,
        start: LocalDate,
        end: LocalDate
    ) = channelFlow {
        val res = api.getUserStatsRange(
            userId = userId,
            startDate = start.toString(),
            endDate = end.toString()
        ).tryBody()
        send(res)
    }.flowOn(Dispatchers.IO)
}
```

**ポイント**:
- カーソルベースのページネーション
- リロード時（cursor == null）はリストをクリア
- 累積リストを管理してページングをサポート

## 完全なUI実装例

### 1. EditorTabScreen - Koin Scopeと状態管理

```kotlin
@Composable
fun EditorTabScreen(
    date: LocalDate,
    onClickItem: (ItemState) -> Unit,
    onClickStats: (userId: Long) -> Unit,
    onAddNewItem: () -> Unit,
    modifier: Modifier = Modifier
) {
    // Koin Scopeで日付ごとに独立したViewModelを管理
    val scope: Scope = getKoin().getScopeOrNull(date.toString())
        ?: getKoin().createScope(date.toString(), named(EditViewScope))
    val viewModel: EditorViewModel = scope.get(parameters = { parametersOf(date.toString()) })

    // 状態の監視
    val user by viewModel.user.collectAsStateWithLifecycle(null)
    val items by viewModel.items.collectAsStateWithLifecycle()
    val isLoading by viewModel.isLoading.collectAsStateWithLifecycle()
    val errorMessage by viewModel.errorMessage.collectAsStateWithLifecycle()
    val stats by viewModel.stats.collectAsStateWithLifecycle()

    // リフレッシュ状態管理
    var isRefreshing by rememberSaveable { mutableStateOf(false) }
    if (isRefreshing) {
        LaunchedEffect(isLoading) {
            isRefreshing = isLoading
        }
    }

    // Content層へ委譲
    EditorTabContent(
        items = items,
        monthlyStats = stats,
        onClickItem = onClickItem,
        onClickRemoveItem = viewModel::removeItem,
        onClickAddNewItem = onAddNewItem,
        isLoading = isLoading,
        isRefreshing = isRefreshing,
        onRefresh = {
            viewModel.sync()
            isRefreshing = true
        },
        onClickStatsCard = { user?.userId?.let { onClickStats(it) } },
        modifier = modifier
    )

    // エラーダイアログ
    if (errorMessage != null) {
        ErrorDialog(
            message = errorMessage?.getErrorMessage(),
            onDismiss = viewModel::clearEvent,
            onRetry = {
                viewModel.clearEvent()
                viewModel.sync()
            }
        )
    }
}
```

**ポイント**:
- `getKoin().getScopeOrNull()` で既存のスコープを取得、なければ作成
- タブごとに独立したViewModelインスタンス
- `rememberSaveable` でリフレッシュ状態を保存
- `LaunchedEffect` でリフレッシュ状態を同期

### 2. LoginScreen - ナビゲーションとイベントハンドリング

```kotlin
@Composable
internal fun LoginScreen(
    navigateToHome: () -> Unit,
    viewModel: LoginViewModel = koinInject()
) {
    val lifecycleOwner = LocalLifecycleOwner.current
    DisposableEffect(lifecycleOwner) {
        lifecycleOwner.lifecycle.addObserver(viewModel)
        onDispose {
            lifecycleOwner.lifecycle.removeObserver(viewModel)
        }
    }

    // 状態の監視
    val isLoading by viewModel.isLoading.collectAsStateWithLifecycle()
    val errorMessage by viewModel.dialogErrorMessage.collectAsStateWithLifecycle()
    val username by viewModel.username.collectAsStateWithLifecycle()
    val password by viewModel.password.collectAsStateWithLifecycle()
    val isPostEnabled by viewModel.isPostEnabled.collectAsStateWithLifecycle()

    // ログイン成功時のナビゲーション
    LaunchedEffect(Unit) {
        viewModel.loginSuccess.collect {
            navigateToHome()
        }
    }

    // Content層
    LoginContent(
        username = { username },
        onUsernameChange = viewModel::updateName,
        password = { password },
        onPasswordChange = viewModel::updatePassword,
        onClickLogin = viewModel::onLogin,
        loginEnabled = { isPostEnabled },
        isLoading = isLoading
    )

    // エラーダイアログ
    if (errorMessage != null) {
        ErrorDialog(
            message = errorMessage?.getErrorMessage(),
            onDismiss = viewModel::clearEvent,
            onRetry = viewModel::clearEvent
        )
    }
}
```

**ポイント**:
- `DisposableEffect` でライフサイクルオブザーバーを登録/解除
- `LaunchedEffect(Unit)` でログイン成功イベントを監視
- `SharedFlow` からのイベントでナビゲーション

### 3. UsersScreen - 無限スクロール

```kotlin
@Composable
fun UsersScreen(
    onClickUser: (userId: Long) -> Unit,
    viewModel: UsersViewModel = koinInject()
) {
    val items by viewModel.items.collectAsStateWithLifecycle()
    val nextCursor by viewModel.nextCursor.collectAsStateWithLifecycle()
    val isLoading by viewModel.isLoading.collectAsStateWithLifecycle()
    val errorMessage by viewModel.errorMessage.collectAsStateWithLifecycle()

    var isRefreshing by rememberSaveable { mutableStateOf(false) }
    if (isRefreshing) {
        LaunchedEffect(isLoading) {
            isRefreshing = isLoading
        }
    }

    UsersContent(
        isLoading = isLoading,
        isRefreshing = isRefreshing,
        items = items,
        nextCursor = nextCursor,
        onClickUser = onClickUser,
        onLoadMore = { viewModel.loadNext(it) },
        onRefresh = {
            viewModel.reload()
            isRefreshing = true
        },
        modifier = Modifier.fillMaxSize()
    )

    if (errorMessage != null) {
        ErrorDialog(
            message = errorMessage?.getErrorMessage(),
            onDismiss = { viewModel.clearErrorMessage() },
            onRetry = { viewModel.reload() }
        )
    }
}

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun UsersContent(
    items: ImmutableList<UserState>,
    nextCursor: Long?,
    isLoading: Boolean,
    isRefreshing: Boolean,
    onClickUser: (userId: Long) -> Unit,
    onLoadMore: (cursor: Long) -> Unit,
    onRefresh: () -> Unit,
    modifier: Modifier = Modifier,
    pullToRefreshState: PullToRefreshState = rememberPullToRefreshState()
) {
    PullToRefreshBox(
        modifier = modifier,
        isRefreshing = isRefreshing,
        state = pullToRefreshState,
        onRefresh = onRefresh,
    ) {
        LazyColumn(modifier = Modifier.fillMaxSize()) {
            items(items = items, key = { i -> i.userId }) { state ->
                UserRow(
                    state = state,
                    onClick = { onClickUser(state.userId) }
                )
            }

            // ページング用のローディングインジケーター
            item {
                if (nextCursor != null && !isLoading) {
                    // 最後のアイテムが表示されたら次のページをロード
                    LaunchedEffect(Unit) {
                        onLoadMore(nextCursor)
                    }
                }
                if (isLoading && items.isNotEmpty()) {
                    Box(
                        modifier = Modifier.fillMaxWidth().padding(16.dp),
                        contentAlignment = Alignment.Center
                    ) {
                        CircularProgressIndicator()
                    }
                }
            }
        }
    }
}
```

**ポイント**:
- `LaunchedEffect(Unit)` で最後のアイテムが表示されたら次のページをロード
- `nextCursor` が `null` の場合は最後のページ
- リフレッシュ時は `reload()` でカーソルをリセット

## データ変換パターン

### Entity -> UIState 変換

```kotlin
// app/src/main/kotlin/.../common/EntityExtension.kt

fun Item.toItemState(): ItemState {
    return ItemState(
        id = id,
        userId = userId,
        date = date,
        inTime = inTime,
        outTime = outTime,
        workRate = workRate,
        expenses = expenses,
        description = description,
        isError = false
    )
}

fun UserEntity.toUiState(): UserState {
    return UserState(
        userId = user.id,
        username = user.username,
        displayName = user.displayName,
        email = user.email ?: "",
        currentMonthStats = currentMonthStats?.toUiState(),
        lastMonthStats = lastMonthStats?.toUiState()
    )
}

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

fun ComparisonType?.toUiState(): StatsComparison.Comparison? {
    return when (this) {
        ComparisonType.UP -> StatsComparison.Comparison.UP
        ComparisonType.DOWN -> StatsComparison.Comparison.DOWN
        ComparisonType.SAME -> StatsComparison.Comparison.SAME
        null -> null
    }
}
```

**ポイント**:
- 拡張関数で変換ロジックを記述
- ドメインモデル -> UIモデルの一方向変換
- null安全な変換

## エラーハンドリングパターン

### ErrorMessage - 統一的なエラー表現

```kotlin
sealed interface ErrorMessage {
    fun getErrorTitle(): String
    fun getErrorMessage(): String

    companion object {
        suspend fun of(throwable: Throwable): ErrorMessage {
            return when (throwable) {
                is IllegalArgumentException -> IllegalArgumentErrorMessage(throwable)
                is ClientRequestException -> ClientRequestErrorMessage(throwable).apply { init() }
                is ServerClosedError -> ServerClosedErrorMessage(throwable)
                is HttpRequestError -> HttpRequestErrorMessage(throwable)
                else -> UnknownErrorMessage(throwable)
            }
        }
    }
}

data class ClientRequestErrorMessage(val exception: ClientRequestException) : ErrorMessage {
    private lateinit var errorResponse: ErrorResponse

    suspend fun init() {
        errorResponse = try {
            exception.response.body<ErrorResponse>()
        } catch (e: Exception) {
            ErrorResponse(message = "Unknown error")
        }
    }

    override fun getErrorTitle(): String = "Error"
    override fun getErrorMessage(): String = errorResponse.message
}

data class ServerClosedErrorMessage(val exception: ServerClosedError) : ErrorMessage {
    override fun getErrorTitle(): String = "Server Error"
    override fun getErrorMessage(): String = "Could not connect to server"
}

data class UnknownErrorMessage(val exception: Throwable) : ErrorMessage {
    override fun getErrorTitle(): String = "Unknown Error"
    override fun getErrorMessage(): String =
        exception.message ?: "An unexpected error occurred"
}

// 複数のResourceからエラーメッセージを取得
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

**ポイント**:
- Sealed interfaceで型安全なエラー表現
- 例外の種類に応じて適切なメッセージを生成
- 複数のResourceからエラーを統合

## UserPreferenceパターン

### ローカルストレージとFlowの統合

```kotlin
class UserPreference(
    private val settings: SerializeObservableSettings,
    private val api: UsersApi,
): KoinComponent {

    // キャッシュからのユーザー情報
    private val currentUserFromCache: Flow<SettingsUser?> by lazy {
        settings.serializableFlow(
            key = USER,
            defaultValue = SettingsUser.EMPTY,
            serializer = SettingsUser.serializer()
        ).map<SettingsUser, SettingsUser?> {
            if (it == SettingsUser.EMPTY) null else it
        }.catch { emit(null) }
         .flowOn(Dispatchers.IO)
    }

    // APIからのユーザー情報
    private val currentUserFromApi: Flow<UserEntity> by lazy {
        flow {
            val user = withContext(Dispatchers.IO) {
                api.getMe().tryBody()
            }
            emit(user)
        }.flowOn(Dispatchers.IO)
    }

    // キャッシュとAPIをマージ
    val currentUser: Flow<SettingsUser> by lazy {
        val cacheFlow = currentUserFromCache.filterNotNull()
        val apiFlow = currentUserFromApi.map { it.toSettings() }
        merge(cacheFlow, apiFlow)
            .flowOn(Dispatchers.IO)
            .distinctUntilChanged()
    }

    // アクティブタブ
    fun getActiveTabFlow(): Flow<List<Tab>> {
        return settings.serializableFlow(
            key = ACTIVE_TABS,
            defaultValue = SettingsActiveTabs(emptyList()),
            serializer = SettingsActiveTabs.serializer()
        ).map { it.tabs }
         .flowOn(Dispatchers.IO)
    }

    // トークン保存
    suspend fun saveToken(token: String) {
        withContext(Dispatchers.IO) {
            settings.putString(TOKEN, token)
        }
    }

    // ユーザーID保存
    suspend fun saveUserId(userId: Long) {
        withContext(Dispatchers.IO) {
            settings.putLong(USER_ID, userId)
        }
    }

    companion object {
        private const val TOKEN = "token"
        private const val USER_ID = "user_id"
        private const val USER = "user"
        private const val ACTIVE_TABS = "active_tabs"
    }
}

// カスタムFlow拡張
inline fun <reified T> SerializeObservableSettings.serializableFlow(
    key: String,
    defaultValue: T,
    serializer: KSerializer<T>
): Flow<T> = callbackFlow {
    trySend(decodeValue(serializer, key, defaultValue))
    val listener = addSerializableListener(key, defaultValue, serializer) {
        trySend(it)
    }
    awaitClose { listener.deactivate() }
}.distinctUntilChanged()
```

**ポイント**:
- `merge` でキャッシュとAPIからのデータを統合
- キャッシュは即座に表示、APIから最新データを取得
- `callbackFlow` でリスナーベースのFlowを作成
- `lazy` で遅延初期化

## Koin DI設定例

### Module定義

```kotlin
val appModule = module {
    // ViewModel
    viewModel { LoginViewModel() }
    viewModel { UsersViewModel() }
    viewModel { (userId: Long) -> StatsViewModel(userId) }

    // Scoped ViewModel
    scope(named(EditViewScope)) {
        scoped { (dateString: String) -> EditorViewModel(dateString) }
    }

    // Repository
    single<ItemRepository> { ItemRepositoryImpl() }
    single<UserRepository> { UserRepositoryImpl() }
    single<LoginRepository> { LoginRepositoryImpl() }

    // DataSource
    single { ItemDataSource() }

    // Preference
    single { UserPreference(get(), get()) }

    // API
    single { ItemsApi(get()) }
    single { UsersApi(get()) }
}

const val EditViewScope = "EditViewScope"
```

**ポイント**:
- `viewModel { }` で標準的なViewModel
- `viewModel { (param: Type) -> }` でパラメータ付きViewModel
- `scope(named())` でスコープ付きViewModel
- Repositoryはシングルトン

これらの実装例を参考に、プロジェクトの実装パターンに従ってコードを書いてください。
