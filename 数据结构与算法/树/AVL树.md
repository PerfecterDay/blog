#  AVL树-平衡二叉搜索树
{docsify-updated}

> https://www.baeldung.com/cs/balanced-bst-from-sorted-list

- [AVL树-平衡二叉搜索树](#avl树-平衡二叉搜索树)
  - [由有序列表创建平衡二叉搜索树](#由有序列表创建平衡二叉搜索树)
    - [Top-Down 方法](#top-down-方法)
    - [Bottom-Up 方法](#bottom-up-方法)
  - [保持平衡性](#保持平衡性)
    - [右旋](#右旋)
    - [左旋](#左旋)
    - [先左旋后右旋](#先左旋后右旋)
    - [先右旋后左旋](#先右旋后左旋)
    - [旋转的选择](#旋转的选择)
  - [AVL树实现代码](#avl树实现代码)

1962 年 G.M.Adelson‑Velsky 和 E.M.Landis 在 论 文 “An algorithm for the organization ofinformation”中提出了「AVL 树」。为了纪念他们，将平衡二叉树称为AVL树。

首先，让我们定义一下平衡二叉搜索树的含义：
+ 平衡二叉树是一棵二叉搜索树树，它的高度是 $O(log(n))$，其中n是树内的节点数。
+ 对于平衡树内的每个节点，左子树的高度与右子树的高度相差不得超过 1。

### 由有序列表创建平衡二叉搜索树
在创建平衡 BST 时，我们需要牢记高度条件。首先，让我们考虑一下将哪个节点作为根节点最好。

**由于我们需要平衡树，所以必须将中间的值作为根节点。然后，我们可以将中间值之前的值添加到树的左边。因此，所有较小的值都将被添加到左侧子树。同样，我们将中间值之后的值添加到树的右边，这样所有较大的值都会添加到右边的子树中。**

不过，获取中间的元素取决于列表的类型。如果列表支持随机存取（如数组），我们可以简单地获取中间元素，将其作为根，然后递归地添加左右子树。我们称这种方法为Top-down法。

但是，如果我们只有指向列表中第一个元素的指针，那么获取中间元素就比较麻烦了。因此，在这种情况下，我们将使用Bottom-up的方法。

#### Top-Down 方法
自上而下的方法使用排序数组来创建平衡的 BST。因此，我们可以在恒定时间内访问数组内的任意索引：
<center><img src="pics/quicklatex.com-bd18cef6ee8eb252f700ebf114b4d5fa_l3.svg" alt=""></center>

该函数的初始调用传递数组 $A$，$L$为0，$R$为 $n-1$，其中$n$为数组的大小。

一开始，我们会检查是否到达了一个空范围。在这种情况下，我们只需返回 null，表示对象为空。

接下来，我们获取中间索引并用 $A[mid]$初始化 root 之后，我们进行两次递归调用。第一次调用的对象是范围$[L, mid-1]$ , 表示索引$mid$之前的元素。另一方面，第二次调用的是范围 $[mid+1, R]$，它对应于索引$mid$之后的元素。

第一次调用返回左子树的根。因此，我们将其赋值给根节点的左指针。同样，第二次调用会返回右侧子树的根节点。因此，我们将其赋值给根节点的右指针。

自顶向下方法的复杂度为 $ O(n)$，其中为$n$数组内部元素的个数。

```
class BalanceBstree extends BsTree {

    public BalanceBstree topDown(int[] arr) {
        BalanceBstree balanceBstree = new BalanceBstree();
        root = build(arr, 0, arr.length - 1);
        return balanceBstree;
    }

    private Node build(int[] arr, int left, int right) {
        if (left > right) {
            return null;
        }

        int mid = (left + right) / 2;
        Node root = new Node(arr[mid]);
        root.left = build(arr, left, mid - 1);
        root.right = build(arr, mid + 1, right);
        return root;
    }
}
```

#### Bottom-Up 方法
自下而上的方法使用链表来构建平衡的 BST。因此，我们会有一个指向列表头部的指针，我们只能在恒定时间内向前移动这个指针。
<center><img src="pics/quicklatex.com-0ab014825b9004386afe016d6b44ec97_l3.svg" alt=""></center>

```
algorithm BottomUpBST(head, L, R):
    // INPUT
    //   head = Pointer to the first element in the linked list
    //   L = The left side of the current range
    //   R = The right side of the current range
    // OUTPUT
    //   The root of the balanced BST

    if L > R:
        return null

    mid <- (L + R) / 2

    // Build left subtree
    root.left <- BottomUpBST(head, L, mid - 1)

    // Assign root value
    root.value <- head.data

    // Move to the next element in the list
    head <- head.next

    // Build right subtree
    root.right <- BottomUpBST(head, mid + 1, R)

    return root
```

### 保持平衡性
AVL 树的特点在于“旋转”操作，它能够在不影响二叉树的中序遍历序列的前提下，使失衡节点重新恢复平衡。换句话说，旋转操作既能保持“二叉搜索树”的性质，也能使树重新变为“平衡二叉树”。  
我们将平衡因子绝对值 > 1 的节点称为“失衡节点”。根据节点失衡情况的不同，旋转操作分为四种：右旋、左旋、先右旋后左旋、先左旋后右旋。下面详细介绍这些旋转操作。

#### 右旋
以该失衡节点为根节点的子树，将该节点记为 node ，其左子节点记为 child ，执行“右旋”操作：
<center><img src="pics/turn-right.jpg" width="60%"></center>

当失衡子树带有 grand_child 时：
<center><img src="pics/turn-right-2.jpg" width="60%"></center>

```
/* 右旋操作 */
TreeNode rightRotate(TreeNode node) {
TreeNode child = node.left;
TreeNode grandChild = child.right;
// 以 child 为原点，将 node 向右旋转
child.right = node;
node.left = grandChild;
// 更新节点高度
updateHeight(node);
updateHeight(child);
// 返回旋转后子树的根节点
return child;
}
```

#### 左旋
<center><img src="pics/turn-left.jpg" width="60%"></center>

当失衡子树带有 grand_child 时：
<center><img src="pics/turn-left-2.jpg" width="60%"></center>

可以观察到，**右旋和左旋操作在逻辑上是镜像对称的，它们分别解决的两种失衡情况也是对称的。**基于对称性，我们只需将右旋的实现代码中的所有的 left 替换为 right ，将所有的 right 替换为 left ，即可得到左旋的实现代码：
```
TreeNode rightRotate(TreeNode node) {
TreeNode child = node.right;
TreeNode grandChild = child.left;
// 以 child 为原点，将 node 向右旋转
child.right = node;
node.right = grandChild;
// 更新节点高度
updateHeight(node);
updateHeight(child);
// 返回旋转后子树的根节点
return child;
}
```

#### 先左旋后右旋
<center><img src="pics/left-then-right.jpg" width="60%"></center>

#### 先右旋后左旋
<center><img src="pics/right-then-left.jpg" width="60%"></center>

#### 旋转的选择
<center><img src="pics/rotate.jpg" width="60%"></center>

节点的「平衡因子 balance factor」定义为节点**左子树的高度减去右子树的高度**，同时规定空节点的平衡因子为 0.
<center><img src="pics/rotate-2.jpg" width="50%"></center>

```
/* 执行旋转操作，使该子树重新恢复平衡 */
private TreeNode rotate(TreeNode node) {
    // 获取节点 node 的平衡因子
    int balanceFactor = balanceFactor(node);
    // 左偏树
    if (balanceFactor > 1) {
        if (balanceFactor(node.left) >= 0) {
            // 右旋
            return rightRotate(node);
        } else {
            // 先左旋后右旋
            node.left = leftRotate(node.left);
            return rightRotate(node);
        }
    }
    // 右偏树
    if (balanceFactor < -1) {
        if (balanceFactor(node.right) <= 0) {
            // 左旋
            return leftRotate(node);
        } else {
            // 先右旋后左旋
            node.right = rightRotate(node.right);
            return leftRotate(node);
        }
    }
    // 平衡树，无须旋转，直接返回
    return node;
}
```

### AVL树实现代码
```
/**
 * File: avl_tree.java
 * Created Time: 2022-12-10
 * Author: krahets (krahets@163.com)
 */

package chapter_tree;

import utils.*;

/* AVL 树 */
class AVLTree {
    TreeNode root; // 根节点

    /* 获取节点高度 */
    public int height(TreeNode node) {
        // 空节点高度为 -1 ，叶节点高度为 0
        return node == null ? -1 : node.height;
    }

    /* 更新节点高度 */
    private void updateHeight(TreeNode node) {
        // 节点高度等于最高子树高度 + 1
        node.height = Math.max(height(node.left), height(node.right)) + 1;
    }

    /* 获取平衡因子 */
    public int balanceFactor(TreeNode node) {
        // 空节点平衡因子为 0
        if (node == null)
            return 0;
        // 节点平衡因子 = 左子树高度 - 右子树高度
        return height(node.left) - height(node.right);
    }

    /* 右旋操作 */
    private TreeNode rightRotate(TreeNode node) {
        TreeNode child = node.left;
        TreeNode grandChild = child.right;
        // 以 child 为原点，将 node 向右旋转
        child.right = node;
        node.left = grandChild;
        // 更新节点高度
        updateHeight(node);
        updateHeight(child);
        // 返回旋转后子树的根节点
        return child;
    }

    /* 左旋操作 */
    private TreeNode leftRotate(TreeNode node) {
        TreeNode child = node.right;
        TreeNode grandChild = child.left;
        // 以 child 为原点，将 node 向左旋转
        child.left = node;
        node.right = grandChild;
        // 更新节点高度
        updateHeight(node);
        updateHeight(child);
        // 返回旋转后子树的根节点
        return child;
    }

    /* 执行旋转操作，使该子树重新恢复平衡 */
    private TreeNode rotate(TreeNode node) {
        // 获取节点 node 的平衡因子
        int balanceFactor = balanceFactor(node);
        // 左偏树
        if (balanceFactor > 1) {
            if (balanceFactor(node.left) >= 0) {
                // 右旋
                return rightRotate(node);
            } else {
                // 先左旋后右旋
                node.left = leftRotate(node.left);
                return rightRotate(node);
            }
        }
        // 右偏树
        if (balanceFactor < -1) {
            if (balanceFactor(node.right) <= 0) {
                // 左旋
                return leftRotate(node);
            } else {
                // 先右旋后左旋
                node.right = rightRotate(node.right);
                return leftRotate(node);
            }
        }
        // 平衡树，无须旋转，直接返回
        return node;
    }

    /* 插入节点 */
    public void insert(int val) {
        root = insertHelper(root, val);
    }

    /* 递归插入节点（辅助方法） */
    private TreeNode insertHelper(TreeNode node, int val) {
        if (node == null)
            return new TreeNode(val);
        /* 1. 查找插入位置并插入节点 */
        if (val < node.val)
            node.left = insertHelper(node.left, val);
        else if (val > node.val)
            node.right = insertHelper(node.right, val);
        else
            return node; // 重复节点不插入，直接返回
        updateHeight(node); // 更新节点高度
        /* 2. 执行旋转操作，使该子树重新恢复平衡 */
        node = rotate(node);
        // 返回子树的根节点
        return node;
    }

    /* 删除节点 */
    public void remove(int val) {
        root = removeHelper(root, val);
    }

    /* 递归删除节点（辅助方法） */
    private TreeNode removeHelper(TreeNode node, int val) {
        if (node == null)
            return null;
        /* 1. 查找节点并删除 */
        if (val < node.val)
            node.left = removeHelper(node.left, val);
        else if (val > node.val)
            node.right = removeHelper(node.right, val);
        else {
            if (node.left == null || node.right == null) {
                TreeNode child = node.left != null ? node.left : node.right;
                // 子节点数量 = 0 ，直接删除 node 并返回
                if (child == null)
                    return null;
                // 子节点数量 = 1 ，直接删除 node
                else
                    node = child;
            } else {
                // 子节点数量 = 2 ，则将中序遍历的下个节点删除，并用该节点替换当前节点
                TreeNode temp = node.right;
                while (temp.left != null) {
                    temp = temp.left;
                }
                node.right = removeHelper(node.right, temp.val);
                node.val = temp.val;
            }
        }
        updateHeight(node); // 更新节点高度
        /* 2. 执行旋转操作，使该子树重新恢复平衡 */
        node = rotate(node);
        // 返回子树的根节点
        return node;
    }

    /* 查找节点 */
    public TreeNode search(int val) {
        TreeNode cur = root;
        // 循环查找，越过叶节点后跳出
        while (cur != null) {
            // 目标节点在 cur 的右子树中
            if (cur.val < val)
                cur = cur.right;
            // 目标节点在 cur 的左子树中
            else if (cur.val > val)
                cur = cur.left;
            // 找到目标节点，跳出循环
            else
                break;
        }
        // 返回目标节点
        return cur;
    }
}

public class avl_tree {
    static void testInsert(AVLTree tree, int val) {
        tree.insert(val);
        System.out.println("\n插入节点 " + val + " 后，AVL 树为");
        PrintUtil.printTree(tree.root);
    }

    static void testRemove(AVLTree tree, int val) {
        tree.remove(val);
        System.out.println("\n删除节点 " + val + " 后，AVL 树为");
        PrintUtil.printTree(tree.root);
    }

    public static void main(String[] args) {
        /* 初始化空 AVL 树 */
        AVLTree avlTree = new AVLTree();

        /* 插入节点 */
        // 请关注插入节点后，AVL 树是如何保持平衡的
        testInsert(avlTree, 1);
        testInsert(avlTree, 2);
        testInsert(avlTree, 3);
        testInsert(avlTree, 4);
        testInsert(avlTree, 5);
        testInsert(avlTree, 8);
        testInsert(avlTree, 7);
        testInsert(avlTree, 9);
        testInsert(avlTree, 10);
        testInsert(avlTree, 6);

        /* 插入重复节点 */
        testInsert(avlTree, 7);

        /* 删除节点 */
        // 请关注删除节点后，AVL 树是如何保持平衡的
        testRemove(avlTree, 8); // 删除度为 0 的节点
        testRemove(avlTree, 5); // 删除度为 1 的节点
        testRemove(avlTree, 4); // 删除度为 2 的节点

        /* 查询节点 */
        TreeNode node = avlTree.search(7);
        System.out.println("\n查找到的节点对象为 " + node + "，节点值 = " + node.val);
    }
}
```