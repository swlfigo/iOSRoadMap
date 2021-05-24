# 二叉树排序

二叉树的前中后序遍历的非递归实现



前，中，后只是指父节点遍历的顺序，前序就是 父节点->左子树->右子树，中序是 左子树->父节点->右子树，后序是 左子树 -> 右子树 ->父节点



## 二叉树定义

```python
class TreeNode:
    def __init__(self, x, L=None, R=None):
        self.val = x
        self.left = L
        self.right = R

def List2TN(lst, needs=None):
    '''
    lst: a leetcode way tree list
    needs: A list of Int. The nodes whose indexes provided in this list would be returned.
    '''
    nit = []
    root = TreeNode(lst[0])
    tnQ = [root]
    i = 1
    if needs and i in needs:
        nit.append(root)
    while i < len(lst):
        cur = tnQ.pop(0)
        if lst[i] != None:
            cur.left = TreeNode(lst[i])
            tnQ.append(cur.left)
            if needs and i in needs:
                nit.append(cur.left)
        i += 1
        if i >= len(lst):
            break
        if lst[i] != None:
            cur.right = TreeNode(lst[i])
            tnQ.append(cur.right)
            if needs and i in needs:
                nit.append(cur.right)
        i += 1
    if needs:
        return root, nit
    else:
        return root
```

## 前序遍历

- [144. 二叉树的前序遍历](https://leetcode.com/problems/binary-tree-preorder-traversal/submissions/)

#### 具体过程

1、首先申请一个新的栈，记为stack。
2、然后将头节点head压入stack中。
3、每次从stack中弹出栈顶节点，记为cur，然后打印cur节点的值。如果cur右孩子不为空的话，将cur的右孩子先压入stack中。最后如果cur的左孩子不为空的话，将cur的左孩子压入stack中。
4、不断重复步骤3，直到stack为空，全部过程结束。

#### 代码实现

```python
class Solution(object):
    def preorderTraversal(self,root):
        """
        根->左->右
        :type root: TreeNode
        :rtype: List[int]
        """
        stack = [root]
        res = []
        while stack:
            cur = stack.pop()
            res.append(cur.val)
            if cur.right:
                stack.append(cur.right)
            if cur.left:
                stack.append(cur.left)
        return res
```

## 中序遍历

- [94. 二叉树的中序遍历](https://leetcode.com/problems/binary-tree-inorder-traversal/)

#### 具体过程

- 1、申请一个新的栈，记为`stack`，申请一个变量`cur`，初始时令`stack`为空,`cur`等于头节点。
- 2、先把`cur`节点压入栈中，对以cur节点为头的整棵子树来说，依次把整棵树的左边界压入栈中，即不断令`cur=cur.left`，然后重复步骤2。
- 3、不断重复步骤2，直到发现`cur`为空，此时从`stack`中弹出一个节点，记为`node`。打印`node`的值，并让`cur=node.right`，然后继续重复步骤2。
- 4、当`stack`为空并且`cur`为空时，整个过程结束。

#### 代码实现

```python
class Solution(object):
    def inorderTraversal(self, root):
        """
        左->根->右
        :type root: TreeNode
        :rtype: List[int]
        """
        stack = []
        cur = root
        res = []
        while stack or cur:
            if cur:
                stack.append(cur)
                cur = cur.left
            else:
                node = stack.pop()
                res.append(node.val)
                cur = node.right
        return res
```

## 后序遍历

- [145. 二叉树的后序遍历](https://leetcode-cn.com/problems/binary-tree-postorder-traversal/)

#### 具体过程

**方法一：使用两个栈实现**

1、申请一个栈，记为s1，然后将头节点压入s1中。
2、从s1中弹出的节点记为cur，然后先把cur的左孩子压入s1中，然后把cur的右孩子压入s1中。
3、在整个过程中，每一个从s1中弹出的节点都放进第二个栈s2中。
4、不断重复步骤2和步骤3，直到s1为空，过程停止。
5、从s2中依次弹出节点并打印，打印的顺序就是后序遍历的顺序了。

**方法二：使用一个栈实现**

1、申请一个栈，记为stack，将头节点压入stack，同时设置两个变量h和c。在整个流程中，h代表最近一次弹出并打印的节点，c代表当前stack的栈顶节点，初始时令h为头节点，c为null。
2、每次令c等于当前stack的栈顶节点，但是不从stack中弹出节点，此时分以下三种情况。
（1）如果c的左孩子不为空，并且h不等于c的左孩子，也不等于c的右孩子，则把c的左孩子压入stack中。
（2）如果情况1不成立，并且c的右孩子不为空，并且h不等于c的右孩子，则把c的右孩子压入stack中。
（3）如果情况1和情况2都不成立，那么从stack中弹出c并打印，然后令h等于c。
3、一直重复步骤2，直到stack为空，过程停止。

#### 代码实现

方法一:

```python
class Solution(object):
    def postorderTraversal(self, root):
        """
        左->右->根
        :type root: TreeNode
        :rtype: List[int]
        """
        stack1 = [root]
        stack2 = []
        while stack1:
            cur = stack1.pop()
            stack2.append(cur.val)
            if cur.left:
                stack1.append(cur.left)
            if cur.right:
                stack1.append(cur.right)
        return stack2[::-1]
```



方法二:

```python
class Solution:
    def postorderTraversal(self, root: TreeNode) -> List[int]:
        if not root:
            return list()
        
        res = list()
        stack = list()
        prev = None

        while root or stack:
            while root:
                stack.append(root)
                root = root.left
            root = stack.pop()
            if not root.right or root.right == prev:
                res.append(root.val)
                prev = root
                root = None
            else:
                stack.append(root)
                root = root.right
        
        return res
```





## 层次遍历

#### bfs

```python
def bfs(root):
    queue = []
    # 根节点加入队列中
    queue.append(root)
    res = []
    while queue:
        temp = queue.pop(0)
        l = temp.left
        r = temp.right
        if l:
            queue.append(l)
        if r:
            queue.append(r)
        res.append(temp.val)
    return res
```





## Reference

http://www.ichenfei.com/2019/05/02/%E4%BA%8C%E5%8F%89%E6%A0%91%E7%9A%84%E5%89%8D%E4%B8%AD%E5%90%8E%E5%BA%8F%E9%81%8D%E5%8E%86%E7%9A%84%E9%9D%9E%E9%80%92%E5%BD%92%E5%AE%9E%E7%8E%B0(Python)/