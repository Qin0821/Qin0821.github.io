---
title: LRU 原理与应用场景（Android）
date: 2026-10-20 21:35:13
---

# LRU 原理与应用场景（Android）

> 大白话版。先说清"LRU 到底是什么、为什么能 O(1)"，再讲 Android 里真实用到的 LruCache / DiskLruCache 的源码要点，最后是常踩的坑和面试追问。

---

## 一、LRU 到底在解决什么问题

一句话：**容量有限，装不下全部数据，那就淘汰"最久没被用过"的那一个。**

背后的假设叫局部性原理——近期用过的数据，接下来大概率还会用；很久没碰的，基本不会再碰。所以"最近使用时间"这个指标就够用了。

注意 LRU 管的是**淘汰策略**，不是淘汰时机。触发时机一般是 put 之后超出容量（Android 的 LruCache 走 `trimToSize`）。

## 二、为什么必须是「哈希表 + 双向链表」

把需求拆开看，数据结构就自动定下来了：

- 要按 key 查 → 必须 O(1) → **哈希表**
- 要区分"谁最新、谁最旧" → 必须有序 → **链表**
- 被访问的元素要挪到最新位置 → 要 O(1) 摘除+插入 → **双向链表**（单链表拿不到前驱，删不掉自己）
- 哈希表的 value 直接指向链表节点 → O(1) 就能定位到节点

于是两个核心操作：

- **get(key)**：哈希表 O(1) 找到节点 → 从链表摘下来 → 挂到头部（最新端）
- **put(key, value)**：key 已存在 → 更新值并移到头部；不存在 → 新建节点挂头部，若超容量则把**尾部节点（最旧）**摘掉，同时从哈希表里删掉它的 key

读写都是 O(1)，空间 O(capacity)。这就是 LRU 的全部。

## 三、手写 LRU（面试高频）

### 版本 A：LinkedHashMap 偷懒写法

```java
public class LruCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;

    public LruCache(int capacity) {
        super(capacity, 0.75f, true); // 关键：accessOrder = true
        this.capacity = capacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity; // 返回 true 就自动删掉最旧的那个
    }
}
```

关键就一个参数 `accessOrder = true`。默认 false 是**插入顺序**，true 才是**访问顺序**——每次 get/put 都会把该节点移到链表尾部（LinkedHashMap 里"尾部 = 最近使用"）。

### 版本 B：自己写双向链表 + HashMap

```java
public class LRUCache<K, V> {
    private static class Node<K, V> {
        K k; V v; Node<K, V> prev, next;
        Node(K k, V v) { this.k = k; this.v = v; }
    }

    private final int capacity;
    private final Map<K, Node<K, V>> map = new HashMap<>();
    private final Node<K, V> head = new Node<>(null, null); // 哨兵：head.next 是最新的
    private final Node<K, V> tail = new Node<>(null, null); // 哨兵：tail.prev 是最旧的

    public LRUCache(int capacity) {
        this.capacity = capacity;
        head.next = tail;
        tail.prev = head;
    }

    public V get(K key) {
        Node<K, V> n = map.get(key);
        if (n == null) return null;
        moveToHead(n);
        return n.v;
    }

    public void put(K key, V value) {
        Node<K, V> n = map.get(key);
        if (n != null) {           // 已存在：更新值 + 提为最新
            n.v = value;
            moveToHead(n);
            return;
        }
        n = new Node<>(key, value);
        map.put(key, n);
        addToHead(n);
        if (map.size() > capacity) {          // 超容量，淘汰最旧
            Node<K, V> oldest = tail.prev;
            remove(oldest);
            map.remove(oldest.k);             // 这一步漏了就泄漏
        }
    }

    private void moveToHead(Node<K, V> n) { remove(n); addToHead(n); }
    private void addToHead(Node<K, V> n) {
        n.next = head.next; n.prev = head;
        head.next.prev = n; head.next = n;
    }
    private void remove(Node<K, V> n) {
        n.prev.next = n.next; n.next.prev = n.prev;
    }
}
```

两个易错点：① 淘汰时**必须同时 `map.remove(key)`**，否则 map 里留着一个谁也拿不到的野节点；② 用哨兵头尾节点能省掉一堆 null 判断，也顺手避免"只有一个节点时删出空指针"。

## 四、Android 的 `LruCache`，比裸 LinkedHashMap 多了什么

`android.util.LruCache` 内部就是一个 `LinkedHashMap(0, 0.75f, true)`，但有几个差别面试常问：

| 点 | 说明 |
| --- | --- |
| 计数单位 | 算的不是"条数"，而是 `sizeOf(key, value)` 的返回值，默认是 1。缓存 Bitmap 必须重写为 `bitmap.getByteCount() / 1024` |
| 淘汰粒度 | `trimToSize` 里是 while 循环，从最旧开始连续删，直到 `totalSize <= maxSize`，一次可能删掉好几个 |
| 线程安全 | 内部 `synchronized (this)`，本身是安全的；但 `entryRemoved` 回调里不要再回调 LruCache，会死锁 |
| 强引用 | map 里存的是**强引用**，缓存对象如果反向持有 Activity/Context，照样泄漏 |
| 统计 | 提供 `hitCount` / `missCount` / `createCount`，可以直接算命中率 |
| 动态调整 | `resize(maxSize)` 改容量后会立即 trim 一次 |

典型用法（图片内存缓存）：

```java
int maxKb = (int) (Runtime.getRuntime().maxMemory() / 1024 / 8); // 一般取可用堆的 1/8
LruCache<String, Bitmap> cache = new LruCache<String, Bitmap>(maxKb) {
    @Override
    protected int sizeOf(String key, Bitmap bmp) {
        return bmp.getByteCount() / 1024;
    }
};
```

## 五、`DiskLruCache`：磁盘版 LRU 的几个坑

JakeWharton 的 DiskLruCache 是磁盘 LRU 的事实标准，Glide、OkHttp 都基于它（或它的变体）。原理要点：

- 一个 `journal` 日志文件记录每次写入（`DIRTY` / `CLEAN` / `REMOVE`），启动时**重放日志重建索引**——这就是它不需要每次扫全目录的原因；
- 写入流程必须是 `edit()` → `commit()`（或 `abort()`）。**没 commit 的条目启动时会被清掉**，这是崩溃自愈的设计；
- `get()` 返回的是快照（snapshot），多个快照可以共用同一个 entry，**全部 close 之后文件才可能被删除**——忘了 close 就删不掉文件；
- **不是多进程安全的**，两个进程共用一个缓存目录会把 journal 写坏。

## 六、Android 里到底哪些地方在用

| 场景 | 具体实现 | 说明 |
| --- | --- | --- |
| 图片内存缓存 | `LruCache`、Glide 的 `LruResourceCache` | Glide 按 `ActivityManager.getMemoryClass()` 的 20%~40% 动态算容量 |
| 图片磁盘缓存 | Glide `DiskLruCacheWrapper`、自研 DiskLruCache | Glide 默认 250MB，分 `AUTOMATIC` / `DATA` / `RESOURCE` / `NONE` 四种策略 |
| Bitmap 复用池 | Glide `BitmapPool`（LruBitmapPool） | 按字节数计容量，滚动列表时避免反复 new 大 Bitmap |
| 正在显示的资源 | Glide `ActiveResources`（弱引用 Map） | 正在展示的图片用弱引用保活，**不放进 LRU**，防止被淘汰后被重新加载 |
| 网络响应缓存 | OkHttp `Cache` | 磁盘缓存按 size 做 LRU 裁剪 |
| View 复用 | `RecycledViewPool` | **不是严格 LRU**，是"按 viewType 分桶的栈"，每个类型默认存 5 个，只进不出排序 |
| 消息对象池 | `Message` 的 sPool | **不是 LRU**，是链表池（FIFO 复用） |
| 连接池 | OkHttp `ConnectionPool` | **不是 LRU**，是按空闲超时（默认 5 分钟）淘汰 |

一句话面试答法：**"内存缓存看 LruCache，磁盘缓存看 DiskLruCache，Glide 两个都用；RecycledViewPool、Message 池、连接池名字像但都不是 LRU，别混为一谈。"**

## 七、LRU 的局限（能说出这个才是加分项）

- **缓存污染**：一次全表扫描 / 顺序遍历，会把所有真正的热点数据全部挤出去，之后命中率暴跌。这是"LRU 对扫描型访问不友好"的经典缺陷。
- **只看新旧，不看频率**：只被访问过一次的冷数据，只要它是"刚访问的"，也会占住位置。
- **改进算法**：LFU（按访问次数）、LRU-K（看最近 K 次访问的时间间隔）、2Q（把缓存拆成"新进队列 + 热点队列"）、ARC（自适应调节）。OS 层更常用 **CLOCK（时钟）算法**——给每页一个引用位，指针转一圈淘汰未引用的，近似 LRU 且不用维护链表；Linux 页缓存用的是 active/inactive 双链表的近似 LRU。

## 八、面试追问速答

- **为什么不用数组或单链表？** 数组中间删除是 O(n)；单链表拿不到前驱节点，无法 O(1) 删除自己。双向链表 + 哈希表才凑齐 O(1)。
- **get 也算一次"使用"吗？** 算，而且实现里必须把节点移到链表头，否则 get 不改变顺序，整个结构就退化成 FIFO。
- **`LinkedHashMap` 开 `accessOrder=true` 有什么副作用？** 每次 get 都会改结构（`modCount++`），所以遍历过程中调 get 会抛 `ConcurrentModificationException`；而且它本身非线程安全，多线程要自己加锁。
- **怎么防内存泄漏？** 内存版 map 是强引用，缓存对象别反向持有 Context/Activity；磁盘版快照用完记得 close，否则对应文件永远删不掉。
- **容量怎么定？** 内存缓存的通行经验是**可用堆的 1/8**（LruCache 作者建议），Glide 这类库会按 `memoryClass` 按比例动态计算；磁盘缓存常见 50~250MB 上限，同时留一个"可用空间低于 10% 就清理"的兜底。
