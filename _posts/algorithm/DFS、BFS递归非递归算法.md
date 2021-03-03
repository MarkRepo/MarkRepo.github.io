---
title: DFS、BFS递归非递归
description: 
categories: algorithm
tags:

---

## DFS 递归

```java
public class TreeNode {
    int val;
    TreeNode left;
    TreeNode right;
    TreeNode(int x) { val = x; }
}
```

```java
List<TreeNode> list = new ArrayList<>();
public List<TreeNode> traversal(TreeNode root) {
    dfs(root);
    return list;
}

private void dfs(TreeNode root) {
    if (root == null) return;
    list.add(root);
    dfs(root.left);
    dfs(root.right);
}
```

## DFS 非递归

```java
public List<TreeNode> traversal(TreeNode root) {
    if (root == null) return null;
    List<TreeNode> res = new ArrayList<>();
    Stack<TreeNode> stack = new Stack<>();
    stack.add(root);
    while(!stack.empty()) {
        TreeNode node = stack.pop();
        res.add(node);
        if (node.right != null) {
            stack.push(node.right);
        }
        if (node.left != null) {
            stack.push(node.left);
        }
    }
    return res;
}
```

## BFS 非递归

```java
public List<TreeNode> traversal(TreeNode root) {
  if (root == null) return null;
  return bfs(root);
}

private List<TreeNode> bfs(TreeNode root) {
  //if (root == null) return null;
  int curNum = 1;   //维护当前层的node数量
  int nextNum = 0;  //维护下一层的node数量
  Queue<TreeNode> queue = new LinkedList<>();
  List<TreeNode> list = new ArrayList<>();
  queue.offer(root);
  while(!queue.isEmpty()) {
    TreeNode node = queue.poll();
    list.add(node);
    curNum--;
    if (node.left != null) {
      queue.add(node.left);
      nextNum++;
    }
    if (node.right != null) {
      queue.add(node.right);
      nextNum++;
    }

    if (curNum == 0) {
      curNum = nextNum;
      nextNum = 0;
    }
  }

  return list;
}
```

## BFS 递归

这里所谓的bfs递归形式，是利用dfs的递归形式，在递归过程中记录每个node的level，然后将属于一个level的node放到一list里面

```java
public List<List<TreeNode>> traversal(TreeNode root) {
    if (root == null) return null;
    List<List<TreeNode>> list = new ArrayList<>();
    dfs(root, 0, list);
    return list;
}

private void dfs(TreeNode root, int level, List<List<TreeNode>> list) {
    if (root == null) return;
    if (level >= list.size()) {
      	// 创建新的list
        List<TreeNode> subList = new ArrayList<>();
        subList.add(root);
        list.add(subList);
    } else {
        list.get(level).add(root);
    }
    dfs(root.left, level+1, list);
    dfs(root.right, level+1, list);
}
```

