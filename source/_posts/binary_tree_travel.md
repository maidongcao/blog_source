title: 二叉树的遍历
tags: [数据结构与算法]
categories:
  - 数据结构与算法
date: 2020-06-12 8:32:56
---
<script src="/js/mermaid.full.min.js"></script>
二叉树的遍历题型:
1. [二叉树的前序遍历](https://leetcode-cn.com/problems/binary-tree-preorder-traversal)
2. [二叉树的中序遍历](https://leetcode-cn.com/problems/binary-tree-inorder-traversal)
3. [二叉树的后序遍历](https://leetcode-cn.com/problems/binary-tree-postorder-traversal)
4.  [二叉树的层次遍历](https://leetcode-cn.com/problems/binary-tree-zigzag-level-order-traversal/)
* [二叉树的前序遍历](https://leetcode-cn.com/problems/binary-tree-preorder-traversal)
    前序遍历的遍历顺序: 遍历根节点->左子树->右子树。
    遍历过程时间时遍历左右子树，根节点位置为读取数据。
    **最左遍历法:** 前序遍历左子树在前，所以遍历是先遍历左子树，由于根节点在左子树前，所以遍历左子树的同时，需要获取节点；直到遍历到左子树的叶节点，再遍历右子树【右子树的遍历也是先遍历完右子树的左子树】。
    遍历过程如图所示。
![二叉树的前序遍历](https://github.com/hezhuhui/gallery/raw/master/leetcode/preorder.jpg)
大致过程为:
   1. 从根节点一直入栈，压到最左侧叶节点，节点压入栈的过程同时获取节点到ans数组。
   2. 到左侧叶子节点后，切换右子树，再重复步骤1。
   3. 直到压到右子树最后一个阶段。
   4. 最后输出的ans即为前序遍历的结果。
    代码如下:
    
```
 public List<Integer> preorderTraversal(TreeNode root) {
        if(root == null){
            return new ArrayList<>();
        }
        Stack<TreeNode> stack = new Stack<>();
        LinkedList<Integer> ans = new LinkedList<>();
        while(!stack.isEmpty() || root != null){
            //左子树一直入栈，压到左叶子节点为止
            while(root != null){
                stack.push(root); //节点入栈
                ans.add(root.val); //节点写入ans
                root = root.left; // 切换到左子树
            }
            //到达左叶子节点，切换到右子树
            root = stack.pop();
            root = root.right;
        }
        return ans;
    }


```
* [二叉树的中序遍历](https://leetcode-cn.com/problems/binary-tree-inorder-traversal)
中序遍历的顺序为:左子树->根节点->右子树，二叉树搜索树根据使用中序遍历，可以从小到大输出有序数组。
 中序遍历遍历的数据的方式[最左遍历法]可以与前序遍历的方式一致，但是由于根节点输出位置在左子树之后。所有输出的根节点需要在出栈的过程。即左子树的左叶子最先直接输出。
 遍历和输出过程如图所示:
 
![二叉树的中序遍历](
https://github.com/hezhuhui/gallery/raw/master/leetcode/inorder.jpg)
 注意遍历方式与前序遍历一致，但是输出节点到ans的时机变为出栈时。
 ````
  public List<Integer> inorderTraversal(TreeNode root) {
         if(root == null){
            return new ArrayList<>();
        }
        Stack<TreeNode> stack = new Stack<>();
        List<Integer> ans = new ArrayList<>();
        while(!stack.isEmpty() || root != null){
            //先将最左压倒底
            while(root != null){
                stack.push(root);
                root = root.left;
            }
            //切换到右子树
            root = stack.pop();
            ans.add(root.val); //输出答案时机为出栈时
            root = root.right;
        }
        return ans;
    }
````
* [二叉树的后序遍历](https://leetcode-cn.com/problems/binary-tree-postorder-traversal)
后序遍历的遍历顺序为左子树->右子树->根节点，输出结果刚好与前序遍历相反。
 **最右遍历法:** 根节点需要在右子树之后，也就是可以通过先将右子树的右侧全部入栈，同时逆序排列，即可得到结果。
遍历过程如图所示。
![二叉树的后序遍历](
https://github.com/hezhuhui/gallery/raw/master/leetcode/postorder.jpg)
图中，先遍历右子树，且遍历过程中将节点输入到数组中，但是最后的结果需要逆序输出。也就是输出的结果为 4 5 2 1 6 2 1[序号的逆序输出] 
实现代码如下:
```
 public List<Integer> postorderTraversal(TreeNode root) {
        if(root == null){
            return new ArrayList<>();
        }
        Stack<TreeNode> stack = new Stack<>();
        LinkedList<Integer> ans = new LinkedList<>();
        while(!stack.isEmpty() || root != null){
            //遍历右子树，同时逆序排到输出结果中。
            while(root != null){
                stack.push(root);
                ans.addFirst(root.val);
                root = root.right;
            }
            //最右侧节点为叶子节点后，切到左子树
            root = stack.pop();
            root = root.left;
        }
        return ans;
    }
```
综上: 二叉树的前中后遍历可以总结为两种方法，最左和最右遍历法。其中中序遍历同时适用最左和最右遍历法。
所以前中后遍历二叉树关键是定义好两点:
1、使用的遍历的方法:最左/最右。
2、输出节点的时机。 
3、输出的结果是否需要逆序。
通用代码模板如下:
```
 public List<Integer> inorderTraversal(TreeNode root) {
         //base case
         if(root == null){
            return new ArrayList<>();
        }
        Stack<TreeNode> stack = new Stack<>();
        List<Integer> ans = new ArrayList<>();
        while(!stack.isEmpty() || root != null){
            //先将最左/最右压到叶子节点
            while(root != null){
                stack.push(root);
                //输出时机
                //如果是前序遍历使用最左输出，根节点位于左子树之前，所示先输出节点
                //如果中序遍历适用最左遍历，由于根节点位于左子树之后所以不能输出结果
                //如果后序遍历使用最右遍历，根节点位于右子树之后【理应先输出根节点】，由于逆序输出，变成了根节点可以先与右子树输出，所以此处需要输出节点
                //最左/最右输出，
                //如果是最左输出，需要节点一直为左子树根节点 root = root.left;
                //如果是最右输出，需要节点一直为右子树根节点 root = root.right;
            }
            //切换到右子树
            root = stack.pop();
            //输出时机
            //根节点位于最左/最右之后，在此输出。因为可以形成从叶节点开始输出结果，得到先输出最左/最右的效果
            // ans.add(root.val); //输出答案时机为出栈时
           //切换时机
           //如果是最左，在此切换为右子树；如果是最右，则刚好相反
           root = root.right;
        }
        return ans;
    }
```
* 层次遍历 [102](https://leetcode-cn.com/problems/binary-tree-level-order-traversal/submissions/) [103](https://leetcode-cn.com/problems/binary-tree-zigzag-level-order-traversal/) [429](https://leetcode-cn.com/problems/n-ary-tree-level-order-traversal/)
层次遍历与前中后序遍历不一样，本质上来说层次遍历是一种bfs，前中后遍历是一种dfs。dfs二叉树是往往是后进先出，即出栈一般是叶节点先出栈，再到非叶子节点直到树的根节点。然而bfs不同的是遍历需要从根节点开始，输出也是从根节点开始，所以是先进先出，所以一般使用队列来实现bfs。
二叉树的层次遍历代码模板:
```
public List<List<Integer>> levelOrder(TreeNode root) {
        List<List<Integer>> ans = new ArrayList<>();
        if(root == null){
            return ans;
        }
        Queue<TreeNode> queue = new LinkedList<>();
        queue.offer(root);
        while(!queue.isEmpty()){
            //遍历一层的元素
            List<Integer> list = new ArrayList<>();
            int len = queue.size();
            //按层输出
            for(int i = 0; i < len; i++){
                   // 每一层元素的出队队和入队的方向决定了层次遍历的规则
                   // 如果需要按照Z输出，需要添加判断条件，使得每一层入队和出队在队列的方向是相反的。
                   //同时注意，入队处不能作为出队处，否则会导致下一层的数据提前出队。
                   // 也就是入队和出队在同一处，会导致刚入队的数据[理应是下一层的数据]，就会立马出队，违反了先进先出原则。
                TreeNode node = queue.poll();
                if(node.left != null){
                    queue.offer(node.left);
                }
                if(node.right != null){
                    queue.offer(node.right);
                }
                 list.add(node.val);
            }
            ans.add(list);
        }
        return ans;
    }
```
