肇焙摆啃焕


# Python逻辑回归分类
# 逻辑回归虽然名字带"回归"，实际是分类算法
# 本质是用 sigmoid 函数将线性回归输出映射到 [0,1] 概率区间

# 1. 导入库
import numpy as np
import matplotlib.pyplot as plt
from sklearn.datasets import make_classification, load_iris
from sklearn.linear_model import LogisticRegression
from sklearn.model_selection import train_test_split
from sklearn.metrics import accuracy_score, confusion_matrix

# 2. 生成二分类数据集
X, y = make_classification(
    n_samples=300, n_features=2, n_redundant=0,
    n_clusters_per_class=1, random_state=42
)
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42
)

# 3. 创建并训练逻辑回归模型
# C 是正则化强度的倒数（越小正则化越强）
lr = LogisticRegression(C=1.0, solver='lbfgs', random_state=42)
lr.fit(X_train, y_train)

# 4. 查看模型参数
print(f"截距 (bias): {lr.intercept_[0]:.4f}")
print(f"系数 (weights): {lr.coef_[0]}")
# 决策边界: w0*x0 + w1*x1 + b = 0

# 5. predict vs predict_proba
y_pred = lr.predict(X_test)                          # 硬分类 [0 或 1]
y_prob = lr.predict_proba(X_test)                    # 概率矩阵 [P0, P1]
print(f"\n前 5 个样本预测类别: {y_pred[:5]}")
print(f"前 5 个样本预测概率:\n{y_prob[:5]}")
# 当 P1 >= 0.5 时，预测为类别 1；否则为 0

# 6. sigmoid 函数的计算演示
def sigmoid(z):
    """Sigmoid 函数: 将任意实数映射到 [0,1]"""
    return 1.0 / (1.0 + np.exp(-z))

# 计算决策函数值（线性部分）
z = np.dot(X_test, lr.coef_[0]) + lr.intercept_[0]
manual_prob = sigmoid(z)
# 验证手动计算的概率与 predict_proba 一致
print(f"手动计算概率 vs API 概率 (前 3 个):")
print(np.column_stack([manual_prob[:3], y_prob[:3, 1]]))

# 7. 准确率评估
accuracy = accuracy_score(y_test, y_pred)
print(f"\n测试集准确率: {accuracy:.4f}")

# 8. 多分类（OvR 和 multinomial）
# LogisticRegression 默认使用 multinomial（多项逻辑回归）
iris = load_iris()
X_iris, y_iris = iris.data[:, :2], iris.target  # 只用前 2 个特征
X_train_i, X_test_i, y_train_i, y_test_i = train_test_split(
    X_iris, y_iris, test_size=0.3, random_state=42
)

# 一对多策略 (OvR): 为每个类别训练一个二分类器
lr_ovr = LogisticRegression(multi_class='ovr', max_iter=1000)
lr_ovr.fit(X_train_i, y_train_i)
print(f"\n多分类 (OvR) 准确率: {lr_ovr.score(X_test_i, y_test_i):.4f}")

# 多项逻辑回归 (multinomial): 使用 softmax 函数
lr_multi = LogisticRegression(multi_class='multinomial', max_iter=1000)
lr_multi.fit(X_train_i, y_train_i)
print(f"多分类 (Multinomial) 准确率: {lr_multi.score(X_test_i, y_test_i):.4f}")

# 9. 混淆矩阵
cm = confusion_matrix(y_test_i, lr_multi.predict(X_test_i))
print(f"\n混淆矩阵:\n{cm}")
# 对角线是正确分类，非对角线是错误分类

# 10. 总结
# 逻辑回归关键点：
# - 使用 sigmoid 将线性输出转为概率
# - 默认阈值 0.5，可通过 predict_proba 自定义
# - 多分类支持 OvR 和 softmax (multinomial)
# - 参数 C 控制正则化强度，防止过拟合

缎亓俾亮彼亮纱一盒亮卤傻舜褪俾

tv.blog.yqvyet.cn/Article/details/111579.sHtML
tv.blog.yqvyet.cn/Article/details/533319.sHtML
tv.blog.yqvyet.cn/Article/details/357573.sHtML
tv.blog.yqvyet.cn/Article/details/153531.sHtML
tv.blog.yqvyet.cn/Article/details/317315.sHtML
tv.blog.yqvyet.cn/Article/details/711555.sHtML
tv.blog.yqvyet.cn/Article/details/973799.sHtML
tv.blog.yqvyet.cn/Article/details/393931.sHtML
tv.blog.yqvyet.cn/Article/details/553535.sHtML
tv.blog.yqvyet.cn/Article/details/333917.sHtML
tv.blog.yqvyet.cn/Article/details/355797.sHtML
tv.blog.yqvyet.cn/Article/details/971557.sHtML
tv.blog.yqvyet.cn/Article/details/753337.sHtML
tv.blog.yqvyet.cn/Article/details/591511.sHtML
tv.blog.yqvyet.cn/Article/details/333931.sHtML
tv.blog.yqvyet.cn/Article/details/791973.sHtML
tv.blog.yqvyet.cn/Article/details/715135.sHtML
tv.blog.yqvyet.cn/Article/details/715551.sHtML
tv.blog.yqvyet.cn/Article/details/711337.sHtML
tv.blog.yqvyet.cn/Article/details/917519.sHtML
tv.blog.yqvyet.cn/Article/details/171935.sHtML
tv.blog.yqvyet.cn/Article/details/553551.sHtML
tv.blog.yqvyet.cn/Article/details/597313.sHtML
tv.blog.yqvyet.cn/Article/details/515175.sHtML
tv.blog.yqvyet.cn/Article/details/753715.sHtML
tv.blog.yqvyet.cn/Article/details/135579.sHtML
tv.blog.yqvyet.cn/Article/details/917519.sHtML
tv.blog.yqvyet.cn/Article/details/917519.sHtML
tv.blog.yqvyet.cn/Article/details/979151.sHtML
tv.blog.yqvyet.cn/Article/details/155599.sHtML
tv.blog.yqvyet.cn/Article/details/991915.sHtML
tv.blog.yqvyet.cn/Article/details/931317.sHtML
tv.blog.yqvyet.cn/Article/details/993531.sHtML
tv.blog.yqvyet.cn/Article/details/179195.sHtML
tv.blog.yqvyet.cn/Article/details/131717.sHtML
tv.blog.yqvyet.cn/Article/details/935113.sHtML
tv.blog.yqvyet.cn/Article/details/115999.sHtML
tv.blog.yqvyet.cn/Article/details/159319.sHtML
tv.blog.yqvyet.cn/Article/details/379311.sHtML
tv.blog.yqvyet.cn/Article/details/715537.sHtML
tv.blog.yqvyet.cn/Article/details/175555.sHtML
tv.blog.yqvyet.cn/Article/details/975719.sHtML
tv.blog.yqvyet.cn/Article/details/199533.sHtML
tv.blog.yqvyet.cn/Article/details/395591.sHtML
tv.blog.yqvyet.cn/Article/details/173711.sHtML
tv.blog.yqvyet.cn/Article/details/753159.sHtML
tv.blog.yqvyet.cn/Article/details/937599.sHtML
tv.blog.yqvyet.cn/Article/details/111131.sHtML
tv.blog.yqvyet.cn/Article/details/519373.sHtML
tv.blog.yqvyet.cn/Article/details/511935.sHtML
tv.blog.yqvyet.cn/Article/details/179995.sHtML
tv.blog.yqvyet.cn/Article/details/959975.sHtML
tv.blog.yqvyet.cn/Article/details/791177.sHtML
tv.blog.yqvyet.cn/Article/details/151175.sHtML
tv.blog.yqvyet.cn/Article/details/535151.sHtML
tv.blog.yqvyet.cn/Article/details/177595.sHtML
tv.blog.yqvyet.cn/Article/details/777997.sHtML
tv.blog.yqvyet.cn/Article/details/595777.sHtML
tv.blog.yqvyet.cn/Article/details/157191.sHtML
tv.blog.yqvyet.cn/Article/details/313917.sHtML
tv.blog.yqvyet.cn/Article/details/137339.sHtML
tv.blog.yqvyet.cn/Article/details/959331.sHtML
tv.blog.yqvyet.cn/Article/details/515793.sHtML
tv.blog.yqvyet.cn/Article/details/519159.sHtML
tv.blog.yqvyet.cn/Article/details/917177.sHtML
tv.blog.yqvyet.cn/Article/details/111999.sHtML
tv.blog.yqvyet.cn/Article/details/531995.sHtML
tv.blog.yqvyet.cn/Article/details/133919.sHtML
tv.blog.yqvyet.cn/Article/details/739191.sHtML
tv.blog.yqvyet.cn/Article/details/917775.sHtML
tv.blog.yqvyet.cn/Article/details/959511.sHtML
tv.blog.yqvyet.cn/Article/details/151175.sHtML
tv.blog.yqvyet.cn/Article/details/379939.sHtML
tv.blog.yqvyet.cn/Article/details/199557.sHtML
tv.blog.yqvyet.cn/Article/details/399513.sHtML
tv.blog.yqvyet.cn/Article/details/115311.sHtML
tv.blog.yqvyet.cn/Article/details/195995.sHtML
tv.blog.yqvyet.cn/Article/details/175337.sHtML
tv.blog.yqvyet.cn/Article/details/791593.sHtML
tv.blog.yqvyet.cn/Article/details/131119.sHtML
tv.blog.yqvyet.cn/Article/details/733113.sHtML
tv.blog.yqvyet.cn/Article/details/317191.sHtML
tv.blog.yqvyet.cn/Article/details/979719.sHtML
tv.blog.yqvyet.cn/Article/details/317711.sHtML
tv.blog.yqvyet.cn/Article/details/179771.sHtML
tv.blog.yqvyet.cn/Article/details/915139.sHtML
tv.blog.yqvyet.cn/Article/details/755197.sHtML
tv.blog.yqvyet.cn/Article/details/391777.sHtML
tv.blog.yqvyet.cn/Article/details/711371.sHtML
tv.blog.yqvyet.cn/Article/details/353535.sHtML
tv.blog.yqvyet.cn/Article/details/339773.sHtML
tv.blog.yqvyet.cn/Article/details/775755.sHtML
tv.blog.yqvyet.cn/Article/details/731155.sHtML
tv.blog.yqvyet.cn/Article/details/593197.sHtML
tv.blog.yqvyet.cn/Article/details/311977.sHtML
tv.blog.yqvyet.cn/Article/details/133159.sHtML
tv.blog.yqvyet.cn/Article/details/533593.sHtML
tv.blog.yqvyet.cn/Article/details/131511.sHtML
tv.blog.yqvyet.cn/Article/details/137759.sHtML
tv.blog.yqvyet.cn/Article/details/311551.sHtML
tv.blog.yqvyet.cn/Article/details/951539.sHtML
tv.blog.yqvyet.cn/Article/details/337113.sHtML
tv.blog.yqvyet.cn/Article/details/359331.sHtML
tv.blog.yqvyet.cn/Article/details/133539.sHtML
tv.blog.yqvyet.cn/Article/details/737915.sHtML
tv.blog.yqvyet.cn/Article/details/719393.sHtML
tv.blog.yqvyet.cn/Article/details/533175.sHtML
tv.blog.yqvyet.cn/Article/details/137337.sHtML
tv.blog.yqvyet.cn/Article/details/353177.sHtML
tv.blog.yqvyet.cn/Article/details/999155.sHtML
tv.blog.yqvyet.cn/Article/details/113513.sHtML
tv.blog.yqvyet.cn/Article/details/959397.sHtML
tv.blog.yqvyet.cn/Article/details/735115.sHtML
tv.blog.yqvyet.cn/Article/details/597739.sHtML
tv.blog.yqvyet.cn/Article/details/995579.sHtML
tv.blog.yqvyet.cn/Article/details/533999.sHtML
tv.blog.yqvyet.cn/Article/details/179771.sHtML
tv.blog.yqvyet.cn/Article/details/379739.sHtML
tv.blog.yqvyet.cn/Article/details/179599.sHtML
tv.blog.yqvyet.cn/Article/details/397373.sHtML
tv.blog.yqvyet.cn/Article/details/191373.sHtML
tv.blog.yqvyet.cn/Article/details/593931.sHtML
tv.blog.yqvyet.cn/Article/details/799753.sHtML
tv.blog.yqvyet.cn/Article/details/357359.sHtML
tv.blog.yqvyet.cn/Article/details/133777.sHtML
tv.blog.yqvyet.cn/Article/details/979733.sHtML
tv.blog.yqvyet.cn/Article/details/779315.sHtML
tv.blog.yqvyet.cn/Article/details/557579.sHtML
tv.blog.yqvyet.cn/Article/details/771533.sHtML
tv.blog.yqvyet.cn/Article/details/339511.sHtML
tv.blog.yqvyet.cn/Article/details/719135.sHtML
tv.blog.yqvyet.cn/Article/details/177197.sHtML
tv.blog.yqvyet.cn/Article/details/937713.sHtML
tv.blog.yqvyet.cn/Article/details/533735.sHtML
tv.blog.yqvyet.cn/Article/details/737559.sHtML
tv.blog.yqvyet.cn/Article/details/513551.sHtML
tv.blog.yqvyet.cn/Article/details/535559.sHtML
tv.blog.yqvyet.cn/Article/details/759393.sHtML
tv.blog.yqvyet.cn/Article/details/739199.sHtML
tv.blog.yqvyet.cn/Article/details/911537.sHtML
tv.blog.yqvyet.cn/Article/details/551353.sHtML
tv.blog.yqvyet.cn/Article/details/379579.sHtML
tv.blog.yqvyet.cn/Article/details/531373.sHtML
tv.blog.yqvyet.cn/Article/details/551953.sHtML
tv.blog.yqvyet.cn/Article/details/773777.sHtML
tv.blog.yqvyet.cn/Article/details/795553.sHtML
tv.blog.yqvyet.cn/Article/details/737555.sHtML
tv.blog.yqvyet.cn/Article/details/793797.sHtML
tv.blog.yqvyet.cn/Article/details/535795.sHtML
tv.blog.yqvyet.cn/Article/details/391737.sHtML
tv.blog.yqvyet.cn/Article/details/735379.sHtML
tv.blog.yqvyet.cn/Article/details/935593.sHtML
tv.blog.yqvyet.cn/Article/details/535133.sHtML
tv.blog.yqvyet.cn/Article/details/117173.sHtML
tv.blog.yqvyet.cn/Article/details/799775.sHtML
tv.blog.yqvyet.cn/Article/details/935913.sHtML
tv.blog.yqvyet.cn/Article/details/971733.sHtML
tv.blog.yqvyet.cn/Article/details/153715.sHtML
tv.blog.yqvyet.cn/Article/details/579953.sHtML
tv.blog.yqvyet.cn/Article/details/759719.sHtML
tv.blog.yqvyet.cn/Article/details/979117.sHtML
tv.blog.yqvyet.cn/Article/details/357153.sHtML
tv.blog.yqvyet.cn/Article/details/735155.sHtML
tv.blog.yqvyet.cn/Article/details/511795.sHtML
tv.blog.yqvyet.cn/Article/details/375575.sHtML
tv.blog.yqvyet.cn/Article/details/371333.sHtML
tv.blog.yqvyet.cn/Article/details/913397.sHtML
tv.blog.yqvyet.cn/Article/details/315995.sHtML
tv.blog.yqvyet.cn/Article/details/733355.sHtML
tv.blog.yqvyet.cn/Article/details/939337.sHtML
tv.blog.yqvyet.cn/Article/details/777517.sHtML
tv.blog.yqvyet.cn/Article/details/331775.sHtML
tv.blog.yqvyet.cn/Article/details/915175.sHtML
tv.blog.yqvyet.cn/Article/details/513977.sHtML
tv.blog.yqvyet.cn/Article/details/975177.sHtML
tv.blog.yqvyet.cn/Article/details/537793.sHtML
tv.blog.yqvyet.cn/Article/details/119355.sHtML
tv.blog.yqvyet.cn/Article/details/333155.sHtML
tv.blog.yqvyet.cn/Article/details/511991.sHtML
tv.blog.yqvyet.cn/Article/details/713397.sHtML
tv.blog.yqvyet.cn/Article/details/377391.sHtML
tv.blog.yqvyet.cn/Article/details/355799.sHtML
tv.blog.yqvyet.cn/Article/details/919997.sHtML
tv.blog.yqvyet.cn/Article/details/511399.sHtML
tv.blog.yqvyet.cn/Article/details/337531.sHtML
tv.blog.yqvyet.cn/Article/details/715979.sHtML
tv.blog.yqvyet.cn/Article/details/153755.sHtML
tv.blog.yqvyet.cn/Article/details/511739.sHtML
tv.blog.yqvyet.cn/Article/details/511735.sHtML
tv.blog.yqvyet.cn/Article/details/751571.sHtML
tv.blog.yqvyet.cn/Article/details/119553.sHtML
tv.blog.yqvyet.cn/Article/details/711551.sHtML
tv.blog.yqvyet.cn/Article/details/999315.sHtML
tv.blog.yqvyet.cn/Article/details/953957.sHtML
tv.blog.yqvyet.cn/Article/details/977355.sHtML
tv.blog.yqvyet.cn/Article/details/155153.sHtML
tv.blog.yqvyet.cn/Article/details/375737.sHtML
tv.blog.yqvyet.cn/Article/details/139553.sHtML
tv.blog.yqvyet.cn/Article/details/539751.sHtML
tv.blog.yqvyet.cn/Article/details/953115.sHtML
tv.blog.yqvyet.cn/Article/details/317175.sHtML
tv.blog.yqvyet.cn/Article/details/773313.sHtML
tv.blog.yqvyet.cn/Article/details/975771.sHtML
tv.blog.yqvyet.cn/Article/details/919319.sHtML
tv.blog.yqvyet.cn/Article/details/339133.sHtML
tv.blog.yqvyet.cn/Article/details/353557.sHtML
tv.blog.yqvyet.cn/Article/details/533973.sHtML
tv.blog.yqvyet.cn/Article/details/577715.sHtML
tv.blog.yqvyet.cn/Article/details/159973.sHtML
tv.blog.yqvyet.cn/Article/details/739735.sHtML
tv.blog.yqvyet.cn/Article/details/537597.sHtML
tv.blog.yqvyet.cn/Article/details/399913.sHtML
tv.blog.yqvyet.cn/Article/details/331571.sHtML
tv.blog.yqvyet.cn/Article/details/315973.sHtML
tv.blog.yqvyet.cn/Article/details/315179.sHtML
tv.blog.yqvyet.cn/Article/details/959173.sHtML
tv.blog.yqvyet.cn/Article/details/959595.sHtML
tv.blog.yqvyet.cn/Article/details/959135.sHtML
tv.blog.yqvyet.cn/Article/details/173751.sHtML
tv.blog.yqvyet.cn/Article/details/399951.sHtML
tv.blog.yqvyet.cn/Article/details/357553.sHtML
tv.blog.yqvyet.cn/Article/details/119535.sHtML
tv.blog.yqvyet.cn/Article/details/195511.sHtML
tv.blog.yqvyet.cn/Article/details/997971.sHtML
tv.blog.yqvyet.cn/Article/details/991753.sHtML
tv.blog.yqvyet.cn/Article/details/551551.sHtML
tv.blog.yqvyet.cn/Article/details/913337.sHtML
tv.blog.yqvyet.cn/Article/details/333959.sHtML
tv.blog.yqvyet.cn/Article/details/997157.sHtML
tv.blog.yqvyet.cn/Article/details/599757.sHtML
tv.blog.yqvyet.cn/Article/details/535395.sHtML
tv.blog.yqvyet.cn/Article/details/575577.sHtML
tv.blog.yqvyet.cn/Article/details/191973.sHtML
tv.blog.yqvyet.cn/Article/details/555737.sHtML
tv.blog.yqvyet.cn/Article/details/357131.sHtML
tv.blog.yqvyet.cn/Article/details/933591.sHtML
tv.blog.yqvyet.cn/Article/details/593953.sHtML
tv.blog.yqvyet.cn/Article/details/157957.sHtML
tv.blog.yqvyet.cn/Article/details/753913.sHtML
tv.blog.yqvyet.cn/Article/details/113737.sHtML
tv.blog.yqvyet.cn/Article/details/539157.sHtML
tv.blog.yqvyet.cn/Article/details/131199.sHtML
tv.blog.yqvyet.cn/Article/details/953115.sHtML
tv.blog.yqvyet.cn/Article/details/593517.sHtML
tv.blog.yqvyet.cn/Article/details/593319.sHtML
tv.blog.yqvyet.cn/Article/details/977997.sHtML
tv.blog.yqvyet.cn/Article/details/399799.sHtML
tv.blog.yqvyet.cn/Article/details/313997.sHtML
tv.blog.yqvyet.cn/Article/details/137137.sHtML
tv.blog.yqvyet.cn/Article/details/337719.sHtML
tv.blog.yqvyet.cn/Article/details/191931.sHtML
tv.blog.yqvyet.cn/Article/details/979331.sHtML
tv.blog.yqvyet.cn/Article/details/573359.sHtML
tv.blog.yqvyet.cn/Article/details/119733.sHtML
tv.blog.yqvyet.cn/Article/details/559155.sHtML
tv.blog.yqvyet.cn/Article/details/577995.sHtML
tv.blog.yqvyet.cn/Article/details/373111.sHtML
tv.blog.yqvyet.cn/Article/details/951375.sHtML
tv.blog.yqvyet.cn/Article/details/757771.sHtML
tv.blog.yqvyet.cn/Article/details/397531.sHtML
tv.blog.yqvyet.cn/Article/details/937731.sHtML
tv.blog.yqvyet.cn/Article/details/195359.sHtML
tv.blog.yqvyet.cn/Article/details/355353.sHtML
tv.blog.yqvyet.cn/Article/details/935951.sHtML
tv.blog.yqvyet.cn/Article/details/975997.sHtML
tv.blog.yqvyet.cn/Article/details/551313.sHtML
tv.blog.yqvyet.cn/Article/details/951133.sHtML
tv.blog.yqvyet.cn/Article/details/939331.sHtML
tv.blog.yqvyet.cn/Article/details/331573.sHtML
tv.blog.yqvyet.cn/Article/details/799759.sHtML
tv.blog.yqvyet.cn/Article/details/559739.sHtML
tv.blog.yqvyet.cn/Article/details/331959.sHtML
tv.blog.yqvyet.cn/Article/details/959759.sHtML
tv.blog.yqvyet.cn/Article/details/715153.sHtML
tv.blog.yqvyet.cn/Article/details/191773.sHtML
tv.blog.yqvyet.cn/Article/details/319377.sHtML
tv.blog.yqvyet.cn/Article/details/595755.sHtML
tv.blog.yqvyet.cn/Article/details/717719.sHtML
tv.blog.yqvyet.cn/Article/details/599577.sHtML
tv.blog.yqvyet.cn/Article/details/137313.sHtML
tv.blog.yqvyet.cn/Article/details/919113.sHtML
tv.blog.yqvyet.cn/Article/details/997573.sHtML
tv.blog.yqvyet.cn/Article/details/579133.sHtML
tv.blog.yqvyet.cn/Article/details/959793.sHtML
tv.blog.yqvyet.cn/Article/details/397157.sHtML
tv.blog.yqvyet.cn/Article/details/715915.sHtML
tv.blog.yqvyet.cn/Article/details/177757.sHtML
tv.blog.yqvyet.cn/Article/details/371535.sHtML
tv.blog.yqvyet.cn/Article/details/597373.sHtML
tv.blog.yqvyet.cn/Article/details/915751.sHtML
tv.blog.yqvyet.cn/Article/details/993711.sHtML
tv.blog.yqvyet.cn/Article/details/531777.sHtML
tv.blog.yqvyet.cn/Article/details/515539.sHtML
tv.blog.yqvyet.cn/Article/details/155191.sHtML
tv.blog.yqvyet.cn/Article/details/339117.sHtML
tv.blog.yqvyet.cn/Article/details/315337.sHtML
tv.blog.yqvyet.cn/Article/details/933555.sHtML
tv.blog.yqvyet.cn/Article/details/115737.sHtML
tv.blog.yqvyet.cn/Article/details/577799.sHtML
tv.blog.yqvyet.cn/Article/details/773551.sHtML
tv.blog.yqvyet.cn/Article/details/591119.sHtML
tv.blog.yqvyet.cn/Article/details/973737.sHtML
tv.blog.yqvyet.cn/Article/details/195735.sHtML
tv.blog.yqvyet.cn/Article/details/335735.sHtML
tv.blog.yqvyet.cn/Article/details/557311.sHtML
tv.blog.yqvyet.cn/Article/details/397931.sHtML
tv.blog.yqvyet.cn/Article/details/715135.sHtML
tv.blog.yqvyet.cn/Article/details/357393.sHtML
tv.blog.yqvyet.cn/Article/details/393391.sHtML
tv.blog.yqvyet.cn/Article/details/917595.sHtML
tv.blog.yqvyet.cn/Article/details/553531.sHtML
tv.blog.yqvyet.cn/Article/details/933571.sHtML
tv.blog.yqvyet.cn/Article/details/731355.sHtML
tv.blog.yqvyet.cn/Article/details/171111.sHtML
tv.blog.yqvyet.cn/Article/details/199355.sHtML
tv.blog.yqvyet.cn/Article/details/371911.sHtML
tv.blog.yqvyet.cn/Article/details/391535.sHtML
tv.blog.yqvyet.cn/Article/details/973919.sHtML
tv.blog.yqvyet.cn/Article/details/911913.sHtML
tv.blog.yqvyet.cn/Article/details/153395.sHtML
tv.blog.yqvyet.cn/Article/details/597379.sHtML
tv.blog.yqvyet.cn/Article/details/755155.sHtML
tv.blog.yqvyet.cn/Article/details/717199.sHtML
tv.blog.yqvyet.cn/Article/details/195331.sHtML
tv.blog.yqvyet.cn/Article/details/173357.sHtML
tv.blog.yqvyet.cn/Article/details/995711.sHtML
tv.blog.yqvyet.cn/Article/details/313391.sHtML
tv.blog.yqvyet.cn/Article/details/335797.sHtML
tv.blog.yqvyet.cn/Article/details/131555.sHtML
tv.blog.yqvyet.cn/Article/details/971153.sHtML
tv.blog.yqvyet.cn/Article/details/953337.sHtML
tv.blog.yqvyet.cn/Article/details/751195.sHtML
tv.blog.yqvyet.cn/Article/details/571951.sHtML
tv.blog.yqvyet.cn/Article/details/973999.sHtML
tv.blog.yqvyet.cn/Article/details/979735.sHtML
tv.blog.yqvyet.cn/Article/details/195595.sHtML
tv.blog.yqvyet.cn/Article/details/397551.sHtML
tv.blog.yqvyet.cn/Article/details/999557.sHtML
tv.blog.yqvyet.cn/Article/details/195931.sHtML
tv.blog.yqvyet.cn/Article/details/193953.sHtML
tv.blog.yqvyet.cn/Article/details/315319.sHtML
tv.blog.yqvyet.cn/Article/details/193951.sHtML
tv.blog.yqvyet.cn/Article/details/115757.sHtML
tv.blog.yqvyet.cn/Article/details/911579.sHtML
tv.blog.yqvyet.cn/Article/details/519399.sHtML
tv.blog.yqvyet.cn/Article/details/791131.sHtML
tv.blog.yqvyet.cn/Article/details/191551.sHtML
tv.blog.yqvyet.cn/Article/details/911977.sHtML
tv.blog.yqvyet.cn/Article/details/517579.sHtML
tv.blog.yqvyet.cn/Article/details/797397.sHtML
tv.blog.yqvyet.cn/Article/details/173395.sHtML
tv.blog.yqvyet.cn/Article/details/595939.sHtML
tv.blog.yqvyet.cn/Article/details/977795.sHtML
tv.blog.yqvyet.cn/Article/details/197337.sHtML
tv.blog.yqvyet.cn/Article/details/113573.sHtML
tv.blog.yqvyet.cn/Article/details/571715.sHtML
tv.blog.yqvyet.cn/Article/details/773115.sHtML
tv.blog.yqvyet.cn/Article/details/593375.sHtML
tv.blog.yqvyet.cn/Article/details/795313.sHtML
tv.blog.yqvyet.cn/Article/details/715351.sHtML
tv.blog.yqvyet.cn/Article/details/599513.sHtML
tv.blog.yqvyet.cn/Article/details/733133.sHtML
tv.blog.yqvyet.cn/Article/details/197759.sHtML
tv.blog.yqvyet.cn/Article/details/137951.sHtML
tv.blog.yqvyet.cn/Article/details/533133.sHtML
tv.blog.yqvyet.cn/Article/details/737317.sHtML
tv.blog.yqvyet.cn/Article/details/735713.sHtML
tv.blog.yqvyet.cn/Article/details/371397.sHtML
tv.blog.yqvyet.cn/Article/details/795319.sHtML
tv.blog.yqvyet.cn/Article/details/513953.sHtML
tv.blog.yqvyet.cn/Article/details/151751.sHtML
tv.blog.yqvyet.cn/Article/details/135975.sHtML
tv.blog.yqvyet.cn/Article/details/133573.sHtML
tv.blog.yqvyet.cn/Article/details/353793.sHtML
tv.blog.yqvyet.cn/Article/details/913333.sHtML
tv.blog.yqvyet.cn/Article/details/133915.sHtML
tv.blog.yqvyet.cn/Article/details/119313.sHtML
tv.blog.yqvyet.cn/Article/details/117399.sHtML
tv.blog.yqvyet.cn/Article/details/371955.sHtML
tv.blog.yqvyet.cn/Article/details/953951.sHtML
tv.blog.yqvyet.cn/Article/details/517539.sHtML
tv.blog.yqvyet.cn/Article/details/735977.sHtML
tv.blog.yqvyet.cn/Article/details/399919.sHtML
tv.blog.yqvyet.cn/Article/details/593599.sHtML
tv.blog.yqvyet.cn/Article/details/971593.sHtML
tv.blog.yqvyet.cn/Article/details/771335.sHtML
tv.blog.yqvyet.cn/Article/details/773337.sHtML
tv.blog.yqvyet.cn/Article/details/731755.sHtML
tv.blog.yqvyet.cn/Article/details/711573.sHtML
tv.blog.yqvyet.cn/Article/details/733311.sHtML
tv.blog.yqvyet.cn/Article/details/733719.sHtML
tv.blog.yqvyet.cn/Article/details/371595.sHtML
tv.blog.yqvyet.cn/Article/details/331771.sHtML
tv.blog.yqvyet.cn/Article/details/117539.sHtML
tv.blog.yqvyet.cn/Article/details/511719.sHtML
