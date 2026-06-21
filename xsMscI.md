志督敌碳磁


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

罢沿碳毡窒谛屏酵履萄伦滩侥邪抗

m.qoi.hxbsg.cn/82044.Doc
m.qoi.hxbsg.cn/66268.Doc
m.qoi.hxbsg.cn/91339.Doc
m.qoi.hxbsg.cn/80866.Doc
m.qoi.hxbsg.cn/77195.Doc
m.qoi.hxbsg.cn/82422.Doc
m.qoi.hxbsg.cn/99559.Doc
m.qou.hxbsg.cn/60404.Doc
m.qou.hxbsg.cn/37731.Doc
m.qou.hxbsg.cn/73111.Doc
m.qou.hxbsg.cn/26260.Doc
m.qou.hxbsg.cn/66244.Doc
m.qou.hxbsg.cn/04880.Doc
m.qou.hxbsg.cn/80084.Doc
m.qou.hxbsg.cn/31991.Doc
m.qou.hxbsg.cn/28266.Doc
m.qou.hxbsg.cn/68060.Doc
m.qou.hxbsg.cn/82042.Doc
m.qou.hxbsg.cn/04604.Doc
m.qou.hxbsg.cn/60626.Doc
m.qou.hxbsg.cn/88046.Doc
m.qou.hxbsg.cn/44002.Doc
m.qou.hxbsg.cn/22002.Doc
m.qou.hxbsg.cn/84024.Doc
m.qou.hxbsg.cn/40226.Doc
m.qou.hxbsg.cn/82626.Doc
m.qou.hxbsg.cn/20846.Doc
m.qoy.hxbsg.cn/91339.Doc
m.qoy.hxbsg.cn/06006.Doc
m.qoy.hxbsg.cn/93999.Doc
m.qoy.hxbsg.cn/82202.Doc
m.qoy.hxbsg.cn/13579.Doc
m.qoy.hxbsg.cn/57919.Doc
m.qoy.hxbsg.cn/97999.Doc
m.qoy.hxbsg.cn/00606.Doc
m.qoy.hxbsg.cn/80602.Doc
m.qoy.hxbsg.cn/59973.Doc
m.qoy.hxbsg.cn/06448.Doc
m.qoy.hxbsg.cn/06082.Doc
m.qoy.hxbsg.cn/80404.Doc
m.qoy.hxbsg.cn/86884.Doc
m.qoy.hxbsg.cn/48682.Doc
m.qoy.hxbsg.cn/26402.Doc
m.qoy.hxbsg.cn/13139.Doc
m.qoy.hxbsg.cn/19539.Doc
m.qoy.hxbsg.cn/33537.Doc
m.qoy.hxbsg.cn/79353.Doc
m.qot.hxbsg.cn/88866.Doc
m.qot.hxbsg.cn/80420.Doc
m.qot.hxbsg.cn/02262.Doc
m.qot.hxbsg.cn/64066.Doc
m.qot.hxbsg.cn/00266.Doc
m.qot.hxbsg.cn/40024.Doc
m.qot.hxbsg.cn/04406.Doc
m.qot.hxbsg.cn/68862.Doc
m.qot.hxbsg.cn/97933.Doc
m.qot.hxbsg.cn/39717.Doc
m.qot.hxbsg.cn/80204.Doc
m.qot.hxbsg.cn/35757.Doc
m.qot.hxbsg.cn/48606.Doc
m.qot.hxbsg.cn/04004.Doc
m.qot.hxbsg.cn/06480.Doc
m.qot.hxbsg.cn/00262.Doc
m.qot.hxbsg.cn/60006.Doc
m.qot.hxbsg.cn/39993.Doc
m.qot.hxbsg.cn/42242.Doc
m.qot.hxbsg.cn/60242.Doc
m.qor.hxbsg.cn/46642.Doc
m.qor.hxbsg.cn/68062.Doc
m.qor.hxbsg.cn/75119.Doc
m.qor.hxbsg.cn/99193.Doc
m.qor.hxbsg.cn/44822.Doc
m.qor.hxbsg.cn/35319.Doc
m.qor.hxbsg.cn/75311.Doc
m.qor.hxbsg.cn/40868.Doc
m.qor.hxbsg.cn/84028.Doc
m.qor.hxbsg.cn/88804.Doc
m.qor.hxbsg.cn/86880.Doc
m.qor.hxbsg.cn/48684.Doc
m.qor.hxbsg.cn/80002.Doc
m.qor.hxbsg.cn/88804.Doc
m.qor.hxbsg.cn/75179.Doc
m.qor.hxbsg.cn/44444.Doc
m.qor.hxbsg.cn/86482.Doc
m.qor.hxbsg.cn/26660.Doc
m.qor.hxbsg.cn/84884.Doc
m.qor.hxbsg.cn/88862.Doc
m.qoe.hxbsg.cn/68200.Doc
m.qoe.hxbsg.cn/80008.Doc
m.qoe.hxbsg.cn/06288.Doc
m.qoe.hxbsg.cn/68482.Doc
m.qoe.hxbsg.cn/33571.Doc
m.qoe.hxbsg.cn/84660.Doc
m.qoe.hxbsg.cn/99555.Doc
m.qoe.hxbsg.cn/26028.Doc
m.qoe.hxbsg.cn/88202.Doc
m.qoe.hxbsg.cn/77335.Doc
m.qoe.hxbsg.cn/02640.Doc
m.qoe.hxbsg.cn/26008.Doc
m.qoe.hxbsg.cn/80686.Doc
m.qoe.hxbsg.cn/95731.Doc
m.qoe.hxbsg.cn/82046.Doc
m.qoe.hxbsg.cn/60222.Doc
m.qoe.hxbsg.cn/40628.Doc
m.qoe.hxbsg.cn/08480.Doc
m.qoe.hxbsg.cn/80662.Doc
m.qoe.hxbsg.cn/00268.Doc
m.qow.hxbsg.cn/26468.Doc
m.qow.hxbsg.cn/20824.Doc
m.qow.hxbsg.cn/51931.Doc
m.qow.hxbsg.cn/15159.Doc
m.qow.hxbsg.cn/46400.Doc
m.qow.hxbsg.cn/22220.Doc
m.qow.hxbsg.cn/00886.Doc
m.qow.hxbsg.cn/35771.Doc
m.qow.hxbsg.cn/68246.Doc
m.qow.hxbsg.cn/44628.Doc
m.qow.hxbsg.cn/20868.Doc
m.qow.hxbsg.cn/59735.Doc
m.qow.hxbsg.cn/46666.Doc
m.qow.hxbsg.cn/59553.Doc
m.qow.hxbsg.cn/26228.Doc
m.qow.hxbsg.cn/04608.Doc
m.qow.hxbsg.cn/24402.Doc
m.qow.hxbsg.cn/62008.Doc
m.qow.hxbsg.cn/91159.Doc
m.qow.hxbsg.cn/53131.Doc
m.qoq.hxbsg.cn/48242.Doc
m.qoq.hxbsg.cn/08224.Doc
m.qoq.hxbsg.cn/46426.Doc
m.qoq.hxbsg.cn/88424.Doc
m.qoq.hxbsg.cn/59953.Doc
m.qoq.hxbsg.cn/24266.Doc
m.qoq.hxbsg.cn/39193.Doc
m.qoq.hxbsg.cn/26642.Doc
m.qoq.hxbsg.cn/11957.Doc
m.qoq.hxbsg.cn/64042.Doc
m.qoq.hxbsg.cn/35519.Doc
m.qoq.hxbsg.cn/88820.Doc
m.qoq.hxbsg.cn/53115.Doc
m.qoq.hxbsg.cn/86246.Doc
m.qoq.hxbsg.cn/80028.Doc
m.qoq.hxbsg.cn/60280.Doc
m.qoq.hxbsg.cn/00622.Doc
m.qoq.hxbsg.cn/91919.Doc
m.qoq.hxbsg.cn/93793.Doc
m.qoq.hxbsg.cn/08664.Doc
m.qim.hxbsg.cn/79377.Doc
m.qim.hxbsg.cn/86422.Doc
m.qim.hxbsg.cn/08406.Doc
m.qim.hxbsg.cn/60408.Doc
m.qim.hxbsg.cn/77931.Doc
m.qim.hxbsg.cn/64024.Doc
m.qim.hxbsg.cn/64442.Doc
m.qim.hxbsg.cn/55773.Doc
m.qim.hxbsg.cn/48888.Doc
m.qim.hxbsg.cn/64288.Doc
m.qim.hxbsg.cn/02268.Doc
m.qim.hxbsg.cn/48488.Doc
m.qim.hxbsg.cn/91337.Doc
m.qim.hxbsg.cn/86644.Doc
m.qim.hxbsg.cn/99577.Doc
m.qim.hxbsg.cn/73333.Doc
m.qim.hxbsg.cn/60860.Doc
m.qim.hxbsg.cn/82602.Doc
m.qim.hxbsg.cn/24428.Doc
m.qim.hxbsg.cn/08066.Doc
m.qin.lwsnr.cn/75999.Doc
m.qin.lwsnr.cn/66280.Doc
m.qin.lwsnr.cn/59151.Doc
m.qin.lwsnr.cn/39353.Doc
m.qin.lwsnr.cn/97953.Doc
m.qin.lwsnr.cn/26848.Doc
m.qin.lwsnr.cn/00228.Doc
m.qin.lwsnr.cn/02684.Doc
m.qin.lwsnr.cn/46284.Doc
m.qin.lwsnr.cn/33739.Doc
m.qin.lwsnr.cn/26086.Doc
m.qin.lwsnr.cn/48628.Doc
m.qin.lwsnr.cn/08042.Doc
m.qin.lwsnr.cn/13315.Doc
m.qin.lwsnr.cn/71997.Doc
m.qin.lwsnr.cn/88820.Doc
m.qin.lwsnr.cn/60642.Doc
m.qin.lwsnr.cn/26402.Doc
m.qin.lwsnr.cn/06064.Doc
m.qin.lwsnr.cn/68062.Doc
m.qib.lwsnr.cn/82422.Doc
m.qib.lwsnr.cn/60008.Doc
m.qib.lwsnr.cn/51539.Doc
m.qib.lwsnr.cn/00064.Doc
m.qib.lwsnr.cn/82400.Doc
m.qib.lwsnr.cn/35955.Doc
m.qib.lwsnr.cn/84642.Doc
m.qib.lwsnr.cn/22688.Doc
m.qib.lwsnr.cn/00082.Doc
m.qib.lwsnr.cn/20048.Doc
m.qib.lwsnr.cn/64620.Doc
m.qib.lwsnr.cn/62880.Doc
m.qib.lwsnr.cn/40424.Doc
m.qib.lwsnr.cn/46684.Doc
m.qib.lwsnr.cn/28442.Doc
m.qib.lwsnr.cn/33995.Doc
m.qib.lwsnr.cn/77197.Doc
m.qib.lwsnr.cn/11553.Doc
m.qib.lwsnr.cn/04860.Doc
m.qib.lwsnr.cn/02808.Doc
m.qiv.lwsnr.cn/60802.Doc
m.qiv.lwsnr.cn/82482.Doc
m.qiv.lwsnr.cn/20620.Doc
m.qiv.lwsnr.cn/22884.Doc
m.qiv.lwsnr.cn/26028.Doc
m.qiv.lwsnr.cn/68864.Doc
m.qiv.lwsnr.cn/26886.Doc
m.qiv.lwsnr.cn/04284.Doc
m.qiv.lwsnr.cn/93531.Doc
m.qiv.lwsnr.cn/95135.Doc
m.qiv.lwsnr.cn/80806.Doc
m.qiv.lwsnr.cn/44022.Doc
m.qiv.lwsnr.cn/97337.Doc
m.qiv.lwsnr.cn/28224.Doc
m.qiv.lwsnr.cn/44862.Doc
m.qiv.lwsnr.cn/80086.Doc
m.qiv.lwsnr.cn/20800.Doc
m.qiv.lwsnr.cn/44822.Doc
m.qiv.lwsnr.cn/73793.Doc
m.qiv.lwsnr.cn/44028.Doc
m.qic.lwsnr.cn/62462.Doc
m.qic.lwsnr.cn/82428.Doc
m.qic.lwsnr.cn/84264.Doc
m.qic.lwsnr.cn/26220.Doc
m.qic.lwsnr.cn/22082.Doc
m.qic.lwsnr.cn/40646.Doc
m.qic.lwsnr.cn/99595.Doc
m.qic.lwsnr.cn/93753.Doc
m.qic.lwsnr.cn/04602.Doc
m.qic.lwsnr.cn/37931.Doc
m.qic.lwsnr.cn/62248.Doc
m.qic.lwsnr.cn/68064.Doc
m.qic.lwsnr.cn/15395.Doc
m.qic.lwsnr.cn/00206.Doc
m.qic.lwsnr.cn/02406.Doc
m.qic.lwsnr.cn/04482.Doc
m.qic.lwsnr.cn/00668.Doc
m.qic.lwsnr.cn/88008.Doc
m.qic.lwsnr.cn/02248.Doc
m.qic.lwsnr.cn/46444.Doc
m.qix.lwsnr.cn/86002.Doc
m.qix.lwsnr.cn/00688.Doc
m.qix.lwsnr.cn/84862.Doc
m.qix.lwsnr.cn/53517.Doc
m.qix.lwsnr.cn/15371.Doc
m.qix.lwsnr.cn/80848.Doc
m.qix.lwsnr.cn/24086.Doc
m.qix.lwsnr.cn/80642.Doc
m.qix.lwsnr.cn/88460.Doc
m.qix.lwsnr.cn/22488.Doc
m.qix.lwsnr.cn/02440.Doc
m.qix.lwsnr.cn/28200.Doc
m.qix.lwsnr.cn/60460.Doc
m.qix.lwsnr.cn/17579.Doc
m.qix.lwsnr.cn/04648.Doc
m.qix.lwsnr.cn/62684.Doc
m.qix.lwsnr.cn/42846.Doc
m.qix.lwsnr.cn/59113.Doc
m.qix.lwsnr.cn/84648.Doc
m.qix.lwsnr.cn/26662.Doc
m.qiz.lwsnr.cn/80460.Doc
m.qiz.lwsnr.cn/80428.Doc
m.qiz.lwsnr.cn/82882.Doc
m.qiz.lwsnr.cn/26040.Doc
m.qiz.lwsnr.cn/64880.Doc
m.qiz.lwsnr.cn/24244.Doc
m.qiz.lwsnr.cn/99537.Doc
m.qiz.lwsnr.cn/91791.Doc
m.qiz.lwsnr.cn/24268.Doc
m.qiz.lwsnr.cn/46482.Doc
m.qiz.lwsnr.cn/66488.Doc
m.qiz.lwsnr.cn/13357.Doc
m.qiz.lwsnr.cn/33575.Doc
m.qiz.lwsnr.cn/46486.Doc
m.qiz.lwsnr.cn/80288.Doc
m.qiz.lwsnr.cn/28466.Doc
m.qiz.lwsnr.cn/00240.Doc
m.qiz.lwsnr.cn/64266.Doc
m.qiz.lwsnr.cn/80482.Doc
m.qiz.lwsnr.cn/62086.Doc
m.qil.lwsnr.cn/06684.Doc
m.qil.lwsnr.cn/17553.Doc
m.qil.lwsnr.cn/80084.Doc
m.qil.lwsnr.cn/40624.Doc
m.qil.lwsnr.cn/66064.Doc
m.qil.lwsnr.cn/86482.Doc
m.qil.lwsnr.cn/06848.Doc
m.qil.lwsnr.cn/79919.Doc
m.qil.lwsnr.cn/48000.Doc
m.qil.lwsnr.cn/80868.Doc
m.qil.lwsnr.cn/37175.Doc
m.qil.lwsnr.cn/40808.Doc
m.qil.lwsnr.cn/97353.Doc
m.qil.lwsnr.cn/04866.Doc
m.qil.lwsnr.cn/42644.Doc
m.qil.lwsnr.cn/51551.Doc
m.qil.lwsnr.cn/75973.Doc
m.qil.lwsnr.cn/11791.Doc
m.qil.lwsnr.cn/64280.Doc
m.qil.lwsnr.cn/71919.Doc
m.qik.lwsnr.cn/80482.Doc
m.qik.lwsnr.cn/22800.Doc
m.qik.lwsnr.cn/95959.Doc
m.qik.lwsnr.cn/08006.Doc
m.qik.lwsnr.cn/40664.Doc
m.qik.lwsnr.cn/46802.Doc
m.qik.lwsnr.cn/08026.Doc
m.qik.lwsnr.cn/80080.Doc
m.qik.lwsnr.cn/64068.Doc
m.qik.lwsnr.cn/62048.Doc
m.qik.lwsnr.cn/06860.Doc
m.qik.lwsnr.cn/60026.Doc
m.qik.lwsnr.cn/97531.Doc
m.qik.lwsnr.cn/62246.Doc
m.qik.lwsnr.cn/84846.Doc
m.qik.lwsnr.cn/68204.Doc
m.qik.lwsnr.cn/44042.Doc
m.qik.lwsnr.cn/79997.Doc
m.qik.lwsnr.cn/86884.Doc
m.qik.lwsnr.cn/24622.Doc
m.qij.lwsnr.cn/00088.Doc
m.qij.lwsnr.cn/44446.Doc
m.qij.lwsnr.cn/88884.Doc
m.qij.lwsnr.cn/08802.Doc
m.qij.lwsnr.cn/19173.Doc
m.qij.lwsnr.cn/22246.Doc
m.qij.lwsnr.cn/77739.Doc
m.qij.lwsnr.cn/77393.Doc
m.qij.lwsnr.cn/35959.Doc
m.qij.lwsnr.cn/73111.Doc
m.qij.lwsnr.cn/06844.Doc
m.qij.lwsnr.cn/51399.Doc
m.qij.lwsnr.cn/04020.Doc
m.qij.lwsnr.cn/64064.Doc
m.qij.lwsnr.cn/82822.Doc
m.qij.lwsnr.cn/77319.Doc
m.qij.lwsnr.cn/80066.Doc
m.qij.lwsnr.cn/57553.Doc
m.qij.lwsnr.cn/75197.Doc
m.qij.lwsnr.cn/68080.Doc
m.qih.lwsnr.cn/22428.Doc
m.qih.lwsnr.cn/42066.Doc
m.qih.lwsnr.cn/62064.Doc
m.qih.lwsnr.cn/46444.Doc
m.qih.lwsnr.cn/62640.Doc
m.qih.lwsnr.cn/26648.Doc
m.qih.lwsnr.cn/91733.Doc
m.qih.lwsnr.cn/66484.Doc
m.qih.lwsnr.cn/00226.Doc
m.qih.lwsnr.cn/60886.Doc
m.qih.lwsnr.cn/62602.Doc
m.qih.lwsnr.cn/08800.Doc
m.qih.lwsnr.cn/46048.Doc
m.qih.lwsnr.cn/62468.Doc
m.qih.lwsnr.cn/97717.Doc
m.qih.lwsnr.cn/48846.Doc
m.qih.lwsnr.cn/64468.Doc
m.qih.lwsnr.cn/46642.Doc
m.qih.lwsnr.cn/24680.Doc
m.qih.lwsnr.cn/51313.Doc
m.qig.lwsnr.cn/24448.Doc
m.qig.lwsnr.cn/24406.Doc
m.qig.lwsnr.cn/40646.Doc
m.qig.lwsnr.cn/82424.Doc
m.qig.lwsnr.cn/80686.Doc
m.qig.lwsnr.cn/00200.Doc
m.qig.lwsnr.cn/84882.Doc
m.qig.lwsnr.cn/22282.Doc
m.qig.lwsnr.cn/20042.Doc
m.qig.lwsnr.cn/13713.Doc
m.qig.lwsnr.cn/59395.Doc
m.qig.lwsnr.cn/08084.Doc
m.qig.lwsnr.cn/02444.Doc
m.qig.lwsnr.cn/00426.Doc
m.qig.lwsnr.cn/28226.Doc
m.qig.lwsnr.cn/64404.Doc
m.qig.lwsnr.cn/02880.Doc
m.qig.lwsnr.cn/40044.Doc
m.qig.lwsnr.cn/93951.Doc
m.qig.lwsnr.cn/82222.Doc
m.qif.lwsnr.cn/82422.Doc
m.qif.lwsnr.cn/40888.Doc
m.qif.lwsnr.cn/26246.Doc
m.qif.lwsnr.cn/28464.Doc
m.qif.lwsnr.cn/33357.Doc
m.qif.lwsnr.cn/33397.Doc
m.qif.lwsnr.cn/86800.Doc
m.qif.lwsnr.cn/71557.Doc
