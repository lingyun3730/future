# Java HashMap：中英文要点

用法：每个点先看中文理解，再背英文说法。回答公式：**原理 → 取舍 → 项目例子**。并发角度的 ConcurrentHashMap 见 `01-multithreading.md`。

## 1. 数据结构（JDK 8）

**数组 + 链表 + 红黑树。**

```
table[0]
table[1] → Node → Node → Node        （链表）
table[2]
table[3] → TreeNode ...               （红黑树）
```

- 数组的每个格子叫 **bucket（桶）**，存的是链表的**头节点**，头节点本身就是一条完整的数据。
- 每个 `Node` 存四样东西：`hash`、`key`、`value`、`next`。
- 节点里存 key，是因为不同 key 可能落在同一个桶里，查找时要逐个比较 key。

**English:** "HashMap is an array of buckets. Each bucket holds a linked list, or a red-black tree when the list gets long. Each node stores the hash, key, value, and a next pointer."

## 2. put 的流程（高频）

1. 数组为空 → 先 `resize()` 初始化（懒初始化，默认长度 16）。
2. 计算 hash，定位桶：`index = (n - 1) & hash`。
3. 桶为空 → 直接放入新节点。
4. 桶不为空：
   - 头节点的 key 相同（hash 相等且 `equals`）→ 覆盖 value。
   - 是红黑树 → 按树的方式插入。
   - 是链表 → 遍历到尾部追加；已有相同 key 则覆盖。链表长度达到 8，尝试转红黑树。
5. 插入新节点后，`size` 超过阈值（`容量 × 0.75`）→ **扩容**。

**English:** "On put, HashMap computes the bucket index from the key's hash. If the bucket is empty, it inserts a node. Otherwise it walks the list or tree, overwrites the value if the key already exists, and appends a new node if not. After inserting, if the size exceeds the threshold, it resizes."

## 3. hash 的计算与下标

```java
static int hash(Object key) {
    int h = key.hashCode();
    return h ^ (h >>> 16);        // 高 16 位异或低 16 位
}
index = (n - 1) & hash;            // n 是 2 的幂
```

- **为什么高位异或低位：** 数组较小时，下标只用到 hash 的低几位。把高位混进来，让高位也参与运算，**减少冲突**。
- **为什么容量是 2 的幂：** `(n - 1) & hash` 等价于取模，但比 `%` 快；并且扩容时元素的新位置只有两种可能（见第 5 节）。

**English:** "HashMap XORs the high 16 bits into the low bits so that the high bits also affect the index, which reduces collisions. The capacity is always a power of two, so `(n - 1) & hash` works as a fast modulo."

## 4. 链表转红黑树

- **条件：** 链表长度 ≥ 8，**并且**数组长度 ≥ 64。如果数组长度 < 64，先**扩容**而不是转树。
- **退化：** 树节点数 ≤ 6，转回链表。
- **为什么阈值是 8：** 在 hash 分布良好时，一个桶里出现 8 个节点的概率极低（泊松分布，约千万分之六）。转树是对极端情况（差的 hash 函数、恶意构造冲突）的保护，把最坏查询从 O(n) 降到 O(log n)。
- **为什么 8 转树、6 退化：** 中间留 7 作缓冲，避免在边界反复转换。

**English:** "When a bucket's list reaches 8 nodes and the table has at least 64 slots, it's converted into a red-black tree, so the worst-case lookup is O(log n) instead of O(n). If the table is smaller than 64, it resizes first. It turns back to a list at 6 nodes, and the gap prevents flip-flopping."

## 5. 扩容 resize

- **触发：** `size > 容量 × 负载因子`。默认容量 16，负载因子 0.75，所以第 13 个元素放入后扩容。
- **过程：** 新数组长度是原来的 **2 倍**，然后把旧数据迁移到新数组。
- **JDK 8 的优化：** 不需要重新计算 hash。因为容量翻倍，新下标只可能是 **原下标** 或 **原下标 + 旧容量**，只要看 hash 多出来的那一位是 0 还是 1。链表迁移后还能保持原来的相对顺序。
- **代价：** 扩容是 O(n)，所以已知大小时，建议**初始化指定容量**，避免多次扩容。
  - 预计放 `N` 个元素，容量约 `N / 0.75 + 1`。

**为什么负载因子是 0.75：** 空间和时间的折中。太大 → 冲突多，查询慢；太小 → 浪费空间，频繁扩容。

**English:** "When the size exceeds capacity times the load factor, which is 0.75 by default, HashMap doubles the array. In JDK 8, a node's new index is either the old index or the old index plus the old capacity, so it doesn't recompute hashes. Resizing costs O(n), so when I know the size, I set the initial capacity."

## 6. key 的要求：hashCode 与 equals

- 判断"相同 key"的顺序：先比 `hash`，再比引用 `==`，最后 `equals`。
- **重写 equals 必须同时重写 hashCode**，并保证：`a.equals(b)` 则 `a.hashCode() == b.hashCode()`。
  - 否则两个逻辑上相同的 key 会落在不同的桶，`get` 找不到。
- **key 最好是不可变对象**（如 `String`、`Integer`）。
  - 如果 key 放入之后，参与 hashCode 的字段被修改，hash 变了，就再也**找不到**这条数据，并造成内存泄漏。
- 允许 **一个 null key**（放在下标 0），允许 null value。

**English:** "If I override equals, I must override hashCode too, otherwise equal keys may land in different buckets and get() fails. Keys should be immutable, because if a field used by hashCode changes after insertion, the entry can't be found anymore."

## 7. 线程不安全

**表现：**
1. **丢失更新：** 两个线程同时 put 到同一个桶，一个覆盖另一个；或者 `size++` 不是原子的，数量不准。
2. **扩容时数据丢失或错乱。**
3. **JDK 7 的死循环：** 多线程扩容时，**头插法**导致链表形成环，`get` 时无限循环，CPU 打满。JDK 8 改用**尾插法**，避免了这个环，但并发下仍然会丢数据。
4. **读到不一致的数据：** 没有可见性保证。

**解决：**
- `ConcurrentHashMap`（首选）。
- `Collections.synchronizedMap`：整个 map 一把锁，性能差。
- `Hashtable`：方法都加 synchronized，已过时，不推荐。

**English:** "HashMap isn't thread safe. Concurrent puts can lose updates, and a concurrent resize can lose or corrupt data. In JDK 7, concurrent resizing could create a cycle in a bucket, causing an infinite loop. JDK 8 fixed the cycle with tail insertion, but it still loses data. I use ConcurrentHashMap instead."

## 8. HashMap vs 其他 Map

| | HashMap | LinkedHashMap | TreeMap | Hashtable | ConcurrentHashMap |
| --- | --- | --- | --- | --- | --- |
| 顺序 | 无序 | 插入顺序（或访问顺序） | 按 key 排序 | 无序 | 无序 |
| 线程安全 | 否 | 否 | 否 | 是（整体锁） | 是（细粒度） |
| null key / value | 允许 | 允许 | key 不允许 | 不允许 | 不允许 |
| 复杂度 | O(1) | O(1) | O(log n) | O(1) | O(1) |

- **LinkedHashMap 的用途：** 把 `accessOrder` 设为 true，并重写 `removeEldestEntry`，就是一个简单的 **LRU 缓存**。

```java
class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;
    LRUCache(int capacity) {
        super(16, 0.75f, true);          // true = 按访问顺序
        this.capacity = capacity;
    }
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;        // 超过容量，删除最久未访问的
    }
}
```

## 9. 常见易错点

- **遍历中修改会抛 `ConcurrentModificationException`**：用 `Iterator.remove()`，或 `removeIf`。（fail-fast，靠 `modCount` 检测。）
- **`new HashMap<>()` 默认不分配数组**，第一次 put 才创建。
- **容量会被调整成 2 的幂：** `new HashMap<>(10)` 实际容量是 16。
- **JDK 8 以后 `hashCode` 相同并不意味着一定冲突**，因为只取低位参与下标。

## 10. 面试高频问题清单（自测）

1. HashMap 的底层结构？JDK 7 和 8 有什么区别？
2. put 的流程？
3. 怎么计算下标？为什么容量是 2 的幂？
4. 为什么要把 hash 高位异或低位？
5. 什么时候链表转红黑树？为什么是 8？
6. 扩容的时机和过程？JDK 8 做了什么优化？
7. 为什么重写 equals 必须重写 hashCode？
8. HashMap 为什么线程不安全？JDK 7 的死循环是怎么来的？
9. HashMap、LinkedHashMap、TreeMap、ConcurrentHashMap 怎么选？
10. 怎么用 LinkedHashMap 实现 LRU？

## 11. 和你项目的连接

| 知识点 | 你的故事 |
| --- | --- |
| 本地缓存的数据结构 | FXCenter 客户端缓存：本地 map 存汇率，查询全部走本地，不访问 server |
| 线程安全的 map | 缓存被多线程读，预加载时重建，要用并发安全的结构或整体替换引用 |
| 预设容量 | 已知要缓存的汇率数量，初始化时指定容量，避免扩容 |
| key 的设计 | 汇率的 key（币种对、日期等）要不可变，并正确实现 equals 和 hashCode |
| LRU | 缓存淘汰策略的备选方案 |

[待补] 实际用的是哪种 map、key 是什么、预加载时怎么保证读写一致（比如构建完整的新 map 再整体替换引用），请按你的真实实现填写。
