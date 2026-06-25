
============================================================
 Python二叉树与遍历 — 节点定义/遍历方式/平衡检查/BST验证
============================================================

二叉树是每个节点最多有两个子节点的树结构，是递归数据结构的经典代表。
Python中常用类来实现二叉树节点，遍历方式分为DFS（深度优先）和BFS（广度优先）。

============================================================
1. 二叉树节点定义
============================================================

from collections import deque

class TreeNode:
    """二叉树节点类"""
    def __init__(self, val=0, left=None, right=None):
        self.val = val          # 节点值
        self.left = left        # 左子节点
        self.right = right      # 右子节点

class BinarySearchTree:
    """二叉搜索树类"""
    def __init__(self):
        self.root = None

    def insert(self, val):
        """向BST中插入值（递归实现）"""
        def _insert(node, val):
            if not node:
                return TreeNode(val)
            if val < node.val:              # 小于当前值，插入左子树
                node.left = _insert(node.left, val)
            else:                           # 大于等于当前值，插入右子树
                node.right = _insert(node.right, val)
            return node
        self.root = _insert(self.root, val)

    def search(self, val):
        """在BST中查找值，返回节点或None"""
        def _search(node, val):
            if not node or node.val == val:
                return node
            if val < node.val:
                return _search(node.left, val)
            return _search(node.right, val)
        return _search(self.root, val)

============================================================
2. 深度优先遍历 DFS（递归实现）
============================================================

def inorder_recursive(root):
    """中序遍历：左 -> 根 -> 右（BST下输出为升序序列）"""
    if not root:
        return []
    return inorder_recursive(root.left) + [root.val] + inorder_recursive(root.right)

def preorder_recursive(root):
    """前序遍历：根 -> 左 -> 右（用于树的结构复制）"""
    if not root:
        return []
    return [root.val] + preorder_recursive(root.left) + preorder_recursive(root.right)

def postorder_recursive(root):
    """后序遍历：左 -> 右 -> 根（用于树的删除操作）"""
    if not root:
        return []
    return postorder_recursive(root.left) + postorder_recursive(root.right) + [root.val]

============================================================
3. 深度优先遍历（迭代实现，用栈模拟递归）
============================================================

def inorder_iterative(root):
    """中序遍历迭代版本，使用显式栈"""
    res, stack = [], []
    curr = root
    while curr or stack:
        while curr:              # 一直往左走，将沿途节点入栈
            stack.append(curr)
            curr = curr.left
        curr = stack.pop()       # 弹出最左节点
        res.append(curr.val)     # 访问节点
        curr = curr.right        # 转向右子树
    return res

def preorder_iterative(root):
    """前序遍历迭代版本"""
    if not root:
        return []
    res, stack = [], [root]
    while stack:
        node = stack.pop()
        res.append(node.val)
        if node.right:           # 右子节点先入栈（后出）
            stack.append(node.right)
        if node.left:            # 左子节点后入栈（先出）
            stack.append(node.left)
    return res

def postorder_iterative(root):
    """后序遍历迭代版本（前序反转法）"""
    if not root:
        return []
    res, stack = [], [root]
    while stack:
        node = stack.pop()
        res.append(node.val)
        if node.left:            # 与前序相反，左子节点先入栈
            stack.append(node.left)
        if node.right:
            stack.append(node.right)
    return res[::-1]             # 反转得到后序

============================================================
4. 层序遍历（BFS广度优先）
============================================================

def level_order(root):
    """层序遍历，按层返回二维列表[[第0层], [第1层], ...]"""
    if not root:
        return []
    res = []
    queue = deque([root])        # 使用双端队列作为队列
    while queue:
        level_size = len(queue)  # 当前层的节点数
        level = []
        for _ in range(level_size):
            node = queue.popleft()
            level.append(node.val)
            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)
        res.append(level)
    return res

============================================================
5. 树的高度计算
============================================================

def tree_height(root):
    """递归计算二叉树的高度（根节点高度为1）"""
    if not root:
        return 0
    return 1 + max(tree_height(root.left), tree_height(root.right))

============================================================
6. 平衡二叉树检查
============================================================

def is_balanced(root):
    """检查二叉树是否平衡（左右子树高度差不超过1）"""
    def _check(node):
        """返回 (是否平衡, 高度)"""
        if not node:
            return True, 0
        left_balanced, left_h = _check(node.left)
        right_balanced, right_h = _check(node.right)
        balanced = left_balanced and right_balanced and abs(left_h - right_h) <= 1
        return balanced, 1 + max(left_h, right_h)
    return _check(root)[0]

============================================================
7. 验证二叉搜索树（BST Validation）
============================================================

def is_valid_bst(root):
    """验证一棵树是否为有效的BST（中序遍历法）"""
    def _inorder(node):
        if not node:
            return []
        return _inorder(node.left) + [node.val] + _inorder(node.right)
    vals = _inorder(root)
    # BST的中序遍历结果应该是严格递增的
    return all(vals[i] < vals[i + 1] for i in range(len(vals) - 1))

def is_valid_bst_range(root):
    """验证BST的另一种方法：递归传递上下界"""
    def _validate(node, low, high):
        if not node:
            return True
        if node.val <= low or node.val >= high:  # 超出范围则非法
            return False
        return (_validate(node.left, low, node.val) and
                _validate(node.right, node.val, high))
    return _validate(root, float('-inf'), float('inf'))

============================================================
使用示例
============================================================
if __name__ == "__main__":
    bst = BinarySearchTree()
    for v in [5, 3, 7, 2, 4, 6, 8]:
        bst.insert(v)
    print("中序遍历(递归):", inorder_recursive(bst.root))      # [2,3,4,5,6,7,8]
    print("中序遍历(迭代):", inorder_iterative(bst.root))      # [2,3,4,5,6,7,8]
    print("前序遍历:", preorder_recursive(bst.root))            # [5,3,2,4,7,6,8]
    print("后序遍历:", postorder_recursive(bst.root))           # [2,4,3,6,8,7,5]
    print("层序遍历:", level_order(bst.root))                   # [[5],[3,7],[2,4,6,8]]
    print("树高度:", tree_height(bst.root))                     # 3
    print("是否平衡:", is_balanced(bst.root))                   # True
    print("是否有效BST:", is_valid_bst(bst.root))              # True

doi.kumchen.cn/85768.Doc
doi.kumchen.cn/45246.Doc
doi.kumchen.cn/19777.Doc
doi.kumchen.cn/10767.Doc
doi.kumchen.cn/59091.Doc
doi.kumchen.cn/23793.Doc
doi.kumchen.cn/55818.Doc
doi.kumchen.cn/52404.Doc
doi.kumchen.cn/24528.Doc
doi.kumchen.cn/48792.Doc
dou.kumchen.cn/52614.Doc
dou.kumchen.cn/68031.Doc
dou.kumchen.cn/06992.Doc
dou.kumchen.cn/54326.Doc
dou.kumchen.cn/95325.Doc
dou.kumchen.cn/73479.Doc
dou.kumchen.cn/94532.Doc
dou.kumchen.cn/31676.Doc
dou.kumchen.cn/35675.Doc
dou.kumchen.cn/23470.Doc
doy.kumchen.cn/18342.Doc
doy.kumchen.cn/61256.Doc
doy.kumchen.cn/30966.Doc
doy.kumchen.cn/87644.Doc
doy.kumchen.cn/43233.Doc
doy.kumchen.cn/52283.Doc
doy.kumchen.cn/56004.Doc
doy.kumchen.cn/14382.Doc
doy.kumchen.cn/57082.Doc
doy.kumchen.cn/00594.Doc
dot.kumchen.cn/53428.Doc
dot.kumchen.cn/18953.Doc
dot.kumchen.cn/80493.Doc
dot.kumchen.cn/61418.Doc
dot.kumchen.cn/88004.Doc
dot.kumchen.cn/27172.Doc
dot.kumchen.cn/59929.Doc
dot.kumchen.cn/46829.Doc
dot.kumchen.cn/70904.Doc
dot.kumchen.cn/30815.Doc
dor.kumchen.cn/08146.Doc
dor.kumchen.cn/68063.Doc
dor.kumchen.cn/93203.Doc
dor.kumchen.cn/30135.Doc
dor.kumchen.cn/68102.Doc
dor.kumchen.cn/07498.Doc
dor.kumchen.cn/26965.Doc
dor.kumchen.cn/44024.Doc
dor.kumchen.cn/18979.Doc
dor.kumchen.cn/39345.Doc
doe.kumchen.cn/73366.Doc
doe.kumchen.cn/29259.Doc
doe.kumchen.cn/64592.Doc
doe.kumchen.cn/33240.Doc
doe.kumchen.cn/01570.Doc
doe.kumchen.cn/09457.Doc
doe.kumchen.cn/88938.Doc
doe.kumchen.cn/34922.Doc
doe.kumchen.cn/23378.Doc
doe.kumchen.cn/14400.Doc
dow.kumchen.cn/55770.Doc
dow.kumchen.cn/39868.Doc
dow.kumchen.cn/43678.Doc
dow.kumchen.cn/37635.Doc
dow.kumchen.cn/31556.Doc
dow.kumchen.cn/97105.Doc
dow.kumchen.cn/64435.Doc
dow.kumchen.cn/66205.Doc
dow.kumchen.cn/23027.Doc
dow.kumchen.cn/80487.Doc
doq.kumchen.cn/80357.Doc
doq.kumchen.cn/31267.Doc
doq.kumchen.cn/99240.Doc
doq.kumchen.cn/86031.Doc
doq.kumchen.cn/22332.Doc
doq.kumchen.cn/27124.Doc
doq.kumchen.cn/89896.Doc
doq.kumchen.cn/76513.Doc
doq.kumchen.cn/13135.Doc
doq.kumchen.cn/42044.Doc
dim.kumchen.cn/48715.Doc
dim.kumchen.cn/48722.Doc
dim.kumchen.cn/51127.Doc
dim.kumchen.cn/24503.Doc
dim.kumchen.cn/51020.Doc
dim.kumchen.cn/78797.Doc
dim.kumchen.cn/61614.Doc
dim.kumchen.cn/06291.Doc
dim.kumchen.cn/59363.Doc
dim.kumchen.cn/52663.Doc
din.kumchen.cn/19799.Doc
din.kumchen.cn/23708.Doc
din.kumchen.cn/21808.Doc
din.kumchen.cn/51935.Doc
din.kumchen.cn/86488.Doc
din.kumchen.cn/96119.Doc
din.kumchen.cn/34578.Doc
din.kumchen.cn/66057.Doc
din.kumchen.cn/28175.Doc
din.kumchen.cn/57136.Doc
dib.kumchen.cn/66365.Doc
dib.kumchen.cn/86022.Doc
dib.kumchen.cn/29112.Doc
dib.kumchen.cn/73545.Doc
dib.kumchen.cn/80841.Doc
dib.kumchen.cn/79213.Doc
dib.kumchen.cn/08682.Doc
dib.kumchen.cn/80222.Doc
dib.kumchen.cn/86787.Doc
dib.kumchen.cn/90677.Doc
div.kumchen.cn/45567.Doc
div.kumchen.cn/88810.Doc
div.kumchen.cn/57013.Doc
div.kumchen.cn/60735.Doc
div.kumchen.cn/10891.Doc
div.kumchen.cn/34346.Doc
div.kumchen.cn/93260.Doc
div.kumchen.cn/00666.Doc
div.kumchen.cn/65250.Doc
div.kumchen.cn/21630.Doc
dic.kumchen.cn/23366.Doc
dic.kumchen.cn/94419.Doc
dic.kumchen.cn/48581.Doc
dic.kumchen.cn/51278.Doc
dic.kumchen.cn/24257.Doc
dic.kumchen.cn/71017.Doc
dic.kumchen.cn/46568.Doc
dic.kumchen.cn/54888.Doc
dic.kumchen.cn/17025.Doc
dic.kumchen.cn/92859.Doc
dix.kumchen.cn/08778.Doc
dix.kumchen.cn/81378.Doc
dix.kumchen.cn/27218.Doc
dix.kumchen.cn/42100.Doc
dix.kumchen.cn/48701.Doc
dix.kumchen.cn/96494.Doc
dix.kumchen.cn/49955.Doc
dix.kumchen.cn/99577.Doc
dix.kumchen.cn/42090.Doc
dix.kumchen.cn/72464.Doc
diz.kumchen.cn/64965.Doc
diz.kumchen.cn/17245.Doc
diz.kumchen.cn/25627.Doc
diz.kumchen.cn/54639.Doc
diz.kumchen.cn/33674.Doc
diz.kumchen.cn/86280.Doc
diz.kumchen.cn/06624.Doc
diz.kumchen.cn/65160.Doc
diz.kumchen.cn/16659.Doc
diz.kumchen.cn/59247.Doc
dil.kumchen.cn/33686.Doc
dil.kumchen.cn/10175.Doc
dil.kumchen.cn/88000.Doc
dil.kumchen.cn/67789.Doc
dil.kumchen.cn/70158.Doc
dil.kumchen.cn/40511.Doc
dil.kumchen.cn/86991.Doc
dil.kumchen.cn/76910.Doc
dil.kumchen.cn/71246.Doc
dil.kumchen.cn/00714.Doc
dik.kumchen.cn/32380.Doc
dik.kumchen.cn/85113.Doc
dik.kumchen.cn/52681.Doc
dik.kumchen.cn/94471.Doc
dik.kumchen.cn/71401.Doc
dik.kumchen.cn/87510.Doc
dik.kumchen.cn/06601.Doc
dik.kumchen.cn/33592.Doc
dik.kumchen.cn/32528.Doc
dik.kumchen.cn/10995.Doc
dij.kumchen.cn/02213.Doc
dij.kumchen.cn/86998.Doc
dij.kumchen.cn/79907.Doc
dij.kumchen.cn/93851.Doc
dij.kumchen.cn/39413.Doc
dij.kumchen.cn/06705.Doc
dij.kumchen.cn/82561.Doc
dij.kumchen.cn/88595.Doc
dij.kumchen.cn/28900.Doc
dij.kumchen.cn/82094.Doc
dih.kumchen.cn/08068.Doc
dih.kumchen.cn/07092.Doc
dih.kumchen.cn/69598.Doc
dih.kumchen.cn/43611.Doc
dih.kumchen.cn/23970.Doc
dih.kumchen.cn/64834.Doc
dih.kumchen.cn/91498.Doc
dih.kumchen.cn/00611.Doc
dih.kumchen.cn/27135.Doc
dih.kumchen.cn/42601.Doc
dig.kumchen.cn/29936.Doc
dig.kumchen.cn/75845.Doc
dig.kumchen.cn/90926.Doc
dig.kumchen.cn/99136.Doc
dig.kumchen.cn/87137.Doc
dig.kumchen.cn/11484.Doc
dig.kumchen.cn/37811.Doc
dig.kumchen.cn/26345.Doc
dig.kumchen.cn/73999.Doc
dig.kumchen.cn/34843.Doc
dif.kumchen.cn/69453.Doc
dif.kumchen.cn/50018.Doc
dif.kumchen.cn/00879.Doc
dif.kumchen.cn/08454.Doc
dif.kumchen.cn/85909.Doc
dif.kumchen.cn/69047.Doc
dif.kumchen.cn/75175.Doc
dif.kumchen.cn/38008.Doc
dif.kumchen.cn/24636.Doc
dif.kumchen.cn/03621.Doc
did.kumchen.cn/37528.Doc
did.kumchen.cn/29201.Doc
did.kumchen.cn/64892.Doc
did.kumchen.cn/94033.Doc
did.kumchen.cn/72887.Doc
did.kumchen.cn/37787.Doc
did.kumchen.cn/27906.Doc
did.kumchen.cn/94015.Doc
did.kumchen.cn/98910.Doc
did.kumchen.cn/64814.Doc
dis.kumchen.cn/63402.Doc
dis.kumchen.cn/66787.Doc
dis.kumchen.cn/90859.Doc
dis.kumchen.cn/66229.Doc
dis.kumchen.cn/16487.Doc
dis.kumchen.cn/98569.Doc
dis.kumchen.cn/81815.Doc
dis.kumchen.cn/20241.Doc
dis.kumchen.cn/06422.Doc
dis.kumchen.cn/92998.Doc
dia.kumchen.cn/29284.Doc
dia.kumchen.cn/35770.Doc
dia.kumchen.cn/36228.Doc
dia.kumchen.cn/26855.Doc
dia.kumchen.cn/32344.Doc
dia.kumchen.cn/29403.Doc
dia.kumchen.cn/24634.Doc
dia.kumchen.cn/04934.Doc
dia.kumchen.cn/50357.Doc
dia.kumchen.cn/55138.Doc
dip.kumchen.cn/96176.Doc
dip.kumchen.cn/43604.Doc
dip.kumchen.cn/66081.Doc
dip.kumchen.cn/89062.Doc
dip.kumchen.cn/63635.Doc
dip.kumchen.cn/21577.Doc
dip.kumchen.cn/57770.Doc
dip.kumchen.cn/62523.Doc
dip.kumchen.cn/09452.Doc
dip.kumchen.cn/00609.Doc
dio.kumchen.cn/26410.Doc
dio.kumchen.cn/17200.Doc
dio.kumchen.cn/88634.Doc
dio.kumchen.cn/00279.Doc
dio.kumchen.cn/72387.Doc
dio.kumchen.cn/85973.Doc
dio.kumchen.cn/06572.Doc
dio.kumchen.cn/51590.Doc
dio.kumchen.cn/33269.Doc
dio.kumchen.cn/31502.Doc
dii.kumchen.cn/23547.Doc
dii.kumchen.cn/08024.Doc
dii.kumchen.cn/79034.Doc
dii.kumchen.cn/61258.Doc
dii.kumchen.cn/60146.Doc
dii.kumchen.cn/78760.Doc
dii.kumchen.cn/84604.Doc
dii.kumchen.cn/10244.Doc
dii.kumchen.cn/69105.Doc
dii.kumchen.cn/82044.Doc
diu.kumchen.cn/00839.Doc
diu.kumchen.cn/27242.Doc
diu.kumchen.cn/50196.Doc
diu.kumchen.cn/63404.Doc
diu.kumchen.cn/25998.Doc
diu.kumchen.cn/06298.Doc
diu.kumchen.cn/35730.Doc
diu.kumchen.cn/04382.Doc
diu.kumchen.cn/99452.Doc
diu.kumchen.cn/24724.Doc
diy.kumchen.cn/83487.Doc
diy.kumchen.cn/03708.Doc
diy.kumchen.cn/63162.Doc
diy.kumchen.cn/73502.Doc
diy.kumchen.cn/77301.Doc
diy.kumchen.cn/99665.Doc
diy.kumchen.cn/23360.Doc
diy.kumchen.cn/69420.Doc
diy.kumchen.cn/50706.Doc
diy.kumchen.cn/93980.Doc
dit.kumchen.cn/42749.Doc
dit.kumchen.cn/36159.Doc
dit.kumchen.cn/99246.Doc
dit.kumchen.cn/70389.Doc
dit.kumchen.cn/16032.Doc
dit.kumchen.cn/36914.Doc
dit.kumchen.cn/52576.Doc
dit.kumchen.cn/89091.Doc
dit.kumchen.cn/02506.Doc
dit.kumchen.cn/90147.Doc
dir.kumchen.cn/02473.Doc
dir.kumchen.cn/00560.Doc
dir.kumchen.cn/59117.Doc
dir.kumchen.cn/40698.Doc
dir.kumchen.cn/80864.Doc
dir.kumchen.cn/27027.Doc
dir.kumchen.cn/70279.Doc
dir.kumchen.cn/74265.Doc
dir.kumchen.cn/57568.Doc
dir.kumchen.cn/92218.Doc
die.kumchen.cn/76646.Doc
die.kumchen.cn/47691.Doc
die.kumchen.cn/21942.Doc
die.kumchen.cn/04842.Doc
die.kumchen.cn/97898.Doc
die.kumchen.cn/68043.Doc
die.kumchen.cn/94553.Doc
die.kumchen.cn/09814.Doc
die.kumchen.cn/06280.Doc
die.kumchen.cn/22579.Doc
diw.kumchen.cn/42404.Doc
diw.kumchen.cn/18366.Doc
diw.kumchen.cn/63815.Doc
diw.kumchen.cn/81386.Doc
diw.kumchen.cn/93009.Doc
diw.kumchen.cn/71842.Doc
diw.kumchen.cn/52693.Doc
diw.kumchen.cn/99601.Doc
diw.kumchen.cn/90847.Doc
diw.kumchen.cn/99236.Doc
diq.kumchen.cn/74252.Doc
diq.kumchen.cn/33192.Doc
diq.kumchen.cn/05266.Doc
diq.kumchen.cn/96575.Doc
diq.kumchen.cn/46857.Doc
diq.kumchen.cn/05801.Doc
diq.kumchen.cn/35579.Doc
diq.kumchen.cn/58156.Doc
diq.kumchen.cn/22376.Doc
diq.kumchen.cn/29156.Doc
dum.kumchen.cn/89027.Doc
dum.kumchen.cn/82001.Doc
dum.kumchen.cn/88472.Doc
dum.kumchen.cn/12634.Doc
dum.kumchen.cn/08572.Doc
dum.kumchen.cn/50153.Doc
dum.kumchen.cn/57548.Doc
dum.kumchen.cn/84185.Doc
dum.kumchen.cn/18319.Doc
dum.kumchen.cn/69931.Doc
dun.kumchen.cn/62983.Doc
dun.kumchen.cn/13856.Doc
dun.kumchen.cn/00137.Doc
dun.kumchen.cn/50592.Doc
dun.kumchen.cn/59722.Doc
dun.kumchen.cn/84606.Doc
dun.kumchen.cn/96349.Doc
dun.kumchen.cn/62509.Doc
dun.kumchen.cn/72298.Doc
dun.kumchen.cn/24529.Doc
dub.kumchen.cn/33348.Doc
dub.kumchen.cn/37098.Doc
dub.kumchen.cn/93426.Doc
dub.kumchen.cn/50512.Doc
dub.kumchen.cn/65825.Doc
dub.kumchen.cn/83536.Doc
dub.kumchen.cn/64648.Doc
dub.kumchen.cn/04983.Doc
dub.kumchen.cn/59127.Doc
dub.kumchen.cn/30677.Doc
duv.kumchen.cn/28209.Doc
duv.kumchen.cn/42581.Doc
duv.kumchen.cn/00697.Doc
duv.kumchen.cn/12979.Doc
duv.kumchen.cn/99118.Doc
duv.kumchen.cn/65609.Doc
duv.kumchen.cn/04112.Doc
duv.kumchen.cn/78117.Doc
duv.kumchen.cn/48471.Doc
duv.kumchen.cn/55877.Doc
duc.kumchen.cn/08586.Doc
duc.kumchen.cn/11269.Doc
duc.kumchen.cn/14288.Doc
duc.kumchen.cn/48890.Doc
duc.kumchen.cn/66967.Doc
duc.kumchen.cn/07170.Doc
duc.kumchen.cn/68142.Doc
duc.kumchen.cn/02132.Doc
duc.kumchen.cn/79022.Doc
duc.kumchen.cn/67415.Doc
dux.kumchen.cn/11251.Doc
dux.kumchen.cn/34672.Doc
dux.kumchen.cn/26908.Doc
dux.kumchen.cn/09253.Doc
dux.kumchen.cn/53958.Doc
dux.kumchen.cn/03595.Doc
dux.kumchen.cn/84511.Doc
dux.kumchen.cn/83063.Doc
dux.kumchen.cn/85782.Doc
dux.kumchen.cn/62903.Doc
duz.kumchen.cn/23600.Doc
duz.kumchen.cn/53384.Doc
duz.kumchen.cn/83109.Doc
duz.kumchen.cn/73846.Doc
duz.kumchen.cn/94898.Doc
duz.kumchen.cn/67498.Doc
duz.kumchen.cn/98273.Doc
duz.kumchen.cn/27966.Doc
duz.kumchen.cn/99890.Doc
duz.kumchen.cn/80245.Doc
dul.kumchen.cn/78764.Doc
dul.kumchen.cn/63197.Doc
dul.kumchen.cn/87594.Doc
dul.kumchen.cn/71416.Doc
dul.kumchen.cn/76177.Doc
dul.kumchen.cn/49642.Doc
dul.kumchen.cn/79456.Doc
dul.kumchen.cn/01742.Doc
dul.kumchen.cn/66132.Doc
dul.kumchen.cn/65486.Doc
duk.kumchen.cn/19573.Doc
duk.kumchen.cn/92898.Doc
duk.kumchen.cn/99978.Doc
duk.kumchen.cn/59333.Doc
duk.kumchen.cn/04362.Doc
duk.kumchen.cn/64387.Doc
duk.kumchen.cn/21380.Doc
duk.kumchen.cn/70499.Doc
duk.kumchen.cn/09879.Doc
duk.kumchen.cn/00225.Doc
duj.kumchen.cn/63663.Doc
duj.kumchen.cn/65813.Doc
duj.kumchen.cn/18957.Doc
duj.kumchen.cn/36602.Doc
duj.kumchen.cn/67486.Doc
duj.kumchen.cn/47400.Doc
duj.kumchen.cn/74024.Doc
duj.kumchen.cn/78389.Doc
duj.kumchen.cn/00634.Doc
duj.kumchen.cn/15500.Doc
duh.kumchen.cn/46998.Doc
duh.kumchen.cn/11035.Doc
duh.kumchen.cn/63249.Doc
duh.kumchen.cn/38599.Doc
duh.kumchen.cn/52225.Doc
duh.kumchen.cn/63799.Doc
duh.kumchen.cn/79938.Doc
duh.kumchen.cn/16962.Doc
duh.kumchen.cn/44342.Doc
duh.kumchen.cn/46606.Doc
dug.kumchen.cn/46767.Doc
dug.kumchen.cn/68310.Doc
dug.kumchen.cn/32846.Doc
dug.kumchen.cn/91576.Doc
dug.kumchen.cn/80906.Doc
dug.kumchen.cn/18059.Doc
dug.kumchen.cn/38793.Doc
dug.kumchen.cn/08142.Doc
dug.kumchen.cn/75324.Doc
dug.kumchen.cn/06499.Doc
duf.kumchen.cn/06226.Doc
duf.kumchen.cn/24083.Doc
duf.kumchen.cn/62294.Doc
duf.kumchen.cn/73415.Doc
duf.kumchen.cn/24403.Doc
duf.kumchen.cn/24416.Doc
duf.kumchen.cn/32757.Doc
duf.kumchen.cn/58803.Doc
duf.kumchen.cn/34578.Doc
duf.kumchen.cn/39595.Doc
dud.kumchen.cn/75059.Doc
dud.kumchen.cn/67076.Doc
dud.kumchen.cn/36722.Doc
dud.kumchen.cn/26099.Doc
dud.kumchen.cn/33335.Doc
dud.kumchen.cn/24328.Doc
dud.kumchen.cn/80480.Doc
dud.kumchen.cn/50778.Doc
dud.kumchen.cn/97429.Doc
dud.kumchen.cn/25982.Doc
dus.kumchen.cn/70554.Doc
dus.kumchen.cn/51851.Doc
dus.kumchen.cn/66479.Doc
dus.kumchen.cn/48728.Doc
dus.kumchen.cn/60225.Doc
dus.kumchen.cn/97348.Doc
dus.kumchen.cn/11509.Doc
dus.kumchen.cn/71797.Doc
dus.kumchen.cn/72721.Doc
dus.kumchen.cn/72212.Doc
