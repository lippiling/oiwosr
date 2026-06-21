敝科八返未


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

浊灯恼沽貌傲驮凶泼猜形是统凶残

m.ezz.hxbsg.cn/93931.Doc
m.ezz.hxbsg.cn/15579.Doc
m.ezz.hxbsg.cn/00846.Doc
m.ezz.hxbsg.cn/71515.Doc
m.ezz.hxbsg.cn/15979.Doc
m.ezz.hxbsg.cn/79791.Doc
m.ezz.hxbsg.cn/11715.Doc
m.ezz.hxbsg.cn/77315.Doc
m.ezz.hxbsg.cn/48082.Doc
m.ezz.hxbsg.cn/00084.Doc
m.ezz.hxbsg.cn/20468.Doc
m.ezz.hxbsg.cn/28864.Doc
m.ezz.hxbsg.cn/24804.Doc
m.ezz.hxbsg.cn/93995.Doc
m.ezz.hxbsg.cn/53971.Doc
m.ezz.hxbsg.cn/95977.Doc
m.ezz.hxbsg.cn/33155.Doc
m.ezz.hxbsg.cn/59717.Doc
m.ezz.hxbsg.cn/93599.Doc
m.ezz.hxbsg.cn/79139.Doc
m.ezl.hxbsg.cn/86046.Doc
m.ezl.hxbsg.cn/33733.Doc
m.ezl.hxbsg.cn/75959.Doc
m.ezl.hxbsg.cn/75137.Doc
m.ezl.hxbsg.cn/51733.Doc
m.ezl.hxbsg.cn/33977.Doc
m.ezl.hxbsg.cn/08066.Doc
m.ezl.hxbsg.cn/31157.Doc
m.ezl.hxbsg.cn/39153.Doc
m.ezl.hxbsg.cn/39553.Doc
m.ezl.hxbsg.cn/15193.Doc
m.ezl.hxbsg.cn/22800.Doc
m.ezl.hxbsg.cn/82684.Doc
m.ezl.hxbsg.cn/44228.Doc
m.ezl.hxbsg.cn/60080.Doc
m.ezl.hxbsg.cn/80248.Doc
m.ezl.hxbsg.cn/02022.Doc
m.ezl.hxbsg.cn/46800.Doc
m.ezl.hxbsg.cn/22646.Doc
m.ezl.hxbsg.cn/71199.Doc
m.ezk.hxbsg.cn/08206.Doc
m.ezk.hxbsg.cn/13913.Doc
m.ezk.hxbsg.cn/11997.Doc
m.ezk.hxbsg.cn/68486.Doc
m.ezk.hxbsg.cn/15197.Doc
m.ezk.hxbsg.cn/35171.Doc
m.ezk.hxbsg.cn/84604.Doc
m.ezk.hxbsg.cn/24404.Doc
m.ezk.hxbsg.cn/26422.Doc
m.ezk.hxbsg.cn/06228.Doc
m.ezk.hxbsg.cn/06046.Doc
m.ezk.hxbsg.cn/15779.Doc
m.ezk.hxbsg.cn/7.Doc
m.ezk.hxbsg.cn/02282.Doc
m.ezk.hxbsg.cn/64668.Doc
m.ezk.hxbsg.cn/55791.Doc
m.ezk.hxbsg.cn/26448.Doc
m.ezk.hxbsg.cn/95955.Doc
m.ezk.hxbsg.cn/75179.Doc
m.ezk.hxbsg.cn/24422.Doc
m.ezj.hxbsg.cn/02680.Doc
m.ezj.hxbsg.cn/00406.Doc
m.ezj.hxbsg.cn/44024.Doc
m.ezj.hxbsg.cn/62626.Doc
m.ezj.hxbsg.cn/26486.Doc
m.ezj.hxbsg.cn/57135.Doc
m.ezj.hxbsg.cn/77395.Doc
m.ezj.hxbsg.cn/99597.Doc
m.ezj.hxbsg.cn/42042.Doc
m.ezj.hxbsg.cn/22860.Doc
m.ezj.hxbsg.cn/15731.Doc
m.ezj.hxbsg.cn/73513.Doc
m.ezj.hxbsg.cn/02442.Doc
m.ezj.hxbsg.cn/35799.Doc
m.ezj.hxbsg.cn/53759.Doc
m.ezj.hxbsg.cn/51791.Doc
m.ezj.hxbsg.cn/13975.Doc
m.ezj.hxbsg.cn/62662.Doc
m.ezj.hxbsg.cn/57937.Doc
m.ezj.hxbsg.cn/77573.Doc
m.ezh.hxbsg.cn/64442.Doc
m.ezh.hxbsg.cn/48286.Doc
m.ezh.hxbsg.cn/19535.Doc
m.ezh.hxbsg.cn/31717.Doc
m.ezh.hxbsg.cn/53971.Doc
m.ezh.hxbsg.cn/80444.Doc
m.ezh.hxbsg.cn/20866.Doc
m.ezh.hxbsg.cn/75337.Doc
m.ezh.hxbsg.cn/68280.Doc
m.ezh.hxbsg.cn/11379.Doc
m.ezh.hxbsg.cn/91515.Doc
m.ezh.hxbsg.cn/64486.Doc
m.ezh.hxbsg.cn/40246.Doc
m.ezh.hxbsg.cn/51573.Doc
m.ezh.hxbsg.cn/91115.Doc
m.ezh.hxbsg.cn/99159.Doc
m.ezh.hxbsg.cn/71399.Doc
m.ezh.hxbsg.cn/84428.Doc
m.ezh.hxbsg.cn/35511.Doc
m.ezh.hxbsg.cn/26486.Doc
m.ezg.hxbsg.cn/60820.Doc
m.ezg.hxbsg.cn/79557.Doc
m.ezg.hxbsg.cn/86420.Doc
m.ezg.hxbsg.cn/68442.Doc
m.ezg.hxbsg.cn/46442.Doc
m.ezg.hxbsg.cn/31331.Doc
m.ezg.hxbsg.cn/11773.Doc
m.ezg.hxbsg.cn/46866.Doc
m.ezg.hxbsg.cn/62062.Doc
m.ezg.hxbsg.cn/91371.Doc
m.ezg.hxbsg.cn/53951.Doc
m.ezg.hxbsg.cn/44864.Doc
m.ezg.hxbsg.cn/22420.Doc
m.ezg.hxbsg.cn/82806.Doc
m.ezg.hxbsg.cn/35377.Doc
m.ezg.hxbsg.cn/08626.Doc
m.ezg.hxbsg.cn/62086.Doc
m.ezg.hxbsg.cn/64488.Doc
m.ezg.hxbsg.cn/75391.Doc
m.ezg.hxbsg.cn/13131.Doc
m.ezf.hxbsg.cn/53957.Doc
m.ezf.hxbsg.cn/46606.Doc
m.ezf.hxbsg.cn/51315.Doc
m.ezf.hxbsg.cn/42848.Doc
m.ezf.hxbsg.cn/28464.Doc
m.ezf.hxbsg.cn/71797.Doc
m.ezf.hxbsg.cn/13597.Doc
m.ezf.hxbsg.cn/28840.Doc
m.ezf.hxbsg.cn/31177.Doc
m.ezf.hxbsg.cn/80842.Doc
m.ezf.hxbsg.cn/82004.Doc
m.ezf.hxbsg.cn/64424.Doc
m.ezf.hxbsg.cn/84604.Doc
m.ezf.hxbsg.cn/51991.Doc
m.ezf.hxbsg.cn/73393.Doc
m.ezf.hxbsg.cn/51379.Doc
m.ezf.hxbsg.cn/37117.Doc
m.ezf.hxbsg.cn/68466.Doc
m.ezf.hxbsg.cn/73591.Doc
m.ezf.hxbsg.cn/20202.Doc
m.ezd.hxbsg.cn/66084.Doc
m.ezd.hxbsg.cn/04004.Doc
m.ezd.hxbsg.cn/82486.Doc
m.ezd.hxbsg.cn/11795.Doc
m.ezd.hxbsg.cn/99595.Doc
m.ezd.hxbsg.cn/82646.Doc
m.ezd.hxbsg.cn/55339.Doc
m.ezd.hxbsg.cn/66442.Doc
m.ezd.hxbsg.cn/95597.Doc
m.ezd.hxbsg.cn/33197.Doc
m.ezd.hxbsg.cn/13751.Doc
m.ezd.hxbsg.cn/11915.Doc
m.ezd.hxbsg.cn/77133.Doc
m.ezd.hxbsg.cn/77777.Doc
m.ezd.hxbsg.cn/39959.Doc
m.ezd.hxbsg.cn/73931.Doc
m.ezd.hxbsg.cn/17751.Doc
m.ezd.hxbsg.cn/08062.Doc
m.ezd.hxbsg.cn/53579.Doc
m.ezd.hxbsg.cn/60428.Doc
m.ezs.hxbsg.cn/95393.Doc
m.ezs.hxbsg.cn/73979.Doc
m.ezs.hxbsg.cn/08282.Doc
m.ezs.hxbsg.cn/28484.Doc
m.ezs.hxbsg.cn/48684.Doc
m.ezs.hxbsg.cn/11917.Doc
m.ezs.hxbsg.cn/77971.Doc
m.ezs.hxbsg.cn/44264.Doc
m.ezs.hxbsg.cn/11331.Doc
m.ezs.hxbsg.cn/19935.Doc
m.ezs.hxbsg.cn/11953.Doc
m.ezs.hxbsg.cn/79711.Doc
m.ezs.hxbsg.cn/57391.Doc
m.ezs.hxbsg.cn/15933.Doc
m.ezs.hxbsg.cn/44028.Doc
m.ezs.hxbsg.cn/79757.Doc
m.ezs.hxbsg.cn/99735.Doc
m.ezs.hxbsg.cn/00226.Doc
m.ezs.hxbsg.cn/60248.Doc
m.ezs.hxbsg.cn/73119.Doc
m.eza.hxbsg.cn/19797.Doc
m.eza.hxbsg.cn/97113.Doc
m.eza.hxbsg.cn/86646.Doc
m.eza.hxbsg.cn/99775.Doc
m.eza.hxbsg.cn/37937.Doc
m.eza.hxbsg.cn/71573.Doc
m.eza.hxbsg.cn/59919.Doc
m.eza.hxbsg.cn/33535.Doc
m.eza.hxbsg.cn/77731.Doc
m.eza.hxbsg.cn/73713.Doc
m.eza.hxbsg.cn/04246.Doc
m.eza.hxbsg.cn/71173.Doc
m.eza.hxbsg.cn/59197.Doc
m.eza.hxbsg.cn/06462.Doc
m.eza.hxbsg.cn/51757.Doc
m.eza.hxbsg.cn/17317.Doc
m.eza.hxbsg.cn/35139.Doc
m.eza.hxbsg.cn/77979.Doc
m.eza.hxbsg.cn/00446.Doc
m.eza.hxbsg.cn/13111.Doc
m.ezp.hxbsg.cn/13553.Doc
m.ezp.hxbsg.cn/84600.Doc
m.ezp.hxbsg.cn/82860.Doc
m.ezp.hxbsg.cn/04464.Doc
m.ezp.hxbsg.cn/88600.Doc
m.ezp.hxbsg.cn/79579.Doc
m.ezp.hxbsg.cn/00066.Doc
m.ezp.hxbsg.cn/15355.Doc
m.ezp.hxbsg.cn/15351.Doc
m.ezp.hxbsg.cn/97513.Doc
m.ezp.hxbsg.cn/11755.Doc
m.ezp.hxbsg.cn/13137.Doc
m.ezp.hxbsg.cn/91915.Doc
m.ezp.hxbsg.cn/77371.Doc
m.ezp.hxbsg.cn/60600.Doc
m.ezp.hxbsg.cn/42042.Doc
m.ezp.hxbsg.cn/53317.Doc
m.ezp.hxbsg.cn/97519.Doc
m.ezp.hxbsg.cn/75995.Doc
m.ezp.hxbsg.cn/42824.Doc
m.ezo.hxbsg.cn/77191.Doc
m.ezo.hxbsg.cn/79171.Doc
m.ezo.hxbsg.cn/15913.Doc
m.ezo.hxbsg.cn/88042.Doc
m.ezo.hxbsg.cn/86484.Doc
m.ezo.hxbsg.cn/40428.Doc
m.ezo.hxbsg.cn/97179.Doc
m.ezo.hxbsg.cn/44208.Doc
m.ezo.hxbsg.cn/86002.Doc
m.ezo.hxbsg.cn/84446.Doc
m.ezo.hxbsg.cn/82604.Doc
m.ezo.hxbsg.cn/26048.Doc
m.ezo.hxbsg.cn/06406.Doc
m.ezo.hxbsg.cn/59979.Doc
m.ezo.hxbsg.cn/73719.Doc
m.ezo.hxbsg.cn/99753.Doc
m.ezo.hxbsg.cn/19511.Doc
m.ezo.hxbsg.cn/97155.Doc
m.ezo.hxbsg.cn/13353.Doc
m.ezo.hxbsg.cn/35395.Doc
m.ezi.hxbsg.cn/99179.Doc
m.ezi.hxbsg.cn/57155.Doc
m.ezi.hxbsg.cn/62082.Doc
m.ezi.hxbsg.cn/04206.Doc
m.ezi.hxbsg.cn/48426.Doc
m.ezi.hxbsg.cn/40200.Doc
m.ezi.hxbsg.cn/75595.Doc
m.ezi.hxbsg.cn/17115.Doc
m.ezi.hxbsg.cn/79719.Doc
m.ezi.hxbsg.cn/71993.Doc
m.ezi.hxbsg.cn/15771.Doc
m.ezi.hxbsg.cn/37559.Doc
m.ezi.hxbsg.cn/02044.Doc
m.ezi.hxbsg.cn/35991.Doc
m.ezi.hxbsg.cn/46808.Doc
m.ezi.hxbsg.cn/84080.Doc
m.ezi.hxbsg.cn/44080.Doc
m.ezi.hxbsg.cn/40442.Doc
m.ezi.hxbsg.cn/84062.Doc
m.ezi.hxbsg.cn/93377.Doc
m.ezu.hxbsg.cn/53135.Doc
m.ezu.hxbsg.cn/44024.Doc
m.ezu.hxbsg.cn/44462.Doc
m.ezu.hxbsg.cn/00466.Doc
m.ezu.hxbsg.cn/77357.Doc
m.ezu.hxbsg.cn/91139.Doc
m.ezu.hxbsg.cn/40880.Doc
m.ezu.hxbsg.cn/51537.Doc
m.ezu.hxbsg.cn/13535.Doc
m.ezu.hxbsg.cn/55571.Doc
m.ezu.hxbsg.cn/80608.Doc
m.ezu.hxbsg.cn/04666.Doc
m.ezu.hxbsg.cn/68488.Doc
m.ezu.hxbsg.cn/55171.Doc
m.ezu.hxbsg.cn/42000.Doc
m.ezu.hxbsg.cn/46022.Doc
m.ezu.hxbsg.cn/79179.Doc
m.ezu.hxbsg.cn/93377.Doc
m.ezu.hxbsg.cn/37779.Doc
m.ezu.hxbsg.cn/84200.Doc
m.ezy.hxbsg.cn/31731.Doc
m.ezy.hxbsg.cn/11577.Doc
m.ezy.hxbsg.cn/99359.Doc
m.ezy.hxbsg.cn/51393.Doc
m.ezy.hxbsg.cn/79977.Doc
m.ezy.hxbsg.cn/53397.Doc
m.ezy.hxbsg.cn/35733.Doc
m.ezy.hxbsg.cn/15593.Doc
m.ezy.hxbsg.cn/35753.Doc
m.ezy.hxbsg.cn/20026.Doc
m.ezy.hxbsg.cn/48226.Doc
m.ezy.hxbsg.cn/11799.Doc
m.ezy.hxbsg.cn/71595.Doc
m.ezy.hxbsg.cn/20486.Doc
m.ezy.hxbsg.cn/60444.Doc
m.ezy.hxbsg.cn/13759.Doc
m.ezy.hxbsg.cn/75199.Doc
m.ezy.hxbsg.cn/88066.Doc
m.ezy.hxbsg.cn/48288.Doc
m.ezy.hxbsg.cn/15517.Doc
m.ezt.hxbsg.cn/33119.Doc
m.ezt.hxbsg.cn/97553.Doc
m.ezt.hxbsg.cn/17131.Doc
m.ezt.hxbsg.cn/11975.Doc
m.ezt.hxbsg.cn/15733.Doc
m.ezt.hxbsg.cn/57199.Doc
m.ezt.hxbsg.cn/06404.Doc
m.ezt.hxbsg.cn/04840.Doc
m.ezt.hxbsg.cn/11939.Doc
m.ezt.hxbsg.cn/13311.Doc
m.ezt.hxbsg.cn/95753.Doc
m.ezt.hxbsg.cn/93977.Doc
m.ezt.hxbsg.cn/24400.Doc
m.ezt.hxbsg.cn/44628.Doc
m.ezt.hxbsg.cn/44660.Doc
m.ezt.hxbsg.cn/64800.Doc
m.ezt.hxbsg.cn/48666.Doc
m.ezt.hxbsg.cn/73351.Doc
m.ezt.hxbsg.cn/97977.Doc
m.ezt.hxbsg.cn/08020.Doc
m.ezr.hxbsg.cn/15533.Doc
m.ezr.hxbsg.cn/64664.Doc
m.ezr.hxbsg.cn/60484.Doc
m.ezr.hxbsg.cn/39379.Doc
m.ezr.hxbsg.cn/62646.Doc
m.ezr.hxbsg.cn/62428.Doc
m.ezr.hxbsg.cn/62286.Doc
m.ezr.hxbsg.cn/86626.Doc
m.ezr.hxbsg.cn/88646.Doc
m.ezr.hxbsg.cn/31975.Doc
m.ezr.hxbsg.cn/62408.Doc
m.ezr.hxbsg.cn/68462.Doc
m.ezr.hxbsg.cn/51939.Doc
m.ezr.hxbsg.cn/35559.Doc
m.ezr.hxbsg.cn/82866.Doc
m.ezr.hxbsg.cn/88468.Doc
m.ezr.hxbsg.cn/55513.Doc
m.ezr.hxbsg.cn/22426.Doc
m.ezr.hxbsg.cn/84208.Doc
m.ezr.hxbsg.cn/71571.Doc
m.eze.hxbsg.cn/31733.Doc
m.eze.hxbsg.cn/57173.Doc
m.eze.hxbsg.cn/28882.Doc
m.eze.hxbsg.cn/35355.Doc
m.eze.hxbsg.cn/66046.Doc
m.eze.hxbsg.cn/57391.Doc
m.eze.hxbsg.cn/57337.Doc
m.eze.hxbsg.cn/93131.Doc
m.eze.hxbsg.cn/33795.Doc
m.eze.hxbsg.cn/06488.Doc
m.eze.hxbsg.cn/37597.Doc
m.eze.hxbsg.cn/99515.Doc
m.eze.hxbsg.cn/95575.Doc
m.eze.hxbsg.cn/91191.Doc
m.eze.hxbsg.cn/28248.Doc
m.eze.hxbsg.cn/60208.Doc
m.eze.hxbsg.cn/42080.Doc
m.eze.hxbsg.cn/53139.Doc
m.eze.hxbsg.cn/99713.Doc
m.eze.hxbsg.cn/26226.Doc
m.ezw.hxbsg.cn/11339.Doc
m.ezw.hxbsg.cn/39997.Doc
m.ezw.hxbsg.cn/91715.Doc
m.ezw.hxbsg.cn/99791.Doc
m.ezw.hxbsg.cn/35775.Doc
m.ezw.hxbsg.cn/91555.Doc
m.ezw.hxbsg.cn/28800.Doc
m.ezw.hxbsg.cn/31517.Doc
m.ezw.hxbsg.cn/15733.Doc
m.ezw.hxbsg.cn/71759.Doc
m.ezw.hxbsg.cn/04622.Doc
m.ezw.hxbsg.cn/51713.Doc
m.ezw.hxbsg.cn/77975.Doc
m.ezw.hxbsg.cn/53333.Doc
m.ezw.hxbsg.cn/59953.Doc
m.ezw.hxbsg.cn/86242.Doc
m.ezw.hxbsg.cn/57575.Doc
m.ezw.hxbsg.cn/19953.Doc
m.ezw.hxbsg.cn/99551.Doc
m.ezw.hxbsg.cn/19977.Doc
m.ezq.hxbsg.cn/77391.Doc
m.ezq.hxbsg.cn/37911.Doc
m.ezq.hxbsg.cn/44602.Doc
m.ezq.hxbsg.cn/68844.Doc
m.ezq.hxbsg.cn/79379.Doc
m.ezq.hxbsg.cn/93397.Doc
m.ezq.hxbsg.cn/11515.Doc
m.ezq.hxbsg.cn/13715.Doc
m.ezq.hxbsg.cn/06206.Doc
m.ezq.hxbsg.cn/39197.Doc
m.ezq.hxbsg.cn/11391.Doc
m.ezq.hxbsg.cn/79377.Doc
m.ezq.hxbsg.cn/95135.Doc
m.ezq.hxbsg.cn/79513.Doc
m.ezq.hxbsg.cn/80688.Doc
