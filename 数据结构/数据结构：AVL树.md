---
title: 数据结构：AVL 树
abbrlink: 54793
date: 2021-07-04 10:50:29
categories: 
 - 数据结构
---

上一篇文章所讲述的【[二叉搜索树](https://www.wrp.cool/posts/46128/)】其实有一个非常严重的漏洞：如果插入的元素都是有序的怎么办？

> 如果按照有序的方式或者近乎有序的方式将元素插入到二叉搜索树当中去，那么此时的二叉搜索树将退化成一个链表的数据结构。于是引入平衡这种机制来防止这种情况的发生，引入平衡的二叉树被成为平衡二叉树。

**平衡：**<font color=blue>一棵树中对于任意节点都有左子树的高度减去右子树的高度的绝对值小于等于1，则称这棵二叉树是平衡的。</font>

# 概述

首先还是先看看百度百科中对于AVL树的定义（[AVL树-百度百科](https://baike.baidu.com/item/AVL%E6%A0%91)）。

**AVL树的特点：**
 - 本身首先是一棵二叉搜索树
 - 带有平衡性：每个节点的左右子树的高度之差的绝对值（**平衡因子**）最多为1。
   也就是说，AVL树，本质上是带了平衡功能的二叉搜索树。

**平衡因子：**左子树的高度减去右子树的高度。（注意：平衡因子的值可能为负，不过这是正常的，平衡因子的正负性能够帮助我们区分二叉树是向左倾斜的还是向右倾斜的）

# 准备工作
> 本质上AVL树还是一种二叉搜索树，但是引入了“平衡因子”的概念来维护树的平衡，所以在树的节点类中要新增加一个`height`变量来维护树的高度。

```java
/**
 * 节点类
 */
private class Node {
    public int e, height;
    public Node left, right;

    public Node(int e, Node left, Node right) {
        this.height = 1; // 新节点默认高度为 1
        this.e = e;
        this.left = left;
        this.right = right;
    }

    @Override
    public String toString() {
        return String.valueOf(e);
    }
}
```

设计两个辅助函数

```java
/**
 * 获取节点高度
 */
public int getHeight(Node node) {
    return node == null ? 0 : node.height;
}

/**
 * 获取平衡因子
 */
public int getBalanceFactor(Node node) {
    return node == null ? 0 : getHeight(node.left) - getHeight(node.right);
}
```

# 树的平衡与失衡
一个节点的平衡因子由有这个节点左子树的高度减去右子树的高度得到的。如果平衡因子的绝对值小于等于一，则这个节点是平衡的，反之则称之为失衡。

<font color=blue>那么什么情况会导致树的失衡呢？</font>
当左右子树的高度本来就相差1的情况下，其中较高的子树再次加1的情况下，此时就会产生失衡。

<font color=blue>那么失衡总共有多少种情况呢？</font>
四种，分别是LL型、RR型、LR型、RL型，L和R分别是Left和Right的缩写。
其中第一个字母代表的是较高的子树是左子树还是右子树，第二个字母代表的是新增的节点是增加在较高子树的左子树还是右子树。
除了这四种失衡的情况，其余的情况就都是平衡的了。

下面为这四种情况分别举个例子：
![20210714225927](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/20210714225927.png)

> 以上的四种情况失衡的节点都是根节点。

# 左旋转与右旋转 平衡 LL和RR

左旋转与右旋转就是对不平衡节点的平衡操作，就拿上面比较简单的LL和RR的例子来演示左旋转与右旋转。

**具体的旋转过程用这两张图来表示：**
![数据结构-AVL-3](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/数据结构-AVL-3.jpg)
![数据结构-AVL-2](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/数据结构-AVL-2.jpg)

> 这里的`D、E、F、G`分别是挂在在`A、B、C`节点下的二叉树，这里假设`D、E、F、G`高度相同。为了方便理解，你可以直接把`D、E、F、G`就当成是单个节点。

<font color=blue>这里最重要的一点是旋转前后整个树依然保持着二叉搜索树的性质，也就是图中的结论在旋转前和旋转后都是成立的。</font>

<font size=5>**编码实现：**</font>

```java
/**
 * 左旋转
 */
private Node leftRotate(Node a) {
    // 暂存
    Node b = a.right;
    Node e = b.left;

    // 旋转
    b.left = a;
    a.right = e;

    // 更新Height
    a.height = Math.max(getHeight(a.left), getHeight(a.right)) + 1;
    b.height = Math.max(getHeight(b.left), getHeight(b.right)) + 1;

    return b;
}

/**
 * 右旋转
 */
private Node rightRotate(Node a) {
    // 暂存
    Node b = a.left;
    Node f = b.right;

    // 旋转
    b.right = a;
    a.left = f;

    // 更新Height
    a.height = Math.max(getHeight(a.left), getHeight(a.right)) + 1;
    b.height = Math.max(getHeight(b.left), getHeight(b.right)) + 1;

    return b;
}
```

# 左旋转与右旋转 平衡 LR和RL

![数据结构-AVL-4](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/数据结构-AVL-4.jpg)

大概的处理过程就是这个样子，先旋转其中一个节点，将LR和RL变成LL或者是RR的形式，最后再旋转一次即可完成对LR和RL的平衡。

# 在二叉搜索树中引入AVL

```java
/**
 * 保持平衡
 */
public Node keepBalance(Node node) {

    // 平衡维护
    int balance = getBalanceFactor(root);

    // LL
    if (balance >= 2 && getBalanceFactor(root.left) >= 0)
        return rightRotate(root);

    // RR
    if (balance <= -2 && getBalanceFactor(root.right) <= 0)
        return leftRotate(root);

    // LR
    if (balance >= 2 && getBalanceFactor(root.left) <= 0) {
        root.left = leftRotate(root.left);
        return rightRotate(root);
    }

    // RL
    if (balance <= -2 && getBalanceFactor(root.right) >= 0) {
        root.right = rightRotate(root.right);
        return leftRotate(root);
    }

    return root;
}

/**
 * 添加节点
 */
public void add(int e) {
    root = add(root, e);
}
private Node add(Node root, int e) {
    if (root == null) {
        size++;
        return new Node(e, null, null);
    }

    if (e < root.e) root.left = add(root.left, e);
    if (e > root.e) root.right = add(root.right, e);

    // 更新Height
    root.height = Math.max(getHeight(root.left), getHeight(root.right)) + 1;

    // 平衡维护
    return keepBalance(root);
}

/**
 * 删除节点
 */
public void delNode(int e) {
    delNode(root, e);
}
private Node delNode(Node root, int e) {
    if (root == null) return null;

    if (e > root.e) {
        root.right = delNode(root.right, e);
        return root;
    }
    if (e < root.e) {
        root.left = delNode(root.left, e);
        return root;
    }

    size --;
    Node retNode;
    if (root.left == null) retNode =  root.right;
    if (root.right == null) retNode = root.left;
    retNode = delMax(root.left);

    // 更新Height
    root.height = Math.max(getHeight(root.left), getHeight(root.right)) + 1;

    // 平衡维护
    return keepBalance(retNode);
}
```