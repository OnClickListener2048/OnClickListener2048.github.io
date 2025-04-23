---
title: "【源码解析】HashMap 核心设计与源码速览 (JDK 8+)"
date: 2023-10-27T10:00:00+08:00
lastmod: 2023-10-27T10:00:00+08:00
author: "OnClickListener" # Change to your name
# authorLink: "Your Link" # Optional: Link to your profile
weight: 1
description: "快速理解 Java HashMap 的核心工作原理，包括数据结构、put/get 流程、哈希冲突处理和扩容机制，辅以关键源码片段。"
# license: "" # Optional: Specify a license
resources:
  - name: "featured-image"
    src: "featured-image.jpg"
tags: ["Java", "HashMap", "源码解析", "数据结构", "集合框架", "后端"]
categories: ["技术", "源码分析"]
toc:
  enable: true # 是否启用目录
# series: ["Java 集合框架源码"] # Optional: Add to a series
# weight: # Optional: for ordering within a series/section
draft: false # Set to false to publish
---

## 前言

`HashMap` 几乎是 Java 面试和开发中的“必考题”和“必备品”。它提供了高效的键值对存储和查找能力（平均时间复杂度 O(1)）。理解其内部实现原理，不仅有助于我们写出更优的代码，也能在面试中脱颖而出。本文旨在快速梳理 `HashMap` (以 JDK 8+ 为基础) 的核心设计和源码要点，助你快速入门。

{{< admonition type=tip title="阅读目标" open=true >}}
通过本文，你将快速理解：
*   `HashMap` 的基本数据结构（数组 + 链表/红黑树）。
*   `put` 和 `get` 操作的核心流程。
*   哈希冲突是如何发生的，以及如何解决（链地址法、树化）。
*   动态扩容（resize）的触发时机和过程。
{{< /admonition >}}

## 核心数据结构：数组 + 链表 / 红黑树

`HashMap` 内部维护了一个 **`Node<K,V>[] table`** 数组，这是其主体结构，也常被称为“桶”（bucket）数组。每个桶（数组元素）可以存放一个 `Node` 节点，或者是一条 `Node` 组成的链表，或者是一棵红黑树（`TreeNode`）。

{{< figure src="featured-image.jpg" title="HashMap 内部结构示意图 (JDK 8+)" width="80%" >}}
<!-- 你需要将上面这行替换为你自己的图片路径，或者删除它 -->
<!-- 图片示意：一个数组，部分索引为null，部分索引指向单个Node，部分索引指向链表，部分索引指向红黑树 -->

{{< admonition type=note title="关键成员变量 (JDK 8+)" >}}
*   `transient Node<K,V>[] table`: 存储数据的桶数组，长度总是 2 的幂次方。
*   `transient Set<Map.Entry<K,V>> entrySet`: 缓存的 entry 集合。
*   `transient int size`: `HashMap` 中存储的键值对数量。
*   `transient int modCount`: 修改次数，用于迭代时的快速失败机制。
*   `int threshold`: 扩容阈值，当 `size` 超过这个值时触发扩容。`threshold = capacity * loadFactor`。
*   `final float loadFactor`: 负载因子，默认为 0.75f。控制数组的填充程度。
*   `static final int TREEIFY_THRESHOLD = 8`: 链表转红黑树的阈值。当一个桶中的链表长度达到 8 时，**并且** `table` 的容量 `capacity` 大于等于 `MIN_TREEIFY_CAPACITY` (64) 时，链表会转化为红黑树。
*   `static final int UNTREEIFY_THRESHOLD = 6`: 红黑树转链表的阈值。当扩容时，如果一个桶中的节点数减少到 6，红黑树会退化回链表。
*   `static final int MIN_TREEIFY_CAPACITY = 64`: 允许链表树化的最小 `table` 容量。如果容量小于此值，即使链表长度达到 8，也只会进行扩容，而不会树化。
{{< /admonition >}}

## `put(K key, V value)` 核心流程

`put` 方法是 `HashMap` 最核心的操作之一。

```java
// JDK 8 简化版 putVal 逻辑示意
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;

    // 1. 初始化 table (如果首次 put)
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length; // resize() 会初始化或扩容

    // 2. 计算索引 i，并检查桶位 p 是否为空
    i = (n - 1) & hash; // 用位运算替代取模，要求 n 是 2 的幂次方
    if ((p = tab[i]) == null) {
        // 桶位为空，直接创建新节点放入
        tab[i] = newNode(hash, key, value, null);
    } else {
        // 桶位不为空（发生哈希冲突）
        Node<K,V> e; K k;
        // 3. 检查桶的第一个节点 p 是否就是我们要找的 key
        if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k)))) {
            e = p; // key 相同
        }
        // 4. 如果第一个节点不是，检查 p 是否是红黑树节点
        else if (p instanceof TreeNode) {
            // 调用红黑树的 putTreeVal 方法查找或插入
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        }
        // 5. 如果不是红黑树，说明是链表
        else {
            // 遍历链表
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    // 到达链表尾部，仍未找到 key，插入新节点
                    p.next = newNode(hash, key, value, null);
                    // 检查链表长度是否达到树化阈值
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 是因为从 0 开始计数
                        treeifyBin(tab, hash); // 尝试树化 (内部会检查 MIN_TREEIFY_CAPACITY)
                    break; // 插入完成，跳出循环
                }
                // 在链表中找到了相同的 key
                if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k)))) {
                    break; // 找到 key，跳出循环，后续会更新 value
                }
                p = e; // 继续下一个节点
            }
        }

        // 6. 如果 e != null，说明 key 已存在
        if (e != null) {
            V oldValue = e.value;
            // 根据 onlyIfAbsent 判断是否替换旧值
            if (!onlyIfAbsent || oldValue == null) {
                e.value = value;
            }
            afterNodeAccess(e); // 空方法，留给 LinkedHashMap 实现
            return oldValue; // 返回旧值
        }
    }

    // 7. 修改计数器 modCount 增加
    ++modCount;

    // 8. 检查 size 是否超过扩容阈值 threshold
    if (++size > threshold)
        resize(); // 进行扩容

    afterNodeInsertion(evict); // 空方法，留给 LinkedHashMap 实现
    return null; // key 不存在，返回 null
}

// hash() 方法：计算 key 的哈希值，并进行扰动处理
static final int hash(Object key) {
    int h;
    // 使用 key 的 hashCode() 并结合高位进行异或运算，减少哈希冲突
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

{{< admonition type=info title="put 流程总结" >}}
1.  **计算 Hash:** 使用 `key` 的 `hashCode()` 并通过 `hash()` 方法（扰动函数）计算最终的哈希值。`null` key 的哈希值为 0。
2.  **计算索引:** 使用 `(table.length - 1) & hash` 计算在 `table` 数组中的索引 `i`。
3.  **处理桶位 `tab[i]`:**
    *   **为空:** 直接创建新 `Node` 放入。
    *   **不为空 (冲突):**
        *   检查第一个节点 `p` 是否匹配 `key`。
        *   若 `p` 是 `TreeNode`，则在红黑树中查找或插入。
        *   若 `p` 是普通 `Node`（链表头），遍历链表：
            *   若找到匹配 `key` 的节点 `e`，跳出循环。
            *   若到达链表末尾仍未找到，创建新 `Node` 并追加到链表尾部。追加后检查链表长度 (`binCount`) 是否达到 `TREEIFY_THRESHOLD - 1`，若是则调用 `treeifyBin` 尝试树化（内部会判断容量是否 >= 64）。
4.  **处理结果:**
    *   若找到已存在的 `key` (节点 `e` 不为 `null`)，则更新其 `value` (如果 `onlyIfAbsent` 为 `false`)，并返回旧 `value`。
    *   若未找到 `key` (插入了新节点)，`size` 加 1。
5.  **检查扩容:** 判断 `size` 是否大于 `threshold`，若是则调用 `resize()`。
6.  **返回:** 如果是插入新节点，返回 `null`。
{{< /admonition >}}

{{< admonition type=warning title="hashCode() 与 equals() 的重要性" >}}
`HashMap` 的正确工作严重依赖于 `key` 对象的 `hashCode()` 和 `equals()` 方法：
*   `hashCode()` 用于快速定位桶的位置。
*   `equals()` 用于在发生哈希冲突时，精确地判断两个 `key` 是否相同。
*   **约定:** 如果 `a.equals(b)` 为 `true`，那么 `a.hashCode()` 必须等于 `b.hashCode()`。反之不一定成立（不同对象可以有相同的哈希码）。
*   如果自定义对象作为 `key`，务必同时正确重写这两个方法。
{{< /admonition >}}

## `get(Object key)` 核心流程

`get` 操作相对简单，主要是在定位到桶之后进行查找。

```java
// JDK 8 getNode 简化逻辑
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;

    // 1. 检查 table 是否初始化，并计算索引
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) { // 定位到桶的首节点 first

        // 2. 检查桶的第一个节点 first 是否匹配
        if (first.hash == hash && // 先比较 hash，效率更高
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first; // 第一个节点就命中，直接返回

        // 3. 如果第一个节点没命中，且后面还有节点 (first.next != null)
        if ((e = first.next) != null) {
            // 4. 检查是否是红黑树
            if (first instanceof TreeNode)
                // 在红黑树中查找
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            // 5. 是链表，遍历查找
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e; // 找到匹配节点，返回
            } while ((e = e.next) != null); // 继续下一个
        }
    }
    // 没找到或 table 未初始化，返回 null
    return null;
}
```

{{< admonition type=info title="get 流程总结" >}}
1.  **计算 Hash 和索引:** 与 `put` 类似。
2.  **定位桶:** 找到 `table` 中对应索引 `i` 的第一个节点 `first`。
3.  **查找:**
    *   如果 `first` 为 `null`，表示 `key` 不存在，返回 `null`。
    *   如果 `first` 的 `key` 匹配，直接返回 `first`。
    *   如果 `first` 不匹配且 `first` 是 `TreeNode`，则在红黑树中查找。
    *   如果 `first` 不匹配且是普通 `Node`（链表），则遍历链表，使用 `equals()` 比较 `key`，找到则返回，遍历完未找到则返回 `null`。
{{< /admonition >}}

## 扩容机制 `resize()`

当 `HashMap` 中的元素数量 `size` 超过了扩容阈值 `threshold` (`capacity * loadFactor`) 时，就会触发 `resize()` 操作，以减少哈希冲突，保持 O(1) 的平均查找效率。

```java
// JDK 8 resize 简化逻辑示意
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;

    // 1. 计算新容量 newCap 和新阈值 newThr
    if (oldCap > 0) {
        // 如果旧容量已达最大值，不再扩容
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 新容量翻倍，不超过最大值
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // 新阈值也翻倍
    }
    // ... (处理旧容量为 0 或旧阈值 > 0 的初始化情况) ...
    else {
        // 使用默认初始容量和负载因子计算
        newCap = DEFAULT_INITIAL_CAPACITY; // 16
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY); // 0.75 * 16 = 12
    }

    // ... (处理 newThr 未计算的情况) ...
    threshold = newThr; // 更新阈值

    // 2. 创建新的桶数组 newTab
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab; // HashMap 指向新数组

    // 3. 将旧数组 oldTab 中的元素迁移到新数组 newTab 中
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) { // 遍历旧数组的每个桶
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null; // 释放旧桶引用
                if (e.next == null) {
                    // 桶中只有一个元素，直接计算新索引放入新数组
                    newTab[e.hash & (newCap - 1)] = e;
                }
                else if (e instanceof TreeNode) {
                    // 如果是红黑树，进行树的拆分 (split)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                }
                else { // 是链表
                    // **JDK 8 优化点：**
                    // 将链表拆分成两条：一条索引不变，另一条索引变为 "原索引 + oldCap"
                    Node<K,V> loHead = null, loTail = null; // 低位链表 (索引不变)
                    Node<K,V> hiHead = null, hiTail = null; // 高位链表 (索引 + oldCap)
                    Node<K,V> next;
                    do {
                        next = e.next;
                        // 根据 (e.hash & oldCap) 的值判断节点去向
                        // 等于 0，索引不变，放入低位链表
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null) loHead = e; else loTail.next = e;
                            loTail = e;
                        }
                        // 不等于 0，索引变为 j + oldCap，放入高位链表
                        else {
                            if (hiTail == null) hiHead = e; else hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);

                    // 将两条链表分别放入新数组的对应位置
                    if (loTail != null) {
                        loTail.next = null; // 断开链表尾部
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null; // 断开链表尾部
                        newTab[j + oldCap] = hiHead; // 新索引位置
                    }
                }
            }
        }
    }
    return newTab;
}
```

{{< admonition type=success title="resize 关键点" >}}
*   **触发条件:** `size > threshold`。
*   **容量变化:** 通常是旧容量的 2 倍 (`oldCap << 1`)。容量始终是 2 的幂次方。
*   **元素迁移:** 需要遍历旧 `table` 中的所有元素，重新计算它们在新 `table` 中的索引，并放入新 `table`。这是一个相对耗时的操作。
*   **JDK 8 优化 (rehash):** 对于链表节点，利用容量是 2 的幂次方的特性。由于新容量 `newCap` 是旧容量 `oldCap` 的 2 倍，元素在新表中的索引要么与旧索引 `j` 相同，要么是 `j + oldCap`。这可以通过判断 `e.hash & oldCap` 是否为 0 来决定。等于 0 则索引不变，否则索引变为 `j + oldCap`。这样，可以将原链表高效地拆分成两条子链表，分别放到新表的对应位置，避免了对每个元素都重新计算完整的 `hash & (newCap - 1)`。
*   **树的处理:** 红黑树节点会调用 `split` 方法进行拆分，逻辑类似，也是根据 `hash & oldCap` 分成两棵（或一棵 + 一条链表）放入新表。如果拆分后节点数少于 `UNTREEIFY_THRESHOLD` (6)，可能会退化回链表。
{{< /admonition >}}

## 总结

`HashMap` 通过精妙的哈希函数、高效的索引计算（位运算）、巧妙的冲突处理（链地址法 + 红黑树）以及动态扩容机制，实现了平均 O(1) 时间复杂度的 `put` 和 `get` 操作。

{{< admonition type=milestone title="快速回顾" >}}
*   **结构:** 数组 + 链表/红黑树。
*   **核心:** `hash()` 计算哈希值（含扰动），`(n-1) & hash` 定位桶。
*   **冲突:** 拉链法，链表过长（8）且容量足够（64）时转红黑树。
*   **扩容:** `size` 超 `threshold` 时触发，容量翻倍，元素 rehash 到新表（JDK 8 有优化）。
*   **关键:** 正确实现 `key` 的 `hashCode()` 和 `equals()`。
{{< /admonition >}}

理解 `HashMap` 的源码是掌握 Java 集合框架的重要一步。虽然本文只做了快速梳理，但抓住了核心脉络。建议读者结合 JDK 源码进行更深入的阅读和调试，体会其设计的精妙之处。


