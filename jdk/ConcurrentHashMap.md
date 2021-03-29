# ConcurrentHashMap 说明

## 属性说明

```java
//最大容量
private static final int MAXIMUM_CAPACITY = 1 << 30;

//默认初始容量
private static final int DEFAULT_CAPACITY = 16;

//数组的最大容量,防止抛出OOM
static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

//最大并行度，仅用于兼容JDK1.7以前版本
private static final int DEFAULT_CONCURRENCY_LEVEL = 16;

//扩容因子
private static final float LOAD_FACTOR = 0.75f;

//链表转红黑树的阈值
static final int TREEIFY_THRESHOLD = 8;

//红黑树退化阈值
static final int UNTREEIFY_THRESHOLD = 6;

//链表转红黑树的最小总量
static final int MIN_TREEIFY_CAPACITY = 64;

//扩容搬运时批量搬运的最小槽位数
private static final int MIN_TRANSFER_STRIDE = 16;


//当前待扩容table的邮戳位,通常是高16位
private static final int RESIZE_STAMP_BITS = 16;

//同时搬运的线程数自增的最大值
private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;

//搬运线程数的标识位，通常是低16位
private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;

static final int MOVED     = -1; // 说明是forwardingNode
static final int TREEBIN   = -2; // 红黑树
static final int RESERVED  = -3; // 原子计算的占位Node
static final int HASH_BITS = 0x7fffffff; // 保证hashcode扰动计算结果为正数

//当前哈希表
transient volatile Node<K,V>[] table;

//下一个哈希表
private transient volatile Node<K,V>[] nextTable;

//计数的基准值
private transient volatile long baseCount;

/**
 * 控制变量，不同场景有不同用途，参考下文
 * 初始化设置了容器的初始化大小时该值还会用于存储初始化的容器大小数量载体使用
 *
 * 0：初始化值，数组为初始化
 * -1：当前正在进行初始化容器
 * 正数：下次扩容是的阈值
 */
private transient volatile int sizeCtl;

//并发搬运过程中CAS获取区段的下限值
private transient volatile int transferIndex
//计数cell初始化或者扩容时基于此字段使用自旋锁
private transient volatile int cellsBusy;

//加速多核CPU计数的cell数组
private transient volatile CounterCell[] counterCells;
```

### 初始化容器

```java
private final Node<K, V>[] initTable() {
    Node<K, V>[] tab;
    int sc;
    // 循环判断是否已经初始化完成，用于多线程并发初始化操作
    // 这样就不会出现有线程提前走到后续的业务流程
    while ((tab = table) == null || tab.length == 0) {
        // 如果已经有其它线程在执行了初始化操作，则提示本线程可以让出cpu使用。具体实现看系统底层
        if ((sc = sizeCtl) < 0) {
            Thread.yield(); // lost initialization race; just spin
        } else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) { // 通过CAS设置状态位的值，如果设置成功则可执行
            // 执行初始化操作
            try {
                // 二次验证是否需要进行初始化操作，多线程一起执行初始化操作。后续的线程总会走进该分支而结束
                if ((tab = table) == null || tab.length == 0) {
                    // 构造函数是定义了该值，则使用自定义值否则使用默认大小
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K, V>[] nt = (Node<K, V>[]) new Node<?, ?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);
                }
            } finally {
                // 还原sizeCtl值
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

## 写操作操作

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
  // 不支持空key和value
  if (key == null || value == null) throw new NullPointerException();
  // 计算hash
  int hash = spread(key.hashCode());
  // 当前put数据进行操作次数
  int binCount = 0;
  // 将堆中对象的引用赋值给线程栈中的局部变量
  for (Node<K, V>[] tab = table; ; ) {
    Node<K, V> f;
    int n, i, fh;
    // 数组初始化
    if (tab == null || (n = tab.length) == 0) {
      tab = initTable();
    } else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) { // 获取数组的下标节点对象，并且当前空
      // 将当前的k/v创建新的bin对象, 过cas方式进行赋值数组操作,成功跳出循环
      if (casTabAt(tab, i, null, new Node<K, V>(hash, key, value, null))) {
        break;
      }
    } else if ((fh = f.hash) == MOVED) { // 如果节点bin的首个对象hash等于moved
      // 帮忙执行数据转换
      tab = helpTransfer(tab, f);
    } else {
      V oldVal = null;
      // 对应hash槽下的bin不为空，并且为触发迁移操作
      // 锁定当前的bin的首node对象，如果当时有其它线程对该node对象进行锁定则需要等待其它操作完成
      synchronized (f) {
        // 二次确认，当前的hash的bin对象是否等于获取的bin对象。// todo回来在看为什么要二次获取
        if (tabAt(tab, i) == f) {
          // bin对象的hash大于等于0，表明是链表模式
          if (fh >= 0) {
            binCount = 1;
            for (Node<K, V> e = f; ; ++binCount) {
              K ek;
              // hash值一致并且key一致进行响应的值操作并且跳出链表循环
              if (e.hash == hash && ((ek = e.key) == key || (ek != null && key.equals(ek)))) {
                oldVal = e.val;
                // 允许覆盖旧的值，则覆盖历史value
                if (!onlyIfAbsent) {
                  e.val = value;
                }
                break;
              }
              Node<K, V> pred = e;
              // 递归获取链表的下一个节点对象，如果空创建node对象加入链表的尾部。跳出链表循环
              if ((e = e.next) == null) {
                pred.next = new Node<K, V>(hash, key, value, null);
                break;
              }
            }
          } else if (f instanceof TreeBin) { // bin 对象已经是红黑树模式
            Node<K, V> p;
            binCount = 2;
            // 添加到红黑树中，并且之前存在该node
            if ((p = ((TreeBin<K, V>) f).putTreeVal(hash, key, value)) != null) {
              oldVal = p.val;
              if (!onlyIfAbsent) {
                p.val = value;
              }
            }
          }
        }
      }
      // 数据添加完成操作
      if (binCount != 0) {
        // 判断bin数据链表数据量是否大于转换成红黑树的阈值
        if (binCount >= TREEIFY_THRESHOLD) {
          // bin转换成红黑树
          treeifyBin(tab, i);
        }
        if (oldVal != null) return oldVal;
        break;
      }
    }
  }
  // ?
  addCount(1L, binCount);
  return null;
}
```

#### 赋值操作整体是一个多步骤的循环操作

（1）初始化 hash 数组，如果当前的 hash 数组不存在则先执行初始化操作  
（2）计算出对应的 key 的槽点下标，并且通过 CAS 操作方式获取数组的 bin 对象。如果 bin 对象不存在，则通过 CAS 进行赋值操作,成功则直接退出循环操作  
（3）bin 节点正在执行数据迁移？  
（4）锁定 bin 的第一个数据值，判断 bin 是否为链表模式。如果是链表模式进行递归查询,找到则进行替换否则创建一个新的 node 对象加入到链表的尾部；如果不是链表模式则将加入到红黑树结构中。  
（5）判断操作的数据次数（只有链接模式才有意义，红黑树估计值为 2）是否满足红黑树转换条件，满足则将对应的 hash 槽转换为红黑树

## 转换红黑树

```java
private final void treeifyBin(Node<K, V>[] tab, int index) {
    Node<K, V> b;
    int n, sc;
    // 数组未初始化则不执行
    if (tab != null) {
        // 如果数组的容量小于红黑树转换最小的设置容器阈值则进行扩容操作
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY) {
            tryPresize(n << 1);
        } else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
            // 通过CAS获取数组对应下标的node对象，并且对应的hash值是正数（非红黑树）
            // 锁住node对象，则隐式的表明对应的节点不能同时进行增删改操作
            synchronized (b) {
                // 多线程同时触发转换操作，这里需要在次判断对应的数组下标处对象是否还是链表对象
                // 红黑数转换完成后，新的node对象是treeNode对象，则不会再次进行转换操作
                if (tabAt(tab, index) == b) {
                    TreeNode<K, V> hd = null, tl = null;
                    // 链表转换成红黑树node结构
                    for (Node<K, V> e = b; e != null; e = e.next) {
                        TreeNode<K, V> p = new TreeNode<K, V>(e.hash, e.key, e.val, null, null);
                        // 如果tl空，则当前p节点设置为第一个节点
                        if ((p.prev = tl) == null) {
                            hd = p;
                        } else {
                            // 双向节点设置赋值
                            tl.next = p;
                        }
                        // 将新创建的node节点设置为最后节点
                        tl = p;
                    }
                    // 创建红黑树bin对象，通过CAS方式将红黑树node进行赋值
                    // 在赋值过程中，查询线程可能会查询到之前的链表结构也可以会查询到替换后的红黑树结构。但是这不影响
                    // 数据的查询操作，因为这是通过原子性的赋值操作
                    setTabAt(tab, index, new TreeBin<K, V>(hd));
                }
            }
        }
    }
}
```

## 协助数据迁移?后续分析

```java
final Node<K, V>[] helpTransfer(Node<K, V>[] tab, Node<K, V> f) {
    Node<K, V>[] nextTab;
    int sc;
    /*
     * 1.旧容器不能为空
     * 2.容器对应的hash位置bin当前的
     * 3.bin的下个bin不为空？等待后续
     */
    if (tab != null
            && (f instanceof ForwardingNode)
            && (nextTab = ((ForwardingNode<K, V>) f).nextTable) != null) {
        // 计算数组还有多少位可以进行扩容操作（二进制位数）
        int rs = resizeStamp(tab.length);
        /*
         * 1.堆中的nextTable对象等于bin的下个nextTab
         * 2.参数数组和堆中的数组是同个对象
         * 3.当前正在进行扩容或者初始化操作
         */
        while (nextTab == nextTable && table == tab && (sc = sizeCtl) < 0) {
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs
                    || sc == rs + 1
                    || sc == rs + MAX_RESIZERS
                    || transferIndex <= 0) break;
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                transfer(tab, nextTab);
                break;
            }
        }
        return nextTab;
    }
    return table;
}
```

## 数组数量添加

```java
// 从 putVal 传入的参数是 1， binCount，binCount 默认是0，只有 hash 冲突了才会大于 1.且他的大小是链表的长度（如果不是红黑数结构的话）。
private final void addCount(long x, int check) {
    CounterCell[] as;
    long b, s;
    // counterCells不为空（有并发，如果没有并发则进行CAS计数添加）或计数数量CAS更新失败
    if ((as = counterCells) != null
            || !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell a;
        long v;
        int m;
        boolean uncontended = true;
        /*
         * 1.counterCells为空（不存在并发，其它线程已操作完成）
         * 2.当前线程探针槽对象null
         * 3.当前线程探针槽对象计数值CAS更新失败
         * 没有并发则进行对线程探针槽进行CAS赋值，如果失败则调用fullAddCount进行循环插入操作
         */
        if (as == null
                || (m = as.length - 1) < 0
                || (a = as[ThreadLocalRandom.getProbe() & m]) == null
                || !(uncontended = U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            // 完整的计数添加
            fullAddCount(x, uncontended);
            return;
        }
        // 如果无需验证，直接推出操作
        if (check <= 1) {
            return;
        }
        // 获取当前的容器数据量总数
        s = sumCount();
    }
    if (check >= 0) {
        Node<K, V>[] tab, nt;
        int n, sc;
        /*
         * 1.总量大于等于扩容阈值
         * 2.数组不为空
         * 3.数组长度小于最大容量
         */
        while (s >= (long) (sc = sizeCtl)
                && (tab = table) != null
                && (n = tab.length) < MAXIMUM_CAPACITY) {
            // 根据 length 得到一个标识
            int rs = resizeStamp(n);
            // 小于0，表示在扩容操作。（-1 初始化不会在此处出现）
            if (sc < 0) {
                /*
                 * 如果 sc 的低 16 位不等于 标识符（校验异常 sizeCtl 变化了）
                 * 如果 sc == 标识符 + 1 （扩容结束了，不再有线程进行扩容）（默认第一个线程设置 sc ==rs 左移 16 位 + 2，
                 * 当第一个线程结束扩容了，就会将 sc 减一。这个时候，sc 就等于 rs + 1）
                 * 如果 sc == 标识符 + 65535（帮助线程数已经达到最大）
                 * 如果 nextTable == null（结束扩容了）
                 * 如果 transferIndex <= 0 (转移状态变化了)
                 * 结束循环
                 **/
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs
                        || sc == rs + 1
                        || sc == rs + MAX_RESIZERS
                        || (nt = nextTable) == null
                        || transferIndex <= 0) {
                    break;
                }
                // 如果可以帮助扩容，那么将 sc 加 1. 表示多了一个线程在帮助扩容
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                    // 扩容
                    transfer(tab, nt);
                }
            }
            // 如果不在扩容，将 sc 更新：标识符左移 16 位 然后 + 2. 也就是变成一个负数。高 16 位是标识符，低 16 位初始是 2.
            else if (U.compareAndSwapInt(this, SIZECTL, sc, (rs << RESIZE_STAMP_SHIFT) + 2)) {
                // 更新 sizeCtl 为负数后，开始扩容。
                transfer(tab, null);
            }
            s = sumCount();
        }
    }
}
```

x 参数表示的此次需要对表中元素的个数加几。check 参数表示是否需要进行扩容检查，大于等于 0 需要进行检查，而我们的 putVal 方法的 binCount 参数最小也是 0 ，因此，每次添加元素都会进行检查。（除非是覆盖操作）

1. 判断计数盒子属性是否是空，如果是空，就尝试修改 baseCount 变量，对该变量进行加 X。
2. 如果计数盒子不是空，或者修改 baseCount 变量失败了，则放弃对 baseCount 进行操作。
3. 如果计数盒子是 null 或者计数盒子的 length 是 0，或者随机取一个位置取于数组长度是 null，那么就对刚刚的元素进行 CAS 赋值。
4. 如果赋值失败，或者满足上面的条件，则调用 fullAddCount 方法重新死循环插入。
5. 这里如果操作 baseCount 失败了（或者计数盒子不是 Null），且对计数盒子赋值成功，那么就检查 check 变量，如果该变量小于等于 1. 直接结束。否则，计算一下 count 变量。
6. 如果 check 大于等于 0 ，说明需要对是否扩容进行检查。
7. 如果 map 的 size 大于 sizeCtl（扩容阈值），且 table 的长度小于 1 << 30，那么就进行扩容。
8. 根据 length 得到一个标识符，然后，判断 sizeCtl 状态，如果小于 0 ，说明要么在初始化，要么在扩容。
9. 如果正在扩容，那么就校验一下数据是否变化了（具体可以看上面代码的注释）。如果检验数据不通过，break。
10. 如果校验数据通过了，那么将 sizeCtl 加一，表示多了一个线程帮助扩容。然后进行扩容。
11. 如果没有在扩容，但是需要扩容。那么就将 sizeCtl 更新，赋值为标识符左移 16 位 —— 一个负数。然后加 2。 表示，已经有一个线程开始扩容了。然后进行扩容。然后再次更新 count，看看是否还需要扩容。

## 总结一下

总结下来看，addCount 方法做了 2 件事情：

1. 对 table 的长度加一。无论是通过修改 baseCount，还是通过使用 CounterCell。当 CounterCell 被初始化了，就优先使用他，不再使用 baseCount。
2. 检查是否需要扩容，或者是否正在扩容。如果需要扩容，就调用扩容方法，如果正在扩容，就帮助其扩容。

有几个要点注意：

1. 第一次调用扩容方法前，sizeCtl 的低 16 位是加 2 的，不是加一。所以 sc == rs + 1 的判断是表示是否完成任务了。因为完成扩容后，sizeCtl == rs + 1。
2. 扩容线程最大数量是 65535，是由于低 16 位的位数限制。
3. 这里也是可以帮助扩容的，类似 helpTransfer 方法。

## 查询操作

```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    // 扰行扰动函数，计算hash
    int h = spread(key.hashCode());
    // 哈希表不为空&并且对应的hash槽不为空&bin头节点不为空
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        // bin的哈希等于key的hash，则是链表查询。判断第一个key是否为查询的值
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        } else if (eh < 0) {
            // bin的哈希是负数，则表明已转为红黑数
            return (p = e.find(h, key)) != null ? p.val : null;
        }

        // 遍历列表查询
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    // 未查询到结果返回
    return null;
}
```

#### 1、如果当前哈希表 table 为 null

哈希表未初始化或者正在初始化未完成，直接返回 null；当时可能其它线程有在初始化，至少在判断 tab==null 的时间点 key 肯定是不存在的，返回 null 符合某一时刻的客观事实。

#### 2、如果读取的 bin 头节点为 null

说明该槽位尚未有节点，直接返回 null。

#### 3、如果读取的 bin 是一个链表

说明头节点是个普通 Node。

（1）如果正在发生链表向红黑树的 treeify 工作，因为 treeify 本身并不破坏旧的链表 bin 的结构，只是在全部 treeify 完成后将头节点一次性替换为新创建的 TreeBin，可以放心读取。

（2）如果正在发生 resize 且当前 bin 正在被 transfer，因为 transfer 本身并不破坏旧的链表 bin 的结构，只是在全部 transfer 完成后将头节点一次性替换为 ForwardingNode，可以放心读取。

（3）如果其它线程正在操作链表，在当前线程遍历链表的任意一个时间点，都有可能同时在发生 add/replace/remove 操作。

- 如果是 add 操作，因为链表的节点新增从 JDK8 以后都采用了后入式，无非是多遍历或者少遍历一个 tailNode。
- 如果是 remove 操作，存在遍历到某个 Node 时，正好有其它线程将其 remove，导致其孤立于整个链表之外；但因为其 next 引用未发生变更，整个链表并没有断开，还是可以照常遍历链表直到 tailNode。
- 如果是 replace 操作，链表的结构未变，只是某个 Node 的 value 发生了变化，没有安全问题。
  <!-- <table><tr><td bgcolor=silver> -->
  <!-- </td></tr></table> -->

结论：对于链表这种线性数据结构，单线程写且插入操作保证是后入式的前提下，并发读取是安全的；不会存在误读、链表断开导致的漏读、读到环状链表等问题。
