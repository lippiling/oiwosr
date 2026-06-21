橇口恼山科


"""
二分搜索进阶 — bisect 模块剖析 + 旋转数组/二维搜索/范围查询
掌握 left 与 right 的区别是正确使用二分的关键
"""

import bisect


def demo_bisect():
    """bisect_left vs bisect_right 的区别"""
    arr = [1, 3, 3, 3, 5, 7]
    target = 3
    L = bisect.bisect_left(arr, target)   # 第一个 >= target
    R = bisect.bisect_right(arr, target)  # 第一个 > target
    print(f"数组: {arr}, 目标: {target}")
    print(f"left={L}, right={R}, 出现次数={R - L}")


def search_rotated(nums: list, target: int) -> int:
    """搜索旋转排序数组 (LeetCode 33)"""
    lo, hi = 0, len(nums) - 1
    while lo <= hi:
        mid = (lo + hi) // 2
        if nums[mid] == target:
            return mid
        if nums[lo] <= nums[mid]:  # 左半有序
            if nums[lo] <= target < nums[mid]:
                hi = mid - 1
            else:
                lo = mid + 1
        else:  # 右半有序
            if nums[mid] < target <= nums[hi]:
                lo = mid + 1
            else:
                hi = mid - 1
    return -1


def first_last(nums: list, target: int) -> list:
    """有序数组中查找 target 的首尾位置 (LeetCode 34)"""
    left = bisect.bisect_left(nums, target)
    if left == len(nums) or nums[left] != target:
        return [-1, -1]
    right = bisect.bisect_right(nums, target) - 1
    return [left, right]


def search_matrix(mat: list, target: int) -> bool:
    """行列有序的二维矩阵搜索 (LeetCode 240)"""
    if not mat or not mat[0]:
        return False
    i, j = 0, len(mat[0]) - 1  # 从右上角开始
    while i < len(mat) and j >= 0:
        if mat[i][j] == target:
            return True
        elif mat[i][j] > target:
            j -= 1  # 当前行右侧都更大
        else:
            i += 1  # 当前列下方都更大
    return False


def count_in_range(arr: list, low: int, high: int) -> int:
    """统计有序数组中 [low, high] 的元素个数"""
    return bisect.bisect_right(arr, high) - bisect.bisect_left(arr, low)


def demo():
    demo_bisect()
    # 旋转数组
    r = [4, 5, 6, 7, 0, 1, 2]
    print(f"旋转数组 {r} 搜索 0: 索引 {search_rotated(r, 0)}")
    print(f"旋转数组搜索 3: {search_rotated(r, 3)} (期望 -1)")
    # 首尾位置
    arr = [1, 2, 3, 3, 3, 4, 5]
    print(f"值 3 首尾位置: {first_last(arr, 3)}")
    # 二维搜索
    m = [[1, 4, 7], [2, 5, 8], [3, 6, 9]]
    print(f"矩阵搜索 5: {search_matrix(m, 5)}, 搜索 10: {search_matrix(m, 10)}")
    # 范围查询
    nums = [1, 3, 5, 7, 9, 11]
    print(f"[3,10] 元素个数: {count_in_range(nums, 3, 10)}")


if __name__ == "__main__":
    demo()

颇庸我疤毡寥牧胺估泼艘备现司卣

m.qxe.qtbzn.cn/04460.Doc
m.qxe.qtbzn.cn/26484.Doc
m.qxe.qtbzn.cn/60266.Doc
m.qxe.qtbzn.cn/02400.Doc
m.qxe.qtbzn.cn/06444.Doc
m.qxe.qtbzn.cn/88000.Doc
m.qxe.qtbzn.cn/42804.Doc
m.qxe.qtbzn.cn/04204.Doc
m.qxe.qtbzn.cn/73175.Doc
m.qxe.qtbzn.cn/40688.Doc
m.qxe.qtbzn.cn/93751.Doc
m.qxe.qtbzn.cn/37119.Doc
m.qxe.qtbzn.cn/48462.Doc
m.qxe.qtbzn.cn/24642.Doc
m.qxe.qtbzn.cn/60884.Doc
m.qxe.qtbzn.cn/42666.Doc
m.qxe.qtbzn.cn/84844.Doc
m.qxe.qtbzn.cn/44840.Doc
m.qxe.qtbzn.cn/57571.Doc
m.qxe.qtbzn.cn/84264.Doc
m.qxw.qtbzn.cn/19779.Doc
m.qxw.qtbzn.cn/99557.Doc
m.qxw.qtbzn.cn/46226.Doc
m.qxw.qtbzn.cn/37579.Doc
m.qxw.qtbzn.cn/02864.Doc
m.qxw.qtbzn.cn/02484.Doc
m.qxw.qtbzn.cn/80424.Doc
m.qxw.qtbzn.cn/22002.Doc
m.qxw.qtbzn.cn/00668.Doc
m.qxw.qtbzn.cn/51133.Doc
m.qxw.qtbzn.cn/48424.Doc
m.qxw.qtbzn.cn/26484.Doc
m.qxw.qtbzn.cn/51553.Doc
m.qxw.qtbzn.cn/88082.Doc
m.qxw.qtbzn.cn/42804.Doc
m.qxw.qtbzn.cn/20862.Doc
m.qxw.qtbzn.cn/15979.Doc
m.qxw.qtbzn.cn/00880.Doc
m.qxw.qtbzn.cn/00860.Doc
m.qxw.qtbzn.cn/68662.Doc
m.qxq.qtbzn.cn/06626.Doc
m.qxq.qtbzn.cn/86024.Doc
m.qxq.qtbzn.cn/75533.Doc
m.qxq.qtbzn.cn/84468.Doc
m.qxq.qtbzn.cn/06086.Doc
m.qxq.qtbzn.cn/82480.Doc
m.qxq.qtbzn.cn/55711.Doc
m.qxq.qtbzn.cn/08404.Doc
m.qxq.qtbzn.cn/84860.Doc
m.qxq.qtbzn.cn/42082.Doc
m.qxq.qtbzn.cn/04828.Doc
m.qxq.qtbzn.cn/08006.Doc
m.qxq.qtbzn.cn/44440.Doc
m.qxq.qtbzn.cn/80228.Doc
m.qxq.qtbzn.cn/59991.Doc
m.qxq.qtbzn.cn/68222.Doc
m.qxq.qtbzn.cn/08448.Doc
m.qxq.qtbzn.cn/42802.Doc
m.qxq.qtbzn.cn/66260.Doc
m.qxq.qtbzn.cn/80020.Doc
m.qzm.qtbzn.cn/59117.Doc
m.qzm.qtbzn.cn/28824.Doc
m.qzm.qtbzn.cn/64228.Doc
m.qzm.qtbzn.cn/04262.Doc
m.qzm.qtbzn.cn/46200.Doc
m.qzm.qtbzn.cn/75915.Doc
m.qzm.qtbzn.cn/82644.Doc
m.qzm.qtbzn.cn/20608.Doc
m.qzm.qtbzn.cn/80024.Doc
m.qzm.qtbzn.cn/15733.Doc
m.qzm.qtbzn.cn/42820.Doc
m.qzm.qtbzn.cn/80686.Doc
m.qzm.qtbzn.cn/15773.Doc
m.qzm.qtbzn.cn/24860.Doc
m.qzm.qtbzn.cn/62222.Doc
m.qzm.qtbzn.cn/80406.Doc
m.qzm.qtbzn.cn/68482.Doc
m.qzm.qtbzn.cn/48660.Doc
m.qzm.qtbzn.cn/86288.Doc
m.qzm.qtbzn.cn/04262.Doc
m.qzn.qtbzn.cn/04860.Doc
m.qzn.qtbzn.cn/24868.Doc
m.qzn.qtbzn.cn/06068.Doc
m.qzn.qtbzn.cn/44888.Doc
m.qzn.qtbzn.cn/28086.Doc
m.qzn.qtbzn.cn/44240.Doc
m.qzn.qtbzn.cn/48844.Doc
m.qzn.qtbzn.cn/82446.Doc
m.qzn.qtbzn.cn/20004.Doc
m.qzn.qtbzn.cn/02600.Doc
m.qzn.qtbzn.cn/11957.Doc
m.qzn.qtbzn.cn/42220.Doc
m.qzn.qtbzn.cn/84822.Doc
m.qzn.qtbzn.cn/79537.Doc
m.qzn.qtbzn.cn/26464.Doc
m.qzn.qtbzn.cn/06804.Doc
m.qzn.qtbzn.cn/60626.Doc
m.qzn.qtbzn.cn/68048.Doc
m.qzn.qtbzn.cn/66688.Doc
m.qzn.qtbzn.cn/57531.Doc
m.qzb.qtbzn.cn/20242.Doc
m.qzb.qtbzn.cn/37937.Doc
m.qzb.qtbzn.cn/02848.Doc
m.qzb.qtbzn.cn/62288.Doc
m.qzb.qtbzn.cn/26062.Doc
m.qzb.qtbzn.cn/64408.Doc
m.qzb.qtbzn.cn/40640.Doc
m.qzb.qtbzn.cn/86044.Doc
m.qzb.qtbzn.cn/04648.Doc
m.qzb.qtbzn.cn/66466.Doc
m.qzb.qtbzn.cn/60862.Doc
m.qzb.qtbzn.cn/28602.Doc
m.qzb.qtbzn.cn/22884.Doc
m.qzb.qtbzn.cn/84826.Doc
m.qzb.qtbzn.cn/68624.Doc
m.qzb.qtbzn.cn/40226.Doc
m.qzb.qtbzn.cn/40604.Doc
m.qzb.qtbzn.cn/24646.Doc
m.qzb.qtbzn.cn/62668.Doc
m.qzb.qtbzn.cn/26084.Doc
m.qzv.qtbzn.cn/48066.Doc
m.qzv.qtbzn.cn/04206.Doc
m.qzv.qtbzn.cn/86088.Doc
m.qzv.qtbzn.cn/66220.Doc
m.qzv.qtbzn.cn/82646.Doc
m.qzv.qtbzn.cn/66066.Doc
m.qzv.qtbzn.cn/88242.Doc
m.qzv.qtbzn.cn/06060.Doc
m.qzv.qtbzn.cn/42246.Doc
m.qzv.qtbzn.cn/93575.Doc
m.qzv.qtbzn.cn/08820.Doc
m.qzv.qtbzn.cn/48460.Doc
m.qzv.qtbzn.cn/28828.Doc
m.qzv.qtbzn.cn/22886.Doc
m.qzv.qtbzn.cn/40260.Doc
m.qzv.qtbzn.cn/39153.Doc
m.qzv.qtbzn.cn/46404.Doc
m.qzv.qtbzn.cn/55733.Doc
m.qzv.qtbzn.cn/24248.Doc
m.qzv.qtbzn.cn/17777.Doc
m.qzc.qtbzn.cn/86806.Doc
m.qzc.qtbzn.cn/35739.Doc
m.qzc.qtbzn.cn/44688.Doc
m.qzc.qtbzn.cn/62686.Doc
m.qzc.qtbzn.cn/80022.Doc
m.qzc.qtbzn.cn/06660.Doc
m.qzc.qtbzn.cn/48466.Doc
m.qzc.qtbzn.cn/08866.Doc
m.qzc.qtbzn.cn/84206.Doc
m.qzc.qtbzn.cn/06200.Doc
m.qzc.qtbzn.cn/60064.Doc
m.qzc.qtbzn.cn/24226.Doc
m.qzc.qtbzn.cn/20448.Doc
m.qzc.qtbzn.cn/79393.Doc
m.qzc.qtbzn.cn/86224.Doc
m.qzc.qtbzn.cn/75119.Doc
m.qzc.qtbzn.cn/59593.Doc
m.qzc.qtbzn.cn/64020.Doc
m.qzc.qtbzn.cn/20068.Doc
m.qzc.qtbzn.cn/44022.Doc
m.qzx.qtbzn.cn/46644.Doc
m.qzx.qtbzn.cn/40844.Doc
m.qzx.qtbzn.cn/00224.Doc
m.qzx.qtbzn.cn/66266.Doc
m.qzx.qtbzn.cn/64624.Doc
m.qzx.qtbzn.cn/64408.Doc
m.qzx.qtbzn.cn/06086.Doc
m.qzx.qtbzn.cn/84646.Doc
m.qzx.qtbzn.cn/08282.Doc
m.qzx.qtbzn.cn/86002.Doc
m.qzx.qtbzn.cn/40444.Doc
m.qzx.qtbzn.cn/06624.Doc
m.qzx.qtbzn.cn/60484.Doc
m.qzx.qtbzn.cn/40222.Doc
m.qzx.qtbzn.cn/60264.Doc
m.qzx.qtbzn.cn/02200.Doc
m.qzx.qtbzn.cn/42242.Doc
m.qzx.qtbzn.cn/62280.Doc
m.qzx.qtbzn.cn/04442.Doc
m.qzx.qtbzn.cn/42242.Doc
m.qzz.qtbzn.cn/00244.Doc
m.qzz.qtbzn.cn/60846.Doc
m.qzz.qtbzn.cn/68284.Doc
m.qzz.qtbzn.cn/31757.Doc
m.qzz.qtbzn.cn/40682.Doc
m.qzz.qtbzn.cn/22266.Doc
m.qzz.qtbzn.cn/82648.Doc
m.qzz.qtbzn.cn/80862.Doc
m.qzz.qtbzn.cn/88082.Doc
m.qzz.qtbzn.cn/04006.Doc
m.qzz.qtbzn.cn/40064.Doc
m.qzz.qtbzn.cn/77951.Doc
m.qzz.qtbzn.cn/86682.Doc
m.qzz.qtbzn.cn/55773.Doc
m.qzz.qtbzn.cn/80486.Doc
m.qzz.qtbzn.cn/08844.Doc
m.qzz.qtbzn.cn/46220.Doc
m.qzz.qtbzn.cn/62882.Doc
m.qzz.qtbzn.cn/88420.Doc
m.qzz.qtbzn.cn/31195.Doc
m.qzl.qtbzn.cn/31397.Doc
m.qzl.qtbzn.cn/68828.Doc
m.qzl.qtbzn.cn/24044.Doc
m.qzl.qtbzn.cn/00008.Doc
m.qzl.qtbzn.cn/15913.Doc
m.qzl.qtbzn.cn/15533.Doc
m.qzl.qtbzn.cn/26024.Doc
m.qzl.qtbzn.cn/08082.Doc
m.qzl.qtbzn.cn/40664.Doc
m.qzl.qtbzn.cn/00888.Doc
m.qzl.qtbzn.cn/44444.Doc
m.qzl.qtbzn.cn/13775.Doc
m.qzl.qtbzn.cn/08222.Doc
m.qzl.qtbzn.cn/00442.Doc
m.qzl.qtbzn.cn/22664.Doc
m.qzl.qtbzn.cn/84600.Doc
m.qzl.qtbzn.cn/40640.Doc
m.qzl.qtbzn.cn/24064.Doc
m.qzl.qtbzn.cn/80222.Doc
m.qzl.qtbzn.cn/64682.Doc
m.qzk.qtbzn.cn/60466.Doc
m.qzk.qtbzn.cn/22228.Doc
m.qzk.qtbzn.cn/02446.Doc
m.qzk.qtbzn.cn/55391.Doc
m.qzk.qtbzn.cn/60440.Doc
m.qzk.qtbzn.cn/80808.Doc
m.qzk.qtbzn.cn/04244.Doc
m.qzk.qtbzn.cn/44222.Doc
m.qzk.qtbzn.cn/33719.Doc
m.qzk.qtbzn.cn/86080.Doc
m.qzk.qtbzn.cn/82240.Doc
m.qzk.qtbzn.cn/44440.Doc
m.qzk.qtbzn.cn/24840.Doc
m.qzk.qtbzn.cn/86428.Doc
m.qzk.qtbzn.cn/20424.Doc
m.qzk.qtbzn.cn/55399.Doc
m.qzk.qtbzn.cn/24208.Doc
m.qzk.qtbzn.cn/68488.Doc
m.qzk.qtbzn.cn/13555.Doc
m.qzk.qtbzn.cn/57339.Doc
m.qzj.qtbzn.cn/53337.Doc
m.qzj.qtbzn.cn/44462.Doc
m.qzj.qtbzn.cn/60808.Doc
m.qzj.qtbzn.cn/68268.Doc
m.qzj.qtbzn.cn/55997.Doc
m.qzj.qtbzn.cn/44464.Doc
m.qzj.qtbzn.cn/86840.Doc
m.qzj.qtbzn.cn/24000.Doc
m.qzj.qtbzn.cn/02200.Doc
m.qzj.qtbzn.cn/86800.Doc
m.qzj.qtbzn.cn/93371.Doc
m.qzj.qtbzn.cn/19315.Doc
m.qzj.qtbzn.cn/42280.Doc
m.qzj.qtbzn.cn/66848.Doc
m.qzj.qtbzn.cn/62286.Doc
m.qzj.qtbzn.cn/80842.Doc
m.qzj.qtbzn.cn/53571.Doc
m.qzj.qtbzn.cn/48642.Doc
m.qzj.qtbzn.cn/44686.Doc
m.qzj.qtbzn.cn/62864.Doc
m.qzh.qtbzn.cn/04066.Doc
m.qzh.qtbzn.cn/44022.Doc
m.qzh.qtbzn.cn/08862.Doc
m.qzh.qtbzn.cn/46664.Doc
m.qzh.qtbzn.cn/39715.Doc
m.qzh.qtbzn.cn/08628.Doc
m.qzh.qtbzn.cn/33977.Doc
m.qzh.qtbzn.cn/28020.Doc
m.qzh.qtbzn.cn/93355.Doc
m.qzh.qtbzn.cn/64268.Doc
m.qzh.qtbzn.cn/88626.Doc
m.qzh.qtbzn.cn/84842.Doc
m.qzh.qtbzn.cn/20000.Doc
m.qzh.qtbzn.cn/75375.Doc
m.qzh.qtbzn.cn/68046.Doc
m.qzh.qtbzn.cn/20642.Doc
m.qzh.qtbzn.cn/68802.Doc
m.qzh.qtbzn.cn/20022.Doc
m.qzh.qtbzn.cn/42646.Doc
m.qzh.qtbzn.cn/68042.Doc
m.qzg.qtbzn.cn/42800.Doc
m.qzg.qtbzn.cn/20048.Doc
m.qzg.qtbzn.cn/31979.Doc
m.qzg.qtbzn.cn/71371.Doc
m.qzg.qtbzn.cn/46044.Doc
m.qzg.qtbzn.cn/84868.Doc
m.qzg.qtbzn.cn/79515.Doc
m.qzg.qtbzn.cn/40040.Doc
m.qzg.qtbzn.cn/37939.Doc
m.qzg.qtbzn.cn/08442.Doc
m.qzg.qtbzn.cn/99955.Doc
m.qzg.qtbzn.cn/40604.Doc
m.qzg.qtbzn.cn/64226.Doc
m.qzg.qtbzn.cn/79331.Doc
m.qzg.qtbzn.cn/00060.Doc
m.qzg.qtbzn.cn/46824.Doc
m.qzg.qtbzn.cn/60286.Doc
m.qzg.qtbzn.cn/44488.Doc
m.qzg.qtbzn.cn/26000.Doc
m.qzg.qtbzn.cn/40662.Doc
m.qzf.qtbzn.cn/22008.Doc
m.qzf.qtbzn.cn/84000.Doc
m.qzf.qtbzn.cn/20288.Doc
m.qzf.qtbzn.cn/62228.Doc
m.qzf.qtbzn.cn/35739.Doc
m.qzf.qtbzn.cn/04288.Doc
m.qzf.qtbzn.cn/88862.Doc
m.qzf.qtbzn.cn/24484.Doc
m.qzf.qtbzn.cn/48840.Doc
m.qzf.qtbzn.cn/02042.Doc
m.qzf.qtbzn.cn/55793.Doc
m.qzf.qtbzn.cn/93771.Doc
m.qzf.qtbzn.cn/39339.Doc
m.qzf.qtbzn.cn/40020.Doc
m.qzf.qtbzn.cn/82244.Doc
m.qzf.qtbzn.cn/48608.Doc
m.qzf.qtbzn.cn/19175.Doc
m.qzf.qtbzn.cn/24826.Doc
m.qzf.qtbzn.cn/60640.Doc
m.qzf.qtbzn.cn/80422.Doc
m.qzd.qtbzn.cn/02440.Doc
m.qzd.qtbzn.cn/00046.Doc
m.qzd.qtbzn.cn/20202.Doc
m.qzd.qtbzn.cn/15931.Doc
m.qzd.qtbzn.cn/22664.Doc
m.qzd.qtbzn.cn/42628.Doc
m.qzd.qtbzn.cn/44262.Doc
m.qzd.qtbzn.cn/60482.Doc
m.qzd.qtbzn.cn/84620.Doc
m.qzd.qtbzn.cn/79975.Doc
m.qzd.qtbzn.cn/28806.Doc
m.qzd.qtbzn.cn/28002.Doc
m.qzd.qtbzn.cn/73357.Doc
m.qzd.qtbzn.cn/84608.Doc
m.qzd.qtbzn.cn/60206.Doc
m.qzd.qtbzn.cn/80446.Doc
m.qzd.qtbzn.cn/17791.Doc
m.qzd.qtbzn.cn/22000.Doc
m.qzd.qtbzn.cn/82628.Doc
m.qzd.qtbzn.cn/68066.Doc
m.qzs.qtbzn.cn/13193.Doc
m.qzs.qtbzn.cn/02022.Doc
m.qzs.qtbzn.cn/08882.Doc
m.qzs.qtbzn.cn/22666.Doc
m.qzs.qtbzn.cn/00424.Doc
m.qzs.qtbzn.cn/17351.Doc
m.qzs.qtbzn.cn/60280.Doc
m.qzs.qtbzn.cn/20228.Doc
m.qzs.qtbzn.cn/42082.Doc
m.qzs.qtbzn.cn/02808.Doc
m.qzs.qtbzn.cn/35995.Doc
m.qzs.qtbzn.cn/73759.Doc
m.qzs.qtbzn.cn/24022.Doc
m.qzs.qtbzn.cn/26046.Doc
m.qzs.qtbzn.cn/62202.Doc
m.qzs.qtbzn.cn/66460.Doc
m.qzs.qtbzn.cn/37735.Doc
m.qzs.qtbzn.cn/46440.Doc
m.qzs.qtbzn.cn/64206.Doc
m.qzs.qtbzn.cn/79733.Doc
m.qza.qtbzn.cn/40866.Doc
m.qza.qtbzn.cn/88202.Doc
m.qza.qtbzn.cn/97193.Doc
m.qza.qtbzn.cn/20606.Doc
m.qza.qtbzn.cn/00226.Doc
m.qza.qtbzn.cn/88244.Doc
m.qza.qtbzn.cn/46680.Doc
m.qza.qtbzn.cn/42648.Doc
m.qza.qtbzn.cn/77957.Doc
m.qza.qtbzn.cn/48628.Doc
m.qza.qtbzn.cn/82202.Doc
m.qza.qtbzn.cn/44628.Doc
m.qza.qtbzn.cn/46464.Doc
m.qza.qtbzn.cn/86606.Doc
m.qza.qtbzn.cn/68804.Doc
m.qza.qtbzn.cn/02402.Doc
m.qza.qtbzn.cn/39733.Doc
m.qza.qtbzn.cn/00800.Doc
m.qza.qtbzn.cn/42468.Doc
m.qza.qtbzn.cn/60622.Doc
m.qzp.qtbzn.cn/84260.Doc
m.qzp.qtbzn.cn/77393.Doc
m.qzp.qtbzn.cn/15575.Doc
m.qzp.qtbzn.cn/42024.Doc
m.qzp.qtbzn.cn/22006.Doc
m.qzp.qtbzn.cn/66266.Doc
m.qzp.qtbzn.cn/08400.Doc
m.qzp.qtbzn.cn/40404.Doc
m.qzp.qtbzn.cn/00202.Doc
m.qzp.qtbzn.cn/48004.Doc
m.qzp.qtbzn.cn/95797.Doc
m.qzp.qtbzn.cn/82848.Doc
m.qzp.qtbzn.cn/02680.Doc
m.qzp.qtbzn.cn/82042.Doc
m.qzp.qtbzn.cn/88864.Doc
