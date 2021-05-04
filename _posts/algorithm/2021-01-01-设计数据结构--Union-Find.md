---
title: Union-Find 算法
description: 用框架思维看待算法套路
categories: algorithm
tags:
---

## 算法详解

Union-Find 算法，也就是常说的并查集算法，主要是解决图论中「动态连通性」问题的。

简单说，动态连通性其实可以抽象成给一幅图连线。

这里所说的「连通」是一种等价关系，也就是说具有如下三个性质：

**1、自反性**：节点`p`和`p`是连通的。

**2、对称性**：如果节点`p`和`q`连通，那么`q`和`p`也连通。

**3、传递性**：如果节点`p`和`q`连通，`q`和`r`连通，那么`p`和`r`也连通。

Union-Find 算法的关键就在于`union`和`connected`函数的效率

思路：使用森林（若干棵树）来表示图的动态连通性，用数组来具体实现这个森林。

怎么用森林来表示连通性呢？我们设定树的每个节点有一个指针指向其父节点，如果是根节点的话，这个指针指向自己。

```c++
class UF {
    // 连通分量个数
    private int count;
    // 存储多棵树，节点 x 的父节点是 parent[x]
    private int[] parent;
    // 记录树的“重量”，表示size[x] 树有几个节点
    private int[] size;

    public UF(int n) {
        // 一开始互不连通
        this.count = n;
        parent = new int[n];
        size = new int[n];
        for (int i = 0; i < n; i++) {
          	// 父节点指针初始指向自己
            parent[i] = i;
          	// 一开始每棵树只有一个节点
            size[i] = 1;
        }
    }

    /* 将 p 和 q 连接 */
    public void union(int p, int q) {
        int rootP = find(p);
        int rootQ = find(q);
        if (rootP == rootQ)
            return;

        // 小树接到大树下面，较平衡
        if (size[rootP] > size[rootQ]) {
            parent[rootQ] = rootP;
            size[rootP] += size[rootQ];
        } else {
            parent[rootP] = rootQ;
            size[rootQ] += size[rootP];
        }
        count--;
    }

  	/* 判断 p 和 q 是否连通 */
    public boolean connected(int p, int q) {
        int rootP = find(p);
        int rootQ = find(q);
        return rootP == rootQ;
    }
  
    /* 返回当前的连通分量个数 */
    public int count() { 
        return count;
    }
  
  	/*路径压缩，使find能以 O(1) 的时间找到某一节点的根节点，相应的，connected和union复杂度都下降为 O(1)。*/
    private int find(int x) {
        while (parent[x] != x) {
            // 进行路径压缩
            parent[x] = parent[parent[x]];
            x = parent[x];
        }
        return x;
    }
}
```

算法的关键点有 3 个：

**1、**用`parent`数组记录每个节点的父节点，相当于指向父节点的指针，所以`parent`数组内实际存储着一个森林（若干棵多叉树）。

**2、**用`size`数组记录着每棵树的重量，目的是让`union`后树依然拥有平衡性，而不会退化成链表，影响操作效率。

**3、**在`find`函数中进行路径压缩，保证任意树的高度保持在常数，使得`union`和`connected`API 时间复杂度为 O(1)。

## 算法应用

首先，Union-Find 算法解决的是图的动态连通性问题，这个算法本身不难，能不能应用出来主要是看你抽象问题的能力，是否能够把原始问题抽象成一个有关图论的问题。

## DFS 的替代方案

leetcode 130 给你一个 M×N 的二维矩阵，其中包含字符`X`和`O`，让你找到矩阵中**完全**被`X`围住的`O`，并且把它们替换成`X`。

解决这个问题的传统方法也不困难，先用 for 循环遍历棋盘的**四边**，用 DFS 算法把那些与边界相连的`O`换成一个特殊字符，比如`#`；然后再遍历整个棋盘，把剩下的`O`换成`X`，把`#`恢复成`O`。这样就能完成题目的要求，时间复杂度 O(MN)。

使用 Union-Find 算法：

**你可以把那些不需要被替换的`O`看成一个拥有独门绝技的门派，它们有一个共同祖师爷叫`dummy`，这些`O`和`dummy`互相连通，而那些需要被替换的`O`与`dummy`不连通**。

<img src="https://mmbiz.qpic.cn/sz_mmbiz_jpg/gibkIz0MVqdFYt2Sv0QHpOlDhRGsaamLRSpAjNAzuQXnv1ojPV2fKcPwjI172uI1SwslwxcovB7IHXRFQR2fiaDQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1" alt="图片" style="zoom:50%;" />



```c++
void solve(char[][] board) {
    if (board.length == 0) return;

    int m = board.length;
    int n = board[0].length;
    // 给 dummy 留一个额外位置
    UF uf = new UF(m * n + 1);
    int dummy = m * n;
    // 将首列和末列的 O 与 dummy 连通
    for (int i = 0; i < m; i++) {
        if (board[i][0] == 'O')
            uf.union(i * n, dummy);
        if (board[i][n - 1] == 'O')
            uf.union(i * n + n - 1, dummy);
    }
    // 将首行和末行的 O 与 dummy 连通
    for (int j = 0; j < n; j++) {
        if (board[0][j] == 'O')
            uf.union(j, dummy);
        if (board[m - 1][j] == 'O')
            uf.union(n * (m - 1) + j, dummy);
    }
    // 方向数组 d 是上下左右搜索的常用手法
    int[][] d = new int[][]{ {1,0}, {0,1}, {0,-1}, {-1,0} };
    for (int i = 1; i < m - 1; i++) 
        for (int j = 1; j < n - 1; j++) 
            if (board[i][j] == 'O')
                // 将此 O 与上下左右的 O 连通
                for (int k = 0; k < 4; k++) {
                    int x = i + d[k][0];
                    int y = j + d[k][1];
                    if (board[x][y] == 'O')
                        uf.union(x * n + y, i * n + j);
                }
    // 所有不和 dummy 连通的 O，都要被替换
    for (int i = 1; i < m - 1; i++) 
        for (int j = 1; j < n - 1; j++) 
            if (!uf.connected(dummy, i * n + j))
                board[i][j] = 'X';
}
```

**主要思路是适时增加虚拟节点，想办法让元素「分门别类」，建立动态连通关系**。

## 判定合法算式

给你一个数组`equations`，装着若干字符串表示的算式。每个算式`equations[i]`长度都是 4，而且只有这两种情况：`a==b`或者`a!=b`，其中`a,b`可以是任意小写字母。你写一个算法，如果`equations`中所有算式都不会互相冲突，返回 true，否则返回 false。

分析：

动态连通性其实就是一种等价关系，具有「自反性」「传递性」和「对称性」，其实`==`关系也是一种等价关系，具有这些性质。所以这个问题用 Union-Find 算法就很自然。

核心思想是，**将`equations`中的算式根据`==`和`!=`分成两部分，先处理`==`算式，使得他们通过相等关系各自勾结成门派；然后处理`!=`算式，检查不等关系是否破坏了相等关系的连通性**。

```c++
boolean equationsPossible(String[] equations) {
    // 26 个英文字母
    UF uf = new UF(26);
    // 先让相等的字母形成连通分量
    for (String eq : equations) {
        if (eq.charAt(1) == '=') {
            char x = eq.charAt(0);
            char y = eq.charAt(3);
            uf.union(x - 'a', y - 'a');
        }
    }
    // 检查不等关系是否打破相等关系的连通性
    for (String eq : equations) {
        if (eq.charAt(1) == '!') {
            char x = eq.charAt(0);
            char y = eq.charAt(3);
            // 如果相等关系成立，就是逻辑冲突
            if (uf.connected(x - 'a', y - 'a'))
                return false;
        }
    }
    return true;
}
```

## 总结

使用 Union-Find 算法，主要是如何把原问题转化成图的动态连通性问题。对于算式合法性问题，可以直接利用等价关系，对于棋盘包围问题，则是利用一个虚拟节点，营造出动态连通特性。

另外，将二维数组映射到一维数组，利用方向数组`d`来简化代码量，都是在写算法时常用的一些小技巧，如果没见过可以注意一下