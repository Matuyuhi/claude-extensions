# UI/Logic分離アーキテクチャガイド

このドキュメントでは、AndroidプロジェクトにおけるUI層とLogic層の完全分離アーキテクチャについて説明します。

## アーキテクチャ概要

このアーキテクチャは、UIとビジネスロジックを完全に分離した層構造を採用しています。

```
+-------------------------------------------------+
|          Content層（Pure UI Composables）        |
|  - *Content.kt (ui module)                      |
|  - UIロジックなし、表示のみ                       |
|  - プロパティとコールバックのみ受け取る           |
+-------------------------------------------------+
                        ^ UIState & Callbacks
+-------------------------------------------------+
|           Screen層（状態管理とイベント処理）      |
|  - *Screen.kt (app module)                      |
|  - collectAsStateWithLifecycle()でState収集      |
|  - イベントをViewModelに委譲                      |
+-------------------------------------------------+
                        ^ StateFlow<UIState>
+-------------------------------------------------+
|      ViewModel層（状態管理とデータ変換）          |
|  - *ViewModel.kt (app module)                   |
|  - Entity -> UIStateへの変換を実施                |
|  - Flow/StateFlowでリアクティブな状態管理         |
+-------------------------------------------------+
                        ^ Entity or Domain Model
+-------------------------------------------------+
|    Repository層（ビジネスロジックとAPI通信）      |
|  - *Repository.kt (repository module)           |
|  - APIとの通信                                   |
|  - 必要に応じてEntity -> Domain Model変換         |
|  - Flow/suspend funでデータを提供                |
+-------------------------------------------------+
                        ^ API Entities
+-------------------------------------------------+
|         DataSource層（変更通知の管理）           |
|  - *DataSource.kt (repository module)           |
|  - SharedFlowでイベント通知                      |
+-------------------------------------------------+
                        ^ API Entities
+-------------------------------------------------+
|       API層（OpenAPI Generator Generated）       |
|  - *Api.kt (openapi module)                     |
|  - *Entity.kt (openapi module)                  |
|  - HTTP通信とシリアライゼーション                 |
+-------------------------------------------------+
```

## モジュール構成

### 1. openapi モジュール（API層）
- **役割**: APIとの通信、JSONシリアライゼーション
- **内容**: OpenAPI Generatorで自動生成されたコード
- **モデル**: `*Entity`, `*RequestEntity`, `*ResponseEntity`

### 2. repository モジュール（Logic層）
- **役割**: ビジネスロジック、データ取得、変更通知
- **内容**: Repository、DataSource、Domain Model
- **モデル**: Domain Model（プレフィックスなし）

### 3. ui モジュール（Pure UI層）
- **役割**: 純粋な表示コンポーネント
- **内容**: `*Content.kt`, UIモデル
- **モデル**: `*State`, その他UIモデル
- **重要**: ViewModelに依存しない、プレビュー可能

### 4. app モジュール（状態管理層）
- **役割**: 状態管理、データ変換、イベント処理
- **内容**: `*Screen.kt`, `*ViewModel.kt`, 変換ロジック
- **モデル**: なし（uiモジュールのモデルを使用）

## データフローの詳細

### データの流れ（下から上）

```
1. API Response (JSON)
     | 自動シリアライズ
     v
2. *Entity (openapi module)
     | Repository層で必要に応じて変換
     v
3. Domain Model または *Entity (repository module)
     | ViewModelで変換 [重要]
     v
4. UIState/*State (ui module)
     | Screen層でStateFlowを監視
     v
5. props (Content層へ渡す)
     |
     v
6. Pure Composable (ui module)
```

### イベントの流れ（上から下）

```
1. User Interaction (Content層)
     | callback
     v
2. Screen層がイベント受信
     | ViewModelのメソッド呼び出し
     v
3. ViewModel層で処理
     | Repository呼び出し
     v
4. Repository層でAPI通信
     | DataSourceへイベント発行
     v
5. DataSource層がSharedFlowで通知
     | Repository層のFlowが再発行
     v
6. ViewModelが新しいデータを受信
     | UIStateに変換
     v
7. Screen層が新しいStateを監視
     | Content層に渡す
     v
8. UI更新
```

## UIモデルの設計原則

### 1. UIで必要な情報のみを持つ

APIのEntityはバックエンドの都合で設計されているため、UIに不要な情報が含まれることがあります。UIモデルは**UIで実際に使用する情報のみ**を含むべきです。

**例：PostState**
```kotlin
// API Entity（openapi module）
@Serializable
data class PostEntity (
    @SerialName(value = "post_id") val postId: Long,
    @SerialName(value = "user") val user: UserEntity,  // ネストしたEntity
    @SerialName(value = "date") val date: LocalDate,
    @SerialName(value = "in_time") val inTime: LocalTime,
    @SerialName(value = "out_time") val outTime: LocalTime,
    @SerialName(value = "work_rate") val workRate: Float,
    @SerialName(value = "expenses") val expenses: Float,
    @SerialName(value = "description") val description: String,
    @SerialName(value = "is_error") val isError: Boolean?,
    @SerialName(value = "created_at") val createdAt: String,  // UIで不要
    @SerialName(value = "updated_at") val updatedAt: String,  // UIで不要
)

// UI Model（ui module）
@Immutable
data class PostState(
    val id: Long,
    val userId: Long,        // UserEntityから必要な情報のみ抽出
    val date: LocalDate,
    val inTime: LocalTime,
    val outTime: LocalTime,
    val workRate: Int,       // Float -> Int変換済み
    val expenses: Int,       // Float -> Int変換済み
    val description: String,
    val isError: Boolean     // nullable解消済み
) {
    // 表示用の計算プロパティ
    val visibleList: List<String>
        get() = listOf(
            date.toDateString,     // "11-22"形式
            inTime.toTimeString,   // " 9:30"形式
            outTime.toTimeString,  // "18:00"形式
            workRate.toString(),
            expenses.toString()
        )
}
```

### 2. UIで直接表示できる形式にする

UIモデルは**UIでそのまま表示できる形式**にすべきです。UIコンポーネント内でフォーマット処理を書かないようにします。

#### 悪い例（UIでフォーマット処理）
```kotlin
@Composable
fun UserRow(createdAt: Instant, playCount: Int) {
    // UI層でフォーマット処理を書いている
    val formattedDate = remember(createdAt) {
        createdAt.toLocalDateTime(TimeZone.currentSystemDefault())
            .let { "${it.year}/${it.monthNumber}/${it.dayOfMonth}" }
    }
    val formattedCount = remember(playCount) {
        when {
            playCount >= 1000 -> "${playCount / 1000}K"
            else -> playCount.toString()
        }
    }

    Text(formattedDate)
    Text(formattedCount)
}
```

#### 良い例（ViewModelで変換済み）
```kotlin
// UI Model（ui module）
data class UserState(
    val createdAt: String,   // "2024/11/22" 形式
    val playCount: String    // "16K" 形式
)

// ViewModel（app module）で変換
class UserViewModel : ViewModel() {
    val user = userRepository.getUser()
        .map { entity ->
            UserState(
                createdAt = entity.createdAt.toDisplayDate(),  // Instant -> "2024/11/22"
                playCount = entity.playCount.toDisplayCount()   // 16000 -> "16K"
            )
        }
        .stateIn(viewModelScope, SharingStarted.Lazily, null)
}

// Content（ui module）
@Composable
fun UserRow(state: UserState) {
    // そのまま表示するだけ
    Text(state.createdAt)
    Text(state.playCount)
}
```

### 3. 計算プロパティで派生状態を提供

UIで使用する派生的な情報は、計算プロパティ（computed property）として実装します。

```kotlin
@Immutable
data class PostState(
    val id: Long,
    val inTime: LocalTime,
    val outTime: LocalTime,
    val workRate: Int,
    val expenses: Int,
) {
    // 新規投稿かどうか
    val isNewPost: Boolean
        get() = id == 0L

    // バリデーション結果
    val isValid: Boolean
        get() = inTime < outTime
                && workRate in 0..100
                && expenses >= 0

    // 表示用リスト
    val visibleList: List<String>
        get() = listOf(
            inTime.toTimeString,
            outTime.toTimeString,
            workRate.toString(),
            expenses.toString()
        )

    companion object {
        val empty = PostState(0L, LocalTime(9, 0), LocalTime(18, 0), 100, 0)

        fun createNew(userId: Long): PostState {
            return empty.copy(userId = userId)
        }
    }
}
```

### 4. Enumで型安全な状態管理

状態を表すString値はEnumに変換します。UIで使用するリソース（色、アイコン等）もEnumのメソッドで提供します。

```kotlin
// UI Model（ui module）
data class StatsComparison(
    val weeklyAverage: Comparison?,
    val totalHours: Comparison?,
) {
    enum class Comparison {
        UP, DOWN, SAME;

        // UIで使用するアイコンを返す
        fun image() = when (this) {
            UP -> LottieImage.Increase
            DOWN -> LottieImage.Decrease
            SAME -> LottieImage.FlatArrow
        }

        // UIで使用する色を返す
        val color @Composable get() = when (this) {
            UP ->  Color(0xFF47D747)
            DOWN -> Color(0xFFCB1251)
            SAME -> Color(0xFF888888)
        }
    }
}

// Content（ui module）
@Composable
fun ComparisonIcon(comparison: StatsComparison.Comparison) {
    // Enumから直接取得
    Icon(
        imageVector = comparison.image().icon,
        tint = comparison.color,
        contentDescription = null
    )
}
```

## ViewModelでの変換パターン

### パターン1: シンプルな変換

```kotlin
class StatsViewModel(val userId: Long): ViewModel() {
    private val userRepository: UserRepository by inject()

    private val resource = userRepository
        .getUserStats(userId, year, month)
        .map { Resource.success(it) }
        .onStart { emit(Resource.loading()) }
        .catch { emit(Resource.error(it)) }
        .stateIn(viewModelScope, SharingStarted.Lazily, Resource.none())

    // Entity -> UIState変換
    val stats = resource.mapNotNull { res ->
        res.data?.let { entity ->
            StatsState(
                userId = userId,
                monthlyStats = entity.monthlyStats.map { it.toMonthWorkStats() },
                totalHours = entity.totalHours,
                totalAverageHours = entity.totalAverageHours,
            )
        }
    }.stateIn(viewModelScope, SharingStarted.Lazily, null)

    // 変換用の拡張関数（ViewModel内で定義）
    private fun MonthlyStatsEntity.toMonthWorkStats(): MonthWorkStats {
        return MonthWorkStats(
            year = year.toInt(),          // Long -> Int
            month = month.toInt(),
            totalHours = totalHours,
            averageHours = averageHours,
            lestHours = lestHours.toFloat(),  // Long -> Float
            comparison = comparison.toStatsComparison()
        )
    }

    private fun ComparisonEntity.toStatsComparison(): StatsComparison {
        return StatsComparison(
            weeklyAverage = weeklyAverageComparison?.let {
                when (it) {
                    ComparisonEntity.WeeklyAverageComparison.up -> StatsComparison.Comparison.UP
                    ComparisonEntity.WeeklyAverageComparison.down -> StatsComparison.Comparison.DOWN
                    ComparisonEntity.WeeklyAverageComparison.same -> StatsComparison.Comparison.SAME
                }
            },
            totalHours = totalHoursComparison?.let {
                when (it) {
                    ComparisonEntity.TotalHoursComparison.up -> StatsComparison.Comparison.UP
                    ComparisonEntity.TotalHoursComparison.down -> StatsComparison.Comparison.DOWN
                    ComparisonEntity.TotalHoursComparison.same -> StatsComparison.Comparison.SAME
                }
            }
        )
    }
}
```

### パターン2: 共通の変換ロジック（EntityExtension.kt）

複数のViewModelで使用する変換ロジックは、`EntityExtension.kt`に定義します。

**ファイルパス**: `app/src/main/kotlin/.../common/EntityExtension.kt`

```kotlin
// MonthlyStatsEntity -> MonthWorkStats変換
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

private fun ComparisonEntity.WeeklyAverageComparison.toUiState(): StatsComparison.Comparison {
    return when (this) {
        ComparisonEntity.WeeklyAverageComparison.up -> StatsComparison.Comparison.UP
        ComparisonEntity.WeeklyAverageComparison.down -> StatsComparison.Comparison.DOWN
        ComparisonEntity.WeeklyAverageComparison.same -> StatsComparison.Comparison.SAME
    }
}

// ViewModelで使用
class StatsViewModel : ViewModel() {
    val stats = resource.mapNotNull { res ->
        res.data?.let { entity ->
            StatsState(
                monthlyStats = entity.monthlyStats.map { it.toUiState() },  // 拡張関数を使用
                // ...
            )
        }
    }.stateIn(viewModelScope, SharingStarted.Lazily, null)
}
```

### パターン3: フォーマット変換（TimeHelper.kt等）

日時や数値のフォーマット変換は、拡張関数として定義します。

**ファイルパス**: `repository/src/main/kotlin/.../TimeHelper.kt`

```kotlin
// LocalDate -> 表示用文字列
val LocalDate.toDisplayDate: String
    get() = "${year}/${monthNumber}/${dayOfMonth}"

// LocalTime -> 表示用文字列
val LocalTime.toDisplayTime: String
    get() = "${hour.to2digit}:${minute.to2digit}"

val Int.to2digit: String
    get() = if (this < 10) "0$this" else this.toString()

// PostState内で使用
internal val LocalDate.toDateString: String
    get() = "$monthNumber-${dayOfMonth.toString().padStart(2, ' ')}"

internal val LocalTime.toTimeString: String
    get() = "${hour.toString().padStart(2, ' ')}:${minute.toString().padStart(2, '0')}"

// ViewModelで使用
class UserViewModel : ViewModel() {
    val user = userRepository.getUser()
        .map { entity ->
            UserState(
                createdAt = entity.createdAt.toDisplayDate(),  // 拡張関数を使用
                // ...
            )
        }
        .stateIn(viewModelScope, SharingStarted.Lazily, null)
}
```

## Screen層とContent層の分離

### Screen層（app module）

**役割**:
- ViewModelの注入
- StateFlowの監視（`collectAsStateWithLifecycle()`）
- イベントハンドラの接続
- ナビゲーション処理
- ダイアログ等の表示制御

**実装例**:

```kotlin
@Composable
fun StatsScreen(
    userId: Long,
    onNavigateBack: () -> Unit,
    viewModel: StatsViewModel = koinInject(parameters = { parametersOf(userId) })
) {
    // StateFlowを監視
    val startDate by viewModel.startDate.collectAsStateWithLifecycle()
    val endDate by viewModel.endDate.collectAsStateWithLifecycle()
    val stats by viewModel.stats.collectAsStateWithLifecycle()
    val isLoading by viewModel.isLoading.collectAsStateWithLifecycle()
    val errorMessage by viewModel.errorMessage.collectAsStateWithLifecycle()

    // Content層に状態を渡す（ViewModelは渡さない）
    StatsContent(
        startDate = startDate,
        endDate = endDate,
        stats = stats,
        isLoading = isLoading,
        onBackClick = onNavigateBack,
        onPreviousPeriod = viewModel::previousPeriod,  // メソッド参照
        onNextPeriod = viewModel::nextPeriod,
    )

    // エラーハンドリング（Screen層で管理）
    errorMessage?.let {
        ErrorDialog(
            message = it.getErrorMessage(),
            onDismiss = { viewModel.clearErrorMessage() },
            onRetry = { viewModel.reload() }
        )
    }
}
```

### Content層（ui module）

**役割**:
- 純粋な表示
- プロパティを受け取ってUIをレンダリング
- イベントをコールバックで通知

**重要な制約**:
- ViewModelに依存しない
- `koinInject()`を使わない
- `collectAsState*()`を使わない
- すべての状態をプロパティで受け取る
- すべてのイベントをコールバックで通知
- Previewが書ける

**実装例**:

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun StatsContent(
    startDate: LocalDate?,
    endDate: LocalDate?,
    stats: StatsState?,
    isLoading: Boolean,
    onPreviousPeriod: () -> Unit,
    onNextPeriod: () -> Unit,
    onBackClick: () -> Unit,
    modifier: Modifier = Modifier
) {
    Scaffold(
        modifier = modifier.fillMaxSize(),
        topBar = {
            TopAppBar(
                title = { Text("Stats") },
                navigationIcon = {
                    IconButton(onClick = onBackClick) {
                        Icon(Icons.AutoMirrored.Filled.ArrowBackIos, contentDescription = "Back")
                    }
                }
            )
        }
    ) { paddingValues ->
        // 受け取った状態をそのまま表示
        stats?.let { statsData ->
            LazyColumn(
                modifier = Modifier.fillMaxSize().padding(paddingValues)
            ) {
                items(statsData.monthlyStats) { monthStat ->
                    WorkStatsRow(
                        workStats = monthStat,
                        onClickStatsCard = {},
                    )
                }
            }
        }

        if (isLoading) {
            LoadingContent(modifier = Modifier.fillMaxSize())
        }
    }
}

// Previewが書ける
@Preview
@Composable
fun StatsContentPreview() {
    StatsContent(
        startDate = LocalDate(2024, 1, 1),
        endDate = LocalDate(2024, 12, 31),
        stats = StatsState(
            userId = 1L,
            startDate = LocalDate(2024, 1, 1),
            endDate = LocalDate(2024, 12, 31),
            monthlyStats = listOf(
                MonthWorkStats(2024, 1, 160f, 8f, 0f, null)
            ),
            totalHours = 160f,
            totalAverageHours = 8f,
            totalLestHours = 0f
        ),
        isLoading = false,
        onPreviousPeriod = {},
        onNextPeriod = {},
        onBackClick = {}
    )
}
```

## Repository層とDataSource層の役割

### Repository層

**役割**:
- APIとの通信
- データのキャッシング（必要に応じて）
- Entity -> Domain Modelの変換（必要に応じて）
- Flowでリアクティブなデータストリームを提供

**実装例**:

```kotlin
interface ItemRepository {
    suspend fun removeItem(itemId: Long): Unit
    suspend fun addItem(item: RequestItemEntity): Unit
    suspend fun updateItem(itemId: Long, item: RequestItemEntity): Unit
    fun getItemsFlow(date: LocalDate, userId: Long): Flow<Resource<List<Item>>>
}

class ItemRepositoryImpl : ItemRepository, KoinComponent {
    private val dataSource: ItemDataSource by inject()
    private val api: ItemsApi by inject()

    override suspend fun addItem(item: RequestItemEntity) {
        withContext(Dispatchers.IO) {
            api.addItem(item).tryBody()
        }.also {
            // DataSourceに変更を通知
            dataSource.action.emit(ItemDataSource.Action.Add(item.id))
        }
    }

    override fun getItemsFlow(date: LocalDate, userId: Long) = channelFlow {
        val range = date.getFirstAndLastDayOfMonth()
        val res = getItems(userId, range.first, range.second)
        send(res)

        // DataSourceの変更を監視して自動更新
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
}
```

### DataSource層

**役割**:
- データの変更イベントを管理
- SharedFlowで変更を通知
- Repository層が購読

**実装例**:

```kotlin
internal class ItemDataSource: KoinComponent {
    sealed interface Action {
        val id: Long
        data class Remove(override val id: Long): Action
        data class Update(override val id: Long): Action
        data class Add(override val id: Long): Action
    }

    // SharedFlowで変更イベントを配信
    val action = MutableSharedFlow<Action>()
}
```

## ベストプラクティス

### 1. UIモデルは@Immutableにする

```kotlin
@Immutable  // Composeの最適化のために必須
data class PostState(
    val id: Long,
    val date: LocalDate,
    // ...
)
```

### 2. リストは ImmutableList を使用

```kotlin
import kotlinx.collections.immutable.ImmutableList
import kotlinx.collections.immutable.persistentListOf
import kotlinx.collections.immutable.toImmutableList

class EditorViewModel : ViewModel() {
    val items: StateFlow<ImmutableList<PostState>> = postList
        .mapNotNull { list ->
            list?.map { it.toPostState() }?.toImmutableList()  // ImmutableListに変換
        }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(), persistentListOf())
}
```

### 3. Content層でViewModelに依存しない

```kotlin
// 悪い例
@Composable
fun StatsContent(viewModel: StatsViewModel) {
    val stats by viewModel.stats.collectAsStateWithLifecycle()
    // ...
}

// 良い例
@Composable
fun StatsContent(
    stats: StatsState?,
    onRefresh: () -> Unit,
    modifier: Modifier = Modifier
) {
    // ...
}
```

### 4. 型変換は必要に応じて実施

```kotlin
// API層のLongをUIで扱いやすいIntに変換
MonthWorkStats(
    year = entity.year.toInt(),    // Long -> Int
    month = entity.month.toInt(),
    // ...
)

// FloatをIntに変換（小数点不要な場合）
PostState(
    workRate = entity.workRate.toInt(),    // Float -> Int
    expenses = entity.expenses.toInt(),
    // ...
)
```

### 5. nullable解消はViewModelで実施

```kotlin
// API Entityがnullableな場合
@Serializable
data class PostEntity(
    val isError: Boolean?,  // nullable
)

// UIStateでは non-null
@Immutable
data class PostState(
    val isError: Boolean   // non-null
)

// ViewModelで変換時にデフォルト値を設定
PostState(
    isError = entity.isError ?: false,  // null時はfalse
)
```

## トラブルシューティング

### Content層でViewModelが必要になった場合

**原因**: 設計ミス。Content層は純粋なUIコンポーネントであるべき。

**解決策**:
1. Screen層で必要な状態をすべて収集する
2. コールバックでイベントをViewModelに渡す
3. どうしても必要な場合は、Screen層とContent層を統合する

### UIモデルの変換が重い場合

**原因**: 大量のデータを変換している、または変換ロジックが複雑

**解決策**:
1. `map`オペレーターで非同期的に変換（Flow上で実行）
2. 必要に応じて`flowOn(Dispatchers.Default)`でディスパッチャーを変更

```kotlin
val stats = resource
    .mapNotNull { res ->
        res.data?.let { entity ->
            // 重い変換処理
            entity.toUiState()
        }
    }
    .flowOn(Dispatchers.Default)  // CPUバウンドな処理を別スレッドで実行
    .stateIn(viewModelScope, SharingStarted.Lazily, null)
```

### 同じ変換ロジックを複数のViewModelで使う場合

**解決策**: `EntityExtension.kt`に拡張関数として定義

```kotlin
// app/src/main/kotlin/.../common/EntityExtension.kt
fun MonthlyStatsEntity.toUiState(): MonthWorkStats {
    return MonthWorkStats(
        year = year.toInt(),
        month = month.toInt(),
        // ...
    )
}

// 複数のViewModelで使用可能
class StatsViewModel : ViewModel() {
    val stats = resource.mapNotNull { it.data?.monthlyStats?.map { it.toUiState() } }
}

class UserViewModel : ViewModel() {
    val user = resource.mapNotNull { it.data?.currentMonthStats?.toUiState() }
}
```
