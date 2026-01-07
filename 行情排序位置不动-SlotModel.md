好的，这是一份针对 **Kotlin Multiplatform (KMP) / Compose Multiplatform** 开发者的完整技术指南。

你可以直接保存为 `.md` 文件或者分享给团队。

***

# 核心揭秘：如何在 KMP 中实现 Binance 风格的“排序切换位移不动”

在 Binance 或 Bitget 的行情列表中，当你滚动到列表中间并切换排序（例如从“默认”切换到“涨跌幅”）时，**视口（Viewport）完全不动**，仅仅是当前视野内的数据发生了变化。

这种体验的核心逻辑在于：**UI 关注的是“位置索引”，而不是“数据身份”。**

## 1. 核心原理：坑位模型 (Slot Model)

在 Compose 的 `LazyColumn` 中，`Key` 决定了 Item 的身份：

*   **❌ 使用数据 ID 作为 Key (常规做法)**：
    Compose 认为：“在这个位置的是 BTC”。如果你重新排序，BTC 跑到了第 100 行，Compose 可能会尝试追踪 BTC 的位置，导致列表跳动。
*   **✅ 使用 Index 作为 Key (行情列表做法)**：
    Compose 认为：“这是第 5 行”。不管数据怎么变，第 5 行永远是第 5 行。当数据源重新排序后，Compose 发现第 5 行的数据变了，它**原地刷新**第 5 行的内容，而不会去滚动列表。

---

## 2. 代码实现指南

### 步骤一：数据模型优化 (Data Layer)

为了保证性能，在列表原地刷新时减少不必要的重绘，务必给数据类加上 `@Immutable` 或 `@Stable` 注解。

```kotlin
import androidx.compose.runtime.Immutable

@Immutable
data class MarketTicker(
    val symbol: String,      // e.g., "BTC/USDT"
    val price: String,       // e.g., "65000.00"
    val changeRate: Double   // e.g., 2.5
)
```

### 步骤二：ViewModel 逻辑 (Logic Layer)

ViewModel 负责输出排序后的**新列表**。

```kotlin
class MarketViewModel : ViewModel() {
    // 原始全量数据
    private val rawData = MockData.getAllTickers()

    // UI 状态
    private val _uiState = MutableStateFlow(MarketUiState())
    val uiState = _uiState.asStateFlow()

    fun sortList(type: SortType) {
        // 关键点：基于当前数据生成一个新的 List
        val sorted = when (type) {
            SortType.PRICE_DESC -> rawData.sortedByDescending { it.price.toDouble() }
            SortType.CHANGE_DESC -> rawData.sortedByDescending { it.changeRate }
            SortType.DEFAULT -> rawData
        }
        
        // 更新 State，UI 会收到新列表
        _uiState.update { it.copy(list = sorted, sortType = type) }
    }
}
```

### 步骤三：Compose UI 实现 (UI Layer)

这是最关键的部分。请注意 `LazyColumn` 中 `items` 的写法。

```kotlin
@Composable
fun MarketListScreen(viewModel: MarketViewModel) {
    val state by viewModel.uiState.collectAsState()
    
    // 1. 保持列表状态
    // 这个 State 记录了 "firstVisibleItemIndex" 和 "scrollOffset"
    val listState = rememberLazyListState()

    Column {
        // 顶部排序按钮区域
        SortHeader(
            currentSort = state.sortType,
            onSortClick = { newSort -> viewModel.sortList(newSort) }
        )

        // 列表区域
        LazyColumn(
            state = listState, // 绑定状态
            modifier = Modifier.fillMaxSize()
        ) {
            // 2. 关键核心：不指定 Key，或者显式指定 Key 为 index
            // ❌ 不要写 key = { item.symbol }
            // ✅ 默认情况就是 key = index
            items(state.list) { ticker ->
                
                // 3. 渲染行 Item
                // 当排序变化时，这个 Row 依然在屏幕的这个位置，
                // 只是传入的 `ticker` 数据变了，导致 Text 内容刷新。
                MarketTickerRow(ticker)
            }
        }
    }
}

@Composable
fun MarketTickerRow(ticker: MarketTicker) {
    Row(
        modifier = Modifier
            .fillMaxWidth()
            .height(60.dp)
            .padding(16.dp),
        horizontalArrangement = Arrangement.SpaceBetween
    ) {
        // 名字
        Text(text = ticker.symbol)
        
        // 价格 (数据变化时，这里直接重组刷新)
        Text(text = ticker.price)
        
        // 涨跌幅
        Box(
            modifier = Modifier.background(
                if (ticker.changeRate >= 0) Color.Green else Color.Red
            )
        ) {
            Text(text = "${ticker.changeRate}%", color = Color.White)
        }
    }
}
```

---

## 3. 性能考量与优化

你可能会担心：**“不使用唯一 ID 做 Key，会不会导致性能问题？”**

在 Android 原生的 `RecyclerView` 时代，如果不告诉 Adapter 具体的 ID，它需要重新绑定整个屏幕的 View，这确实比较重。

但在 **Compose** 中，开销非常小，原因如下：

1.  **智能重组 (Smart Recomposition)**：Compose 会比较新旧数据的 `equals`。如果你使用了 `@Immutable` data class，且某个 Item 的数据在新旧列表中碰巧一样（或者只变了价格），Compose 只会重绘变动的部分。
2.  **视口限制**：屏幕上通常只能显示 10-15 个币种。切换排序时，Compose 只需要计算并重绘这 10-15 个可见区域的组件。这对于现代手机（哪怕是低端机）来说，计算量也是忽略不计的。

## 4. 总结：如何做到“位置不动”

| 行为                 | 你的做法 (可能)                    | Binance 的做法                |
|:-------------------|:-----------------------------|:---------------------------|
| **LazyColumn Key** | `key = { it.symbol }`        | **不传 Key** (默认为 Index)     |
| **Compose 理解**     | 追踪特定的币 (BTC)                 | 追踪列表的行号 (Row 5)            |
| **排序后表现**          | BTC 跑到第 100 行 -> **列表跳动或追踪** | 第 5 行变成了 ETH -> **原地刷新内容** |

**一句话口诀：**
> **想要列表跟人走（追踪数据），Key 用 ID；**
> **想要列表原地守（类似行情），Key 用 Index。**