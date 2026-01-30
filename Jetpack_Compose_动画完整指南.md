# Jetpack Compose 动画完整指南

> 面向有8年Android开发经验的开发者，从View体系过渡到Compose动画的全面指南

---

## 目录

- [一、Compose动画概述](#一compose动画概述)
- [二、高阶动画API](#二高阶动画api)
- [三、基础动画API](#三基础动画api)
- [四、低级动画API](#四低级动画api)
- [五、动画规格AnimationSpec](#五动画规格animationspec)
- [六、Easing缓动函数](#六easing缓动函数)
- [七、底层实现原理](#七底层实现原理)
- [八、性能优化](#八性能优化)
- [九、与传统View动画对比](#九与传统view动画对比)

---

## 一、Compose动画概述

### 1.1 声明式动画 vs 命令式动画

**传统View体系（命令式）**
```kotlin
// 传统View动画：手动控制动画
ObjectAnimator.ofFloat(view, "translationX", 0f, 100f).apply {
    duration = 300
    start()
}
```

**Compose（声明式）**
```kotlin
// Compose动画：通过状态驱动
var offset by remember { mutableStateOf(0f) }
Box(
    Modifier.offset { IntOffset(offset.roundToInt(), 0) }
)

// 改变状态，动画自动播放
offset = 100f
```

**核心区别：**

| 特性 | View体系 | Compose |
|------|----------|---------|
| **控制方式** | 命令式：调用start()等方法 | 声明式：改变State值 |
| **状态管理** | 需要手动跟踪动画状态 | 自动由State管理 |
| **取消/暂停** | 需要手动调用cancel() | 自动处理，无需干预 |
| **组合性** | 动画难以组合 | 动画天然可组合 |
| **类型安全** | 反射调用属性值 | 编译时类型检查 |
| **Recompose** | N/A | 动画值变化自动触发重绘 |

### 1.2 动画状态管理

**State-driven Animation（状态驱动的动画）**

Compose动画的核心思想：**动画只是状态的过渡**

```kotlin
// 定义状态
enum class CardState { Expanded, Collapsed }

var cardState by remember { mutableStateOf(CardState.Collapsed) }

// 状态驱动动画
val size by animateDpAsState(
    targetValue = if (cardState == CardState.Expanded) 300.dp else 100.dp,
    animationSpec = spring(stiffness = Spring.StiffnessMedium)
)

// UI根据状态渲染
Card(modifier = Modifier.size(size)) { /* ... */ }
```

**单向数据流在动画中的应用：**

```
用户交互 → State改变 → 动画计算 → Recompose → 渲染
    ↑                                              ↓
    └──────────────── 反馈循环 ────────────────────┘
```

### 1.3 Compose动画分层架构

```
┌─────────────────────────────────────────────┐
│   高阶动画API（最易用，覆盖常见场景）        │
│   - AnimatedVisibility                       │
│   - AnimatedContent                          │
│   - Crossfade                                │
│   - animateContentSize                       │
├─────────────────────────────────────────────┤
│   基础动画API（中等难度，灵活控制）           │
│   - animate*AsState                          │
│   - rememberInfiniteTransition               │
│   - updateTransition                         │
├─────────────────────────────────────────────┤
│   低级动画API（最灵活，需要手动管理）         │
│   - Animatable                               │
│   - Animation                                │
│   - TargetBasedAnimation                     │
├─────────────────────────────────────────────┤
│   底层机制                                   │
│   - AnimationSpec                            │
│   - Easing                                   │
│   - AnimationState                           │
└─────────────────────────────────────────────┘
```

---

## 二、高阶动画API

### 2.1 AnimatedVisibility - 可见性动画

**用途：** 控制Composable的进入和退出动画

**核心概念：**
- 自动处理进入/退出逻辑
- 支持多种内置动画效果
- 可以自定义EnterTransition和ExitTransition

**EnterTransition类型：**
- `fadeIn` - 淡入
- `expandIn` - 展开
- `expandHorizontally` - 水平展开
- `expandVertically` - 垂直展开
- `slideIn` - 滑入
- `slideInHorizontally` - 水平滑入
- `slideInVertically` - 垂直滑入
- `scaleIn` - 缩放进入

**ExitTransition类型：**
- `fadeOut` - 淡出
- `shrinkOut` - 收缩
- `shrinkHorizontally` - 水平收缩
- `shrinkVertically` - 垂直收缩
- `slideOut` - 滑出
- `slideOutHorizontally` - 水平滑出
- `slideOutVertically` - 垂直滑出
- `scaleOut` - 缩放退出

**组合方式：**
```kotlin
// 使用 + 组合多个动画效果
val enterTransition = fadeIn() + expandVertically()
val exitTransition = fadeOut() + shrinkVertically()
```

**[Demo1] 展开/折叠列表项**
```
[GIF演示：点击按钮，列表项依次展开，带淡入+展开效果]
```

**[Demo2] 不同方向的进入退出动画**
```
[GIF演示：展示8种不同方向的动画效果对比]
```

### 2.2 AnimatedContent - 内容切换动画

**用途：** 在两个内容之间切换时播放动画

**核心概念：**
- 类似Fragment转场动画
- 支持自定义尺寸、位置、透明度动画
- 使用`sizeTransform`控制内容尺寸变化

**内置Transition：**
- `sizeTransform` - 控制内容大小变化
- `contentAlignment` - 控制内容对齐方式

**[Demo3] 图片切换动画（淡入淡出+缩放）**
```
[GIF演示：点击按钮，两张图片之间切换，带缩放和淡入淡出效果]
```

### 2.3 Crossfade - 淡入淡出动画

**用途：** 最简单的内容切换动画，只做透明度变化

**核心概念：**
- AnimatedContent的简化版本
- 只有淡入淡出效果
- 适用于简单场景

**[Demo4] 多个界面状态切换**
```
[GIF演示：在Loading、Success、Error三个状态之间切换]
```

### 2.4 animateContentSize - 尺寸变化动画

**用途：** 当内容尺寸改变时，自动播放尺寸过渡动画

**核心概念：**
- 自动监听内容尺寸变化
- 无需手动计算动画
- 适用于动态内容场景

**[Demo5] 文本展开/收起**
```
[GIF演示：点击按钮，长文本平滑展开/收起]
```

---

## 三、基础动画API

### 3.1 animateAsState - 单值动画

**用途：** 将单个状态值的变化动画化

**支持的类型：**
- `animateFloatAsState`
- `animateDpAsState`
- `animateSizeAsState`
- `animateOffsetAsState`
- `animateRectAsState`
- `animateIntAsState`
- `animateIntOffsetAsState`
- `animateColorAsState`

**核心参数：**
- `targetValue` - 目标值（必须）
- `animationSpec` - 动画规格（可选，默认spring）
- `label` - 调试标签（可选）
- `finishedListener` - 动画完成回调（可选）

**工作原理：**
```kotlin
// 当targetValue改变时，自动创建动画
val size by animateDpAsState(
    targetValue = if (expanded) 100.dp else 50.dp,
    animationSpec = tween(300)
)
```

**[Demo6] 按钮点击缩放效果**
```
[GIF演示：点击按钮，按钮有缩放反馈]
```

**[Demo7] 颜色渐变动画**
```
[GIF演示：颜色在多种颜色之间平滑过渡]
```

### 3.2 animateDecay - 减速动画

**用途：** 模拟物理减速效果（如滑动后的惯性）

**核心概念：**
- 基于初速度计算减速轨迹
- 常用于手势交互
- 需要配合`rememberSplineBasedDecay`使用

**[Demo8] 滑动惯性效果**
```
[GIF演示：拖动卡片释放后，卡片带惯性滑动]
```

### 3.3 rememberInfiniteTransition - 无限循环动画

**用途：** 创建永不停止的动画循环

**核心概念：**
- 返回`InfiniteTransition`实例
- 通过`animateColor`、`animateFloat`、`animateValue`添加动画
- 所有动画会同步运行

**适用场景：**
- Loading动画
- 呼吸效果
- 旋转指示器
- 背景动画

**[Demo9] 加载旋转动画**
```
[GIF演示：无限旋转的加载图标]
```

**[Demo10] 呼吸灯效果**
```
[GIF演示：透明度循环变化的呼吸灯效果]
```

### 3.4 updateTransition - 复杂状态切换动画

**用途：** 在多个状态之间切换，每个状态可以包含多个动画属性

**核心概念：**
- 使用`Transition`对象管理多个动画
- 通过`animate*`扩展函数定义属性动画
- 状态变化时所有属性同步动画

**使用步骤：**
1. 定义状态枚举
2. 使用`updateTransition`创建Transition
3. 为每个状态的每个属性定义动画
4. 使用动画值渲染UI

**[Demo11] 卡片展开/收起多属性动画**
```
[GIF演示：卡片展开时，宽度、高度、圆角、透明度同时变化]
```

**[Demo12] 多状态按钮动画**
```
[GIF演示：按钮在Normal、Pressed、Disabled三种状态下变化]
```

---

## 四、低级动画API

### 4.1 Animatable - 可暂停、可取消的动画

**用途：** 最灵活的动画控制，支持暂停、恢复、取消、倒放

**核心概念：**
- 单值动画控制器
- 完全控制动画生命周期
- 支持协程操作

**关键方法：**
- `animateTo(targetValue)` - 动画到目标值
- `snapTo(value)` - 瞬间跳到指定值
- `stop()` - 停止动画
- `runBlocking { anim.animateTo(...) }` - 阻塞式动画
- `animateDecay(velocity)` - 减速动画

**特性：**
- 自动处理并发调用（新动画会取消旧动画）
- 支持动画值监听
- 支持动画完成回调

**[Demo13] 手动控制进度的动画**
```
[GIF演示：通过Slider手动控制动画进度]
```

**[Demo14] 动画暂停/恢复/取消**
```
[GIF演示：播放中可暂停、恢复、取消动画]
```

### 4.2 TargetBasedAnimation - 基于目标的动画

**用途：** 手动控制动画帧的计算，不自动运行

**核心概念：**
- 定义动画但不自动播放
- 通过`playTime`获取当前帧的值
- 适用于需要完全控制动画进度的场景

**使用场景：**
- 视频播放器进度条
- 手动拖拽控制动画
- 与自定义时间轴集成

**[Demo15] 自定义动画控制**
```
[GIF演示：通过时间轴控制动画播放]
```

### 4.3 DecayAnimation - 衰减动画

**用途：** 模拟物理减速运动

**核心概念：**
- 基于初速度计算每一帧的值
- 使用`SplineBasedDecay`或其他衰减算法
- 速度会逐渐减到0

**[Demo16] 滚动减速效果**
```
[GIF演示：手指滑动后，内容带惯性滚动]
```

---

## 五、动画规格AnimationSpec

### 5.1 tween - 基于时间的插值器

**用途：** 在指定时间内完成动画，可使用Easing控制速度曲线

**参数：**
- `durationMillis` - 动画时长（ms）
- `delayMillis` - 延迟时间（ms）
- `easing` - 缓动函数（默认FastOutSlowInEasing）

**特点：**
- 最基础的动画规格
- 适合需要精确控制时长的场景
- 支持延迟启动

**[Demo17] 不同时长的tween对比**
```
[GIF演示：100ms、300ms、1000ms三种时长的动画对比]
```

### 5.2 spring - 物理弹簧动画

**用途：** 模拟弹簧物理效果，更自然的交互体验

**参数：**
- `dampingRatio` - 阻尼比（控制弹跳次数）
  - `0f` - 永不停止
  - `0.5f` - 稍微弹跳
  - `1f` - 无弹跳
  - >1f - 过阻尼
- `stiffness` - 刚度（控制动画速度）
  - `StiffnessHigh` - 快速
  - `StiffnessMedium` - 中等
  - `StiffnessLow` - 慢速
  - `StiffnessVeryLow` - 很慢

**特点：**
- Material Design推荐的默认动画
- 符合物理直觉
- 自动调整时长（无需指定duration）

**[Demo18] 不同stiffness/dampingRatio效果**
```
[GIF演示：9种不同参数组合的弹簧效果]
```

**[Demo19] 拖拽释放弹簧效果**
```
[GIF演示：拖拽卡片释放后，卡片弹回原位]
```

### 5.3 keyframes - 关键帧动画

**用途：** 在特定时间点定义特定值，创建复杂动画路径

**参数：**
- `durationMillis` - 总时长
- `delayMillis` - 延迟时间
- 通过`at`和`with`定义关键帧

**特点：**
- 精确控制每个时间点的值
- 可以创建复杂的运动轨迹
- 支持不同的Easing曲线

**[Demo20] 复杂路径动画**
```
[GIF演示：物体按自定义路径运动]
```

### 5.4 repeatable - 可重复动画

**用途：** 动画重复播放指定次数

**参数：**
- `iterations` - 重复次数（1表示重复一次，总共播放2次）
- `animationSpec` - 子动画规格
- `repeatMode` - 重复模式
  - `RepeatMode.Restart` - 从头开始
  - `RepeatMode.Reverse` - 反向播放
- `initialStartOffset` - 初始偏移

**[Demo21] 重复播放的动画**
```
[GIF演示：动画重复3次后停止]
```

### 5.5 infiniteRepeatable - 无限重复动画

**用途：** 动画永不停止地循环播放

**参数：**
- `animationSpec` - 子动画规格
- `repeatMode` - 重复模式
- `initialStartOffset` - 初始偏移

**使用场景：**
- Loading动画
- 背景效果
- 指示器

**[Demo22] 无限弹跳效果**
```
[GIF演示：无限循环的弹跳动画]
```

### 5.6 snap - 瞬间切换

**用途：** 不播放动画，直接跳到目标值

**特点：**
- duration = 0
- 通常与其他动画组合使用

**[Demo23] 结合其他动画使用**
```
[GIF演示：部分属性瞬间变化，部分属性平滑过渡]
```

---

## 六、Easing缓动函数

### 6.1 Easing的作用

Easing控制动画速度随时间的变化，即"加速-减速"曲线。

```
Easing函数签名：
fun Easing(fraction: Float): Float

fraction: 0f → 1f (时间进度)
返回值: 0f → 1f (动画值进度)
```

### 6.2 内置Easing

| Easing | 效果 | 适用场景 |
|--------|------|----------|
| `FastOutSlowInEasing` | 开始快，中间快，结束慢 | 通用场景，Material默认 |
| `LinearOutSlowInEasing` | 线性开始，结束慢 | 进入动画 |
| `FastOutLinearEasing` | 开始快，线性结束 | 退出动画 |
| `LinearEasing` | 匀速 | 不推荐，显得生硬 |
| `CubicBezierEasing` | 自定义贝塞尔曲线 | 复杂效果 |

**[Demo24] 可视化展示不同Easing曲线**
```
[GIF演示：5种Easing的曲线和对应的动画效果]
```

**[Demo25] 实际应用中的Easing选择**
```
[GIF演示：同一动画使用不同Easing的效果对比]
```

### 6.3 自定义Easing

```kotlin
// 自定义Easing
val customEasing = Easing { fraction ->
    fraction * fraction * fraction // 自定义曲线
}
```

---

## 七、底层实现原理

### 7.1 Compose渲染流程

```
┌─────────────────────────────────────────────────────────┐
│  Compose 渲染管道                                        │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. Composition                                         │
│     - 构建Composable树                                   │
│     - 创建SlotTable                                     │
│     - 记录状态依赖                                       │
│     - 动画状态在这里注册和观察                           │
│                                                         │
│  2. Layout (Measure + Layout)                          │
│     - 计算每个节点尺寸                                   │
│     - 确定每个节点位置                                   │
│     - 动画影响尺寸时，触发Measure重新计算                │
│     - Modifier.offset, Modifier.size等在这里应用        │
│                                                         │
│  3. Draw                                                │
│     - 绘制到Canvas                                      │
│     - 动画影响绘制参数（alpha, scale, rotation等）       │
│     - Modifier.graphicsLayer在这里应用                   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**动画在哪个阶段介入？**

- **属性动画**（如大小、位置）：在Layout阶段
  - 影响`MeasureResult`
  - 触发父节点重新Layout
  - 消耗：中等

- **绘制动画**（如颜色、透明度、缩放、旋转）：在Draw阶段
  - 只影响`DrawScope`
  - 不触发Layout重新计算
  - 消耗：最低（推荐优先使用）

- **可见性动画**：通过在Composition中插入/移除节点实现
  - AnimatedVisibility/AnimatedContent
  - 涉及Composition树的变化
  - 消耗：最高

**渲染流程详细图解：**

```
┌─────────────────────────────────────────────────────────────┐
│  用户交互：点击按钮 → 状态改变                               │
│  state.value = newValue                                     │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ↓
┌─────────────────────────────────────────────────────────────┐
│  Snapshot系统检测到状态变化                                  │
│  - 记录变化的State对象                                       │
│  - 计算最小需要Recompose的范围                               │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ↓
┌─────────────────────────────────────────────────────────────┐
│  Composition Phase                                         │
│  - 标记需要Recompose的Composable                             │
│  - 调用被标记Composable的函数                                │
│  - 执行remember、LaunchedEffect等副作用                      │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ↓
┌─────────────────────────────────────────────────────────────┐
│  Layout Phase (如果动画影响布局)                            │
│  - Measure: 计算每个节点的新尺寸                             │
│  - Layout: 确定每个节点的新位置                              │
│  - 动画值影响measure/layout约束                              │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ↓
┌─────────────────────────────────────────────────────────────┐
│  Draw Phase                                                 │
│  - 将Layout结果绘制到Canvas                                 │
│  - 应用Modifier.graphicsLayer等绘制动画                     │
│  - 提交给GPU渲染                                            │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ↓
┌─────────────────────────────────────────────────────────────┐
│  下一帧动画继续                                              │
│  - 动画值更新 → 重复上述流程                                │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 Recompose机制

**Recompose触发条件：**
```kotlin
var visible by remember { mutableStateOf(true) }

// visible改变 → 触发Recompose
if (visible) {
    Content()
}
```

**动画值变化如何触发Recompose：**

```kotlin
// animateAsState内部实现（简化版）
@Composable
fun animateFloatAsState(
    targetValue: Float,
    animationSpec: AnimationSpec<Float> = spring()
): State<Float> {
    val animatable = remember { Animatable(targetValue) }
    
    LaunchedEffect(targetValue) {
        animatable.animateTo(targetValue, animationSpec)
    }
    
    return animatable.asState()
}
```

**关键点：**
1. `Animatable`的值是`State<Float>`类型
2. 每帧更新时，`State.value`改变
3. `State`被读取的地方会标记为需要Recompose
4. Compose调度器执行Recompose

**Recompose优化：**
- Compose会跳过未变化的子树
- 使用`remember`避免不必要的重建
- 使用`derivedStateOf`缓存计算结果

### 7.3 Snapshot系统

**Snapshot的作用：**
- 管理Compose中的状态读写
- 实现可观察性（Observable）
- 支持原子性状态更新

**动画中的Snapshot：**

```kotlin
// 动画每一帧都在新的Snapshot中更新
while (true) {
    val currentTime = frameTimeNanos
    val value = calculateValue(currentTime)
    
    // 在Snapshot中更新状态
    Snapshot.withMutableSnapshot {
        animatable.updateState(value)
    }
    
    // 触发Recompose
    composeScheduler.scheduleFrame()
}
```

**Snapshot关键概念：**
- `MutableSnapshot` - 可变快照，用于写入
- `Snapshot.withMutableSnapshot` - 在快照中执行操作
- `apply()` - 提交快照，触发监听器

### 7.4 AnimationFrameClock

**作用：** 控制动画帧的调度，与VSync同步

**工作流程：**
```
1. 动画开始
   ↓
2. 注册AnimationFrameClock
   ↓
3. 等待VSync信号（约16.67ms）
   ↓
4. onAnimationFrame(timeMillis) 回调
   ↓
5. 计算当前帧的动画值
   ↓
6. 更新State，触发Recompose
   ↓
7. 循环回到步骤3
```

**接口定义：**
```kotlin
interface AnimationFrameClock {
    suspend fun <R> withFrameNanos(
        onFrame: (frameTimeNanos: Long) -> R
    ): R
}
```

**实现：**
- Android平台使用`Choreographer`
- 其他平台有各自的实现

### 7.5 Coroutines在动画中的应用

**为什么使用协程？**
- 动画需要持续执行多个帧
- 每帧之间需要等待VSync
- 需要支持取消、暂停
- 协程的suspend机制非常适合

**LaunchedEffect在动画中的作用：**

```kotlin
LaunchedEffect(targetValue) {
    // 在Composable作用域内启动协程
    animatable.animateTo(targetValue)
    
    // Composable离开作用域时，自动取消协程
}
```

**suspend动画函数：**

```kotlin
// Animatable.animateTo 是suspend函数
suspend fun animateTo(
    targetValue: T,
    animationSpec: AnimationSpec<T> = ...
): AnimationResult<T> {
    while (!finished) {
        val value = getNextValue()
        state.value = value
        delayFrame() // 等待下一帧
    }
    return AnimationResult(...)
}
```

**协程的取消：**
- Composable销毁时，LaunchedEffect自动取消
- 新动画启动时，旧动画自动取消
- 手动调用`animatable.stop()`取消动画

---

## 八、性能优化

### 8.1 避免不必要的Recompose

**问题示例：**
```kotlin
// 每帧都Recompose整个Box
@Composable
fun BadExample() {
    var progress by remember { mutableStateOf(0f) }
    
    LaunchedEffect(Unit) {
        while (true) {
            delay(16)
            progress += 0.01f
        }
    }
    
    // 整个Box每帧都Recompose
    Box {
        Text("Progress: $progress")
        OtherExpensiveComposable()
    }
}
```

**优化方案：**
```kotlin
// 使用Modifier.graphicsLayer避免Recompose
@Composable
fun GoodExample() {
    var progress by remember { mutableStateOf(0f) }
    
    LaunchedEffect(Unit) {
        while (true) {
            delay(16)
            progress += 0.01f
        }
    }
    
    Box {
        Text("Progress: $progress")
        // 使用graphicsLayer，只更新绘制层
        OtherExpensiveComposable(
            modifier = Modifier.graphicsLayer {
                scaleX = progress
                scaleY = progress
            }
        )
    }
}
```

**[Demo26] 优化前后的性能对比**
```
[GIF演示：使用工具对比优化前后的Recompose次数]
```

### 8.2 使用remember优化动画

**记住动画控制器：**
```kotlin
// ✅ 正确：使用remember
@Composable
fun CorrectExample() {
    val animatable = remember { Animatable(0f) }
    LaunchedEffect(Unit) {
        animatable.animateTo(1f)
    }
}

// ❌ 错误：每次Recompose都创建新动画
@Composable
fun WrongExample() {
    val animatable = Animatable(0f)
    LaunchedEffect(Unit) {
        animatable.animateTo(1f)
    }
}
```

**[Demo27] remember的正确使用**
```
[GIF演示：展示正确和错误两种方式的区别]
```

### 8.3 animate*AsState vs Animatable的选择

| 场景 | 推荐API | 原因 |
|------|---------|------|
| 简单的单值动画 | `animate*AsState` | 最简单，自动管理 |
| 需要暂停/恢复 | `Animatable` | 完全控制 |
| 手势驱动动画 | `Animatable` | 可响应手势 |
| 无限循环 | `rememberInfiniteTransition` | 专用API |
| 多状态切换 | `updateTransition` | 统一管理 |

**[Demo28] 两种方式的场景对比**
```
[GIF演示：同一需求用两种API实现，展示适用场景]
```

### 8.4 LaunchedEffect和SideEffect的正确使用

**LaunchedEffect：**
- 在Composable作用域内启动协程
- Composable销毁时自动取消
- 适合启动动画

**SideEffect：**
- 在Recompose成功后执行
- 不影响组合
- 适合更新非Compose状态

**示例：**
```kotlin
@Composable
fun AnimationExample() {
    var expanded by remember { mutableStateOf(false) }
    val size by animateDpAsState(if (expanded) 200.dp else 100.dp)
    
    LaunchedEffect(expanded) {
        // 启动动画
    }
    
    SideEffect {
        // 动画值改变后执行，更新外部状态
        externalCallback(size)
    }
}
```

### 8.5 使用graphicsLayer优化绘制动画

**适用场景：**
- 透明度、缩放、旋转、位移
- 不需要影响Layout的动画

**优势：**
- 不触发Recompose
- 直接操作绘制矩阵
- 性能最优

**示例：**
```kotlin
val animatedScale by animateFloatAsState(if (pressed) 0.95f else 1f)

Box(
    modifier = Modifier
        .graphicsLayer {
            scaleX = animatedScale
            scaleY = animatedScale
        }
) {
    // 内容不会Recompose
}
```

---

## 九、与传统View动画对比

### 9.1 完整对比表

| 特性 | View动画 | Compose动画 | 说明 |
|------|----------|-------------|------|
| **触发方式** | 调用start()等方法 | 改变State值 | Compose更直观 |
| **状态管理** | 需要手动跟踪 | 自动由State管理 | Compose自动处理 |
| **取消/暂停** | 需要手动调用cancel() | 自动处理 | Compose无内存泄漏风险 |
| **组合性** | 动画难以组合 | 动画天然可组合 | Compose支持复杂组合 |
| **类型安全** | 反射调用属性值 | 编译时类型检查 | Compose更安全 |
| **Recompose** | N/A | 动画值变化自动触发重绘 | Compose声明式特性 |
| **手势集成** | 需要手动处理 | 天然集成 | Compose更简单 |
| **测试** | 需要等待动画 | 可控制时间 | Compose更容易测试 |
| **调试** | 较难追踪 | 有调试工具支持 | Compose Layout Inspector |
| **性能** | 可能过度绘制 | 自动跳过未变化节点 | Compose更优 |
| **协程支持** | 不支持 | 原生支持 | Compose更现代 |

### 9.2 迁移示例

**View体系 → Compose**

**示例1：透明度动画**

View体系：
```kotlin
val animator = ObjectAnimator.ofFloat(view, "alpha", 0f, 1f)
animator.duration = 300
animator.start()
```

Compose：
```kotlin
var visible by remember { mutableStateOf(false) }
val alpha by animateFloatAsState(if (visible) 1f else 0f)
Box(modifier = Modifier.graphicsLayer { this.alpha = alpha })
```

**示例2：位移动画**

View体系：
```kotlin
val animator = ObjectAnimator.ofFloat(view, "translationX", 0f, 100f)
animator.duration = 300
animator.start()
```

Compose：
```kotlin
var offset by remember { mutableStateOf(0f) }
val animatedOffset by animateFloatAsState(offset)
Box(modifier = Modifier.offset { IntOffset(animatedOffset.roundToInt(), 0) })
```

**示例3：可见性动画**

View体系：
```kotlin
val animIn = AnimationUtils.loadAnimation(context, R.anim.fade_in)
view.startAnimation(animIn)
```

Compose：
```kotlin
var visible by remember { mutableStateOf(false) }
AnimatedVisibility(visible) {
    Content()
}
```

### 9.3 Compose动画的优势总结

1. **声明式，更直观**：代码即意图，易于理解和维护
2. **自动管理，无泄漏**：自动取消，无需手动清理
3. **类型安全，编译检查**：避免运行时错误
4. **组合性强，灵活扩展**：轻松组合多个动画
5. **性能优化，自动跳过**：只更新需要重绘的部分
6. **协程支持，现代架构**：与Kotlin协程深度集成

---

## 附录

### 依赖添加

```gradle
dependencies {
    implementation "androidx.compose.animation:animation:1.5.0"
    // 或
    implementation platform("androidx.compose:compose-bom:2023.10.01")
    implementation "androidx.compose.animation:animation"
}
```

### 学习资源

- [官方文档 - Compose Animation](https://developer.android.com/jetpack/compose/animation)
- [Compose Animation Codelab](https://developer.android.com/codelabs/jetpack-compose-animation)
- [Now in Android - 动画实践](https://github.com/android/nowinandroid)

---

**文档版本：** 1.0
**更新日期：** 2026-01-31
**适用于：** Compose 1.5+
