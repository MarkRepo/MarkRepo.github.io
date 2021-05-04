---
title: LFU 算法
description: 用框架思维看待算法套路
categories: algorithm
tags:
---

## LFU 算法设计

LFU 算法相当于是淘汰访问频次最低的数据，如果访问频次最低的数据有多条，需要淘汰最旧的数据。

要求你写一个类，接受一个`capacity`参数，实现`get`和`put`方法, 时间复杂度O(1):

```c++
class LFUCache {
    // 构造容量为 capacity 的缓存
    public LFUCache(int capacity) {}
    // 在缓存中查询 key
    public int get(int key) {}
    // 将 key 和 val 存入缓存
    public void put(int key, int val) {}
}
```

分析，LFU算法需求如下：

1、调用`get(key)`方法时，要返回该`key`对应的`val`。

2、只要用`get`或者`put`方法访问一次某个`key`，该`key`的`freq（频率）`就要加一。

3、如果在容量满了的时候进行插入，则需要将`freq`最小的`key`删除，如果最小的`freq`对应多个`key`，则删除其中最旧的那一个。

好的，我们希望能够在 O(1) 的时间内解决这些需求，可以使用基本数据结构来逐个击破：

**1、**使用一个`HashMap`存储`key`到`val`的映射，就可以快速计算`get(key)`。

```
HashMap<Integer, Integer> keyToVal;
```

**2、**使用一个`HashMap`存储`key`到`freq`的映射，就可以快速操作`key`对应的`freq`。

```
HashMap<Integer, Integer> keyToFreq;
```

**3、**这个需求是 LFU 算法的核心，所以我们分开说。

​	**3.1**、首先，肯定是需要`freq`到`key`的映射，用来找到最小`freq`对应的`key`。

​	**3.2、**将`freq`最小的`key`删除，那你就得快速得到当前所有`key`最小的`freq`是多少。想要时间复杂度 O(1) 的话，

​			肯定不能遍历一遍去找，那就用一个变量`minFreq`来记录当前最小的`freq`吧。

​	**3.3、**可能有多个`key`拥有相同的`freq`，所以 **`freq`对`key`是一对多的关系**，即一个`freq`对应一个`key`的列表。

​	**3.4、**希望`freq`对应的`key`的列表是**存在时序**的，便于快速查找并删除最旧的`key`。

​	**3.5、**希望**能够快速删除`key`列表中的任何一个`key`**，因为如果频次为`freq`的某个`key`被访问，

​			 那么它的频次就会变成`freq+1`，就应该从`freq`对应的`key`列表中删除，加到`freq+1`对应的`key`的列表中。

`LinkedHashSet`顾名思义，是链表和哈希集合的结合体。链表不能快速访问链表节点，但是插入元素具有时序；哈希集合中的元素无序，但是可以对元素进行快速的访问和删除。

```cpp
class LFUCache {
    // key 到 val 的映射，我们后文称为 KV 表
    HashMap<Integer, Integer> keyToVal;
    // key 到 freq 的映射，我们后文称为 KF 表
    HashMap<Integer, Integer> keyToFreq;
    // freq 到 key 列表的映射，我们后文称为 FK 表
    HashMap<Integer, LinkedHashSet<Integer>> freqToKeys;
    // 记录最小的频次
    int minFreq;
    // 记录 LFU 缓存的最大容量
    int cap;

    public LFUCache(int capacity) {
        keyToVal = new HashMap<>();
        keyToFreq = new HashMap<>();
        freqToKeys = new HashMap<>();
        this.cap = capacity;
        this.minFreq = 0;
    }

    public int get(int key) {
      if (!keyToVal.containsKey(key)) {
        return -1;
    	}
    	// 增加 key 对应的 freq是 LFU 算法的核心，抽象成一个函数increaseFreq
    	increaseFreq(key);
    	return keyToVal.get(key);
    }

  	// put 逻辑参考下图
    public void put(int key, int val) {
      if (this.cap <= 0) return;

      /* 若 key 已存在，修改对应的 val 即可 */
      if (keyToVal.containsKey(key)) {
          keyToVal.put(key, val);
          // key 对应的 freq 加一
          increaseFreq(key);
          return;
      }

      /* key 不存在，需要插入 */
      /* 容量已满的话需要淘汰一个 freq 最小的 key */
      if (this.cap <= keyToVal.size()) {
          removeMinFreqKey();
      }

      /* 插入 key 和 val，对应的 freq 为 1 */
      // 插入 KV 表
      keyToVal.put(key, val);
      // 插入 KF 表
      keyToFreq.put(key, 1);
      // 插入 FK 表
      freqToKeys.putIfAbsent(1, new LinkedHashSet<>());
      freqToKeys.get(1).add(key);
      // 插入新 key 后最小的 freq 肯定是 1
      this.minFreq = 1;
		}
  
    private void removeMinFreqKey() {
      // freq 最小的 key 列表
      LinkedHashSet<Integer> keyList = freqToKeys.get(this.minFreq);
      // 其中最先被插入的那个 key 就是该被淘汰的 key
      int deletedKey = keyList.iterator().next();
      /* 更新 FK 表 */
      keyList.remove(deletedKey);
      if (keyList.isEmpty()) {
          freqToKeys.remove(this.minFreq);
          // 问：这里需要更新 minFreq 的值吗？
         	/*这里没必要更新minFreq变量，想想removeMinFreqKey这个函数是在什么时候调用？在put方法中插入新key时可能调用。
         	而你回头看put的代码，插入新key时一定会把minFreq更新成 1，所以说即便这里minFreq变了，我们也不需要管它。*/
      }
      /* 更新 KV 表 */
      keyToVal.remove(deletedKey);
      /* 更新 KF 表 */
      keyToFreq.remove(deletedKey);
  	}
  
    private void increaseFreq(int key) {
      int freq = keyToFreq.get(key);
      /* 更新 KF 表 */
      keyToFreq.put(key, freq + 1);
      /* 更新 FK 表 */
      // 将 key 从 freq 对应的列表中删除
      freqToKeys.get(freq).remove(key);
      // 将 key 加入 freq + 1 对应的列表中
      freqToKeys.putIfAbsent(freq + 1, new LinkedHashSet<>());
      freqToKeys.get(freq + 1).add(key);
      // 如果 freq 对应的列表空了，移除这个 freq
      if (freqToKeys.get(freq).isEmpty()) {
          freqToKeys.remove(freq);
          // 如果这个 freq 恰好是 minFreq，更新 minFreq
          if (freq == this.minFreq) {
              this.minFreq++;
          }
      }
  }

}
```

LFU 的逻辑不难理解，但是写代码实现并不容易，因为你看我们要维护`KV`表，`KF`表，`FK`表三个映射，特别容易出错。对于这种情况，labuladong 教你三个技巧：

**1、**不要企图上来就实现算法的所有细节，而应该自顶向下，逐步求精，先写清楚主函数的逻辑框架，然后再一步步实现细节。

**2、**搞清楚映射关系，如果我们更新了某个`key`对应的`freq`，那么就要同步修改`KF`表和`FK`表，这样才不会出问题。

**3、**画图，画图，画图，重要的话说三遍，把逻辑比较复杂的部分用流程图画出来，然后根据图来写代码，可以极大减少出错的概率。

<img src="/Users/markfqwu/Library/Application Support/typora-user-images/image-20210303233423518.png" alt="image-20210303233423518" style="zoom:50%;" />