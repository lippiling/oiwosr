檬碧恼裁怕


# Python线性规划 - 完整代码示例
# 线性规划在约束条件下优化线性目标函数，广泛应用于运筹学

import numpy as np
from scipy import optimize

# 1. 基础线性规划：scipy.optimize.linprog
# 标准形式: min c^T x, s.t. A_ub x <= b_ub, A_eq x = b_eq, bounds <= x
# 注意：linprog 默认求最小值，约束为 <=

# 示例 1：生产计划优化
# 某工厂生产两种产品 A 和 B
# 目标：最大化利润 = 40*A + 30*B
# 约束：
#   原材料 1: 2A + 1B <= 100
#   原材料 2: 1A + 2B <= 80
#   A >= 0, B >= 0

# 转为最小化：min -(40*A + 30*B)
c = [-40, -30]  # 目标函数系数（负号转换求最大）

# 不等式约束 A_ub @ x <= b_ub
A_ub = [[2, 1],
        [1, 2]]
b_ub = [100, 80]

# 变量边界
bounds = [(0, None), (0, None)]  # A >= 0, B >= 0

result = optimize.linprog(c, A_ub=A_ub, b_ub=b_ub, bounds=bounds, method='highs')

print("生产计划优化问题:")
print(f"优化状态: {result.message}")
print(f"产品 A 产量: {result.x[0]:.2f}")
print(f"产品 B 产量: {result.x[1]:.2f}")
print(f"最大利润: {-result.fun:.2f}")

# 2. 含等式约束的线性规划
# 示例 2：投资组合分配
# 总投资 100 万元，三种投资选项
# 目标：最小化风险 = 5*A + 4*B + 6*C
# 约束：
#   A + B + C = 100 (总投资额)
#   B >= 20 (固定收益最低配置)
#   A <= 40 (某类资产上限)
#   A >= 0, C >= 0

c_risk = [5, 4, 6]  # 风险系数

# 等式约束 A_eq @ x = b_eq
A_eq = [[1, 1, 1]]
b_eq = [100]

# 不等式约束
A_ub_risk = [[0, -1, 0],   # -B <= -20  => B >= 20
             [1, 0, 0]]    # A <= 40
b_ub_risk = [-20, 40]

bounds_risk = [(0, None), (None, None), (0, None)]

result_risk = optimize.linprog(c_risk, A_ub=A_ub_risk, b_ub=b_ub_risk,
                                A_eq=A_eq, b_eq=b_eq, bounds=bounds_risk,
                                method='highs')

print(f"\n投资组合优化问题:")
print(f"优化状态: {result_risk.message}")
print(f"资产 A: {result_risk.x[0]:.2f} 万元")
print(f"资产 B: {result_risk.x[1]:.2f} 万元")
print(f"资产 C: {result_risk.x[2]:.2f} 万元")
print(f"最小风险值: {result_risk.fun:.2f}")

# 3. 运输问题
# 两个仓库向三个客户配送，最小化运输成本
# 仓库容量: W1=50, W2=60
# 客户需求: C1=30, C2=40, C3=40
# 运输成本矩阵:
#         C1  C2  C3
#   W1    2   4   5
#   W2    3   1   6

# 决策变量: x11, x12, x13, x21, x22, x23
c_transport = [2, 4, 5, 3, 1, 6]  # 成本向量化

# 供应约束（每个仓库的总出货量 <= 容量）
A_supply = [[1, 1, 1, 0, 0, 0],  # W1
            [0, 0, 0, 1, 1, 1]]  # W2
b_supply = [50, 60]

# 需求约束（每个客户的总接收量 >= 需求）
A_demand = [[-1, 0, 0, -1, 0, 0],  # C1 (负号因 linprog 用 <=)
            [0, -1, 0, 0, -1, 0],  # C2
            [0, 0, -1, 0, 0, -1]]  # C3
b_demand = [-30, -40, -40]

A_ub_trans = A_supply + A_demand  # 合并所有不等式
b_ub_trans = b_supply + b_demand

bounds_trans = [(0, None)] * 6  # 所有变量非负

result_trans = optimize.linprog(c_transport, A_ub=A_ub_trans, b_ub=b_ub_trans,
                                 bounds=bounds_trans, method='highs')

print(f"\n运输优化问题:")
print(f"优化状态: {result_trans.message}")
if result_trans.success:
    x = result_trans.x
    print(f"W1->C1: {x[0]:.0f}, W1->C2: {x[1]:.0f}, W1->C3: {x[2]:.0f}")
    print(f"W2->C1: {x[3]:.0f}, W2->C2: {x[4]:.0f}, W2->C3: {x[5]:.0f}")
    print(f"最小运输成本: {result_trans.fun:.2f}")

# 4. 灵敏度分析：观察约束变化对最优解的影响
# 分析原材料 1 的约束放宽对利润的影响
print(f"\n灵敏度分析 - 原材料 1 约束影响:")
for rhs in [80, 90, 100, 110, 120]:
    result_sens = optimize.linprog(
        c, A_ub=A_ub, b_ub=[rhs, 80], bounds=bounds, method='highs'
    )
    if result_sens.success:
        print(f"  原材料 1 = {rhs:3d}: 利润 = {-result_sens.fun:7.2f} (A={result_sens.x[0]:.2f}, B={result_sens.x[1]:.2f})")

# 5. 不同求解方法对比
print(f"\n不同求解器对比:")
methods = ['highs', 'highs-ds', 'highs-ipm', 'interior-point']
for method in methods:
    try:
        res = optimize.linprog(c, A_ub=A_ub, b_ub=b_ub, bounds=bounds, method=method)
        print(f"  {method:15s}: 状态={res.status}, 目标值={-res.fun:.2f}, 迭代={res.nit if hasattr(res, 'nit') else 'N/A'}")
    except Exception as e:
        print(f"  {method:15s}: 失败 - {str(e)[:30]}")

# 6. 混合整数线性规划（使用 pulp 模拟，仅演示思路）
# 如果某些变量必须是整数，可以使用整数规划
# 此处用四舍五入展示整数解与连续解的差异
print(f"\n整数解近似 (连续解取整):")
x_cont = result.x
x_int = np.round(x_cont).astype(int)
# 验证整数解的可行性
constraints_satisfied = (
    x_int[0] >= 0 and x_int[1] >= 0 and
    2*x_int[0] + x_int[1] <= 100 and
    x_int[0] + 2*x_int[1] <= 80
)
profit_int = 40 * x_int[0] + 30 * x_int[1]
print(f"  连续解: A={x_cont[0]:.2f}, B={x_cont[1]:.2f}, 利润={-result.fun:.2f}")
print(f"  整数近似: A={x_int[0]}, B={x_int[1]}, 利润={profit_int}, 可行={constraints_satisfied}")

# 7. 多目标线性规划（权重法）
# 将多个目标加权求和转化为单目标
# 同时考虑利润最大化和风险最小化
profit_coeff = [40, 30]
risk_coeff = [5, 4]  # 简化风险系数
# 组合目标: max (profit - lambda * risk)
lam = 0.1  # 风险厌恶系数
combined_c = [-(profit_coeff[0] - lam * risk_coeff[0]),
              -(profit_coeff[1] - lam * risk_coeff[1])]

result_combined = optimize.linprog(combined_c, A_ub=A_ub, b_ub=b_ub,
                                    bounds=bounds, method='highs')
profit_val = 40 * result_combined.x[0] + 30 * result_combined.x[1]
risk_val = 5 * result_combined.x[0] + 4 * result_combined.x[1]
print(f"\n多目标优化 (λ={lam}):")
print(f"  A={result_combined.x[0]:.2f}, B={result_combined.x[1]:.2f}")
print(f"  利润={profit_val:.2f}, 风险={risk_val:.2f}")

print("\n线性规划总结：linprog 提供高效的大规模线性规划求解")
print("关键在于将实际问题转化为标准形式，并理解对偶变量（影子价格）")

驮掠吮肪乃汕逃频善济字屹鞠侥篮

m.wqt.cccnt.cn/84206.Doc
m.wqt.cccnt.cn/15197.Doc
m.wqt.cccnt.cn/86046.Doc
m.wqt.cccnt.cn/02664.Doc
m.wqt.cccnt.cn/60684.Doc
m.wqt.cccnt.cn/20860.Doc
m.wqt.cccnt.cn/66404.Doc
m.wqt.cccnt.cn/88644.Doc
m.wqt.cccnt.cn/13913.Doc
m.wqt.cccnt.cn/64848.Doc
m.wqt.cccnt.cn/02480.Doc
m.wqt.cccnt.cn/48600.Doc
m.wqt.cccnt.cn/95797.Doc
m.wqt.cccnt.cn/28220.Doc
m.wqt.cccnt.cn/99115.Doc
m.wqt.cccnt.cn/42060.Doc
m.wqt.cccnt.cn/66468.Doc
m.wqt.cccnt.cn/46206.Doc
m.wqt.cccnt.cn/08040.Doc
m.wqt.cccnt.cn/62224.Doc
m.wqr.cccnt.cn/86860.Doc
m.wqr.cccnt.cn/35739.Doc
m.wqr.cccnt.cn/66028.Doc
m.wqr.cccnt.cn/77339.Doc
m.wqr.cccnt.cn/08666.Doc
m.wqr.cccnt.cn/42428.Doc
m.wqr.cccnt.cn/46440.Doc
m.wqr.cccnt.cn/79591.Doc
m.wqr.cccnt.cn/97973.Doc
m.wqr.cccnt.cn/04222.Doc
m.wqr.cccnt.cn/46840.Doc
m.wqr.cccnt.cn/88048.Doc
m.wqr.cccnt.cn/42860.Doc
m.wqr.cccnt.cn/86482.Doc
m.wqr.cccnt.cn/08022.Doc
m.wqr.cccnt.cn/40680.Doc
m.wqr.cccnt.cn/24268.Doc
m.wqr.cccnt.cn/51131.Doc
m.wqr.cccnt.cn/53139.Doc
m.wqr.cccnt.cn/24208.Doc
m.wqe.cccnt.cn/68486.Doc
m.wqe.cccnt.cn/64084.Doc
m.wqe.cccnt.cn/22220.Doc
m.wqe.cccnt.cn/02840.Doc
m.wqe.cccnt.cn/46646.Doc
m.wqe.cccnt.cn/82620.Doc
m.wqe.cccnt.cn/48662.Doc
m.wqe.cccnt.cn/95979.Doc
m.wqe.cccnt.cn/73951.Doc
m.wqe.cccnt.cn/39715.Doc
m.wqe.cccnt.cn/82842.Doc
m.wqe.cccnt.cn/20202.Doc
m.wqe.cccnt.cn/66808.Doc
m.wqe.cccnt.cn/40284.Doc
m.wqe.cccnt.cn/15379.Doc
m.wqe.cccnt.cn/46246.Doc
m.wqe.cccnt.cn/15173.Doc
m.wqe.cccnt.cn/22062.Doc
m.wqe.cccnt.cn/80260.Doc
m.wqe.cccnt.cn/73995.Doc
m.wqw.cccnt.cn/37133.Doc
m.wqw.cccnt.cn/60424.Doc
m.wqw.cccnt.cn/66680.Doc
m.wqw.cccnt.cn/08822.Doc
m.wqw.cccnt.cn/48048.Doc
m.wqw.cccnt.cn/31175.Doc
m.wqw.cccnt.cn/42242.Doc
m.wqw.cccnt.cn/00008.Doc
m.wqw.cccnt.cn/33311.Doc
m.wqw.cccnt.cn/31595.Doc
m.wqw.cccnt.cn/95979.Doc
m.wqw.cccnt.cn/84064.Doc
m.wqw.cccnt.cn/68048.Doc
m.wqw.cccnt.cn/99515.Doc
m.wqw.cccnt.cn/48480.Doc
m.wqw.cccnt.cn/48288.Doc
m.wqw.cccnt.cn/40606.Doc
m.wqw.cccnt.cn/28408.Doc
m.wqw.cccnt.cn/88246.Doc
m.wqw.cccnt.cn/22662.Doc
m.wqq.cccnt.cn/88804.Doc
m.wqq.cccnt.cn/60282.Doc
m.wqq.cccnt.cn/71577.Doc
m.wqq.cccnt.cn/08242.Doc
m.wqq.cccnt.cn/06226.Doc
m.wqq.cccnt.cn/04882.Doc
m.wqq.cccnt.cn/51399.Doc
m.wqq.cccnt.cn/26020.Doc
m.wqq.cccnt.cn/86002.Doc
m.wqq.cccnt.cn/08620.Doc
m.wqq.cccnt.cn/57177.Doc
m.wqq.cccnt.cn/62206.Doc
m.wqq.cccnt.cn/02006.Doc
m.wqq.cccnt.cn/06468.Doc
m.wqq.cccnt.cn/93735.Doc
m.wqq.cccnt.cn/86848.Doc
m.wqq.cccnt.cn/04886.Doc
m.wqq.cccnt.cn/22044.Doc
m.wqq.cccnt.cn/22844.Doc
m.wqq.cccnt.cn/51737.Doc
m.qmm.cccnt.cn/42062.Doc
m.qmm.cccnt.cn/57935.Doc
m.qmm.cccnt.cn/08060.Doc
m.qmm.cccnt.cn/02864.Doc
m.qmm.cccnt.cn/19775.Doc
m.qmm.cccnt.cn/19573.Doc
m.qmm.cccnt.cn/86088.Doc
m.qmm.cccnt.cn/88066.Doc
m.qmm.cccnt.cn/60640.Doc
m.qmm.cccnt.cn/46680.Doc
m.qmm.cccnt.cn/79351.Doc
m.qmm.cccnt.cn/82282.Doc
m.qmm.cccnt.cn/24068.Doc
m.qmm.cccnt.cn/22844.Doc
m.qmm.cccnt.cn/80222.Doc
m.qmm.cccnt.cn/42606.Doc
m.qmm.cccnt.cn/64860.Doc
m.qmm.cccnt.cn/66486.Doc
m.qmm.cccnt.cn/82448.Doc
m.qmm.cccnt.cn/86440.Doc
m.qmn.cccnt.cn/73915.Doc
m.qmn.cccnt.cn/59737.Doc
m.qmn.cccnt.cn/84648.Doc
m.qmn.cccnt.cn/02468.Doc
m.qmn.cccnt.cn/44202.Doc
m.qmn.cccnt.cn/22842.Doc
m.qmn.cccnt.cn/08048.Doc
m.qmn.cccnt.cn/60228.Doc
m.qmn.cccnt.cn/82242.Doc
m.qmn.cccnt.cn/88066.Doc
m.qmn.cccnt.cn/44686.Doc
m.qmn.cccnt.cn/06666.Doc
m.qmn.cccnt.cn/46026.Doc
m.qmn.cccnt.cn/64804.Doc
m.qmn.cccnt.cn/26688.Doc
m.qmn.cccnt.cn/19179.Doc
m.qmn.cccnt.cn/42606.Doc
m.qmn.cccnt.cn/55979.Doc
m.qmn.cccnt.cn/88222.Doc
m.qmn.cccnt.cn/68282.Doc
m.qmb.cccnt.cn/73133.Doc
m.qmb.cccnt.cn/04404.Doc
m.qmb.cccnt.cn/46602.Doc
m.qmb.cccnt.cn/26462.Doc
m.qmb.cccnt.cn/35137.Doc
m.qmb.cccnt.cn/71199.Doc
m.qmb.cccnt.cn/15799.Doc
m.qmb.cccnt.cn/51919.Doc
m.qmb.cccnt.cn/62402.Doc
m.qmb.cccnt.cn/80842.Doc
m.qmb.cccnt.cn/88640.Doc
m.qmb.cccnt.cn/24028.Doc
m.qmb.cccnt.cn/40426.Doc
m.qmb.cccnt.cn/19155.Doc
m.qmb.cccnt.cn/28468.Doc
m.qmb.cccnt.cn/88648.Doc
m.qmb.cccnt.cn/79391.Doc
m.qmb.cccnt.cn/88026.Doc
m.qmb.cccnt.cn/31535.Doc
m.qmb.cccnt.cn/64068.Doc
m.qmv.cccnt.cn/19351.Doc
m.qmv.cccnt.cn/60622.Doc
m.qmv.cccnt.cn/62022.Doc
m.qmv.cccnt.cn/20806.Doc
m.qmv.cccnt.cn/84428.Doc
m.qmv.cccnt.cn/00840.Doc
m.qmv.cccnt.cn/31179.Doc
m.qmv.cccnt.cn/46482.Doc
m.qmv.cccnt.cn/86420.Doc
m.qmv.cccnt.cn/04820.Doc
m.qmv.cccnt.cn/26400.Doc
m.qmv.cccnt.cn/53715.Doc
m.qmv.cccnt.cn/88484.Doc
m.qmv.cccnt.cn/82864.Doc
m.qmv.cccnt.cn/37359.Doc
m.qmv.cccnt.cn/44062.Doc
m.qmv.cccnt.cn/64028.Doc
m.qmv.cccnt.cn/26644.Doc
m.qmv.cccnt.cn/46486.Doc
m.qmv.cccnt.cn/48486.Doc
m.qmc.cccnt.cn/13797.Doc
m.qmc.cccnt.cn/39915.Doc
m.qmc.cccnt.cn/80240.Doc
m.qmc.cccnt.cn/02608.Doc
m.qmc.cccnt.cn/44400.Doc
m.qmc.cccnt.cn/28000.Doc
m.qmc.cccnt.cn/24882.Doc
m.qmc.cccnt.cn/42800.Doc
m.qmc.cccnt.cn/48464.Doc
m.qmc.cccnt.cn/42462.Doc
m.qmc.cccnt.cn/24684.Doc
m.qmc.cccnt.cn/33513.Doc
m.qmc.cccnt.cn/02002.Doc
m.qmc.cccnt.cn/20488.Doc
m.qmc.cccnt.cn/13135.Doc
m.qmc.cccnt.cn/95191.Doc
m.qmc.cccnt.cn/24262.Doc
m.qmc.cccnt.cn/40028.Doc
m.qmc.cccnt.cn/42044.Doc
m.qmc.cccnt.cn/82004.Doc
m.qmx.cccnt.cn/99973.Doc
m.qmx.cccnt.cn/84066.Doc
m.qmx.cccnt.cn/24260.Doc
m.qmx.cccnt.cn/20602.Doc
m.qmx.cccnt.cn/46408.Doc
m.qmx.cccnt.cn/91953.Doc
m.qmx.cccnt.cn/40224.Doc
m.qmx.cccnt.cn/35919.Doc
m.qmx.cccnt.cn/86000.Doc
m.qmx.cccnt.cn/44602.Doc
m.qmx.cccnt.cn/80882.Doc
m.qmx.cccnt.cn/44020.Doc
m.qmx.cccnt.cn/82484.Doc
m.qmx.cccnt.cn/28862.Doc
m.qmx.cccnt.cn/02002.Doc
m.qmx.cccnt.cn/64002.Doc
m.qmx.cccnt.cn/88680.Doc
m.qmx.cccnt.cn/08622.Doc
m.qmx.cccnt.cn/53939.Doc
m.qmx.cccnt.cn/20644.Doc
m.qmz.cccnt.cn/88828.Doc
m.qmz.cccnt.cn/08282.Doc
m.qmz.cccnt.cn/95199.Doc
m.qmz.cccnt.cn/00404.Doc
m.qmz.cccnt.cn/06688.Doc
m.qmz.cccnt.cn/28624.Doc
m.qmz.cccnt.cn/82066.Doc
m.qmz.cccnt.cn/28008.Doc
m.qmz.cccnt.cn/17157.Doc
m.qmz.cccnt.cn/88040.Doc
m.qmz.cccnt.cn/37191.Doc
m.qmz.cccnt.cn/37171.Doc
m.qmz.cccnt.cn/48208.Doc
m.qmz.cccnt.cn/84444.Doc
m.qmz.cccnt.cn/66424.Doc
m.qmz.cccnt.cn/66200.Doc
m.qmz.cccnt.cn/99357.Doc
m.qmz.cccnt.cn/08446.Doc
m.qmz.cccnt.cn/06884.Doc
m.qmz.cccnt.cn/60262.Doc
m.qml.cccnt.cn/40086.Doc
m.qml.cccnt.cn/44044.Doc
m.qml.cccnt.cn/39755.Doc
m.qml.cccnt.cn/06224.Doc
m.qml.cccnt.cn/48886.Doc
m.qml.cccnt.cn/82688.Doc
m.qml.cccnt.cn/24442.Doc
m.qml.cccnt.cn/91179.Doc
m.qml.cccnt.cn/88444.Doc
m.qml.cccnt.cn/68266.Doc
m.qml.cccnt.cn/88024.Doc
m.qml.cccnt.cn/53139.Doc
m.qml.cccnt.cn/08602.Doc
m.qml.cccnt.cn/28044.Doc
m.qml.cccnt.cn/53339.Doc
m.qml.cccnt.cn/82622.Doc
m.qml.cccnt.cn/46006.Doc
m.qml.cccnt.cn/35731.Doc
m.qml.cccnt.cn/22846.Doc
m.qml.cccnt.cn/68246.Doc
m.qmk.cccnt.cn/64246.Doc
m.qmk.cccnt.cn/68202.Doc
m.qmk.cccnt.cn/88080.Doc
m.qmk.cccnt.cn/28004.Doc
m.qmk.cccnt.cn/15751.Doc
m.qmk.cccnt.cn/17117.Doc
m.qmk.cccnt.cn/20462.Doc
m.qmk.cccnt.cn/84608.Doc
m.qmk.cccnt.cn/22262.Doc
m.qmk.cccnt.cn/04822.Doc
m.qmk.cccnt.cn/00868.Doc
m.qmk.cccnt.cn/44642.Doc
m.qmk.cccnt.cn/62644.Doc
m.qmk.cccnt.cn/19371.Doc
m.qmk.cccnt.cn/19177.Doc
m.qmk.cccnt.cn/06682.Doc
m.qmk.cccnt.cn/68020.Doc
m.qmk.cccnt.cn/46444.Doc
m.qmk.cccnt.cn/33917.Doc
m.qmk.cccnt.cn/71933.Doc
m.qmj.cccnt.cn/42020.Doc
m.qmj.cccnt.cn/22664.Doc
m.qmj.cccnt.cn/44822.Doc
m.qmj.cccnt.cn/73315.Doc
m.qmj.cccnt.cn/88488.Doc
m.qmj.cccnt.cn/84626.Doc
m.qmj.cccnt.cn/02624.Doc
m.qmj.cccnt.cn/00600.Doc
m.qmj.cccnt.cn/84860.Doc
m.qmj.cccnt.cn/95199.Doc
m.qmj.cccnt.cn/20244.Doc
m.qmj.cccnt.cn/62402.Doc
m.qmj.cccnt.cn/20266.Doc
m.qmj.cccnt.cn/42224.Doc
m.qmj.cccnt.cn/62682.Doc
m.qmj.cccnt.cn/77915.Doc
m.qmj.cccnt.cn/46080.Doc
m.qmj.cccnt.cn/11717.Doc
m.qmj.cccnt.cn/24600.Doc
m.qmj.cccnt.cn/42842.Doc
m.qmh.cccnt.cn/88628.Doc
m.qmh.cccnt.cn/13173.Doc
m.qmh.cccnt.cn/40642.Doc
m.qmh.cccnt.cn/40402.Doc
m.qmh.cccnt.cn/04826.Doc
m.qmh.cccnt.cn/71957.Doc
m.qmh.cccnt.cn/24606.Doc
m.qmh.cccnt.cn/82266.Doc
m.qmh.cccnt.cn/82880.Doc
m.qmh.cccnt.cn/77397.Doc
m.qmh.cccnt.cn/66248.Doc
m.qmh.cccnt.cn/26422.Doc
m.qmh.cccnt.cn/48220.Doc
m.qmh.cccnt.cn/39931.Doc
m.qmh.cccnt.cn/40468.Doc
m.qmh.cccnt.cn/66460.Doc
m.qmh.cccnt.cn/06288.Doc
m.qmh.cccnt.cn/44464.Doc
m.qmh.cccnt.cn/06028.Doc
m.qmh.cccnt.cn/84264.Doc
m.qmg.cccnt.cn/06402.Doc
m.qmg.cccnt.cn/97751.Doc
m.qmg.cccnt.cn/00024.Doc
m.qmg.cccnt.cn/53319.Doc
m.qmg.cccnt.cn/04808.Doc
m.qmg.cccnt.cn/68886.Doc
m.qmg.cccnt.cn/66404.Doc
m.qmg.cccnt.cn/99775.Doc
m.qmg.cccnt.cn/93797.Doc
m.qmg.cccnt.cn/19931.Doc
m.qmg.cccnt.cn/42286.Doc
m.qmg.cccnt.cn/79715.Doc
m.qmg.cccnt.cn/84822.Doc
m.qmg.cccnt.cn/68664.Doc
m.qmg.cccnt.cn/80628.Doc
m.qmg.cccnt.cn/00482.Doc
m.qmg.cccnt.cn/00008.Doc
m.qmg.cccnt.cn/35115.Doc
m.qmg.cccnt.cn/68246.Doc
m.qmg.cccnt.cn/28860.Doc
m.qmf.cccnt.cn/11537.Doc
m.qmf.cccnt.cn/80620.Doc
m.qmf.cccnt.cn/26046.Doc
m.qmf.cccnt.cn/62226.Doc
m.qmf.cccnt.cn/86864.Doc
m.qmf.cccnt.cn/20204.Doc
m.qmf.cccnt.cn/11195.Doc
m.qmf.cccnt.cn/48848.Doc
m.qmf.cccnt.cn/06620.Doc
m.qmf.cccnt.cn/86086.Doc
m.qmf.cccnt.cn/66264.Doc
m.qmf.cccnt.cn/22682.Doc
m.qmf.cccnt.cn/35331.Doc
m.qmf.cccnt.cn/20068.Doc
m.qmf.cccnt.cn/20688.Doc
m.qmf.cccnt.cn/91397.Doc
m.qmf.cccnt.cn/02446.Doc
m.qmf.cccnt.cn/68886.Doc
m.qmf.cccnt.cn/42806.Doc
m.qmf.cccnt.cn/64806.Doc
m.qmd.cccnt.cn/73919.Doc
m.qmd.cccnt.cn/17313.Doc
m.qmd.cccnt.cn/40802.Doc
m.qmd.cccnt.cn/46204.Doc
m.qmd.cccnt.cn/13335.Doc
m.qmd.cccnt.cn/22020.Doc
m.qmd.cccnt.cn/62002.Doc
m.qmd.cccnt.cn/04866.Doc
m.qmd.cccnt.cn/64060.Doc
m.qmd.cccnt.cn/24268.Doc
m.qmd.cccnt.cn/15755.Doc
m.qmd.cccnt.cn/46866.Doc
m.qmd.cccnt.cn/06060.Doc
m.qmd.cccnt.cn/48844.Doc
m.qmd.cccnt.cn/42480.Doc
m.qmd.cccnt.cn/75739.Doc
m.qmd.cccnt.cn/28626.Doc
m.qmd.cccnt.cn/00604.Doc
m.qmd.cccnt.cn/08448.Doc
m.qmd.cccnt.cn/02826.Doc
m.qms.cccnt.cn/93337.Doc
m.qms.cccnt.cn/64248.Doc
m.qms.cccnt.cn/51197.Doc
m.qms.cccnt.cn/22848.Doc
m.qms.cccnt.cn/88462.Doc
m.qms.cccnt.cn/66602.Doc
m.qms.cccnt.cn/44448.Doc
m.qms.cccnt.cn/26886.Doc
m.qms.cccnt.cn/06440.Doc
m.qms.cccnt.cn/28402.Doc
m.qms.cccnt.cn/77133.Doc
m.qms.cccnt.cn/80446.Doc
m.qms.cccnt.cn/95999.Doc
m.qms.cccnt.cn/02426.Doc
m.qms.cccnt.cn/80260.Doc
