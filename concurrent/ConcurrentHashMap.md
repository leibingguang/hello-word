# put 过程

```java
  /** Implementation for put and putIfAbsent */
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
          	// 初始化table
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
          	// 数组bucket为空，CAS替换
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
          	// 扩容中，协助扩容
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
              	// 通过synchronized获取数组相应下标节点的锁
                synchronized (f) {
                  	// 判断数组第一个节点的值是否已经发生变更，如果已经发生变更，则需要重新循环
                  	// 所有元素的插入都是基于首节点的，没有次判断会导致node丢失
                    if (tabAt(tab, i) == f) {
                      	// hash值大于0表示链表
                        if (fh >= 0) {
                          	// binCount用来计算数组具体小标槽中元素个数
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                              	// 存在key，值替换
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                              	// 追加到Node链表尾节点
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        else if (f instanceof TreeBin) {//树操作
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
              	// 判断数据索引位置节点个数是否超过8，超过的话需要树化
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                  	// 表示替换值，不需要重新统计元素个数
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        addCount(1L, binCount);
        return null;
    }
```



## 1. 数组初始化

```java
/**
 * 初始化table数组，默认大小为16， 设置扩容阀值为12
 */
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
      	// 如果表格正在初始化或者resize中时，释放cpu时钟，等待下一次循环
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
      	// 通过CAS标记sizeCtl为-1，表示正在初始化
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);
                }
            } finally {
              	// 下一次需要扩容的值，3/4最大容量的值
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```



## 2. 数据bucket为空

```java
/**
 * CAS替换节点
 */
if (casTabAt(tab, i, null,
             new Node<K,V>(hash, key, value, null)))
  break;
```

## 3. 数组bucket不为空

1. synchronized 锁住node节点
2. 替换原值或插入链表尾部
3. 判断是否需要树化

## 4. 协助扩容

```java
/**
 * 如果其他线程正在扩容，则协助扩容
 */
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
      	// 扩容戳：根据扩容前数据大小生成
        int rs = resizeStamp(tab.length);
      	// 判断线程是否扩容完毕
        while (nextTab == nextTable && table == tab &&
               (sc = sizeCtl) < 0) {
            // 扩容戳不一致：表示不在一个扩容周期内
          	// 最大协助线程数为：RESIZE_STAMP_SHIFT=2^16-1 sizeCtl的低16位决定
          	// transferIndex小于0：表示所有Node都已分配完毕
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
          	// 扩容线程数+1， 协助扩容
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

## 5. 重新计算CHM元素个数

```java
/**
 * 统计元素个数
 * 如果table太小并且没有resize，则触发resize，如果已经处于resize状态，则协助resize
 * 由于resize是滞后处理的，调整完成后需要重新检查是否需要再次resize
 * @param x： 表示增加的元素个数
 * @param check：表示是否需要检查扩容，小于0-不需要检查扩容 大于0-存在冲突的时候需要检查扩容
 */
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
  	// 已经出现竞态条件的情况下(countCells不为空或者CAS baseCount失败),
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell a; long v; int m;
        boolean uncontended = true; //是否冲突标识，默认为没有冲突
      	// 满足以下三个条件之一则调用fullAndCount函数初始化/扩容CountCell数组，并修改countCell值
      	// 1. CountCell数组为空或者CountCell数组元素个数等于0
      	// 2. 随机位置的数据元素为空
      	// 3. CAS数据元素值失败
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
              U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            fullAddCount(x, uncontended);//初始化/扩容CountCell数组，并设置元素值
            return;
        }
        if (check <= 1) // 链表长度小于等于1，不考虑扩容
            return;
        s = sumCount();// 统计ConcurrentHashMap元素个数：baseCount + sum(countCells[i])
    }
  	// binCount值大于0，检查是否需要扩容
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
      	// 扩容条件：sizeCtl(扩容阀值)小于元素个数，table不为空，tab没达到ConcurrentHashMap最大值(1<<30)
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {
            int rs = resizeStamp(n);// 根据数据大小生成扩容戳，低16位表示table大小
            if (sc < 0) {// 表示正在初始化或者正在扩容
              	// 判断是否需要协助扩容
              	// (sc >>> RESIZE_STAMP_SHIFT) != rs : 不在同一个扩容周期，无需协助
              	// sc == rs + 1: 扩容结束
              	// sc == rs + MAX_RESIZERS: 帮助线程已经达到最大值
              	// (nt = nextTable) == null: 扩容结束
              	// transferIndex <= 0: 表示所有的 transfer 任务都被领取完了，没有剩余的 hash 桶来给自己自己好这个线程来做 transfer
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
          	// table不处于扩容状态，则修改sizeCtl值，触发扩容
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
            s = sumCount();
        }
    }
}
```



# hash值计算

hash值枚举：

 * MOVED     = -1; // 表示需要进行协助扩容
 * TREEBIN   = -2; // 红黑树根节点
 * RESERVED  = -3; // hash for transient reservations
 * HASH_BITS = 0x7fffffff; // 正常的节点hash值结算

```java
/**
 * 1. key的hash值高低16位异或
 * 2. 异或结果 & 0x7fffffff
 */
static final int spread(int h) {
    return (h ^ (h >>> 16)) & HASH_BITS;
}
```



# CounterCell处理并发冲突

```java
// 总体逻辑：创建大小为2的countCells数组，通过随机值与数据大小与操作，判断索引位置是否存在值，不存在的话通过cas修改cellbusy状态，并设置索引下标位置的值，如果存在值，通过cas直接修改索引小标的值，如果修改失败的话，对counterCells进行扩容，循环直到成功
private final void fullAddCount(long x, boolean wasUncontended) {
    int h;
  	// 随机值没有初始化，强制初始化
    if ((h = ThreadLocalRandom.getProbe()) == 0) {
        ThreadLocalRandom.localInit();      // force initialization
        h = ThreadLocalRandom.getProbe();
        wasUncontended = true;// 初始化之后设值为无冲突
    }
    boolean collide = false;                // True if last slot nonempty
    for (;;) {
        CounterCell[] as; CounterCell a; int n; long v;
      	// 1. CountCell数组已经初始化过了
        if ((as = counterCells) != null && (n = as.length) > 0) {
            if ((a = as[(n - 1) & h]) == null) {// 具体下标的CounterCell没有初始化
                if (cellsBusy == 0) {            // Try to attach new Cell
                    CounterCell r = new CounterCell(x); // Optimistic create
                  	// CAS成功，设值CounterCells下标元素为r
                    if (cellsBusy == 0 &&
                        U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                        boolean created = false;
                        try {               // Recheck under lock
                            CounterCell[] rs; int m, j;
                            if ((rs = counterCells) != null &&
                                (m = rs.length) > 0 &&
                                rs[j = (m - 1) & h] == null) {
                                rs[j] = r;
                                created = true;
                            }
                        } finally {
                            cellsBusy = 0;
                        }
                        if (created)
                            break;
                        continue;           // Slot is now non-empty
                    }
                }
                collide = false;
            }
          	// 存在冲突，重新生成随机值后进入下一轮循环
            else if (!wasUncontended)       // CAS already known to fail
                wasUncontended = true;      // Continue after rehash
          	// CAS设值CellValue的value值，在不存在冲突的情况下置换成功，结束统计
            else if (U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))
                break;
          	// 正在扩容 || CounterCells数组大小大于CPU个数时，不需要扩容，collide标记为false，重新计算随机值，进入下一轮循环
            else if (counterCells != as || n >= NCPU)
                collide = false;            // At max size or stale
          	// 通过了上一步，表示存在冲突，需要扩容，重新计算随机值，进入下一轮循环，为了再给一次修改的机会？
            else if (!collide)
                collide = true;
          	// 扩容 CounterCell数组大小*2
            else if (cellsBusy == 0 &&
                     U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                try {
                    if (counterCells == as) {// Expand table unless stale
                        CounterCell[] rs = new CounterCell[n << 1];
                        for (int i = 0; i < n; ++i)
                            rs[i] = as[i];
                        counterCells = rs;
                    }
                } finally {
                    cellsBusy = 0;
                }
                collide = false;
                continue;                   // Retry with expanded table
            }
            h = ThreadLocalRandom.advanceProbe(h);
        }
      	// 初始化大小为2的counterCells数组， CELLSBUSY=0表示不存在竞争
        else if (cellsBusy == 0 && counterCells == as &&
                 U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
            boolean init = false;
            try {                           // Initialize table
                if (counterCells == as) {
                    CounterCell[] rs = new CounterCell[2];
                    rs[h & 1] = new CounterCell(x);
                    counterCells = rs;
                    init = true;
                }
            } finally {
                cellsBusy = 0;
            }
            if (init)
                break;
        }
      	// 在CountCell数组初始化期间，如果线程竞争激烈的话，CAS baseCount增加
        else if (U.compareAndSwapLong(this, BASECOUNT, v = baseCount, v + x))
            break;                          // Fall back on using base
    }
}
```



1. 触发CounterCell数组的条件是：通过CAS设置baseCount值失败，当CounterCell数组创建成功之后，就不会修改baseCount值了
2. 默认CounterCell数组大小为2，在CountCell数组创建成功之前，并发情况下还是通过baseCount记录元素个数
3. CounterCell值的修改主要有以下几种方式：
   1. CounterCell数组不存在的时候，初始化数组时创建CounterCell对象(初始化值)
   2. CounterCell数据创建后，如果对应下标不存在CounterCell对象，则创建（初始化值）
   3. 通过CAS修改CounterCell值
4. CounterCell数组扩容：
   1. 扩容条件：CounterCell数组大小小于CPU数量 && 存在CounterCell CAS设值失败
   2. CAS失败后会再给一次机会，会重新计算随机值，再次CAS，如果还是失败，则在满足扩容条件的情况下进行扩容
   3. 扩容：CounterCell数组扩大为原来的2倍，并迁移对象
5. cellsBusy在CounterCell中的应用，以下场景下会将cellsBusy设置为1
   1. 初始化CounterCell数据
   2. 给CounterCell数组设置CounterCell对象
   3. CounterCell数组扩容的时候

# 扩容

```java
/**
 * 转移bucket，扩容
 * @param tab 原有table数组
 * @param nextTab 扩容后的table数组
 */
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
  	// tab.length/8/NCPU > 16 ? tab.length/8/NCPU > 16 : 16
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // 最小步长为16
  	// 如果扩容table没有初始化，则需要先进行初始化，大小为2倍原有数据大小
    if (nextTab == null) {            // initiating
        try {
            @SuppressWarnings("unchecked")
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        transferIndex = n;// 用于定位转移bin，从大到小分配bin给不同的线程
    }
    int nextn = nextTab.length;
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);//扩容前数据，如果相应节点转移完成，则设置节点为fwd
    boolean advance = true;//标记是否需要继续寻找需要扩容转移的节点
    boolean finishing = false; // 表示当前线程的扩容任务已经完成
    for (int i = 0, bound = 0;;) {// i/bound表示当前线程转移的边界
        Node<K,V> f; int fh;
      	// while循环主要功能为多线程transafer情况下，计算每个线程处理的区间
        while (advance) {
            int nextIndex, nextBound;
          	// 当前线程已经分配过，无需再次分配，通过--i来循转移的步长内的所有链表
          	// finishing 扩容完成
            if (--i >= bound || finishing)
                advance = false;
          	// 分配完毕
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
          	// 为当前线程分配待转移区域
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                bound = nextBound;//下界
                i = nextIndex - 1;// 上界
                advance = false;
            }
        }
      	// 已经转移完成
      	// i<0：上界小于0，表示无可分配区间
      	// i>=n || i+n>=nextn ：当前线程执行结束后，如果没有其他线程在继续处理扩容，则会设置i=n，重新检查？
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
          	// 扩容完成
            if (finishing) {
                nextTable = null;
                table = nextTab; 
                sizeCtl = (n << 1) - (n >>> 1);//更新阀值
                return;
            }
          	// 当前线程扩容结束，协助扩容线程数-1
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
              	// 不相等：表示还有其他线程在扩容，结束当前线程扩容流程
              	// 相等：表示没有其他线程在扩容，需要设置table数据为扩容后数组
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
        else if ((f = tabAt(tab, i)) == null)// bin上没有元素，设置为fwd
            advance = casTabAt(tab, i, null, fwd);
        else if ((fh = f.hash) == MOVED)// bin已经被转移了，需要重新分配当前线程协助扩容区间
            advance = true; // already processed
        else {// 转移
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    if (fh >= 0) {
                      	// runBit表示链表首节点的hash & 扩容前数据大小
                        int runBit = fh & n;
                      	// 链表首节点
                        Node<K,V> lastRun = f;
                      	// 遍历链表
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                          	// b表示下一个节点的hash & 扩容前数组大小
                            int b = p.hash & n;
                          	// 不等于：表示两个节点扩容后不在同一条链上
                            if (b != runBit) {
                                runBit = b;// runBit 向后移
                                lastRun = p; // lastRun 节点向后移
                            }
                        }
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                      	// 生成两个链表
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                    else if (f instanceof TreeBin) {
                       // ...
                    }
                }
            }
        }
    }
}
```

扩容操作：

1. 计算扩容步长：stride = tab.length/8/NCPU > 16 ? tab.length/8/NCPU > 16 : 16
2. 初始化nextTable，两倍原有table大小
3. 计算当前线程负责转移区间，通过循环从高到低逐个转移链表，对已转移的bucket标记为ForwardingNode
4. 转移链表：
   1. 计算lastRun，lastRun之后的所有节点都处于高位/低位
   2. 循环当前链表，生成高位链表和低位链表(hash & n)
   3. 设置扩容过后数组的高低位链表

# sizeCtl 的功能

1. sizeCtl用于控制table的初始化和扩容操作，主要枚举值如下：
   1. sizeCtl = -1 : table初始化中
   2. sizeCtl = -N : 正在扩容，参与扩容的线程数为sizeCtl低16位的值-1， 高16位表示扩容戳
   3. sizeCtl > 0 ：表示下一次触发扩容的阀值

2. 修改sizeCtl值的地方主要有：
   1. initTable() ：初始化table
   2. addCount()：统计元素个数
   3. helpTransfer()：协助扩容，判断是否需要协助，需要的话调用transfer()进行扩容
   4. transfer()：扩容具体逻辑
