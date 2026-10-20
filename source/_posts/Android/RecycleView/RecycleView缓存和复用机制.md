---
title: RecyclerView 缓存与复用机制
date: 2026-10-21 00:09:42
---

# RecyclerView 缓存与复用机制

> 对应面试题 Q027（P0）。第一部分是完整的答题内容，第二部分起补充五个容器的精确定义、布局阶段和 bind 的作用。

---

## 一、四级缓存到底是什么（先把容易记错的属性分清）

| 层 | 存活周期 | 容量 | 命中后 | 匹配依据 |
| --- | --- | --- | --- | --- |
| mAttachedScrap | **一次 layout 内**（pre/post 之间） | 无限，随本次布局 | 不 bind | layoutPosition + viewType 校验 |
| mChangedScrap | pre-layout 到 post-layout | 同上 | **post-layout 要 bind** | 仅 viewType 不变者 |
| mCachedViews | 跨 layout 长期存在 | **全局默认 2**（不是每类型 2） | **不 bind** | 无 stable id 按 layoutPosition；有则按 id |
| ViewCacheExtension | 自己管 | 自己管 | 自己定 | 自己实现有效性判断 |
| RecycledViewPool | 进程内长期 | **每个 viewType 默认 5** | **必须 bind** | 仅按 viewType 分桶 |

两个高频误答先排掉：`mCachedViews` 的 2 是**整个 RecyclerView 共享 2 个**，而 pool 的 5 才是**按 viewType 各 5 个**；还有 `onViewRecycled` **只在进 pool 时调用，进 mCachedViews 时不调用**——因为 cache 里的 view 还保有绑定状态，告诉 Adapter“已回收”是错的。

## 二、取一个 ViewHolder 的完整查找链

`tryGetViewHolderForPositionByDeadline` 的顺序：

1. `mChangedScrap`：仅当处于 pre-layout 或数据有 change 时查。
2. `mAttachedScrap`：要求 `layoutPosition` 精确相等、`!isInvalid()`、非 removed；命中就打上 `FLAG_RETURNED_FROM_SCRAP`。
3. **hidden view**：`mChildHelper.findHiddenNonRemovedView(position)`——被 remove 但还在跑动画、仍然 attached 的 view 也算“可复用”，这正是方法名里 `OrHidden` 的来历。
4. `mCachedViews`：同样按 layoutPosition 匹配，命中就 `remove(i)` 并从缓存中摘出。
5. `ViewCacheExtension`。
6. `RecycledViewPool.getRecycledView(viewType)`。
7. 都没有 → `createViewHolder()`。

第 2、4 步命中后还有一道闸：`validateViewHolderForOffsetPosition(holder)`。它专门校验三件事——holder 是不是被标记 removed、`mPosition` 是否越界（越界会直接抛异常，这就是“复用到已不存在位置”的报错来源）、**以及 viewType 是否发生变化**（`mAdapter.getItemViewType(holder.mPosition) != holder.getItemViewType()` 则判定失效）。校验不过就把 holder 扔回去并强制 rebind。

## 三、判断“要不要重新 bind”的三个开关

命中一个 holder 之后，真正决定 bind 与否的是：

- `!holder.isBound()`：`FLAG_BOUND` 没置位 → 必须 bind。
- `holder.needsUpdate()`：`FLAG_UPDATE` 被置 → 必须 bind。
- `holder.isInvalid()`：`FLAG_INVALID` 被置，**或者 position 为负**（位置信息不可信）→ 必须 bind。

所以正确的心智模型是：**scrap/cache 命中 = 省掉 create + bind，但要过有效性校验；pool 命中 = 只省掉 create，bind 一定发生**（pool 在 `putRecycledView` 时已经 `resetInternal()` 清掉了 `FLAG_BOUND` 和数据引用）。

一个关键差异：`mCachedViews` 的匹配**在 Adapter 有 stable id（`setHasStableIds(true)`）时改为按 id 匹配**，走 `getScrapOrCachedViewForId`，并且要求该 id 出现在 `mUnmodifiedIds` 里——`notifyItemChanged` 会把 id 从这里移除，于是这个 view 即使还在 cache 里也不会被复用，必须重建。这就是“stable id 提升复用率”的确切含义。

## 四、回收侧：什么进 cache、什么进 pool

`recycleViewHolderInternal` 的判定顺序是：

- 带 `FLAG_INVALID / FLAG_REMOVED / FLAG_IGNORE / FLAG_RETURNED_FROM_SCRAP` 的，**直接跳过 cache 进 pool**（甚至直接丢弃）。`FLAG_RETURNED_FROM_SCRAP` 专指“动画还没跑完的 view 不能被抢走”，这是动画正确性的保护。
- 否则进 `mCachedViews`；若 cache 已满，就把**队首（最旧的）**弹出 → `addViewHolderToRecycledViewPool` → 此时才 `dispatchViewRecycled` → 回调 `onViewRecycled`。

注意 `detach` 与 `remove` 的区别：`detach` 只是从父容器摘下、view 仍被 RecyclerView 持有（可进 scrap/cache），`remove` 才是彻底走回收。这解释了为什么动画期间 view 还在，却已经不参与正常复用了。

## 五、刷新（notify）各走哪条路

- **结构变更**（insert/remove/move）：`mStructureChanged` 置位，并且会调用 `offsetPositionRecordsForInsert/Remove/Move` 去**同步修正 `mCachedViews` 里每个 holder 的 position**。所以插入数据不会让 cache 整体失效，命中依然不 bind——这正是“位置一致”必须由 Adapter 维护的原因。
- **内容变更**（notifyItemChanged）：走 **pre-layout + post-layout 两遍布局**。第一遍用旧位置算动画起止点；变化但 viewType 不变的 view 落进 `mChangedScrap`，第二遍真正布局时重 bind。
- **全量变更**（notifyDataSetChanged）：设 `mStructureChanged`，跳过 pre-layout，**所有 view 失效**，全部 rebind、无动画、无明显复用。
- `setAdapter / swapAdapter`：cache 与 pool 会被清空。
- 高版本还有 `GapWorker` 的 **prefetch**：在上一帧空闲时提前把下一个 item 取出来 bind，本质上也是往 cache/pool 里预热。

## 六、追问一：滚动和刷新时各层怎么用

滚动的关键是 **cache 是 FIFO 且只有 2 个**，所以命中率有方向性：

- 小幅来回滚动（上下抖）——刚离屏的 view 恰好还躺在 cache 里，命中后**不 bind**，不掉帧。
- 快速同向滚动——需要的 position 早就被挤进 pool 了，命中后**要 bind**，然后是 create。这就是“快速滑动卡顿”的机制性来源。

所以调 `setItemViewCacheSize()` 只对“就近回滚”有效，对大幅滚动几乎无用；大幅滚动要做的是**降低 bind 成本**（bind 里别做重活），不是加大缓存。

刷新时按上面三条路分别说：结构变更靠 position offset 保住 cache；内容变更靠 pre/post layout + `mChangedScrap` 保动画、post 阶段重 bind；全量变更等于放弃复用。

## 七、追问二：共享 pool 的四个前提

pool **只按 viewType 分桶、且只有 pool 能跨 Adapter/RecyclerView 共享，cache 不行**。能共享的前提：

1. **viewType 在全局唯一映射到同一种 item**。两个 Adapter 各自从 0 开始编号就会串味，建议用全局枚举分配 viewType。
2. **LayoutParams 约束兼容**。复用过来的 view 带着原 parent 的 LayoutParams，横向列表若宽度依赖父约束，换列表就错——要么只共享同构列表，要么让 item 自适应宽度。
3. **回收清理对称**。`onViewRecycled` 里必须解绑 listener、停动画、清图片，否则 A 列表的状态会带到 B。
4. **容量要重算**。共享后驻留时间变长，用 `setMaxRecycledViews(viewType, n)` 按类型收紧，避免内存膨胀。

收益是嵌套 RV 的 inflate 和首次 bind 大幅减少；翻车症状固定是三件套：**内容串行、高度跳变、点击位置错**。

额外一句：stable ID 与缓存不是同一件事——它解决的是“notify 之后能否按 id 找回同一个 holder”，改变不了绑定契约，也**不能用 position 当业务 id**，否则 `mUnmodifiedIds`、position offset、diff 动画会连环错位。

---

## 八、补充一：五个容器分别是什么

### 1. mAttachedScrap（`ArrayList<ViewHolder>`）

- **定义**：一次布局过程中，临时存放“本来 attached 在 RecyclerView 上、被 detach 下来等待重新摆放”的 ViewHolder。
- **进入条件（什么情况的 item 会进来）**：
  - 布局开始时**仍 attached 在 RecyclerView 上**的所有子 view。`dispatchLayoutStep1/2` 在调用 `LayoutManager.onLayoutChildren` 之前会执行 `detachAndScrapAttachedViews(recycler)`，把当前屏幕上的孩子逐个摘进 scrap——所以进入者是“**这一帧还在屏幕上、需要在本次布局里重新摆放**的 view”。
  - 布局过程中被 LayoutManager 主动临时摘下的 view（`detachAndScrapView`），例如为了先测后摆或调整锚点。
  - 判定分支在 `Recycler.scrapView`：holder 带 `FLAG_REMOVED / FLAG_INVALID`，或**未被标为 updated**，或 `canReuseUpdatedViewHolder(holder)` 为真（change 带了 payload、动画认为可以直接复用同一个 holder）→ 进 `mAttachedScrap`；否则进 `mChangedScrap`。
- **不会进来**：**已经划出屏幕的 view**。滑动出界的 child 会在 fill 阶段被 `removeAndRecycleView` 交回 Recycler，走 recycle 路径进 `mCachedViews` 或 pool；处在动画中、带 `FLAG_RETURNED_FROM_SCRAP` 的 view 也不会被再次取用。
- **命中条件**：`layoutPosition` 完全相等、未标 invalid、未被 remove。命中后直接重新 attach，**不触发 onBindViewHolder，也不回调 onViewRecycled**。
- **生命周期**：只在 `dispatchLayout` 内有效，布局结束即清空——它不是跨帧缓存。
- **本质**：它是“布局算法的工作区”，用于支持布局过程中对同一批 view 的反复摆放（例如 LinearLayoutManager 先测后摆）。

> **问：mAttachedScrap 会存放“原本可见、但现在被划出屏幕”的 item 吗？**
>
> **答：不会。** 它只装“本次布局开始时仍 attached 的 view”。一个原本可见的 view 确实会在布局开始时先被摘进 scrap，但如果随后在 fill 阶段被判定为出界，它会被从 scrap 取出并走 recycle 进 `mCachedViews`/pool，**最终不会留在 scrap 里**。跨帧保存“刚划出屏幕”的 view 是 `mCachedViews` 的职责。

### 2. mChangedScrap（`ArrayList<ViewHolder>`）

- **定义**：`mAttachedScrap` 的兄弟容器，专门放**内容被 notify 改动、需要参与预测动画**的 ViewHolder。
- **进入条件（什么情况的 item 会进来）**：`Recycler.scrapView` 的另一条分支——holder 被标为 **updated**（`FLAG_UPDATE`，即经历过 `notifyItemChanged` 这类“内容变了”的更新）、不带 `FLAG_REMOVED / FLAG_INVALID`，并且 `canReuseUpdatedViewHolder(holder)` 为 **false**（默认 `DefaultItemAnimator` 在**没有 payload** 时返回 false）→ 进 `mChangedScrap`。
  - 换句话说：**“内容变了、又没有 payload、需要做旧→新过渡动画”的 view 才来这里**；带 payload 的 change 会被判定为可复用，走 `mAttachedScrap`，只做局部重绑。
  - 另一个前提是 pre-layout 已经启用，否则这个容器根本不会被填充。
- **与 attachedScrap 的分工**：`mAttachedScrap` 中的 holder 在 pre-layout 与 post-layout 之间不能复用（因为布局位置会变，视为已失效）；`mChangedScrap` 中的 holder 恰恰是**唯一允许跨 pre/post 两段保留**的，因为动画需要它从旧状态过渡到新状态。
- **限制**：只有 viewType 未变的 view 才会进这里；viewType 变了就没有稳定的过渡形态，只能重建。
- **结局**：pre-layout 结束后，post-layout 布局时这些 holder 会被重新 bind（内容已经变了），因此它命中不 bind 这条规律对它不成立。
- **可见性**：只在 `mState.mInPreLayout` 为 true 时才去查它（`getChangedScrapViewForPosition` 先按 layoutPosition 找，有 stable id 时再按 id 找）。

### 3. mCachedViews（`ArrayList<ViewHolder>`）

- **定义**：第一级真正意义上的“复用缓存”，保存刚刚离开屏幕、**位置连续**的 ViewHolder，且**保留完整绑定状态**。
- **进入条件（什么情况的 item 会进来）**：入口是 `Recycler.recycleViewHolderInternal`，触发者是滑动/布局中 `removeAndRecycleView` 一类的回收动作。典型画像就是——**刚刚离开屏幕、状态仍然有效（`FLAG_BOUND` 未被破坏）、没有 remove/invalid 标记、且 cache 未满**的 view。
  - 被排除、直接丢进 pool 的情况：容量为 0（`setItemViewCacheSize(0)`）；带 `FLAG_INVALID / FLAG_REMOVED / FLAG_IGNORE / FLAG_RETURNED_FROM_SCRAP`；带 `FLAG_NOT_RECYCLABLE`（直接丢弃，连 pool 都不进）。
  - 数据变更的影响：`notifyItemInserted/Removed/Moved` 会通过 `offsetPositionRecordsForInsert/Remove/Move` **修正 cache 里 holder 的 position**，让它们仍然对应同一条数据；`notifyItemChanged` 会把对应 id 从 `mUnmodifiedIds` 移除（有 stable id 时），使这个 holder 虽然还在 cache 里也命中不了；`setAdapter / swapAdapter` 会整体清空。
  - 在里面的 view 一定是“位置连续、紧邻可见区”的那几个，这正是容量只有 2 的原因。
- **容量**：`DEFAULT_CACHE_SIZE = 2`，用 `setItemViewCacheSize(int)` 调整；它是**整个 RecyclerView 共享的 2 个，不是每个 viewType 2 个**。
- **FIFO 语义**：满了以后从队首（最旧的）弹出，走 `resetInternal` 后进 pool。
- **命中效果**：直接 attach，**不 bind**；即便如此仍要过 `validateViewHolderForOffsetPosition`。
- **匹配规则**：Adapter 无 stable id 时按 `layoutPosition` 精确相等匹配；有 stable id 时按 id 匹配并额外校验该 id 属于 `mUnmodifiedIds`。
- **注意**：进 cache **不会**触发 `onViewRecycled`，因为它还“有主”。

### 4. ViewCacheExtension（抽象类）

- **定义**：开发者挂钩点，`getViewForPositionAndType(Recycler, position, type)` 返回一个 view 或 null。
- **位置**：在查找链的第 5 位——cache 之后、pool 之前。
- **特点**：RecyclerView 完全不校验你返回的 view 是否正确，有效性与状态清理全部由实现者负责；一旦返回错误的 view，症状是内容错位且极难排查。因此实践中基本无人使用，属于“知道它存在、知道为什么不用”的层次。

### 5. RecycledViewPool（`SparseArray<ScrapData>`）

- **定义**：按 viewType 分桶的 ViewHolder 池，是**唯一支持跨 Adapter、跨 RecyclerView 共享**的一层。
- **容量**：`DEFAULT_MAX_SCRAP = 5`，**每种 viewType 各自 5 个**；用 `setMaxRecycledViews(viewType, max)` 按类型调整。
- **入池时**：`putRecycledView` 会先 `resetInternal()`——清空 `FLAG_BOUND`、数据引用、`mPosition`，并 `dispatchViewRecycled` 回调 `onViewRecycled`；带 `FLAG_RETURNED_FROM_SCRAP` 的 view 不会被放入。
- **出池时**：只按 viewType 取，**取出后必须 `bindViewHolder`**，因为边界状态已经清干净了。
- **使用方式**：`RecyclerView.setRecycledViewPool(sharedPool)`、`LayoutManager.setRecycledViewPool()`；嵌套列表的常见优化就是内层复用外层的 pool。
- **代价**：共享后 view 驻留时间变长，内存占用上升，需要配合容量调整。

> 记忆主线：**scrap 是布局内的临时工作区，cache 是跨帧的“带数据复用”，extension 是自定义钩子，pool 是跨类型/跨列表的“空壳复用”。**

### 6. 一张表区分“同一批 view 在不同时刻去哪”

| 场景 | 走哪个容器 | 原因 |
| --- | --- | --- |
| 布局开始，屏幕上原有的 view 要重新摆放 | `mAttachedScrap` | 仍 attached，本次布局还要用 |
| 布局/滚动中被判定出界、滑出屏幕 | `mCachedViews`（满了转 pool） | 走 recycle 路径，跨帧保留状态 |
| `notifyItemChanged` 且**无 payload**、需要过渡动画 | `mChangedScrap` | 要保留旧形态做动画，post-layout 再 bind |
| `notifyItemChanged` 且**带 payload** | `mAttachedScrap` | 判定可复用同一 holder，只做局部重绑 |
| `notifyItemRemoved` 后仍在跑动画 | hidden view（`findHiddenNonRemovedView`）+ `FLAG_RETURNED_FROM_SCRAP` | 被保护，不许被抢用 |
| 被回收且 cache 已满、或状态已失效 | `RecycledViewPool` | 状态已 reset，只能按 viewType 复用 |
| `notifyDataSetChanged` / `setAdapter` 之后 | 全部失效，直接 create + bind | 没有可信的位置与旧状态 |

---

## 九、补充二：布局的各个阶段

### 1. 三个官方布局阶段（`dispatchLayout`）

RecyclerView 一次布局会依次走 `dispatchLayoutStep1/2/3`：

| 阶段 | 名称 | 做什么 | 关键标志 |
| --- | --- | --- | --- |
| dispatchLayoutStep1 | pre-layout（预测布局 / 预测动画准备） | 用**旧数据**跑一次 `onLayoutChildren`，记录每个 view 的“动画前位置”；把变化中的 view 放进 `mChangedScrap`；算出消失/出现/移动的集合供动画使用 | `mState.mInPreLayout = mState.mRunPredictiveAnimations` |
| dispatchLayoutStep2 | post-layout（真实布局） | `mState.mInPreLayout = false`，用**新数据**真正摆放所有 view，必要时重新 bind；这一步才是用户看到的最终布局 | `mState.mItemCount` 更新 |
| dispatchLayoutStep3 | 动画与收尾 | 处理“pre-layout 时可见、真实布局后消失”的 view 的回收；执行 `mItemAnimator.runPendingAnimations()`；`mLayout.onLayoutCompleted()`；清空 scrap、清理 `ViewInfoStore`、回收动画遗留的 view | `mState.mDeletedInvisibleItemCountSincePreviousLayout` |

为什么必须两遍布局：只有一个 View 在一帧内既保留旧位置又知道新位置，`ItemAnimator` 才能算出位移/渐变的起止点。这也是 `notifyItemChanged` 能有“原地淡入淡出”的原因。

补充两个前提条件（很多人说不清）：

- `mRunSimpleAnimations`：有 `ItemAnimator`、且数据集发生的是会被感知的变化时才为真。
- `mRunPredictiveAnimations`：在简单动画的基础上，**还要求 LayoutManager 支持预测动画**（`supportsPredictiveItemAnimations()`，Linear/Grid 支持，部分自定义 LM 不支持）；只有它为真，pre-layout 才真正启用，`mChangedScrap` 才会被查询。

### 2. 除 pre-layout / post-layout 之外，还有哪些阶段

- **onMeasure 中的提前布局**：如果 `onMeasure` 时还来不及布局，会在这里直接跑 `dispatchLayoutStep1/2`，把布局提前到测量阶段完成。
- **帧间隙的 prefetch（预取）**：`GapWorker` 在下一帧到来前的空闲时间调用 `LayoutManager.collectAdjacentPrefetchPositions`，提前把即将进入屏幕的 item 取出来并 bind。它发生在布局之外，是 25.1 之后新增的“抢在滑动前把活干完”的机制。
- **动画阶段（ItemAnimator）**：`animateAppearance / animateDisappearance / animateChange / animateMove` + `runPendingAnimations`；动画期间相关 view 带 `FLAG_RETURNED_FROM_SCRAP`，受到“不被拿走”的保护，动画结束后才允许回收。
- **回收阶段（recycle）**：滑动中 `removeAndRecycleView` 触发 detach → cache → pool 的迁移，与布局阶段交错发生，是“滚动时缓存一直在换手”的原因。
- **绘制阶段（onDraw / onDrawOver）**：`ItemDecoration` 的两层绘制分别画在 item 之下与之上；不属于布局，但同属 measure-layout-draw 一次遍历。

### 3. 常被问到的 ViewHolder 状态位

| 标志 | 含义 |
| --- | --- |
| FLAG_BOUND | 已绑定，位置可信 |
| FLAG_UPDATE | 数据可能变了，需要重 bind |
| FLAG_INVALID | 必须重 bind，且不能直接复用 |
| FLAG_REMOVED | 已被移除（动画期间仍可能存在） |
| FLAG_RETURNED_FROM_SCRAP | 来自 scrap / 动画中，不许被抢用 |
| FLAG_NOT_RECYCLABLE | 不可回收，直接丢弃 |
| FLAG_ADAPTER_POSITION_UNKNOWN | 在真实布局前 position 未知 |
| FLAG_MOVED | 发生了移动动画 |

---

## 十、补充三：bind 的作用是什么

**一句话：bind 是“把复用的空壳 View 变成代表某个 position 数据的那一帧视图”的动作。**

### 1. 调用链与它到底改了什么

```
Recycler.tryGetViewHolderForPositionByDeadline
  → bindViewHolder(holder, position)
       holder.mPosition = position            // 位置生效
       holder.mItemId = getItemId(position)   // 有 stable id 时
       holder.setFlags(FLAG_BOUND,
           FLAG_BOUND | FLAG_UPDATE | FLAG_INVALID | FLAG_ADAPTER_POSITION_UNKNOWN)
                                              // 置 BOUND，清掉三个“需要重来”的标志
       adapter.bindViewHolder(holder, position)
            → onBindViewHolder(holder, position, payloads)
```

它做的事情可以归为三类：

1. **数据 → 视图的投影**：`createViewHolder` 只负责 inflate 和 `findViewById`，拿到一个没有任何数据的空壳；bind 才把 item 数据显示到各个子 view 上。
2. **维护 holder 的状态与身份**：写入 `mPosition`、`mItemId`，置 `FLAG_BOUND` 并清掉 `FLAG_UPDATE / FLAG_INVALID / FLAG_ADAPTER_POSITION_UNKNOWN`。**bind 完成后这个 holder 才被认为是“位置可信、可直接复用”的**——这也解释了第三节那三个开关的来历：它们检的就是 bind 有没有被失效掉。
3. **承载局部刷新**：带 payload 的 `notifyItemChanged(pos, payload)` 会走到三参 `onBindViewHolder`，只更新变化的部分，避免整块重绑。

### 2. 为什么必须有 bind

屏幕上只有 M 个 View，数据有 N 条，且 `M ≪ N`。复用机制解决的是“View 不够用”的问题，而 **bind 解决的是“这个 View 现在代表哪条数据”的问题**——没有 bind，池子里捞出来的 view 依然是上一条数据的样子。create 是昂贵的（inflate + 查找子 view），bind 是廉价的，所以复用的价值在于**把 create 摊薄**。

### 3. 什么时候一定会 bind

- pool 命中（状态已被 `resetInternal` 清空）；
- **viewType 与当前位置要求的不一致**（`validateViewHolderForOffsetPosition` 判定失效）；
- holder 被标 `FLAG_UPDATE` 或 `FLAG_INVALID`，或 `mPosition` 为负、越界；
- 新建 holder；
- `notifyDataSetChanged` / `setAdapter` 之后的全量重绑。

### 4. 工程约束（面试常追问“bind 里能做什么”）

- bind 在**主线程执行**，只做映射与轻量格式化；IO、图片解码、复杂计算都应异步化或前置缓存，否则直接体现为滚动掉帧（顺接 Q028）。
- 不要用 `onCreateViewHolder` 里的 `position`，也不要在 bind 时把 `position` 捕获进 listener 闭包——用 `getBindingAdapterPosition()` / `getAbsoluteAdapterPosition()`，否则复用后点击错位。
- 重活交给 `DiffUtil` / payload 局部更新，而不是靠加大缓存掩盖。
- 清理动作的对称点在 `onViewRecycled`（仅对进 pool 的 view 触发），bind 里加的监听、动画、图片请求要在这里或 `onViewDetachedFromWindow` 对应释放。
- 若在 bind 中做过一次性的昂贵初始化，用 `onViewAttachedToWindow`/首次 bind 标记区分，而不是每次 bind 重复执行。
