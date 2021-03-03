---
title: BFS算法套路框架
description: 
categories: algorithm
tags:
---

## 二叉树的最近公共祖先

给定一个二叉树，找到该树中两个指定节点的最近公共祖先

所有二叉树的套路都是一样的：

```c
void traverse(TreeNode root) {
    // 前序遍历
    traverse(root.left)
    // 中序遍历
    traverse(root.right)
    // 后序遍历
}
```

所以，只要看到二叉树的问题，先把这个框架写出来准没问题：

```c
TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
    TreeNode left = lowestCommonAncestor(root.left, p, q);
    TreeNode right = lowestCommonAncestor(root.right, p, q);
}
```

现在我们思考如何添加一些细节，把框架改造成解法。

**labuladong 告诉你，遇到任何递归型的问题，无非就是灵魂三问**：

**1、这个函数是干嘛的**？

**2、这个函数参数中的变量是什么？**

**3、得到函数的递归结果，你应该干什么**？

呵呵，看到这灵魂三问，你有没有感觉到熟悉？本号的动态规划系列文章，篇篇都在说的动态规划套路，首先要明确的是什么？是不是要明确「定义」「状态」「选择」，这仨不就是上面的灵魂三问吗？

下面我们就来看看如何回答这灵魂三问。

## 解法思路

 **首先看第一个问题，这个函数是干嘛的**？或者说，你给我描述一下`lowestCommonAncestor`这个函数的「定义」吧。

描述：给该函数输入三个参数`root`，`p`，`q`，它会返回一个节点。

情况 1，如果`p`和`q`都在以`root`为根的树中，函数返回的即是`p`和`q`的最近公共祖先节点。

情况 2，那如果`p`和`q`都不在以`root`为根的树中怎么办呢？函数理所当然地返回`null`呗。

情况 3，那如果`p`和`q`只有一个存在于`root`为根的树中呢？函数就会返回那个节点。

题目说了输入的`p`和`q`一定存在于以`root`为根的树中，但是递归过程中，以上三种情况都有可能发生，所以说这里要定义清楚，后续这些定义都会在代码中体现。

OK，第一个问题就解决了，把这个定义记在脑子里，无论发生什么，都不要怀疑这个定义的正确性，这是我们写递归函数的基本素养。



**然后来看第二个问题，这个函数的参数中，变量是什么**？或者说，你描述一个这个函数的「状态」吧。

描述：函数参数中的变量是`root`，因为根据框架，`lowestCommonAncestor(root)`会递归调用`root.left`和`root.right`；至于`p`和`q`，我们要求它俩的公共祖先，它俩肯定不会变化的。

第二个问题也解决了，你也可以理解这是「状态转移」，每次递归在做什么？不就是在把「以`root`为根」转移成「以`root`的子节点为根」，不断缩小问题规模嘛？



**最后来看第三个问题，得到函数的递归结果，你该干嘛**？或者说，得到递归调用的结果后，你做什么「选择」？

这就像动态规划系列问题，怎么做选择，需要观察问题的性质，找规律。那么我们就得分析这个「最近公共祖先节点」有什么特点呢？刚才说了函数中的变量是`root`参数，所以这里都要围绕`root`节点的情况来展开讨论。

先想 base case，如果`root`为空，肯定得返回`null`。如果`root`本身就是`p`或者`q`，比如说`root`就是`p`节点吧，如果`q`存在于以`root`为根的树中，显然`root`就是最近公共祖先；即使`q`不存在于以`root`为根的树中，按照情况 3 的定义，也应该返回`root`节点。

以上两种情况的 base case 就可以把框架代码填充一点了：

```c
TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
    // 两种情况的 base case
    if (root == null) return null;
    if (root == p || root == q) return root;

    TreeNode left = lowestCommonAncestor(root.left, p, q);
    TreeNode right = lowestCommonAncestor(root.right, p, q);
}
```

现在就要面临真正的挑战了，用递归调用的结果`left`和`right`来搞点事情。根据刚才第一个问题中对函数的定义，我们继续分情况讨论：

情况 1，如果`p`和`q`都在以`root`为根的树中，那么`left`和`right`一定分别是`p`和`q`（从 base case 看出来的）。

情况 2，如果`p`和`q`都不在以`root`为根的树中，直接返回`null`。

情况 3，如果`p`和`q`只有一个存在于`root`为根的树中，函数返回该节点。

明白了上面三点，可以直接看解法代码了：

```c
TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
    // base case
    if (root == null) return null;
    if (root == p || root == q) return root;

    TreeNode left = lowestCommonAncestor(root.left, p, q);
    TreeNode right = lowestCommonAncestor(root.right, p, q);
    // 情况 1
    if (left != null && right != null) {
        return root;
    }
    // 情况 2
    if (left == null && right == null) {
        return null;
    }
    // 情况 3
    return left == null ? right : left;
}
```

对于情况 1，你肯定有疑问，`left`和`right`非空，分别是`p`和`q`，可以说明`root`是它们的公共祖先，但能确定`root`就是「最近」公共祖先吗？

这就是一个巧妙的地方了，**因为这里是二叉树的后序遍历啊**！前序遍历可以理解为是从上往下，而后序遍历是从下往上，就好比从`p`和`q`出发往上走，第一次相交的节点就是这个`root`，你说这是不是最近公共祖先呢？

综上，二叉树的最近公共祖先就计算出来了。

## 总结

LCA 就是后序遍历，后序遍历有从下往上的特性，前序遍历有从上往下的特性