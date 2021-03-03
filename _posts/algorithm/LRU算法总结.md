---
title: LRU 算法
description: 
categories: algorithm
tags:

---

## LRU 算法设计

LRU（Least Recently Used 最近最少使用） 算法就是一种缓存淘汰策略。

首先要接收一个 `capacity` 参数作为缓存的最大容量，然后实现两个 API，一个是 `put(key, val)` 方法存入键值对，另一个是 `get(key)` 方法获取 `key`对应的 `val`，如果 `key` 不存在则返回 -1。

注意哦，`get` 和 `put` 方法必须都是 `O(1)` 的时间复杂度。

分析：要让 `put` 和 `get` 方法的时间复杂度为 O(1)，我们可以总结出 `cache` 这个数据结构必要的条件：

1、显然 `cache` 中的元素必须有时序，以区分最近使用的和久未使用的数据，当容量满了之后要删除最久未使用的那个元素腾位置。

2、我们要在 `cache` 中快速找某个 `key` 是否已存在并得到对应的 `val`；

3、每次访问 `cache` 中的某个 `key`，需要将这个元素变为最近使用的，也就是说 `cache` 要支持在任意位置快速插入和删除元素。

LRU 缓存算法的核心数据结构就是哈希链表，双向链表和哈希表的结合体。这个数据结构长这样：

![图片](https://mmbiz.qpic.cn/sz_mmbiz_jpg/gibkIz0MVqdHkcPqjzoDYrtO88MrDuPB5TN0Pr0iax20pqyeWibyjDtapiaCaJChucMTjhlibwyHBToIyaLqkr2Tdxw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## 代码实现

```c++
class Node {
    public:
  		int key, val;
    	Node *next, *prev;
    public Node(int k, int v) {
        this->key = k;
        this->val = v;
    }
}

//然后依靠我们的 Node 类型构建一个双链表，实现几个 LRU 算法必须的 API：
class DoubleList {  
    // 头尾虚节点
    private:
      Node *head, *tail;  
    // 链表元素数
    private:
  		int size;

    public DoubleList() {
        // 初始化双向链表的数据
        head = new Node(0, 0);
        tail = new Node(0, 0);
        head->next = tail;
        tail->prev = head;
        size = 0;
    }

    // 在链表尾部添加节点 x，时间 O(1)
    public void addLast(Node* x) {
        x->prev = tail->prev;
        x->next = tail;
        tail->prev->next = x;
        tail->prev = x;
        size++;
    }

    // 删除链表中的 x 节点（x 一定存在）
    // 由于是双链表且给的是目标 Node 节点，时间 O(1)
    public void remove(Node* x) {
        x->prev->next = x->next;
        x->next->prev = x->prev;
        size--;
    }

    // 删除链表中第一个节点，并返回该节点，时间 O(1)
    public Node* removeFirst() {
        if (head->next == tail)
            return null;
        Node* first = head->next;
        remove(first);
        return first;
    }

    // 返回链表长度，时间 O(1)
    public int size() { return size; }

}

class LRUCache {
    // key -> Node(key, val)
    private:
  		HashMap<int, Node*> map;
    // Node(k1, v1) <-> Node(k2, v2)...
  		DoubleList cache;
    // 最大容量
  		int cap;

    public LRUCache(int capacity) {
      this->cap = capacity;
      map = new HashMap<>();
      cache = new DoubleList();
    }
  
  	/* 
  	由于我们要同时维护一个双链表 cache 和一个哈希表 map，很容易漏掉一些操作，比如说删除某个 key 时，
  	在 cache 中删除了对应的 Node，但是却忘记在 map 中删除 key。
  	解决这种问题的有效方法是：在这两种数据结构之上提供一层抽象 API。
  	说的有点玄幻，实际上很简单，就是尽量让 LRU 的主方法 get 和 put 避免直接操作 ma 和 cache 的细节。
  	因此实现下面几个封装函数
  	*/
  
		private:
  		/* 将某个 key 提升为最近使用的 */
      void makeRecently(int key) {
    		Node* x = map.get(key);
    		// 先从链表中删除这个节点
    		cache.remove(x);
    		// 重新插到队尾
    		cache.addLast(x);
			}

			/* 添加最近使用的元素 */
  		void addRecently(int key, int val) {
    		Node* x = new Node(key, val);
    		// 链表尾部就是最近使用的元素
    		cache.addLast(x);
    		// 别忘了在 map 中添加 key 的映射
    		map.put(key, x);
			}

			/* 删除某一个 key */
 			void deleteKey(int key) {
    		Node* x = map.get(key);
    		// 从链表中删除
    		cache.remove(x);
    		// 从 map 中删除
    		map.remove(key);
			}

			/* 删除最久未使用的元素 */
  		void removeLeastRecently() {
    		// 链表头部的第一个元素就是最久未使用的
    		Node* deletedNode = cache.removeFirst();
    		// 同时别忘了从 map 中删除它的 key
    		int deletedKey = deletedNode.key;
    		map.remove(deletedKey);
			}
  
    public:
  		int get(int key) {
    		if (!map.containsKey(key)) {
        	return -1;
    		}
    		// 将该数据提升为最近使用的
    		makeRecently(key);
    		return map.get(key).val;
			}
  
			void put(int key, int val) {
    		if (map.containsKey(key)) {
        	// 删除旧的数据
        	deleteKey(key);
        	// 新插入的数据为最近使用的数据
        	addRecently(key, val);
        	return;
    		}

    		if (cap == cache.size()) {
        	// 删除最久未使用的元素
        	removeLeastRecently();
    		}
    		// 添加为最近使用的元素
    		addRecently(key, val);
			}
}
```



