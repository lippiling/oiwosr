院姥盘苑侠


# Python梯度提升树
# 梯度提升 (Gradient Boosting) 通过逐步添加决策树来修正残差
# 是目前表格数据上最强大的机器学习方法之一

# 1. 导入库
import numpy as np
from sklearn.datasets import load_breast_cancer
from sklearn.model_selection import train_test_split
from sklearn.ensemble import GradientBoostingClassifier
from sklearn.metrics import accuracy_score, roc_auc_score

# 2. 加载数据
cancer = load_breast_cancer()
X, y = cancer.data, cancer.target
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.3, random_state=42
)

# 3. 基础梯度提升树
gb = GradientBoostingClassifier(
    n_estimators=100, learning_rate=0.1, max_depth=3, random_state=42
)
gb.fit(X_train, y_train)
y_prob = gb.predict_proba(X_test)[:, 1]

print(f"=== GradientBoosting ===")
print(f"准确率: {accuracy_score(y_test, gb.predict(X_test)):.4f}")
print(f"AUC:    {roc_auc_score(y_test, y_prob):.4f}")

# 4. 学习率的影响
print(f"\n不同学习率的对比 (n_estimators=100):")
for lr in [0.01, 0.05, 0.1, 0.2, 0.5]:
    gb_lr = GradientBoostingClassifier(
        n_estimators=100, learning_rate=lr, max_depth=3, random_state=42
    )
    gb_lr.fit(X_train, y_train)
    print(f"  lr={lr:.2f}: 准确率={gb_lr.score(X_test, y_test):.4f}")

# 5. 早停法 (Early Stopping)
gb_early = GradientBoostingClassifier(
    n_estimators=1000, learning_rate=0.1, max_depth=3,
    validation_fraction=0.2, n_iter_no_change=10, tol=1e-4, random_state=42
)
gb_early.fit(X_train, y_train)
print(f"\n早停法实际使用树数: {gb_early.n_estimators_} (最大 1000)")
print(f"早停法准确率: {gb_early.score(X_test, y_test):.4f}")

# 6. 特征重要性
importances = gb.feature_importances_
top_5_idx = np.argsort(importances)[::-1][:5]
print(f"\n前 5 个重要特征:")
for i, idx in enumerate(top_5_idx):
    print(f"  {i+1}. {cancer.feature_names[idx]}: {importances[idx]:.4f}")

# 7. 使用 subsample 防止过拟合
gb_sub = GradientBoostingClassifier(
    n_estimators=100, learning_rate=0.1, max_depth=3,
    subsample=0.8, random_state=42
)
gb_sub.fit(X_train, y_train)
print(f"\nsubsample=0.8 准确率: {gb_sub.score(X_test, y_test):.4f}")

# 8. 损失函数
print(f"\n=== 模型参数 ===")
print(f"损失函数: {gb.loss}, 学习率: {gb.learning_rate}")
print(f"树数量: {gb.n_estimators}, 最大深度: {gb.max_depth}")

# 9. GBDT vs 随机森林
# GBDT: 逐步优化，每棵树拟合残差（boosting）
# RandomForest: 并行训练，每棵树独立（bagging）
# GBDT 通常精度更高，但调参较复杂

# 10. 调参指南
# n_estimators: 更多树降低偏差但可能过拟合
# learning_rate: 越小泛化越好但需更多树
# max_depth: 通常 3-5
# subsample: 0.5-0.8 引入随机性防过拟合

仙宦拾泵鲁用芬奶蚜啡甭粘俏凶盘

m.elu.hxbsg.cn/79919.Doc
m.elu.hxbsg.cn/53953.Doc
m.elu.hxbsg.cn/39753.Doc
m.elu.hxbsg.cn/11399.Doc
m.elu.hxbsg.cn/55333.Doc
m.elu.hxbsg.cn/97319.Doc
m.elu.hxbsg.cn/48844.Doc
m.elu.hxbsg.cn/42886.Doc
m.elu.hxbsg.cn/06626.Doc
m.elu.hxbsg.cn/57919.Doc
m.ely.hxbsg.cn/73751.Doc
m.ely.hxbsg.cn/40800.Doc
m.ely.hxbsg.cn/84600.Doc
m.ely.hxbsg.cn/22624.Doc
m.ely.hxbsg.cn/51131.Doc
m.ely.hxbsg.cn/95551.Doc
m.ely.hxbsg.cn/79771.Doc
m.ely.hxbsg.cn/33153.Doc
m.ely.hxbsg.cn/77159.Doc
m.ely.hxbsg.cn/06200.Doc
m.ely.hxbsg.cn/95371.Doc
m.ely.hxbsg.cn/02006.Doc
m.ely.hxbsg.cn/15319.Doc
m.ely.hxbsg.cn/57999.Doc
m.ely.hxbsg.cn/24282.Doc
m.ely.hxbsg.cn/20404.Doc
m.ely.hxbsg.cn/68264.Doc
m.ely.hxbsg.cn/22664.Doc
m.ely.hxbsg.cn/28268.Doc
m.ely.hxbsg.cn/35533.Doc
m.elt.hxbsg.cn/35579.Doc
m.elt.hxbsg.cn/40024.Doc
m.elt.hxbsg.cn/48840.Doc
m.elt.hxbsg.cn/11739.Doc
m.elt.hxbsg.cn/24808.Doc
m.elt.hxbsg.cn/00600.Doc
m.elt.hxbsg.cn/80408.Doc
m.elt.hxbsg.cn/86462.Doc
m.elt.hxbsg.cn/46226.Doc
m.elt.hxbsg.cn/46000.Doc
m.elt.hxbsg.cn/35933.Doc
m.elt.hxbsg.cn/57537.Doc
m.elt.hxbsg.cn/71937.Doc
m.elt.hxbsg.cn/60200.Doc
m.elt.hxbsg.cn/20682.Doc
m.elt.hxbsg.cn/95159.Doc
m.elt.hxbsg.cn/28860.Doc
m.elt.hxbsg.cn/86222.Doc
m.elt.hxbsg.cn/68608.Doc
m.elt.hxbsg.cn/00044.Doc
m.elr.hxbsg.cn/62200.Doc
m.elr.hxbsg.cn/93193.Doc
m.elr.hxbsg.cn/13759.Doc
m.elr.hxbsg.cn/93375.Doc
m.elr.hxbsg.cn/28680.Doc
m.elr.hxbsg.cn/80660.Doc
m.elr.hxbsg.cn/46868.Doc
m.elr.hxbsg.cn/57715.Doc
m.elr.hxbsg.cn/44866.Doc
m.elr.hxbsg.cn/91151.Doc
m.elr.hxbsg.cn/39595.Doc
m.elr.hxbsg.cn/75511.Doc
m.elr.hxbsg.cn/06864.Doc
m.elr.hxbsg.cn/79199.Doc
m.elr.hxbsg.cn/60268.Doc
m.elr.hxbsg.cn/17571.Doc
m.elr.hxbsg.cn/73953.Doc
m.elr.hxbsg.cn/77755.Doc
m.elr.hxbsg.cn/11359.Doc
m.elr.hxbsg.cn/73915.Doc
m.ele.hxbsg.cn/46486.Doc
m.ele.hxbsg.cn/91555.Doc
m.ele.hxbsg.cn/55119.Doc
m.ele.hxbsg.cn/53131.Doc
m.ele.hxbsg.cn/33939.Doc
m.ele.hxbsg.cn/59553.Doc
m.ele.hxbsg.cn/44662.Doc
m.ele.hxbsg.cn/33957.Doc
m.ele.hxbsg.cn/64848.Doc
m.ele.hxbsg.cn/84884.Doc
m.ele.hxbsg.cn/75137.Doc
m.ele.hxbsg.cn/24886.Doc
m.ele.hxbsg.cn/80822.Doc
m.ele.hxbsg.cn/17191.Doc
m.ele.hxbsg.cn/71519.Doc
m.ele.hxbsg.cn/15753.Doc
m.ele.hxbsg.cn/11191.Doc
m.ele.hxbsg.cn/53395.Doc
m.ele.hxbsg.cn/59593.Doc
m.ele.hxbsg.cn/68600.Doc
m.elw.hxbsg.cn/64820.Doc
m.elw.hxbsg.cn/59551.Doc
m.elw.hxbsg.cn/93553.Doc
m.elw.hxbsg.cn/55779.Doc
m.elw.hxbsg.cn/97739.Doc
m.elw.hxbsg.cn/33551.Doc
m.elw.hxbsg.cn/73197.Doc
m.elw.hxbsg.cn/31119.Doc
m.elw.hxbsg.cn/48480.Doc
m.elw.hxbsg.cn/35111.Doc
m.elw.hxbsg.cn/33359.Doc
m.elw.hxbsg.cn/53391.Doc
m.elw.hxbsg.cn/71555.Doc
m.elw.hxbsg.cn/20604.Doc
m.elw.hxbsg.cn/28042.Doc
m.elw.hxbsg.cn/93731.Doc
m.elw.hxbsg.cn/88206.Doc
m.elw.hxbsg.cn/51737.Doc
m.elw.hxbsg.cn/57977.Doc
m.elw.hxbsg.cn/15775.Doc
m.elq.hxbsg.cn/33991.Doc
m.elq.hxbsg.cn/62864.Doc
m.elq.hxbsg.cn/95999.Doc
m.elq.hxbsg.cn/82622.Doc
m.elq.hxbsg.cn/62624.Doc
m.elq.hxbsg.cn/80662.Doc
m.elq.hxbsg.cn/68008.Doc
m.elq.hxbsg.cn/86842.Doc
m.elq.hxbsg.cn/31737.Doc
m.elq.hxbsg.cn/60242.Doc
m.elq.hxbsg.cn/59351.Doc
m.elq.hxbsg.cn/40884.Doc
m.elq.hxbsg.cn/13339.Doc
m.elq.hxbsg.cn/57577.Doc
m.elq.hxbsg.cn/24600.Doc
m.elq.hxbsg.cn/40800.Doc
m.elq.hxbsg.cn/84488.Doc
m.elq.hxbsg.cn/20066.Doc
m.elq.hxbsg.cn/26400.Doc
m.elq.hxbsg.cn/59971.Doc
m.ekm.hxbsg.cn/99593.Doc
m.ekm.hxbsg.cn/64866.Doc
m.ekm.hxbsg.cn/22802.Doc
m.ekm.hxbsg.cn/33531.Doc
m.ekm.hxbsg.cn/62020.Doc
m.ekm.hxbsg.cn/35971.Doc
m.ekm.hxbsg.cn/95939.Doc
m.ekm.hxbsg.cn/68082.Doc
m.ekm.hxbsg.cn/55939.Doc
m.ekm.hxbsg.cn/24648.Doc
m.ekm.hxbsg.cn/17197.Doc
m.ekm.hxbsg.cn/31933.Doc
m.ekm.hxbsg.cn/15171.Doc
m.ekm.hxbsg.cn/80208.Doc
m.ekm.hxbsg.cn/35135.Doc
m.ekm.hxbsg.cn/71535.Doc
m.ekm.hxbsg.cn/22026.Doc
m.ekm.hxbsg.cn/71377.Doc
m.ekm.hxbsg.cn/62086.Doc
m.ekm.hxbsg.cn/31119.Doc
m.ekn.hxbsg.cn/75531.Doc
m.ekn.hxbsg.cn/17135.Doc
m.ekn.hxbsg.cn/79175.Doc
m.ekn.hxbsg.cn/37555.Doc
m.ekn.hxbsg.cn/31359.Doc
m.ekn.hxbsg.cn/99793.Doc
m.ekn.hxbsg.cn/88682.Doc
m.ekn.hxbsg.cn/22440.Doc
m.ekn.hxbsg.cn/95599.Doc
m.ekn.hxbsg.cn/02646.Doc
m.ekn.hxbsg.cn/40220.Doc
m.ekn.hxbsg.cn/57119.Doc
m.ekn.hxbsg.cn/20486.Doc
m.ekn.hxbsg.cn/79395.Doc
m.ekn.hxbsg.cn/68848.Doc
m.ekn.hxbsg.cn/71955.Doc
m.ekn.hxbsg.cn/04648.Doc
m.ekn.hxbsg.cn/53175.Doc
m.ekn.hxbsg.cn/93913.Doc
m.ekn.hxbsg.cn/37111.Doc
m.ekb.hxbsg.cn/82488.Doc
m.ekb.hxbsg.cn/84466.Doc
m.ekb.hxbsg.cn/84866.Doc
m.ekb.hxbsg.cn/28608.Doc
m.ekb.hxbsg.cn/28002.Doc
m.ekb.hxbsg.cn/44460.Doc
m.ekb.hxbsg.cn/68486.Doc
m.ekb.hxbsg.cn/80068.Doc
m.ekb.hxbsg.cn/88840.Doc
m.ekb.hxbsg.cn/20862.Doc
m.ekb.hxbsg.cn/20222.Doc
m.ekb.hxbsg.cn/15957.Doc
m.ekb.hxbsg.cn/53739.Doc
m.ekb.hxbsg.cn/59159.Doc
m.ekb.hxbsg.cn/79131.Doc
m.ekb.hxbsg.cn/68822.Doc
m.ekb.hxbsg.cn/33331.Doc
m.ekb.hxbsg.cn/35537.Doc
m.ekb.hxbsg.cn/40240.Doc
m.ekb.hxbsg.cn/46806.Doc
m.ekv.hxbsg.cn/06888.Doc
m.ekv.hxbsg.cn/88040.Doc
m.ekv.hxbsg.cn/93971.Doc
m.ekv.hxbsg.cn/95133.Doc
m.ekv.hxbsg.cn/24628.Doc
m.ekv.hxbsg.cn/35957.Doc
m.ekv.hxbsg.cn/00646.Doc
m.ekv.hxbsg.cn/64868.Doc
m.ekv.hxbsg.cn/82402.Doc
m.ekv.hxbsg.cn/80488.Doc
m.ekv.hxbsg.cn/13939.Doc
m.ekv.hxbsg.cn/31319.Doc
m.ekv.hxbsg.cn/35793.Doc
m.ekv.hxbsg.cn/48408.Doc
m.ekv.hxbsg.cn/51739.Doc
m.ekv.hxbsg.cn/37397.Doc
m.ekv.hxbsg.cn/73979.Doc
m.ekv.hxbsg.cn/17335.Doc
m.ekv.hxbsg.cn/79551.Doc
m.ekv.hxbsg.cn/13553.Doc
m.ekc.hxbsg.cn/37133.Doc
m.ekc.hxbsg.cn/93733.Doc
m.ekc.hxbsg.cn/88282.Doc
m.ekc.hxbsg.cn/17513.Doc
m.ekc.hxbsg.cn/02406.Doc
m.ekc.hxbsg.cn/17135.Doc
m.ekc.hxbsg.cn/24826.Doc
m.ekc.hxbsg.cn/77137.Doc
m.ekc.hxbsg.cn/31391.Doc
m.ekc.hxbsg.cn/99331.Doc
m.ekc.hxbsg.cn/28664.Doc
m.ekc.hxbsg.cn/33573.Doc
m.ekc.hxbsg.cn/99935.Doc
m.ekc.hxbsg.cn/91359.Doc
m.ekc.hxbsg.cn/71315.Doc
m.ekc.hxbsg.cn/31113.Doc
m.ekc.hxbsg.cn/42486.Doc
m.ekc.hxbsg.cn/35573.Doc
m.ekc.hxbsg.cn/79713.Doc
m.ekc.hxbsg.cn/64044.Doc
m.ekx.hxbsg.cn/88082.Doc
m.ekx.hxbsg.cn/91339.Doc
m.ekx.hxbsg.cn/86848.Doc
m.ekx.hxbsg.cn/66002.Doc
m.ekx.hxbsg.cn/99153.Doc
m.ekx.hxbsg.cn/93993.Doc
m.ekx.hxbsg.cn/40626.Doc
m.ekx.hxbsg.cn/26244.Doc
m.ekx.hxbsg.cn/06846.Doc
m.ekx.hxbsg.cn/86644.Doc
m.ekx.hxbsg.cn/42422.Doc
m.ekx.hxbsg.cn/79199.Doc
m.ekx.hxbsg.cn/62602.Doc
m.ekx.hxbsg.cn/71997.Doc
m.ekx.hxbsg.cn/71993.Doc
m.ekx.hxbsg.cn/42044.Doc
m.ekx.hxbsg.cn/37157.Doc
m.ekx.hxbsg.cn/86684.Doc
m.ekx.hxbsg.cn/93333.Doc
m.ekx.hxbsg.cn/99951.Doc
m.ekz.hxbsg.cn/15771.Doc
m.ekz.hxbsg.cn/66266.Doc
m.ekz.hxbsg.cn/37791.Doc
m.ekz.hxbsg.cn/24288.Doc
m.ekz.hxbsg.cn/00444.Doc
m.ekz.hxbsg.cn/57533.Doc
m.ekz.hxbsg.cn/57715.Doc
m.ekz.hxbsg.cn/22280.Doc
m.ekz.hxbsg.cn/75531.Doc
m.ekz.hxbsg.cn/79351.Doc
m.ekz.hxbsg.cn/93373.Doc
m.ekz.hxbsg.cn/95371.Doc
m.ekz.hxbsg.cn/35991.Doc
m.ekz.hxbsg.cn/71955.Doc
m.ekz.hxbsg.cn/75937.Doc
m.ekz.hxbsg.cn/28844.Doc
m.ekz.hxbsg.cn/66228.Doc
m.ekz.hxbsg.cn/82608.Doc
m.ekz.hxbsg.cn/66446.Doc
m.ekz.hxbsg.cn/28462.Doc
m.ekl.hxbsg.cn/59913.Doc
m.ekl.hxbsg.cn/06626.Doc
m.ekl.hxbsg.cn/93573.Doc
m.ekl.hxbsg.cn/75775.Doc
m.ekl.hxbsg.cn/77539.Doc
m.ekl.hxbsg.cn/82200.Doc
m.ekl.hxbsg.cn/97551.Doc
m.ekl.hxbsg.cn/86406.Doc
m.ekl.hxbsg.cn/06882.Doc
m.ekl.hxbsg.cn/80880.Doc
m.ekl.hxbsg.cn/86024.Doc
m.ekl.hxbsg.cn/00080.Doc
m.ekl.hxbsg.cn/80220.Doc
m.ekl.hxbsg.cn/99351.Doc
m.ekl.hxbsg.cn/79591.Doc
m.ekl.hxbsg.cn/79599.Doc
m.ekl.hxbsg.cn/55951.Doc
m.ekl.hxbsg.cn/04628.Doc
m.ekl.hxbsg.cn/02260.Doc
m.ekl.hxbsg.cn/62806.Doc
m.ekk.hxbsg.cn/1.Doc
m.ekk.hxbsg.cn/48206.Doc
m.ekk.hxbsg.cn/00224.Doc
m.ekk.hxbsg.cn/80666.Doc
m.ekk.hxbsg.cn/08208.Doc
m.ekk.hxbsg.cn/86660.Doc
m.ekk.hxbsg.cn/04046.Doc
m.ekk.hxbsg.cn/15951.Doc
m.ekk.hxbsg.cn/95393.Doc
m.ekk.hxbsg.cn/48460.Doc
m.ekk.hxbsg.cn/19973.Doc
m.ekk.hxbsg.cn/04602.Doc
m.ekk.hxbsg.cn/31933.Doc
m.ekk.hxbsg.cn/97537.Doc
m.ekk.hxbsg.cn/91391.Doc
m.ekk.hxbsg.cn/75339.Doc
m.ekk.hxbsg.cn/60424.Doc
m.ekk.hxbsg.cn/91595.Doc
m.ekk.hxbsg.cn/73575.Doc
m.ekk.hxbsg.cn/31977.Doc
m.ekj.hxbsg.cn/15571.Doc
m.ekj.hxbsg.cn/99797.Doc
m.ekj.hxbsg.cn/20680.Doc
m.ekj.hxbsg.cn/06284.Doc
m.ekj.hxbsg.cn/51715.Doc
m.ekj.hxbsg.cn/11971.Doc
m.ekj.hxbsg.cn/80860.Doc
m.ekj.hxbsg.cn/02884.Doc
m.ekj.hxbsg.cn/99335.Doc
m.ekj.hxbsg.cn/33797.Doc
m.ekj.hxbsg.cn/06642.Doc
m.ekj.hxbsg.cn/19559.Doc
m.ekj.hxbsg.cn/79977.Doc
m.ekj.hxbsg.cn/59153.Doc
m.ekj.hxbsg.cn/62260.Doc
m.ekj.hxbsg.cn/46808.Doc
m.ekj.hxbsg.cn/44284.Doc
m.ekj.hxbsg.cn/35539.Doc
m.ekj.hxbsg.cn/48444.Doc
m.ekj.hxbsg.cn/11735.Doc
m.ekh.hxbsg.cn/71579.Doc
m.ekh.hxbsg.cn/82822.Doc
m.ekh.hxbsg.cn/91979.Doc
m.ekh.hxbsg.cn/59735.Doc
m.ekh.hxbsg.cn/91177.Doc
m.ekh.hxbsg.cn/48086.Doc
m.ekh.hxbsg.cn/53175.Doc
m.ekh.hxbsg.cn/91775.Doc
m.ekh.hxbsg.cn/97711.Doc
m.ekh.hxbsg.cn/31939.Doc
m.ekh.hxbsg.cn/99553.Doc
m.ekh.hxbsg.cn/84668.Doc
m.ekh.hxbsg.cn/02828.Doc
m.ekh.hxbsg.cn/44286.Doc
m.ekh.hxbsg.cn/31317.Doc
m.ekh.hxbsg.cn/44602.Doc
m.ekh.hxbsg.cn/79377.Doc
m.ekh.hxbsg.cn/13573.Doc
m.ekh.hxbsg.cn/57935.Doc
m.ekh.hxbsg.cn/11533.Doc
m.ekg.hxbsg.cn/84448.Doc
m.ekg.hxbsg.cn/46828.Doc
m.ekg.hxbsg.cn/40208.Doc
m.ekg.hxbsg.cn/20222.Doc
m.ekg.hxbsg.cn/84640.Doc
m.ekg.hxbsg.cn/24044.Doc
m.ekg.hxbsg.cn/17335.Doc
m.ekg.hxbsg.cn/55351.Doc
m.ekg.hxbsg.cn/13933.Doc
m.ekg.hxbsg.cn/51171.Doc
m.ekg.hxbsg.cn/66264.Doc
m.ekg.hxbsg.cn/68040.Doc
m.ekg.hxbsg.cn/53359.Doc
m.ekg.hxbsg.cn/44684.Doc
m.ekg.hxbsg.cn/35171.Doc
m.ekg.hxbsg.cn/97933.Doc
m.ekg.hxbsg.cn/48286.Doc
m.ekg.hxbsg.cn/06664.Doc
m.ekg.hxbsg.cn/84688.Doc
m.ekg.hxbsg.cn/53773.Doc
m.ekf.hxbsg.cn/68046.Doc
m.ekf.hxbsg.cn/15577.Doc
m.ekf.hxbsg.cn/53793.Doc
m.ekf.hxbsg.cn/60026.Doc
m.ekf.hxbsg.cn/97377.Doc
m.ekf.hxbsg.cn/71115.Doc
m.ekf.hxbsg.cn/99311.Doc
m.ekf.hxbsg.cn/51197.Doc
m.ekf.hxbsg.cn/71757.Doc
m.ekf.hxbsg.cn/77539.Doc
m.ekf.hxbsg.cn/80828.Doc
m.ekf.hxbsg.cn/79117.Doc
m.ekf.hxbsg.cn/97971.Doc
m.ekf.hxbsg.cn/48806.Doc
m.ekf.hxbsg.cn/95751.Doc
m.ekf.hxbsg.cn/53311.Doc
m.ekf.hxbsg.cn/75539.Doc
m.ekf.hxbsg.cn/11955.Doc
m.ekf.hxbsg.cn/51133.Doc
m.ekf.hxbsg.cn/82880.Doc
m.ekd.hxbsg.cn/46486.Doc
m.ekd.hxbsg.cn/55539.Doc
m.ekd.hxbsg.cn/79519.Doc
m.ekd.hxbsg.cn/15719.Doc
m.ekd.hxbsg.cn/80426.Doc
