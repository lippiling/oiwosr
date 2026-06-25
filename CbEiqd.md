
============================================================
 Python Trie前缀树实现 — 节点定义/插入搜索/自动补全/通配符
============================================================

Trie（前缀树/字典树）是一种高效的字符串检索结构，利用字符串的公共前缀来
减少查询时间。广泛应用于自动补全、拼写检查、IP路由等场景。

============================================================
1. Trie 节点定义
============================================================

class TrieNode:
    """前缀树节点类"""
    def __init__(self):
        self.children = {}          # 子节点字典 {字符: TrieNode}
        self.is_end = False         # 标记此节点是否为某个单词的结尾
        self.count = 0              # 经过此节点的单词数量（用于统计）

class Trie:
    """前缀树类"""
    def __init__(self):
        self.root = TrieNode()

============================================================
2. 插入操作（Insert）
============================================================

    def insert(self, word):
        """向Trie中插入一个单词"""
        node = self.root
        for ch in word:
            if ch not in node.children:
                node.children[ch] = TrieNode()  # 创建新节点
            node = node.children[ch]
            node.count += 1          # 经过此节点的单词数+1
        node.is_end = True           # 标记单词结尾

============================================================
3. 搜索操作（Search）
============================================================

    def search(self, word):
        """精确搜索：判断word是否在Trie中"""
        node = self._find_node(word)
        return node is not None and node.is_end

    def _find_node(self, prefix):
        """找到前缀对应的最后一个节点（辅助方法）"""
        node = self.root
        for ch in prefix:
            if ch not in node.children:
                return None          # 前缀不存在
            node = node.children[ch]
        return node

    def starts_with(self, prefix):
        """判断是否存在以prefix为前缀的单词"""
        return self._find_node(prefix) is not None

============================================================
4. 删除操作（Delete）
============================================================

    def delete(self, word):
        """从Trie中删除一个单词"""
        def _delete(node, word, depth):
            """递归删除，自底向上清理无用节点"""
            if depth == len(word):
                if not node.is_end:  # 单词不存在
                    return False
                node.is_end = False  # 取消单词结尾标记
                return len(node.children) == 0  # 如果是叶子节点则返回True
            ch = word[depth]
            if ch not in node.children:
                return False         # 单词不存在
            should_delete = _delete(node.children[ch], word, depth + 1)
            if should_delete:
                del node.children[ch]  # 删除无用节点
                node.count -= 1
                return len(node.children) == 0 and not node.is_end
            node.count -= 1
            return False
        _delete(self.root, word, 0)

============================================================
5. 自动补全实现（Auto-Complete）
============================================================

    def auto_complete(self, prefix):
        """返回所有以prefix开头的单词"""
        node = self._find_node(prefix)
        if not node:
            return []                # 没有匹配前缀
        results = []
        self._dfs_collect(node, list(prefix), results)
        return results

    def _dfs_collect(self, node, path_chars, results):
        """DFS收集所有以当前节点为前缀的完整单词"""
        if node.is_end:
            results.append(''.join(path_chars))  # 当前路径是一个完整单词
        for ch in sorted(node.children):          # 按字母顺序遍历
            path_chars.append(ch)
            self._dfs_collect(node.children[ch], path_chars, results)
            path_chars.pop()                      # 回溯

============================================================
6. 通配符搜索（Word Search with Wildcard）
============================================================

    def search_with_wildcard(self, word):
        """搜索支持通配符 '.' 匹配任意单个字符"""
        def _search(node, word, idx):
            if idx == len(word):
                return node.is_end
            ch = word[idx]
            if ch == '.':
                # 通配符：尝试所有子节点
                for child in node.children.values():
                    if _search(child, word, idx + 1):
                        return True
                return False
            else:
                if ch not in node.children:
                    return False
                return _search(node.children[ch], word, idx + 1)
        return _search(self.root, word, 0)

============================================================
7. 前缀匹配统计
============================================================

    def prefix_count(self, prefix):
        """统计有多少个单词以prefix为前缀"""
        node = self._find_node(prefix)
        if not node:
            return 0
        # 统计以该节点为根的子树中所有单词的数量
        def _count_words(node):
            total = 1 if node.is_end else 0
            for child in node.children.values():
                total += _count_words(child)
            return total
        return _count_words(node)

    def word_count(self):
        """返回Trie中单词总数"""
        return self.prefix_count('')

============================================================
8. Trie vs set 性能对比
============================================================
# Trie的优势：
#   - 前缀搜索（starts_with、auto_complete）是Trie的独有能力
#   - 在大量字符串的公共前缀很长时更省内存
#   - O(L)时间复杂度，L为单词长度，与存储的单词数量无关
#
# set/dict的优势：
#   - 精确查找速度更快（哈希表O(1) vs Trie O(L)）
#   - 内存占用通常更小（没有额外的指针开销）
#   - Python内置，使用更方便

============================================================
使用示例
============================================================
if __name__ == "__main__":
    trie = Trie()
    words = ["苹果", "苹果手机", "苹果电脑", "安卓", "安卓系统", "香蕉"]
    for w in words:
        trie.insert(w)

    print("搜索'苹果':", trie.search("苹果"))            # True
    print("搜索'橘子':", trie.search("橘子"))            # False
    print("前缀'苹果':", trie.starts_with("苹果"))        # True

    # 自动补全测试
    suggestions = trie.auto_complete("苹果")
    print("'苹果'的补全建议:", suggestions)               # ['苹果', '苹果手机', '苹果电脑']

    # 通配符搜索（英文示例）
    eng_trie = Trie()
    for w in ["cat", "car", "bat", "bar", "catalog"]:
        eng_trie.insert(w)
    print("通配符c.t:", eng_trie.search_with_wildcard("c.t"))  # True (cat)
    print("通配符.og:", eng_trie.search_with_wildcard(".og"))  # False (无匹配)

    # 前缀统计
    print("'苹果'相关的单词数:", trie.prefix_count("苹果"))   # 3

    # 删除测试
    trie.delete("苹果")
    print("删除'苹果'后搜索:", trie.search("苹果"))           # False
    print("删除后'苹果手机'仍然存在:", trie.search("苹果手机"))  # True

dcd.baoquan026.cn/28646.Doc
dcd.baoquan026.cn/66608.Doc
dcd.baoquan026.cn/20808.Doc
dcd.baoquan026.cn/26624.Doc
dcd.baoquan026.cn/42882.Doc
dcd.baoquan026.cn/28400.Doc
dcd.baoquan026.cn/88262.Doc
dcd.baoquan026.cn/44844.Doc
dcd.baoquan026.cn/62426.Doc
dcd.baoquan026.cn/60486.Doc
dcs.baoquan026.cn/20828.Doc
dcs.baoquan026.cn/26426.Doc
dcs.baoquan026.cn/24220.Doc
dcs.baoquan026.cn/20862.Doc
dcs.baoquan026.cn/24820.Doc
dcs.baoquan026.cn/64820.Doc
dcs.baoquan026.cn/02602.Doc
dcs.baoquan026.cn/82406.Doc
dcs.baoquan026.cn/82602.Doc
dcs.baoquan026.cn/64420.Doc
dca.baoquan026.cn/08426.Doc
dca.baoquan026.cn/86886.Doc
dca.baoquan026.cn/84004.Doc
dca.baoquan026.cn/26844.Doc
dca.baoquan026.cn/00864.Doc
dca.baoquan026.cn/68660.Doc
dca.baoquan026.cn/26028.Doc
dca.baoquan026.cn/06886.Doc
dca.baoquan026.cn/48460.Doc
dca.baoquan026.cn/46082.Doc
dcp.baoquan026.cn/62028.Doc
dcp.baoquan026.cn/20044.Doc
dcp.baoquan026.cn/24224.Doc
dcp.baoquan026.cn/64622.Doc
dcp.baoquan026.cn/86442.Doc
dcp.baoquan026.cn/46426.Doc
dcp.baoquan026.cn/22628.Doc
dcp.baoquan026.cn/26222.Doc
dcp.baoquan026.cn/06844.Doc
dcp.baoquan026.cn/88226.Doc
dco.baoquan026.cn/08244.Doc
dco.baoquan026.cn/08448.Doc
dco.baoquan026.cn/46802.Doc
dco.baoquan026.cn/86684.Doc
dco.baoquan026.cn/26886.Doc
dco.baoquan026.cn/64688.Doc
dco.baoquan026.cn/60408.Doc
dco.baoquan026.cn/20062.Doc
dco.baoquan026.cn/62400.Doc
dco.baoquan026.cn/48266.Doc
dci.baoquan026.cn/20280.Doc
dci.baoquan026.cn/46662.Doc
dci.baoquan026.cn/42422.Doc
dci.baoquan026.cn/00684.Doc
dci.baoquan026.cn/60048.Doc
dci.baoquan026.cn/46428.Doc
dci.baoquan026.cn/48684.Doc
dci.baoquan026.cn/02820.Doc
dci.baoquan026.cn/86240.Doc
dci.baoquan026.cn/24008.Doc
dcu.baoquan026.cn/04682.Doc
dcu.baoquan026.cn/02846.Doc
dcu.baoquan026.cn/20426.Doc
dcu.baoquan026.cn/46200.Doc
dcu.baoquan026.cn/20224.Doc
dcu.baoquan026.cn/35773.Doc
dcu.baoquan026.cn/28666.Doc
dcu.baoquan026.cn/28040.Doc
dcu.baoquan026.cn/88860.Doc
dcu.baoquan026.cn/06824.Doc
dcy.baoquan026.cn/26888.Doc
dcy.baoquan026.cn/62424.Doc
dcy.baoquan026.cn/68088.Doc
dcy.baoquan026.cn/57793.Doc
dcy.baoquan026.cn/75137.Doc
dcy.baoquan026.cn/24646.Doc
dcy.baoquan026.cn/22668.Doc
dcy.baoquan026.cn/48840.Doc
dcy.baoquan026.cn/62460.Doc
dcy.baoquan026.cn/86022.Doc
dct.baoquan026.cn/22828.Doc
dct.baoquan026.cn/42200.Doc
dct.baoquan026.cn/60480.Doc
dct.baoquan026.cn/86224.Doc
dct.baoquan026.cn/28006.Doc
dct.baoquan026.cn/84244.Doc
dct.baoquan026.cn/08088.Doc
dct.baoquan026.cn/00482.Doc
dct.baoquan026.cn/22866.Doc
dct.baoquan026.cn/82044.Doc
dcr.baoquan026.cn/44408.Doc
dcr.baoquan026.cn/62664.Doc
dcr.baoquan026.cn/64468.Doc
dcr.baoquan026.cn/62082.Doc
dcr.baoquan026.cn/62048.Doc
dcr.baoquan026.cn/06860.Doc
dcr.baoquan026.cn/28260.Doc
dcr.baoquan026.cn/60484.Doc
dcr.baoquan026.cn/04008.Doc
dcr.baoquan026.cn/60880.Doc
dce.baoquan026.cn/68422.Doc
dce.baoquan026.cn/66400.Doc
dce.baoquan026.cn/82686.Doc
dce.baoquan026.cn/00060.Doc
dce.baoquan026.cn/86084.Doc
dce.baoquan026.cn/02080.Doc
dce.baoquan026.cn/64486.Doc
dce.baoquan026.cn/57177.Doc
dce.baoquan026.cn/04848.Doc
dce.baoquan026.cn/26664.Doc
dcw.baoquan026.cn/15115.Doc
dcw.baoquan026.cn/17171.Doc
dcw.baoquan026.cn/44824.Doc
dcw.baoquan026.cn/84440.Doc
dcw.baoquan026.cn/48020.Doc
dcw.baoquan026.cn/00668.Doc
dcw.baoquan026.cn/42646.Doc
dcw.baoquan026.cn/11177.Doc
dcw.baoquan026.cn/26240.Doc
dcw.baoquan026.cn/73931.Doc
dcq.baoquan026.cn/68622.Doc
dcq.baoquan026.cn/24206.Doc
dcq.baoquan026.cn/66424.Doc
dcq.baoquan026.cn/71771.Doc
dcq.baoquan026.cn/48642.Doc
dcq.baoquan026.cn/06280.Doc
dcq.baoquan026.cn/00826.Doc
dcq.baoquan026.cn/22646.Doc
dcq.baoquan026.cn/24640.Doc
dcq.baoquan026.cn/20268.Doc
dxm.baoquan026.cn/00868.Doc
dxm.baoquan026.cn/37151.Doc
dxm.baoquan026.cn/40044.Doc
dxm.baoquan026.cn/08646.Doc
dxm.baoquan026.cn/24688.Doc
dxm.baoquan026.cn/24002.Doc
dxm.baoquan026.cn/60026.Doc
dxm.baoquan026.cn/86446.Doc
dxm.baoquan026.cn/26626.Doc
dxm.baoquan026.cn/66062.Doc
dxn.baoquan026.cn/84688.Doc
dxn.baoquan026.cn/62622.Doc
dxn.baoquan026.cn/44482.Doc
dxn.baoquan026.cn/86080.Doc
dxn.baoquan026.cn/40808.Doc
dxn.baoquan026.cn/91519.Doc
dxn.baoquan026.cn/84468.Doc
dxn.baoquan026.cn/77355.Doc
dxn.baoquan026.cn/68682.Doc
dxn.baoquan026.cn/02264.Doc
dxb.baoquan026.cn/22206.Doc
dxb.baoquan026.cn/24202.Doc
dxb.baoquan026.cn/08248.Doc
dxb.baoquan026.cn/06888.Doc
dxb.baoquan026.cn/88044.Doc
dxb.baoquan026.cn/06220.Doc
dxb.baoquan026.cn/88466.Doc
dxb.baoquan026.cn/48464.Doc
dxb.baoquan026.cn/84808.Doc
dxb.baoquan026.cn/08424.Doc
dxv.baoquan026.cn/82242.Doc
dxv.baoquan026.cn/66888.Doc
dxv.baoquan026.cn/46880.Doc
dxv.baoquan026.cn/55959.Doc
dxv.baoquan026.cn/08208.Doc
dxv.baoquan026.cn/86462.Doc
dxv.baoquan026.cn/06448.Doc
dxv.baoquan026.cn/26248.Doc
dxv.baoquan026.cn/88688.Doc
dxv.baoquan026.cn/51799.Doc
dxc.baoquan026.cn/80080.Doc
dxc.baoquan026.cn/64668.Doc
dxc.baoquan026.cn/08486.Doc
dxc.baoquan026.cn/13793.Doc
dxc.baoquan026.cn/84288.Doc
dxc.baoquan026.cn/64462.Doc
dxc.baoquan026.cn/62440.Doc
dxc.baoquan026.cn/42068.Doc
dxc.baoquan026.cn/88206.Doc
dxc.baoquan026.cn/46888.Doc
dxx.baoquan026.cn/02624.Doc
dxx.baoquan026.cn/40426.Doc
dxx.baoquan026.cn/00088.Doc
dxx.baoquan026.cn/02400.Doc
dxx.baoquan026.cn/46884.Doc
dxx.baoquan026.cn/88224.Doc
dxx.baoquan026.cn/51197.Doc
dxx.baoquan026.cn/53951.Doc
dxx.baoquan026.cn/97315.Doc
dxx.baoquan026.cn/60028.Doc
dxz.baoquan026.cn/66424.Doc
dxz.baoquan026.cn/40806.Doc
dxz.baoquan026.cn/22604.Doc
dxz.baoquan026.cn/08004.Doc
dxz.baoquan026.cn/00466.Doc
dxz.baoquan026.cn/48240.Doc
dxz.baoquan026.cn/40026.Doc
dxz.baoquan026.cn/26484.Doc
dxz.baoquan026.cn/26800.Doc
dxz.baoquan026.cn/31337.Doc
dxl.baoquan026.cn/26242.Doc
dxl.baoquan026.cn/62480.Doc
dxl.baoquan026.cn/64886.Doc
dxl.baoquan026.cn/88224.Doc
dxl.baoquan026.cn/46400.Doc
dxl.baoquan026.cn/99311.Doc
dxl.baoquan026.cn/39557.Doc
dxl.baoquan026.cn/42226.Doc
dxl.baoquan026.cn/22062.Doc
dxl.baoquan026.cn/28660.Doc
dxk.baoquan026.cn/44062.Doc
dxk.baoquan026.cn/46802.Doc
dxk.baoquan026.cn/97539.Doc
dxk.baoquan026.cn/46246.Doc
dxk.baoquan026.cn/88066.Doc
dxk.baoquan026.cn/73199.Doc
dxk.baoquan026.cn/24620.Doc
dxk.baoquan026.cn/02260.Doc
dxk.baoquan026.cn/88686.Doc
dxk.baoquan026.cn/80048.Doc
dxj.baoquan026.cn/22022.Doc
dxj.baoquan026.cn/48402.Doc
dxj.baoquan026.cn/06604.Doc
dxj.baoquan026.cn/64688.Doc
dxj.baoquan026.cn/15919.Doc
dxj.baoquan026.cn/68426.Doc
dxj.baoquan026.cn/82402.Doc
dxj.baoquan026.cn/20284.Doc
dxj.baoquan026.cn/84222.Doc
dxj.baoquan026.cn/62288.Doc
dxh.baoquan026.cn/42460.Doc
dxh.baoquan026.cn/31977.Doc
dxh.baoquan026.cn/53553.Doc
dxh.baoquan026.cn/42466.Doc
dxh.baoquan026.cn/26280.Doc
dxh.baoquan026.cn/00868.Doc
dxh.baoquan026.cn/40402.Doc
dxh.baoquan026.cn/44622.Doc
dxh.baoquan026.cn/46662.Doc
dxh.baoquan026.cn/80886.Doc
dxg.baoquan026.cn/84444.Doc
dxg.baoquan026.cn/44824.Doc
dxg.baoquan026.cn/84620.Doc
dxg.baoquan026.cn/44846.Doc
dxg.baoquan026.cn/62228.Doc
dxg.baoquan026.cn/02484.Doc
dxg.baoquan026.cn/20686.Doc
dxg.baoquan026.cn/28660.Doc
dxg.baoquan026.cn/60228.Doc
dxg.baoquan026.cn/66686.Doc
dxf.baoquan026.cn/04884.Doc
dxf.baoquan026.cn/84642.Doc
dxf.baoquan026.cn/46208.Doc
dxf.baoquan026.cn/88408.Doc
dxf.baoquan026.cn/53511.Doc
dxf.baoquan026.cn/24888.Doc
dxf.baoquan026.cn/80484.Doc
dxf.baoquan026.cn/62044.Doc
dxf.baoquan026.cn/26804.Doc
dxf.baoquan026.cn/40044.Doc
dxd.baoquan026.cn/08606.Doc
dxd.baoquan026.cn/82048.Doc
dxd.baoquan026.cn/02028.Doc
dxd.baoquan026.cn/37135.Doc
dxd.baoquan026.cn/46644.Doc
dxd.baoquan026.cn/39977.Doc
dxd.baoquan026.cn/86480.Doc
dxd.baoquan026.cn/88640.Doc
dxd.baoquan026.cn/08248.Doc
dxd.baoquan026.cn/24424.Doc
dxs.baoquan026.cn/40082.Doc
dxs.baoquan026.cn/46848.Doc
dxs.baoquan026.cn/02068.Doc
dxs.baoquan026.cn/06000.Doc
dxs.baoquan026.cn/62622.Doc
dxs.baoquan026.cn/39795.Doc
dxs.baoquan026.cn/08402.Doc
dxs.baoquan026.cn/24448.Doc
dxs.baoquan026.cn/20884.Doc
dxs.baoquan026.cn/04802.Doc
dxa.baoquan026.cn/46640.Doc
dxa.baoquan026.cn/24422.Doc
dxa.baoquan026.cn/60866.Doc
dxa.baoquan026.cn/40848.Doc
dxa.baoquan026.cn/60084.Doc
dxa.baoquan026.cn/04086.Doc
dxa.baoquan026.cn/02684.Doc
dxa.baoquan026.cn/46806.Doc
dxa.baoquan026.cn/64624.Doc
dxa.baoquan026.cn/24080.Doc
dxp.baoquan026.cn/44864.Doc
dxp.baoquan026.cn/28846.Doc
dxp.baoquan026.cn/46280.Doc
dxp.baoquan026.cn/82080.Doc
dxp.baoquan026.cn/68262.Doc
dxp.baoquan026.cn/46084.Doc
dxp.baoquan026.cn/00006.Doc
dxp.baoquan026.cn/66880.Doc
dxp.baoquan026.cn/84222.Doc
dxp.baoquan026.cn/48088.Doc
dxo.baoquan026.cn/42204.Doc
dxo.baoquan026.cn/42066.Doc
dxo.baoquan026.cn/48402.Doc
dxo.baoquan026.cn/44228.Doc
dxo.baoquan026.cn/68880.Doc
dxo.baoquan026.cn/19595.Doc
dxo.baoquan026.cn/60448.Doc
dxo.baoquan026.cn/82220.Doc
dxo.baoquan026.cn/88084.Doc
dxo.baoquan026.cn/40646.Doc
dxi.baoquan026.cn/59171.Doc
dxi.baoquan026.cn/44240.Doc
dxi.baoquan026.cn/62688.Doc
dxi.baoquan026.cn/60282.Doc
dxi.baoquan026.cn/80886.Doc
dxi.baoquan026.cn/84644.Doc
dxi.baoquan026.cn/40408.Doc
dxi.baoquan026.cn/84460.Doc
dxi.baoquan026.cn/66004.Doc
dxi.baoquan026.cn/00822.Doc
dxu.baoquan026.cn/80064.Doc
dxu.baoquan026.cn/22626.Doc
dxu.baoquan026.cn/48248.Doc
dxu.baoquan026.cn/66884.Doc
dxu.baoquan026.cn/37771.Doc
dxu.baoquan026.cn/88860.Doc
dxu.baoquan026.cn/06220.Doc
dxu.baoquan026.cn/62286.Doc
dxu.baoquan026.cn/08602.Doc
dxu.baoquan026.cn/42604.Doc
dxy.baoquan026.cn/46422.Doc
dxy.baoquan026.cn/46226.Doc
dxy.baoquan026.cn/80460.Doc
dxy.baoquan026.cn/68080.Doc
dxy.baoquan026.cn/88228.Doc
dxy.baoquan026.cn/46202.Doc
dxy.baoquan026.cn/26044.Doc
dxy.baoquan026.cn/86028.Doc
dxy.baoquan026.cn/08602.Doc
dxy.baoquan026.cn/66042.Doc
dxt.baoquan026.cn/26420.Doc
dxt.baoquan026.cn/40460.Doc
dxt.baoquan026.cn/82422.Doc
dxt.baoquan026.cn/62004.Doc
dxt.baoquan026.cn/20402.Doc
dxt.baoquan026.cn/66644.Doc
dxt.baoquan026.cn/86026.Doc
dxt.baoquan026.cn/06262.Doc
dxt.baoquan026.cn/00260.Doc
dxt.baoquan026.cn/84686.Doc
dxr.baoquan026.cn/68802.Doc
dxr.baoquan026.cn/86804.Doc
dxr.baoquan026.cn/66208.Doc
dxr.baoquan026.cn/22080.Doc
dxr.baoquan026.cn/11971.Doc
dxr.baoquan026.cn/11513.Doc
dxr.baoquan026.cn/02282.Doc
dxr.baoquan026.cn/60466.Doc
dxr.baoquan026.cn/24062.Doc
dxr.baoquan026.cn/00284.Doc
dxe.baoquan026.cn/66602.Doc
dxe.baoquan026.cn/64026.Doc
dxe.baoquan026.cn/00002.Doc
dxe.baoquan026.cn/26668.Doc
dxe.baoquan026.cn/46264.Doc
dxe.baoquan026.cn/40884.Doc
dxe.baoquan026.cn/35915.Doc
dxe.baoquan026.cn/06260.Doc
dxe.baoquan026.cn/11579.Doc
dxe.baoquan026.cn/20824.Doc
dxw.baoquan026.cn/33911.Doc
dxw.baoquan026.cn/60004.Doc
dxw.baoquan026.cn/62888.Doc
dxw.baoquan026.cn/20028.Doc
dxw.baoquan026.cn/64846.Doc
dxw.baoquan026.cn/86640.Doc
dxw.baoquan026.cn/44062.Doc
dxw.baoquan026.cn/06868.Doc
dxw.baoquan026.cn/66042.Doc
dxw.baoquan026.cn/26662.Doc
dxq.baoquan026.cn/26468.Doc
dxq.baoquan026.cn/48826.Doc
dxq.baoquan026.cn/73313.Doc
dxq.baoquan026.cn/80482.Doc
dxq.baoquan026.cn/88008.Doc
dxq.baoquan026.cn/48082.Doc
dxq.baoquan026.cn/86442.Doc
dxq.baoquan026.cn/42444.Doc
dxq.baoquan026.cn/40602.Doc
dxq.baoquan026.cn/66664.Doc
dzm.baoquan026.cn/66888.Doc
dzm.baoquan026.cn/26260.Doc
dzm.baoquan026.cn/44866.Doc
dzm.baoquan026.cn/20682.Doc
dzm.baoquan026.cn/26666.Doc
dzm.baoquan026.cn/24024.Doc
dzm.baoquan026.cn/60248.Doc
dzm.baoquan026.cn/02048.Doc
dzm.baoquan026.cn/20880.Doc
dzm.baoquan026.cn/80282.Doc
dzn.baoquan026.cn/84846.Doc
dzn.baoquan026.cn/46666.Doc
dzn.baoquan026.cn/82404.Doc
dzn.baoquan026.cn/00240.Doc
dzn.baoquan026.cn/02862.Doc
dzn.baoquan026.cn/42208.Doc
dzn.baoquan026.cn/82224.Doc
dzn.baoquan026.cn/80248.Doc
dzn.baoquan026.cn/86864.Doc
dzn.baoquan026.cn/06220.Doc
dzb.baoquan026.cn/44842.Doc
dzb.baoquan026.cn/60002.Doc
dzb.baoquan026.cn/28608.Doc
dzb.baoquan026.cn/00042.Doc
dzb.baoquan026.cn/24240.Doc
dzb.baoquan026.cn/66268.Doc
dzb.baoquan026.cn/44022.Doc
dzb.baoquan026.cn/42082.Doc
dzb.baoquan026.cn/40648.Doc
dzb.baoquan026.cn/40468.Doc
dzv.baoquan026.cn/04444.Doc
dzv.baoquan026.cn/22220.Doc
dzv.baoquan026.cn/04224.Doc
dzv.baoquan026.cn/26482.Doc
dzv.baoquan026.cn/80668.Doc
dzv.baoquan026.cn/04806.Doc
dzv.baoquan026.cn/80444.Doc
dzv.baoquan026.cn/84264.Doc
dzv.baoquan026.cn/20282.Doc
dzv.baoquan026.cn/64400.Doc
dzc.baoquan026.cn/44480.Doc
dzc.baoquan026.cn/68882.Doc
dzc.baoquan026.cn/86264.Doc
dzc.baoquan026.cn/84800.Doc
dzc.baoquan026.cn/68486.Doc
dzc.baoquan026.cn/08080.Doc
dzc.baoquan026.cn/48204.Doc
dzc.baoquan026.cn/60080.Doc
dzc.baoquan026.cn/62662.Doc
dzc.baoquan026.cn/24866.Doc
dzx.baoquan026.cn/28606.Doc
dzx.baoquan026.cn/08008.Doc
dzx.baoquan026.cn/46420.Doc
dzx.baoquan026.cn/66002.Doc
dzx.baoquan026.cn/80020.Doc
dzx.baoquan026.cn/26088.Doc
dzx.baoquan026.cn/20022.Doc
dzx.baoquan026.cn/46606.Doc
dzx.baoquan026.cn/84040.Doc
dzx.baoquan026.cn/84024.Doc
dzz.baoquan026.cn/24602.Doc
dzz.baoquan026.cn/44248.Doc
dzz.baoquan026.cn/00826.Doc
dzz.baoquan026.cn/24086.Doc
dzz.baoquan026.cn/46846.Doc
dzz.baoquan026.cn/84404.Doc
dzz.baoquan026.cn/64402.Doc
dzz.baoquan026.cn/84448.Doc
dzz.baoquan026.cn/26064.Doc
dzz.baoquan026.cn/22000.Doc
dzl.baoquan026.cn/80088.Doc
dzl.baoquan026.cn/60822.Doc
dzl.baoquan026.cn/48000.Doc
dzl.baoquan026.cn/62048.Doc
dzl.baoquan026.cn/44648.Doc
dzl.baoquan026.cn/02888.Doc
dzl.baoquan026.cn/62680.Doc
dzl.baoquan026.cn/80006.Doc
dzl.baoquan026.cn/06406.Doc
dzl.baoquan026.cn/02266.Doc
dzk.baoquan026.cn/88248.Doc
dzk.baoquan026.cn/40806.Doc
dzk.baoquan026.cn/60680.Doc
dzk.baoquan026.cn/64266.Doc
dzk.baoquan026.cn/68406.Doc
dzk.baoquan026.cn/26668.Doc
dzk.baoquan026.cn/62442.Doc
dzk.baoquan026.cn/80868.Doc
dzk.baoquan026.cn/00284.Doc
dzk.baoquan026.cn/46444.Doc
dzj.baoquan026.cn/24628.Doc
dzj.baoquan026.cn/88880.Doc
dzj.baoquan026.cn/40802.Doc
dzj.baoquan026.cn/80286.Doc
dzj.baoquan026.cn/00842.Doc
dzj.baoquan026.cn/88024.Doc
dzj.baoquan026.cn/04444.Doc
dzj.baoquan026.cn/60000.Doc
dzj.baoquan026.cn/00882.Doc
dzj.baoquan026.cn/08600.Doc
