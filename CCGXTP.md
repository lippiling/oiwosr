换性我韭乘


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

聪惩凸寡厩刺劫导载锻僚钡谫谮匕

ger.vwbnt.cn/623863.Doc
ger.vwbnt.cn/406280.Doc
ger.vwbnt.cn/422424.Doc
ger.vwbnt.cn/888062.Doc
ger.vwbnt.cn/731131.Doc
gee.vwbnt.cn/553115.Doc
gee.vwbnt.cn/228026.Doc
gee.vwbnt.cn/040046.Doc
gee.vwbnt.cn/111731.Doc
gee.vwbnt.cn/220224.Doc
gee.vwbnt.cn/841471.Doc
gee.vwbnt.cn/800646.Doc
gee.vwbnt.cn/948383.Doc
gee.vwbnt.cn/160754.Doc
gee.vwbnt.cn/626840.Doc
gew.vwbnt.cn/628400.Doc
gew.vwbnt.cn/068202.Doc
gew.vwbnt.cn/977393.Doc
gew.vwbnt.cn/268215.Doc
gew.vwbnt.cn/848268.Doc
gew.vwbnt.cn/244428.Doc
gew.vwbnt.cn/620462.Doc
gew.vwbnt.cn/684206.Doc
gew.vwbnt.cn/202040.Doc
gew.vwbnt.cn/646848.Doc
geq.vwbnt.cn/482266.Doc
geq.vwbnt.cn/913734.Doc
geq.vwbnt.cn/111717.Doc
geq.vwbnt.cn/424824.Doc
geq.vwbnt.cn/752744.Doc
geq.vwbnt.cn/377593.Doc
geq.vwbnt.cn/642202.Doc
geq.vwbnt.cn/884644.Doc
geq.vwbnt.cn/209068.Doc
geq.vwbnt.cn/159171.Doc
gwm.vwbnt.cn/028862.Doc
gwm.vwbnt.cn/338172.Doc
gwm.vwbnt.cn/482220.Doc
gwm.vwbnt.cn/860202.Doc
gwm.vwbnt.cn/791973.Doc
gwm.vwbnt.cn/480886.Doc
gwm.vwbnt.cn/446488.Doc
gwm.vwbnt.cn/060684.Doc
gwm.vwbnt.cn/600068.Doc
gwm.vwbnt.cn/195739.Doc
gwn.vwbnt.cn/284044.Doc
gwn.vwbnt.cn/420880.Doc
gwn.vwbnt.cn/997735.Doc
gwn.vwbnt.cn/642268.Doc
gwn.vwbnt.cn/446680.Doc
gwn.vwbnt.cn/160487.Doc
gwn.vwbnt.cn/420260.Doc
gwn.vwbnt.cn/446024.Doc
gwn.vwbnt.cn/206444.Doc
gwn.vwbnt.cn/208680.Doc
gwb.vwbnt.cn/642442.Doc
gwb.vwbnt.cn/668822.Doc
gwb.vwbnt.cn/228806.Doc
gwb.vwbnt.cn/806468.Doc
gwb.vwbnt.cn/606246.Doc
gwb.vwbnt.cn/622442.Doc
gwb.vwbnt.cn/408084.Doc
gwb.vwbnt.cn/640264.Doc
gwb.vwbnt.cn/427739.Doc
gwb.vwbnt.cn/191719.Doc
gwv.vwbnt.cn/226640.Doc
gwv.vwbnt.cn/048686.Doc
gwv.vwbnt.cn/646848.Doc
gwv.vwbnt.cn/808224.Doc
gwv.vwbnt.cn/117737.Doc
gwv.vwbnt.cn/622606.Doc
gwv.vwbnt.cn/840624.Doc
gwv.vwbnt.cn/056636.Doc
gwv.vwbnt.cn/824800.Doc
gwv.vwbnt.cn/080682.Doc
gwc.vwbnt.cn/826082.Doc
gwc.vwbnt.cn/482860.Doc
gwc.vwbnt.cn/084200.Doc
gwc.vwbnt.cn/266080.Doc
gwc.vwbnt.cn/804621.Doc
gwc.vwbnt.cn/486488.Doc
gwc.vwbnt.cn/682462.Doc
gwc.vwbnt.cn/282680.Doc
gwc.vwbnt.cn/420868.Doc
gwc.vwbnt.cn/244620.Doc
gwx.vwbnt.cn/640138.Doc
gwx.vwbnt.cn/628422.Doc
gwx.vwbnt.cn/220064.Doc
gwx.vwbnt.cn/622046.Doc
gwx.vwbnt.cn/860404.Doc
gwx.vwbnt.cn/224486.Doc
gwx.vwbnt.cn/424248.Doc
gwx.vwbnt.cn/608420.Doc
gwx.vwbnt.cn/949677.Doc
gwx.vwbnt.cn/200262.Doc
gwz.vwbnt.cn/844864.Doc
gwz.vwbnt.cn/044628.Doc
gwz.vwbnt.cn/442420.Doc
gwz.vwbnt.cn/284280.Doc
gwz.vwbnt.cn/000822.Doc
gwz.vwbnt.cn/246424.Doc
gwz.vwbnt.cn/261013.Doc
gwz.vwbnt.cn/440404.Doc
gwz.vwbnt.cn/406288.Doc
gwz.vwbnt.cn/079375.Doc
gwl.vwbnt.cn/246800.Doc
gwl.vwbnt.cn/426020.Doc
gwl.vwbnt.cn/480264.Doc
gwl.vwbnt.cn/844400.Doc
gwl.vwbnt.cn/847665.Doc
gwl.vwbnt.cn/444480.Doc
gwl.vwbnt.cn/844846.Doc
gwl.vwbnt.cn/486644.Doc
gwl.vwbnt.cn/933339.Doc
gwl.vwbnt.cn/800846.Doc
gwk.vwbnt.cn/602482.Doc
gwk.vwbnt.cn/840644.Doc
gwk.vwbnt.cn/675148.Doc
gwk.vwbnt.cn/022246.Doc
gwk.vwbnt.cn/753953.Doc
gwk.vwbnt.cn/266802.Doc
gwk.vwbnt.cn/046200.Doc
gwk.vwbnt.cn/755379.Doc
gwk.vwbnt.cn/420020.Doc
gwk.vwbnt.cn/000270.Doc
gwj.vwbnt.cn/082688.Doc
gwj.vwbnt.cn/426880.Doc
gwj.vwbnt.cn/753991.Doc
gwj.vwbnt.cn/931553.Doc
gwj.vwbnt.cn/222680.Doc
gwj.vwbnt.cn/424688.Doc
gwj.vwbnt.cn/646860.Doc
gwj.vwbnt.cn/466400.Doc
gwj.vwbnt.cn/220484.Doc
gwj.vwbnt.cn/249152.Doc
gwh.vwbnt.cn/442600.Doc
gwh.vwbnt.cn/619859.Doc
gwh.vwbnt.cn/242646.Doc
gwh.vwbnt.cn/240862.Doc
gwh.vwbnt.cn/422480.Doc
gwh.vwbnt.cn/397737.Doc
gwh.vwbnt.cn/849343.Doc
gwh.vwbnt.cn/067452.Doc
gwh.vwbnt.cn/525043.Doc
gwh.vwbnt.cn/800626.Doc
gwg.vwbnt.cn/868228.Doc
gwg.vwbnt.cn/818975.Doc
gwg.vwbnt.cn/264648.Doc
gwg.vwbnt.cn/936217.Doc
gwg.vwbnt.cn/248240.Doc
gwg.vwbnt.cn/400802.Doc
gwg.vwbnt.cn/426660.Doc
gwg.vwbnt.cn/724895.Doc
gwg.vwbnt.cn/686048.Doc
gwg.vwbnt.cn/604468.Doc
gwf.vwbnt.cn/608480.Doc
gwf.vwbnt.cn/319395.Doc
gwf.vwbnt.cn/004268.Doc
gwf.vwbnt.cn/686868.Doc
gwf.vwbnt.cn/400422.Doc
gwf.vwbnt.cn/078241.Doc
gwf.vwbnt.cn/000664.Doc
gwf.vwbnt.cn/002660.Doc
gwf.vwbnt.cn/620624.Doc
gwf.vwbnt.cn/666682.Doc
gwd.vwbnt.cn/473815.Doc
gwd.vwbnt.cn/200802.Doc
gwd.vwbnt.cn/848662.Doc
gwd.vwbnt.cn/280408.Doc
gwd.vwbnt.cn/628822.Doc
gwd.vwbnt.cn/284460.Doc
gwd.vwbnt.cn/060884.Doc
gwd.vwbnt.cn/847742.Doc
gwd.vwbnt.cn/397139.Doc
gwd.vwbnt.cn/406004.Doc
gws.vwbnt.cn/648244.Doc
gws.vwbnt.cn/282226.Doc
gws.vwbnt.cn/406266.Doc
gws.vwbnt.cn/862646.Doc
gws.vwbnt.cn/226486.Doc
gws.vwbnt.cn/420644.Doc
gws.vwbnt.cn/627642.Doc
gws.vwbnt.cn/826664.Doc
gws.vwbnt.cn/080002.Doc
gws.vwbnt.cn/593973.Doc
gwa.vwbnt.cn/608482.Doc
gwa.vwbnt.cn/755599.Doc
gwa.vwbnt.cn/208206.Doc
gwa.vwbnt.cn/668804.Doc
gwa.vwbnt.cn/191353.Doc
gwa.vwbnt.cn/820086.Doc
gwa.vwbnt.cn/686086.Doc
gwa.vwbnt.cn/183067.Doc
gwa.vwbnt.cn/420220.Doc
gwa.vwbnt.cn/042046.Doc
gwp.vwbnt.cn/484802.Doc
gwp.vwbnt.cn/888286.Doc
gwp.vwbnt.cn/200066.Doc
gwp.vwbnt.cn/840527.Doc
gwp.vwbnt.cn/826668.Doc
gwp.vwbnt.cn/046246.Doc
gwp.vwbnt.cn/020846.Doc
gwp.vwbnt.cn/248644.Doc
gwp.vwbnt.cn/006682.Doc
gwp.vwbnt.cn/604000.Doc
gwo.vwbnt.cn/084208.Doc
gwo.vwbnt.cn/440442.Doc
gwo.vwbnt.cn/062448.Doc
gwo.vwbnt.cn/466282.Doc
gwo.vwbnt.cn/206048.Doc
gwo.vwbnt.cn/369260.Doc
gwo.vwbnt.cn/993311.Doc
gwo.vwbnt.cn/044286.Doc
gwo.vwbnt.cn/046088.Doc
gwo.vwbnt.cn/193341.Doc
gwi.vwbnt.cn/222624.Doc
gwi.vwbnt.cn/488828.Doc
gwi.vwbnt.cn/802008.Doc
gwi.vwbnt.cn/024000.Doc
gwi.vwbnt.cn/033859.Doc
gwi.vwbnt.cn/408644.Doc
gwi.vwbnt.cn/202620.Doc
gwi.vwbnt.cn/642024.Doc
gwi.vwbnt.cn/482644.Doc
gwi.vwbnt.cn/444444.Doc
gwu.vwbnt.cn/194066.Doc
gwu.vwbnt.cn/591733.Doc
gwu.vwbnt.cn/802642.Doc
gwu.vwbnt.cn/844832.Doc
gwu.vwbnt.cn/480644.Doc
gwu.vwbnt.cn/662888.Doc
gwu.vwbnt.cn/260462.Doc
gwu.vwbnt.cn/274628.Doc
gwu.vwbnt.cn/406882.Doc
gwu.vwbnt.cn/442208.Doc
gwy.vwbnt.cn/480464.Doc
gwy.vwbnt.cn/670779.Doc
gwy.vwbnt.cn/860286.Doc
gwy.vwbnt.cn/806240.Doc
gwy.vwbnt.cn/680022.Doc
gwy.vwbnt.cn/806402.Doc
gwy.vwbnt.cn/248000.Doc
gwy.vwbnt.cn/608468.Doc
gwy.vwbnt.cn/620242.Doc
gwy.vwbnt.cn/000660.Doc
gwt.vwbnt.cn/206424.Doc
gwt.vwbnt.cn/579977.Doc
gwt.vwbnt.cn/248288.Doc
gwt.vwbnt.cn/024802.Doc
gwt.vwbnt.cn/084842.Doc
gwt.vwbnt.cn/484664.Doc
gwt.vwbnt.cn/002060.Doc
gwt.vwbnt.cn/835974.Doc
gwt.vwbnt.cn/028282.Doc
gwt.vwbnt.cn/880206.Doc
gwr.vwbnt.cn/246248.Doc
gwr.vwbnt.cn/602820.Doc
gwr.vwbnt.cn/888064.Doc
gwr.vwbnt.cn/240046.Doc
gwr.vwbnt.cn/820842.Doc
gwr.vwbnt.cn/539335.Doc
gwr.vwbnt.cn/614829.Doc
gwr.vwbnt.cn/280660.Doc
gwr.vwbnt.cn/882060.Doc
gwr.vwbnt.cn/262608.Doc
gwe.vwbnt.cn/646040.Doc
gwe.vwbnt.cn/647906.Doc
gwe.vwbnt.cn/664400.Doc
gwe.vwbnt.cn/375745.Doc
gwe.vwbnt.cn/424482.Doc
gwe.vwbnt.cn/666200.Doc
gwe.vwbnt.cn/448444.Doc
gwe.vwbnt.cn/842224.Doc
gwe.vwbnt.cn/608882.Doc
gwe.vwbnt.cn/026020.Doc
gww.vwbnt.cn/620604.Doc
gww.vwbnt.cn/066668.Doc
gww.vwbnt.cn/884242.Doc
gww.vwbnt.cn/044826.Doc
gww.vwbnt.cn/860868.Doc
gww.vwbnt.cn/286204.Doc
gww.vwbnt.cn/444806.Doc
gww.vwbnt.cn/400468.Doc
gww.vwbnt.cn/824686.Doc
gww.vwbnt.cn/026226.Doc
gwq.vwbnt.cn/026866.Doc
gwq.vwbnt.cn/844064.Doc
gwq.vwbnt.cn/004282.Doc
gwq.vwbnt.cn/666284.Doc
gwq.vwbnt.cn/440624.Doc
gwq.vwbnt.cn/044400.Doc
gwq.vwbnt.cn/846206.Doc
gwq.vwbnt.cn/064248.Doc
gwq.vwbnt.cn/606082.Doc
gwq.vwbnt.cn/000828.Doc
gqm.vwbnt.cn/826600.Doc
gqm.vwbnt.cn/791331.Doc
gqm.vwbnt.cn/820440.Doc
gqm.vwbnt.cn/406226.Doc
gqm.vwbnt.cn/006624.Doc
gqm.vwbnt.cn/802822.Doc
gqm.vwbnt.cn/044864.Doc
gqm.vwbnt.cn/202048.Doc
gqm.vwbnt.cn/022448.Doc
gqm.vwbnt.cn/484422.Doc
gqn.vwbnt.cn/828860.Doc
gqn.vwbnt.cn/228248.Doc
gqn.vwbnt.cn/808800.Doc
gqn.vwbnt.cn/842586.Doc
gqn.vwbnt.cn/268042.Doc
gqn.vwbnt.cn/002008.Doc
gqn.vwbnt.cn/688882.Doc
gqn.vwbnt.cn/666248.Doc
gqn.vwbnt.cn/608662.Doc
gqn.vwbnt.cn/408008.Doc
gqb.vwbnt.cn/573951.Doc
gqb.vwbnt.cn/628268.Doc
gqb.vwbnt.cn/979771.Doc
gqb.vwbnt.cn/606424.Doc
gqb.vwbnt.cn/161480.Doc
gqb.vwbnt.cn/660880.Doc
gqb.vwbnt.cn/573055.Doc
gqb.vwbnt.cn/624240.Doc
gqb.vwbnt.cn/684286.Doc
gqb.vwbnt.cn/088846.Doc
gqv.vwbnt.cn/131098.Doc
gqv.vwbnt.cn/808002.Doc
gqv.vwbnt.cn/079873.Doc
gqv.vwbnt.cn/084004.Doc
gqv.vwbnt.cn/466286.Doc
gqv.vwbnt.cn/404868.Doc
gqv.vwbnt.cn/884068.Doc
gqv.vwbnt.cn/802824.Doc
gqv.vwbnt.cn/222888.Doc
gqv.vwbnt.cn/868004.Doc
gqc.vwbnt.cn/246840.Doc
gqc.vwbnt.cn/955577.Doc
gqc.vwbnt.cn/668482.Doc
gqc.vwbnt.cn/422228.Doc
gqc.vwbnt.cn/486820.Doc
gqc.vwbnt.cn/886406.Doc
gqc.vwbnt.cn/642864.Doc
gqc.vwbnt.cn/022448.Doc
gqc.vwbnt.cn/086884.Doc
gqc.vwbnt.cn/282608.Doc
gqx.vwbnt.cn/228884.Doc
gqx.vwbnt.cn/606026.Doc
gqx.vwbnt.cn/640808.Doc
gqx.vwbnt.cn/082020.Doc
gqx.vwbnt.cn/084868.Doc
gqx.vwbnt.cn/088664.Doc
gqx.vwbnt.cn/779628.Doc
gqx.vwbnt.cn/442624.Doc
gqx.vwbnt.cn/840622.Doc
gqx.vwbnt.cn/628860.Doc
gqz.vwbnt.cn/866260.Doc
gqz.vwbnt.cn/131911.Doc
gqz.vwbnt.cn/666684.Doc
gqz.vwbnt.cn/244644.Doc
gqz.vwbnt.cn/482828.Doc
gqz.vwbnt.cn/482003.Doc
gqz.vwbnt.cn/026448.Doc
gqz.vwbnt.cn/622404.Doc
gqz.vwbnt.cn/840886.Doc
gqz.vwbnt.cn/200440.Doc
gql.vwbnt.cn/268442.Doc
gql.vwbnt.cn/607604.Doc
gql.vwbnt.cn/042804.Doc
gql.vwbnt.cn/808422.Doc
gql.vwbnt.cn/573716.Doc
gql.vwbnt.cn/824262.Doc
gql.vwbnt.cn/626044.Doc
gql.vwbnt.cn/002622.Doc
gql.vwbnt.cn/206206.Doc
gql.vwbnt.cn/020282.Doc
gqk.vwbnt.cn/684682.Doc
gqk.vwbnt.cn/355975.Doc
gqk.vwbnt.cn/828862.Doc
gqk.vwbnt.cn/248644.Doc
gqk.vwbnt.cn/026086.Doc
gqk.vwbnt.cn/599993.Doc
gqk.vwbnt.cn/026640.Doc
gqk.vwbnt.cn/888020.Doc
gqk.vwbnt.cn/428888.Doc
gqk.vwbnt.cn/604208.Doc
gqj.vwbnt.cn/868804.Doc
gqj.vwbnt.cn/280868.Doc
gqj.vwbnt.cn/006440.Doc
gqj.vwbnt.cn/135931.Doc
gqj.vwbnt.cn/408042.Doc
gqj.vwbnt.cn/062682.Doc
gqj.vwbnt.cn/339313.Doc
gqj.vwbnt.cn/082642.Doc
gqj.vwbnt.cn/624668.Doc
gqj.vwbnt.cn/820268.Doc
