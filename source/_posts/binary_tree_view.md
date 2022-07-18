title: 二叉树视图/节点求和问题
tags: [数据结构与算法]
categories:
  - 数据结构与算法
date: 2020-05-01 10:14:56
---
<script src="/js/mermaid.full.min.js"></script>
## 问题分类
* 左视图/右视图节点问题:求和/列出节点值
* 左节点/右叶子节点问题:求和/列出节点值
* 俯视图
## 求解技巧
1. 左/右视图问题（以右视图为例：[二叉树的右视图](https://leetcode-cn.com/problems/binary-tree-right-side-view/)）
* 问题描述: 以右视图为例，想象自己站在它的右侧，按照从顶部到底部的顺序，返回从右侧所能看到的节点值。
* 分析：
<u>本质求二叉树每一层的最右边的节点</u>，可以使用bfs对二叉树进行从左到右层次遍历，取每一层最右边的节点加入列表。
* 代码实现
 通过层次遍历获取每一层数据的通用模板
```
  public T readTree(TreeNode root) {
        // base case
        List<Integer> ans = new ArrayList<>();
        if(root == null){
            return T;
        }
        // 定义bfs使用的队列
        Queue<TreeNode> queue = new LinkedList<>();
        // 先加入根节点
        queue.offer(root);
        while (!queue.isEmpty()){
            // 求每一层节点的个数
            int size = queue.size();
            // 遍历每一层的节点
            for(int i = 0; i < size; i++){
                TreeNode tmp = queue.poll();
                /**
                * 此处实现每一层的数据操作
                * 以右视图为例，此处可添加代码
                *  
                    // 每一层最右边的节点加入结果集合
                   if(i == size - 1){
                    ans.add(tmp.val);
                }
                **/
                //添加下一层数据到队列，从左往右添加
                if(tmp.left != null){
                    queue.offer(tmp.left);
                }
                if(tmp.right != null){
                    queue.offer(tmp.right);
                }
            }
        }
        return T;
    }

```

2. 左节点/右叶子节点问题：[左叶子之和](https://leetcode-cn.com/problems/sum-of-left-leaves/)
* 问题描述: 以求左叶子为例，题目要求获取所有左子树的叶子节点之和，不包含非叶子节点。
* 分析: 注意与左视图问题区别：问题可以转化为求所有叶子节点中，左叶子之和。如果使用bfs，由于是层次遍历会导致左右子树的属性丢失，所有使用dfs遍历左子树和右子树，同时标记当前遍历处于哪个子树（通过参数标识）。
* 代码实现
```

class Solution {
    int sum = 0;
    public int sumOfLeftLeaves(TreeNode root) {
        if(root == null) {
            return 0;
        }
        if(root.left != null){
            dfs(root.left,true);
        }
        if(root.right != null){
            dfs(root.right,false);
        }
        return sum;
    }
    void dfs(TreeNode root,Boolean isLeft){
        // 该节点为叶子节点并且是左节点
        if(root.left == null && root.right == null && isLeft){
            sum += root.val;
        }
        if(root.left != null){
            dfs(root.left,true);
        }
        if(root.right != null){
            dfs(root.right,false);
        }
    }
}
```

3. 俯视图
* 问题描述:想象自身从二叉树的上方俯视二叉树，以根节点为界，列出根节点右侧的节点。
* 分析:
   1. 从俯视图右侧的看到的节点，不一定为右子树节点，因为如果根节点的右子树为null，但是左子树一直往右偏移，可能越过根节点的正下方，从而出现在俯视图的根节点右侧。
   2. 每一层的最右节点不一定为俯视图看到节点，因为如果上层的最右节点挡住了当前层的节点也就看不到当前节点了。
   3. 根据以上分析，关键点：**记录每一个节点相对于root节点的偏移量a，再根据偏移量a与当前结果数组大小比较，判断当前节点偏移量是否大于数组大小，如果大于表示当前节点偏移量不会被遮挡，否则会被遮挡。**
   4. 由于偏移量a的获取，需要根据上一层偏移量的大小确定，也就是**依赖纵向关系，所以使用dfs不能使用bfs**。
  
 * 代码实现
 ```
class Solution {
    List<Integer> ans = new ArrayList<>();
    public List<Integer> rightSideView(TreeNode root) {
        if(root == null){
            return new ArrayList<>();
        }
        ans.add(root.val);
        if(root.left != null){
            dfs(root.left,  -1);
        }
        if(root.right != null ){
            dfs(root.right,  1);
        }
        return ans;
    }
    //index为偏移量，向左偏移量-1，向右+1
    void dfs(TreeNode root, int index){
    // 当节点不为null，且不阻挡时，加入结果集合
        if(root != null && index >= ans.size()){
            ans.add(root.val);
        }
        if(root.left != null){
            dfs(root.left, index - 1);
        }
        if(root.right != null ){
            dfs(root.right, index + 1);
        }
    }
}
```
## 总结
1.bfs和dfs遍历二叉树的使用场景:
* **如果当前节点的属性依赖上一节点传递，则使用dfs**。如需要依赖上一节点来确定当前节点是左节点/右节点或者节点相对根节点偏移量。
* **如果当前节点的属性依赖兄弟节点传递，则使用bfs**。如需要确定当前节点位于该层的位置。
