#  BST-二叉搜索树
{docsify-updated}
> https://www.baeldung.com/cs/sorting-binary-tree  

- [BST-二叉搜索树](#bst-二叉搜索树)
	- [BST-二叉搜索树的定义](#bst-二叉搜索树的定义)
	- [BST-二叉搜索树的构造](#bst-二叉搜索树的构造)


### BST-二叉搜索树的定义
二叉搜索树、二叉排序树或者叫BST，是一棵满足以下条件的二叉树：
+ 对于每个节点，左子树的节点的值都小于该节点值
+ 对于每个节点，右子树的节点的值都大于该节点值
+ 递归地，左子树和右子树也是BST

BST 的中序遍历就是一个排序。

### BST-二叉搜索树的构造
构造BST：
<center><img src="pics/bst-construct.svg" alt=""></center>

```
public class BinarySearchTree {
    public static void main(String[] args) {
//        BsTree bsTree = new BsTree();
//        bsTree.insert(8);
//        bsTree.insert(2);
//        bsTree.insert(3);
//        bsTree.insert(9);
//        bsTree.insert(10);
//        bsTree.insert(1);
//        bsTree.insert(20);
//        System.out.println("in order");
//        bsTree.inorderTraverse();
//        System.out.println("pre order");
//        bsTree.preorderTraverse();
//        System.out.println("post order");
//        bsTree.postorderTraverse();
//        bsTree.bfsTraverse();

        int[] arr = new int[6];
        for (int i = 0; i < 6; i++) {
            arr[i] = i;
        }
        BalanceBstree balanceBstree = new BalanceBstree();
        balanceBstree.topDown(arr);

        System.out.println("in order");
        balanceBstree.inorderTraverse();
        System.out.println("pre order");
        balanceBstree.preorderTraverse();
        System.out.println("post order");
        balanceBstree.postorderTraverse();

    }
}

class Node {
    int value;
    Node left;
    Node right;

    public Node(int value) {
        this.value = value;
        this.left = null;
        this.right = null;
    }
}

class BsTree {
    Node root = null;

    public BsTree insert(int value) {
        root = insert(this.root, value);
        return this;
    }

    private Node insert(Node node, int value) {
        if (node == null) {
            node = new Node(value);
            return node;
        } else {
            if (value < node.value) {
                node.left = insert(node.left, value);
            } else {
                node.right = insert(node.right, value);
            }
        }
        return node;
    }

    public void inorderTraverse() {
        recInorderTraverse(this.root);
        System.out.println("-----------------");
        iteInorderTraverse(this.root);
    }

    public void preorderTraverse() {
        itePreorderTraverse(this.root);
    }

    public void postorderTraverse() {
        itePostOderTraverse(this.root);
    }

    /*
     * 递归的中序遍历，返回的排序的数列
     * */
    private void recInorderTraverse(Node node) {
        if (node != null) {
            recInorderTraverse(node.left);
            System.out.println(node.value);
            recInorderTraverse(node.right);
        }
    }

    /*
     * 借助栈实现迭代的中序遍历，返回的排序的数列
     * */
    private void iteInorderTraverse(Node node) {
        if (node == null) {
            return;
        }
        Stack<Node> stack = new Stack();
        while (stack.size() > 0 || node != null) {
            if (node != null) {
                stack.push(node);
                node = node.left;
            } else {
                Node curr = stack.pop();
                System.out.println(curr.value);
                node = curr.right;
            }
        }
    }

    /*
     * 借助栈实现迭代的前序遍历，返回的排序的数列
     * */
    private void itePreorderTraverse(Node node) {
        if (node == null) {
            return;
        }
        Stack<Node> stack = new Stack();
        stack.push(node);
        while (!stack.isEmpty()) {
            Node indexNode = stack.pop();
            System.out.println(indexNode.value);
            if (indexNode.right != null) {
                stack.push(indexNode.right);
            }
            if (indexNode.left != null) {
                stack.push(indexNode.left);
            }
        }
    }

    /*
     * 借助栈实现迭代的后序遍历，返回的排序的数列
     * */
        private void itePostOderTraverse(Node node) {
            if (node == null) {
                return;
            }
            Stack<Node> stack = new Stack<>();
            Node prev = null;
            while (node != null || !stack.isEmpty()) {
                while (node != null) {
                    stack.push(node);
                    if (node.left!=null){
                        node = node.left;
                    }else {
                        node = node.right;
                    }
                }
                node = stack.pop();
                if (node.right == null || node.right == prev) {
                    System.out.println(node.value);
                    prev = node;
                    node = null;
                } else {
                    stack.push(node);
                    node = node.right;
                }
            }
        }

    public void bfsTraverse() {
        bfs(this.root);
    }

    /*
     * 借助队列实现广度优先遍历-层序遍历
     * */
    private void bfs(Node node) {
        if (node == null) {
            return;
        }
        Queue<Node> queue = new LinkedList();
        queue.add(node);
        while (!queue.isEmpty()) {
            Node indexNode = queue.poll();
            System.out.println(indexNode.value);
            if (indexNode.left != null) {
                queue.add(indexNode.left);
            }
            if (indexNode.right != null) {
                queue.add(indexNode.right);
            }
        }
    }
}
```