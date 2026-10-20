# 思路
## 软优化
* 预加载
* 异步加载
* 懒加载
## 硬优化
* 通过Profile找到性能问题，然后针对性修复
* View布局优化，可以通过ConstraintLayout来减少层级，也可以通过自定义View来优化布局
* 尽量避免在onCreateViewHolder、onBindViewHolder中new操作。比如点击事项可以让类实现onClickListener接口，onCreateViewHolder中imageView.setOnClickListener(this)。然后在onBindViewHolder中imageView.setTag。又或者SpanText做好缓存，避免每次滚动都解析。
* 底部加载更多：使用notifyItemRangeInserted代替notifyDataSetChanged。后者会触发2、3次onCreateViewHolder；
* glide首次加载图片会创建线程池，耗时约50ms，可以移到App打开时的欢迎界面就创建好；
* 在Adpater的 onViewRecycled 中把图片和评论remove后存入缓存池。这一步主要为了滚动流畅；
* 共享 RecyclerView 池 - 通过分析，我们观察到创建不同的 ViewHolder 需要更长的时间。为尽量减少该过程，我们创建了一个共享 RecyclerView 池。这还有助于消除了针对类似 Feed 创建 ViewHolder 的费用。
* 从 Rx 迁移到协程 - 内存消耗也会导致频繁的 GC 调用，并且 RX 线程数量超过 100 时线程数量非常高。
* 采用 Coil 进行图片加载 - Glide 会在加载图片时引发问题，特别是在通过 Jetpack Compose 构建的组件中时。我们还发现，在 LazyColumn 中加载图片时，呈现阈值条很高。这些情况促成了客户采用 Coil 进行图片加载。
* 旧的代码清理和重构 - 移除旧的代码和实验有助于从界面中移除不必要的隐藏视图，并且有助于以更好的方式重写部分界面。

# 常见问题
* 图文笔记等重资源回收方法比较耗时，可以把非即时的回收任务放到异步线程中执行；
* 数据打点调用频繁，数据的拼装、发送都可以放到异步线程中执行；
* 对于一些跨进程的操作，比如从Context中拿Service获取WiFi状态，我们只需要监听WIFI状态变化并存到全局变量中，onBind中只读该变量即可；
* RecycleView滑动时，避免执行item的初始化操作，如视频的prepare，仅展示封面即可；
* 复杂多文、富本场景，用StaticLayout对文本进行预渲染；
* 需要重复计算的size要设置为全局变量static，避免每次计算。比如每张图片的宽高，表情图片的15sp大小；
* 采用LruCache缓存最新的32个表情的drawable，这样可以加快常见表情的解析速度；
* 在Thread中按List.count() 解析item的xml布局，并存放在LinkedList 中（为了节省内存，我最多限制8条）。在onCreateViewHolder中进行 .poll
* 在Thread中按List.count() 预创建图片和评论的缓存池：LinkedList 和 LinkedList<评论TextView>。在item显示的时候。从缓冲池中取，而不是new；



