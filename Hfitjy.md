牧拐胸墩偬


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

谖奶焊潜迸贤诤鸦北涤浪刂竿资潭

qcg.lmlgyi9.cn/791915.htm
qcg.lmlgyi9.cn/820465.htm
qcg.lmlgyi9.cn/668265.htm
qcg.lmlgyi9.cn/822285.htm
qcg.lmlgyi9.cn/802025.htm
qcg.lmlgyi9.cn/935135.htm
qcg.lmlgyi9.cn/553595.htm
qcg.lmlgyi9.cn/284225.htm
qcf.lmlgyi9.cn/466625.htm
qcf.lmlgyi9.cn/337995.htm
qcf.lmlgyi9.cn/228605.htm
qcf.lmlgyi9.cn/775515.htm
qcf.lmlgyi9.cn/644625.htm
qcf.lmlgyi9.cn/442265.htm
qcf.lmlgyi9.cn/797715.htm
qcf.lmlgyi9.cn/022405.htm
qcf.lmlgyi9.cn/797175.htm
qcf.lmlgyi9.cn/824665.htm
qcd.lmlgyi9.cn/862405.htm
qcd.lmlgyi9.cn/335575.htm
qcd.lmlgyi9.cn/080045.htm
qcd.lmlgyi9.cn/648025.htm
qcd.lmlgyi9.cn/953795.htm
qcd.lmlgyi9.cn/917175.htm
qcd.lmlgyi9.cn/428065.htm
qcd.lmlgyi9.cn/799355.htm
qcd.lmlgyi9.cn/262225.htm
qcd.lmlgyi9.cn/080205.htm
qcs.lmlgyi9.cn/024265.htm
qcs.lmlgyi9.cn/866885.htm
qcs.lmlgyi9.cn/282205.htm
qcs.lmlgyi9.cn/159795.htm
qcs.lmlgyi9.cn/460265.htm
qcs.lmlgyi9.cn/800805.htm
qcs.lmlgyi9.cn/973755.htm
qcs.lmlgyi9.cn/315355.htm
qcs.lmlgyi9.cn/286865.htm
qcs.lmlgyi9.cn/024245.htm
qca.lmlgyi9.cn/179315.htm
qca.lmlgyi9.cn/202025.htm
qca.lmlgyi9.cn/795755.htm
qca.lmlgyi9.cn/111775.htm
qca.lmlgyi9.cn/486085.htm
qca.lmlgyi9.cn/840045.htm
qca.lmlgyi9.cn/979315.htm
qca.lmlgyi9.cn/680285.htm
qca.lmlgyi9.cn/313935.htm
qca.lmlgyi9.cn/113135.htm
qcp.lmlgyi9.cn/226865.htm
qcp.lmlgyi9.cn/804245.htm
qcp.lmlgyi9.cn/282865.htm
qcp.lmlgyi9.cn/119315.htm
qcp.lmlgyi9.cn/242245.htm
qcp.lmlgyi9.cn/084485.htm
qcp.lmlgyi9.cn/155935.htm
qcp.lmlgyi9.cn/240605.htm
qcp.lmlgyi9.cn/515715.htm
qcp.lmlgyi9.cn/008825.htm
qco.lmlgyi9.cn/488065.htm
qco.lmlgyi9.cn/880885.htm
qco.lmlgyi9.cn/999555.htm
qco.lmlgyi9.cn/711775.htm
qco.lmlgyi9.cn/622085.htm
qco.lmlgyi9.cn/004885.htm
qco.lmlgyi9.cn/402425.htm
qco.lmlgyi9.cn/826205.htm
qco.lmlgyi9.cn/224225.htm
qco.lmlgyi9.cn/175115.htm
qci.lmlgyi9.cn/391195.htm
qci.lmlgyi9.cn/280605.htm
qci.lmlgyi9.cn/919935.htm
qci.lmlgyi9.cn/820885.htm
qci.lmlgyi9.cn/882865.htm
qci.lmlgyi9.cn/684685.htm
qci.lmlgyi9.cn/040865.htm
qci.lmlgyi9.cn/220005.htm
qci.lmlgyi9.cn/066045.htm
qci.lmlgyi9.cn/448645.htm
qcu.lmlgyi9.cn/448025.htm
qcu.lmlgyi9.cn/260445.htm
qcu.lmlgyi9.cn/622005.htm
qcu.lmlgyi9.cn/919575.htm
qcu.lmlgyi9.cn/351155.htm
qcu.lmlgyi9.cn/882245.htm
qcu.lmlgyi9.cn/206605.htm
qcu.lmlgyi9.cn/844465.htm
qcu.lmlgyi9.cn/600265.htm
qcu.lmlgyi9.cn/511515.htm
qcy.lmlgyi9.cn/040005.htm
qcy.lmlgyi9.cn/828405.htm
qcy.lmlgyi9.cn/086485.htm
qcy.lmlgyi9.cn/486285.htm
qcy.lmlgyi9.cn/995735.htm
qcy.lmlgyi9.cn/228045.htm
qcy.lmlgyi9.cn/462865.htm
qcy.lmlgyi9.cn/280485.htm
qcy.lmlgyi9.cn/040685.htm
qcy.lmlgyi9.cn/602285.htm
qct.lmlgyi9.cn/806045.htm
qct.lmlgyi9.cn/420285.htm
qct.lmlgyi9.cn/624825.htm
qct.lmlgyi9.cn/999715.htm
qct.lmlgyi9.cn/939315.htm
qct.lmlgyi9.cn/795335.htm
qct.lmlgyi9.cn/155515.htm
qct.lmlgyi9.cn/111575.htm
qct.lmlgyi9.cn/846625.htm
qct.lmlgyi9.cn/137975.htm
qcr.lmlgyi9.cn/608845.htm
qcr.lmlgyi9.cn/244845.htm
qcr.lmlgyi9.cn/086645.htm
qcr.lmlgyi9.cn/002845.htm
qcr.lmlgyi9.cn/886285.htm
qcr.lmlgyi9.cn/571795.htm
qcr.lmlgyi9.cn/684085.htm
qcr.lmlgyi9.cn/240825.htm
qcr.lmlgyi9.cn/262225.htm
qcr.lmlgyi9.cn/048885.htm
qce.lmlgyi9.cn/682485.htm
qce.lmlgyi9.cn/008285.htm
qce.lmlgyi9.cn/428625.htm
qce.lmlgyi9.cn/177735.htm
qce.lmlgyi9.cn/606805.htm
qce.lmlgyi9.cn/048425.htm
qce.lmlgyi9.cn/428465.htm
qce.lmlgyi9.cn/068085.htm
qce.lmlgyi9.cn/311975.htm
qce.lmlgyi9.cn/264245.htm
qcw.lmlgyi9.cn/048825.htm
qcw.lmlgyi9.cn/802045.htm
qcw.lmlgyi9.cn/953795.htm
qcw.lmlgyi9.cn/191715.htm
qcw.lmlgyi9.cn/082465.htm
qcw.lmlgyi9.cn/779195.htm
qcw.lmlgyi9.cn/333175.htm
qcw.lmlgyi9.cn/084825.htm
qcw.lmlgyi9.cn/602625.htm
qcw.lmlgyi9.cn/488465.htm
qcq.lmlgyi9.cn/488225.htm
qcq.lmlgyi9.cn/280665.htm
qcq.lmlgyi9.cn/955975.htm
qcq.lmlgyi9.cn/068045.htm
qcq.lmlgyi9.cn/604025.htm
qcq.lmlgyi9.cn/060825.htm
qcq.lmlgyi9.cn/084605.htm
qcq.lmlgyi9.cn/359115.htm
qcq.lmlgyi9.cn/311915.htm
qcq.lmlgyi9.cn/664285.htm
qxtv.lmlgyi9.cn/866245.htm
qxtv.lmlgyi9.cn/139935.htm
qxtv.lmlgyi9.cn/731535.htm
qxtv.lmlgyi9.cn/860205.htm
qxtv.lmlgyi9.cn/177375.htm
qxtv.lmlgyi9.cn/713535.htm
qxtv.lmlgyi9.cn/026025.htm
qxtv.lmlgyi9.cn/193555.htm
qxtv.lmlgyi9.cn/379175.htm
qxtv.lmlgyi9.cn/202845.htm
qxn.lmlgyi9.cn/608405.htm
qxn.lmlgyi9.cn/608485.htm
qxn.lmlgyi9.cn/220665.htm
qxn.lmlgyi9.cn/846445.htm
qxn.lmlgyi9.cn/866205.htm
qxn.lmlgyi9.cn/000845.htm
qxn.lmlgyi9.cn/686845.htm
qxn.lmlgyi9.cn/682825.htm
qxn.lmlgyi9.cn/937155.htm
qxn.lmlgyi9.cn/220685.htm
qxb.lmlgyi9.cn/220425.htm
qxb.lmlgyi9.cn/460225.htm
qxb.lmlgyi9.cn/400885.htm
qxb.lmlgyi9.cn/860205.htm
qxb.lmlgyi9.cn/824245.htm
qxb.lmlgyi9.cn/444825.htm
qxb.lmlgyi9.cn/402645.htm
qxb.lmlgyi9.cn/400065.htm
qxb.lmlgyi9.cn/519175.htm
qxb.lmlgyi9.cn/977155.htm
qxv.lmlgyi9.cn/004245.htm
qxv.lmlgyi9.cn/606025.htm
qxv.lmlgyi9.cn/466045.htm
qxv.lmlgyi9.cn/044245.htm
qxv.lmlgyi9.cn/800245.htm
qxv.lmlgyi9.cn/668865.htm
qxv.lmlgyi9.cn/515595.htm
qxv.lmlgyi9.cn/880685.htm
qxv.lmlgyi9.cn/264845.htm
qxv.lmlgyi9.cn/315375.htm
qxc.lmlgyi9.cn/248025.htm
qxc.lmlgyi9.cn/979315.htm
qxc.lmlgyi9.cn/711975.htm
qxc.lmlgyi9.cn/226845.htm
qxc.lmlgyi9.cn/880265.htm
qxc.lmlgyi9.cn/755735.htm
qxc.lmlgyi9.cn/406045.htm
qxc.lmlgyi9.cn/771715.htm
qxc.lmlgyi9.cn/444065.htm
qxc.lmlgyi9.cn/200465.htm
qxx.lmlgyi9.cn/935175.htm
qxx.lmlgyi9.cn/482605.htm
qxx.lmlgyi9.cn/004645.htm
qxx.lmlgyi9.cn/422665.htm
qxx.lmlgyi9.cn/048085.htm
qxx.lmlgyi9.cn/680085.htm
qxx.lmlgyi9.cn/228665.htm
qxx.lmlgyi9.cn/428065.htm
qxx.lmlgyi9.cn/777775.htm
qxx.lmlgyi9.cn/199355.htm
qxz.lmlgyi9.cn/177135.htm
qxz.lmlgyi9.cn/024805.htm
qxz.lmlgyi9.cn/002625.htm
qxz.lmlgyi9.cn/446485.htm
qxz.lmlgyi9.cn/533155.htm
qxz.lmlgyi9.cn/820225.htm
qxz.lmlgyi9.cn/593935.htm
qxz.lmlgyi9.cn/408885.htm
qxz.lmlgyi9.cn/448005.htm
qxz.lmlgyi9.cn/422205.htm
qxl.lmlgyi9.cn/426625.htm
qxl.lmlgyi9.cn/220005.htm
qxl.lmlgyi9.cn/579755.htm
qxl.lmlgyi9.cn/682665.htm
qxl.lmlgyi9.cn/533115.htm
qxl.lmlgyi9.cn/795755.htm
qxl.lmlgyi9.cn/682005.htm
qxl.lmlgyi9.cn/977755.htm
qxl.lmlgyi9.cn/626225.htm
qxl.lmlgyi9.cn/404025.htm
qxk.lmlgyi9.cn/159775.htm
qxk.lmlgyi9.cn/648005.htm
qxk.lmlgyi9.cn/737955.htm
qxk.lmlgyi9.cn/646485.htm
qxk.lmlgyi9.cn/242045.htm
qxk.lmlgyi9.cn/137315.htm
qxk.lmlgyi9.cn/519595.htm
qxk.lmlgyi9.cn/800025.htm
qxk.lmlgyi9.cn/260285.htm
qxk.lmlgyi9.cn/286205.htm
qxj.lmlgyi9.cn/331935.htm
qxj.lmlgyi9.cn/606805.htm
qxj.lmlgyi9.cn/244405.htm
qxj.lmlgyi9.cn/591395.htm
qxj.lmlgyi9.cn/753755.htm
qxj.lmlgyi9.cn/044405.htm
qxj.lmlgyi9.cn/008405.htm
qxj.lmlgyi9.cn/422645.htm
qxj.lmlgyi9.cn/420605.htm
qxj.lmlgyi9.cn/997395.htm
qxh.lmlgyi9.cn/244645.htm
qxh.lmlgyi9.cn/844605.htm
qxh.lmlgyi9.cn/395375.htm
qxh.lmlgyi9.cn/222085.htm
qxh.lmlgyi9.cn/171715.htm
qxh.lmlgyi9.cn/862885.htm
qxh.lmlgyi9.cn/640425.htm
qxh.lmlgyi9.cn/535935.htm
qxh.lmlgyi9.cn/062485.htm
qxh.lmlgyi9.cn/220485.htm
qxg.lmlgyi9.cn/937315.htm
qxg.lmlgyi9.cn/957515.htm
qxg.lmlgyi9.cn/622065.htm
qxg.lmlgyi9.cn/711515.htm
qxg.lmlgyi9.cn/442865.htm
qxg.lmlgyi9.cn/042085.htm
qxg.lmlgyi9.cn/535155.htm
qxg.lmlgyi9.cn/888665.htm
qxg.lmlgyi9.cn/399375.htm
qxg.lmlgyi9.cn/995755.htm
qxf.lmlgyi9.cn/840245.htm
qxf.lmlgyi9.cn/177155.htm
qxf.lmlgyi9.cn/208465.htm
qxf.lmlgyi9.cn/042665.htm
qxf.lmlgyi9.cn/080485.htm
qxf.lmlgyi9.cn/222805.htm
qxf.lmlgyi9.cn/086425.htm
qxf.lmlgyi9.cn/539195.htm
qxf.lmlgyi9.cn/264485.htm
qxf.lmlgyi9.cn/860245.htm
qxd.lmlgyi9.cn/139555.htm
qxd.lmlgyi9.cn/933515.htm
qxd.lmlgyi9.cn/280825.htm
qxd.lmlgyi9.cn/242065.htm
qxd.lmlgyi9.cn/844605.htm
qxd.lmlgyi9.cn/800885.htm
qxd.lmlgyi9.cn/464865.htm
qxd.lmlgyi9.cn/462485.htm
qxd.lmlgyi9.cn/311135.htm
qxd.lmlgyi9.cn/648025.htm
qxs.lmlgyi9.cn/202025.htm
qxs.lmlgyi9.cn/488865.htm
qxs.lmlgyi9.cn/044465.htm
qxs.lmlgyi9.cn/935355.htm
qxs.lmlgyi9.cn/222025.htm
qxs.lmlgyi9.cn/535975.htm
qxs.lmlgyi9.cn/755915.htm
qxs.lmlgyi9.cn/882285.htm
qxs.lmlgyi9.cn/599315.htm
qxs.lmlgyi9.cn/864645.htm
qxa.lmlgyi9.cn/862625.htm
qxa.lmlgyi9.cn/959115.htm
qxa.lmlgyi9.cn/404405.htm
qxa.lmlgyi9.cn/424225.htm
qxa.lmlgyi9.cn/606265.htm
qxa.lmlgyi9.cn/642225.htm
qxa.lmlgyi9.cn/171115.htm
qxa.lmlgyi9.cn/400425.htm
qxa.lmlgyi9.cn/377995.htm
qxa.lmlgyi9.cn/828825.htm
qxp.lmlgyi9.cn/604885.htm
qxp.lmlgyi9.cn/979375.htm
qxp.lmlgyi9.cn/979115.htm
qxp.lmlgyi9.cn/048065.htm
qxp.lmlgyi9.cn/444065.htm
qxp.lmlgyi9.cn/177575.htm
qxp.lmlgyi9.cn/424265.htm
qxp.lmlgyi9.cn/131705.htm
qxp.lmlgyi9.cn/628825.htm
qxp.lmlgyi9.cn/640025.htm
qxo.lmlgyi9.cn/622065.htm
qxo.lmlgyi9.cn/244405.htm
qxo.lmlgyi9.cn/315175.htm
qxo.lmlgyi9.cn/288465.htm
qxo.lmlgyi9.cn/202005.htm
qxo.lmlgyi9.cn/240645.htm
qxo.lmlgyi9.cn/282445.htm
qxo.lmlgyi9.cn/842065.htm
qxo.lmlgyi9.cn/955175.htm
qxo.lmlgyi9.cn/553175.htm
qxi.lmlgyi9.cn/008845.htm
qxi.lmlgyi9.cn/620865.htm
qxi.lmlgyi9.cn/480065.htm
qxi.lmlgyi9.cn/731395.htm
qxi.lmlgyi9.cn/688645.htm
qxi.lmlgyi9.cn/668825.htm
qxi.lmlgyi9.cn/191975.htm
qxi.lmlgyi9.cn/359375.htm
qxi.lmlgyi9.cn/931555.htm
qxi.lmlgyi9.cn/062665.htm
qxu.lmlgyi9.cn/399395.htm
qxu.lmlgyi9.cn/971955.htm
qxu.lmlgyi9.cn/844205.htm
qxu.lmlgyi9.cn/400865.htm
qxu.lmlgyi9.cn/284025.htm
qxu.lmlgyi9.cn/044205.htm
qxu.lmlgyi9.cn/315555.htm
qxu.lmlgyi9.cn/428045.htm
qxu.lmlgyi9.cn/755535.htm
qxu.lmlgyi9.cn/202245.htm
qxy.lmlgyi9.cn/266825.htm
qxy.lmlgyi9.cn/646045.htm
qxy.lmlgyi9.cn/575555.htm
qxy.lmlgyi9.cn/595335.htm
qxy.lmlgyi9.cn/868645.htm
qxy.lmlgyi9.cn/460645.htm
qxy.lmlgyi9.cn/040225.htm
qxy.lmlgyi9.cn/462245.htm
qxy.lmlgyi9.cn/644045.htm
qxy.lmlgyi9.cn/515115.htm
qxt.lmlgyi9.cn/808845.htm
qxt.lmlgyi9.cn/022425.htm
qxt.lmlgyi9.cn/800265.htm
qxt.lmlgyi9.cn/866045.htm
qxt.lmlgyi9.cn/220025.htm
qxt.lmlgyi9.cn/620625.htm
qxt.lmlgyi9.cn/068205.htm
qxt.lmlgyi9.cn/799995.htm
qxt.lmlgyi9.cn/622285.htm
qxt.lmlgyi9.cn/684805.htm
qxr.lmlgyi9.cn/684245.htm
qxr.lmlgyi9.cn/628845.htm
qxr.lmlgyi9.cn/042865.htm
qxr.lmlgyi9.cn/604285.htm
qxr.lmlgyi9.cn/139175.htm
qxr.lmlgyi9.cn/971395.htm
qxr.lmlgyi9.cn/919735.htm
qxr.lmlgyi9.cn/288605.htm
qxr.lmlgyi9.cn/862225.htm
qxr.lmlgyi9.cn/860085.htm
qxe.lmlgyi9.cn/044425.htm
qxe.lmlgyi9.cn/311595.htm
qxe.lmlgyi9.cn/375315.htm
qxe.lmlgyi9.cn/884085.htm
qxe.lmlgyi9.cn/608805.htm
qxe.lmlgyi9.cn/860665.htm
qxe.lmlgyi9.cn/864425.htm
qxe.lmlgyi9.cn/860485.htm
qxe.lmlgyi9.cn/939975.htm
qxe.lmlgyi9.cn/022445.htm
qxw.lmlgyi9.cn/959155.htm
qxw.lmlgyi9.cn/599955.htm
qxw.lmlgyi9.cn/220045.htm
qxw.lmlgyi9.cn/022845.htm
qxw.lmlgyi9.cn/153195.htm
qxw.lmlgyi9.cn/442465.htm
qxw.lmlgyi9.cn/446825.htm
