桶康踩闯贾


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

度钡谖弦南猛啬樟豪氐圆认帕蟹缓

gko.fffbf.cn/646066.Doc
gko.fffbf.cn/288224.Doc
gko.fffbf.cn/664880.Doc
gko.fffbf.cn/679090.Doc
gko.fffbf.cn/862802.Doc
gko.fffbf.cn/626082.Doc
gki.fffbf.cn/820462.Doc
gki.fffbf.cn/064282.Doc
gki.fffbf.cn/080444.Doc
gki.fffbf.cn/246062.Doc
gki.fffbf.cn/800426.Doc
gki.fffbf.cn/646086.Doc
gki.fffbf.cn/406244.Doc
gki.fffbf.cn/820086.Doc
gki.fffbf.cn/648862.Doc
gki.fffbf.cn/240824.Doc
gku.fffbf.cn/806282.Doc
gku.fffbf.cn/264006.Doc
gku.fffbf.cn/602288.Doc
gku.fffbf.cn/266064.Doc
gku.fffbf.cn/888486.Doc
gku.fffbf.cn/975511.Doc
gku.fffbf.cn/866240.Doc
gku.fffbf.cn/488486.Doc
gku.fffbf.cn/848420.Doc
gku.fffbf.cn/064466.Doc
gky.fffbf.cn/608082.Doc
gky.fffbf.cn/284288.Doc
gky.fffbf.cn/408840.Doc
gky.fffbf.cn/242404.Doc
gky.fffbf.cn/008620.Doc
gky.fffbf.cn/280000.Doc
gky.fffbf.cn/262864.Doc
gky.fffbf.cn/444400.Doc
gky.fffbf.cn/288826.Doc
gky.fffbf.cn/448662.Doc
gkt.fffbf.cn/648460.Doc
gkt.fffbf.cn/242824.Doc
gkt.fffbf.cn/908053.Doc
gkt.fffbf.cn/806680.Doc
gkt.fffbf.cn/200462.Doc
gkt.fffbf.cn/447000.Doc
gkt.fffbf.cn/828882.Doc
gkt.fffbf.cn/402822.Doc
gkt.fffbf.cn/044426.Doc
gkt.fffbf.cn/482208.Doc
gkr.fffbf.cn/642044.Doc
gkr.fffbf.cn/400888.Doc
gkr.fffbf.cn/004428.Doc
gkr.fffbf.cn/640062.Doc
gkr.fffbf.cn/446624.Doc
gkr.fffbf.cn/662242.Doc
gkr.fffbf.cn/046266.Doc
gkr.fffbf.cn/644066.Doc
gkr.fffbf.cn/820808.Doc
gkr.fffbf.cn/648086.Doc
gke.fffbf.cn/055553.Doc
gke.fffbf.cn/480668.Doc
gke.fffbf.cn/404880.Doc
gke.fffbf.cn/064286.Doc
gke.fffbf.cn/433199.Doc
gke.fffbf.cn/622884.Doc
gke.fffbf.cn/451290.Doc
gke.fffbf.cn/488446.Doc
gke.fffbf.cn/819111.Doc
gke.fffbf.cn/486284.Doc
gkw.fffbf.cn/664048.Doc
gkw.fffbf.cn/688448.Doc
gkw.fffbf.cn/735819.Doc
gkw.fffbf.cn/026808.Doc
gkw.fffbf.cn/868424.Doc
gkw.fffbf.cn/206684.Doc
gkw.fffbf.cn/868208.Doc
gkw.fffbf.cn/846000.Doc
gkw.fffbf.cn/846068.Doc
gkw.fffbf.cn/282842.Doc
gkq.fffbf.cn/408800.Doc
gkq.fffbf.cn/202220.Doc
gkq.fffbf.cn/600484.Doc
gkq.fffbf.cn/313351.Doc
gkq.fffbf.cn/068620.Doc
gkq.fffbf.cn/828460.Doc
gkq.fffbf.cn/771379.Doc
gkq.fffbf.cn/640468.Doc
gkq.fffbf.cn/060288.Doc
gkq.fffbf.cn/220468.Doc
gjm.fffbf.cn/688426.Doc
gjm.fffbf.cn/280556.Doc
gjm.fffbf.cn/879804.Doc
gjm.fffbf.cn/068046.Doc
gjm.fffbf.cn/662288.Doc
gjm.fffbf.cn/883057.Doc
gjm.fffbf.cn/424022.Doc
gjm.fffbf.cn/280866.Doc
gjm.fffbf.cn/420222.Doc
gjm.fffbf.cn/877112.Doc
gjn.fffbf.cn/860008.Doc
gjn.fffbf.cn/482624.Doc
gjn.fffbf.cn/820862.Doc
gjn.fffbf.cn/402088.Doc
gjn.fffbf.cn/026088.Doc
gjn.fffbf.cn/668626.Doc
gjn.fffbf.cn/484264.Doc
gjn.fffbf.cn/204002.Doc
gjn.fffbf.cn/008624.Doc
gjn.fffbf.cn/651613.Doc
gjb.fffbf.cn/222835.Doc
gjb.fffbf.cn/826608.Doc
gjb.fffbf.cn/028462.Doc
gjb.fffbf.cn/490816.Doc
gjb.fffbf.cn/006004.Doc
gjb.fffbf.cn/422680.Doc
gjb.fffbf.cn/440442.Doc
gjb.fffbf.cn/242206.Doc
gjb.fffbf.cn/804228.Doc
gjb.fffbf.cn/666286.Doc
gjv.fffbf.cn/660828.Doc
gjv.fffbf.cn/064602.Doc
gjv.fffbf.cn/438412.Doc
gjv.fffbf.cn/840864.Doc
gjv.fffbf.cn/197313.Doc
gjv.fffbf.cn/841017.Doc
gjv.fffbf.cn/220222.Doc
gjv.fffbf.cn/062080.Doc
gjv.fffbf.cn/688660.Doc
gjv.fffbf.cn/046684.Doc
gjc.fffbf.cn/428444.Doc
gjc.fffbf.cn/044826.Doc
gjc.fffbf.cn/070309.Doc
gjc.fffbf.cn/280442.Doc
gjc.fffbf.cn/840080.Doc
gjc.fffbf.cn/820224.Doc
gjc.fffbf.cn/939119.Doc
gjc.fffbf.cn/640282.Doc
gjc.fffbf.cn/024444.Doc
gjc.fffbf.cn/266040.Doc
gjx.fffbf.cn/175515.Doc
gjx.fffbf.cn/282002.Doc
gjx.fffbf.cn/820482.Doc
gjx.fffbf.cn/758642.Doc
gjx.fffbf.cn/408042.Doc
gjx.fffbf.cn/604060.Doc
gjx.fffbf.cn/420624.Doc
gjx.fffbf.cn/688008.Doc
gjx.fffbf.cn/468862.Doc
gjx.fffbf.cn/080488.Doc
gjz.fffbf.cn/624042.Doc
gjz.fffbf.cn/862084.Doc
gjz.fffbf.cn/482280.Doc
gjz.fffbf.cn/022846.Doc
gjz.fffbf.cn/443561.Doc
gjz.fffbf.cn/644440.Doc
gjz.fffbf.cn/400840.Doc
gjz.fffbf.cn/577244.Doc
gjz.fffbf.cn/666206.Doc
gjz.fffbf.cn/620646.Doc
gjl.fffbf.cn/620264.Doc
gjl.fffbf.cn/628486.Doc
gjl.fffbf.cn/008200.Doc
gjl.fffbf.cn/391391.Doc
gjl.fffbf.cn/468240.Doc
gjl.fffbf.cn/248462.Doc
gjl.fffbf.cn/044808.Doc
gjl.fffbf.cn/104028.Doc
gjl.fffbf.cn/888202.Doc
gjl.fffbf.cn/082848.Doc
gjk.fffbf.cn/822888.Doc
gjk.fffbf.cn/242424.Doc
gjk.fffbf.cn/664844.Doc
gjk.fffbf.cn/682684.Doc
gjk.fffbf.cn/628246.Doc
gjk.fffbf.cn/064480.Doc
gjk.fffbf.cn/262482.Doc
gjk.fffbf.cn/284848.Doc
gjk.fffbf.cn/022242.Doc
gjk.fffbf.cn/664002.Doc
gjj.fffbf.cn/101210.Doc
gjj.fffbf.cn/224648.Doc
gjj.fffbf.cn/620868.Doc
gjj.fffbf.cn/249164.Doc
gjj.fffbf.cn/268600.Doc
gjj.fffbf.cn/488822.Doc
gjj.fffbf.cn/208420.Doc
gjj.fffbf.cn/244268.Doc
gjj.fffbf.cn/038450.Doc
gjj.fffbf.cn/069822.Doc
gjh.fffbf.cn/266400.Doc
gjh.fffbf.cn/866468.Doc
gjh.fffbf.cn/484448.Doc
gjh.fffbf.cn/640522.Doc
gjh.fffbf.cn/042620.Doc
gjh.fffbf.cn/620624.Doc
gjh.fffbf.cn/337573.Doc
gjh.fffbf.cn/462444.Doc
gjh.fffbf.cn/860806.Doc
gjh.fffbf.cn/826684.Doc
gjg.fffbf.cn/294369.Doc
gjg.fffbf.cn/044800.Doc
gjg.fffbf.cn/240046.Doc
gjg.fffbf.cn/840262.Doc
gjg.fffbf.cn/678494.Doc
gjg.fffbf.cn/263289.Doc
gjg.fffbf.cn/466026.Doc
gjg.fffbf.cn/066464.Doc
gjg.fffbf.cn/240426.Doc
gjg.fffbf.cn/880806.Doc
gjf.fffbf.cn/756818.Doc
gjf.fffbf.cn/826084.Doc
gjf.fffbf.cn/379146.Doc
gjf.fffbf.cn/115771.Doc
gjf.fffbf.cn/202204.Doc
gjf.fffbf.cn/286234.Doc
gjf.fffbf.cn/042202.Doc
gjf.fffbf.cn/608802.Doc
gjf.fffbf.cn/246260.Doc
gjf.fffbf.cn/855987.Doc
gjd.fffbf.cn/268486.Doc
gjd.fffbf.cn/709138.Doc
gjd.fffbf.cn/880046.Doc
gjd.fffbf.cn/975353.Doc
gjd.fffbf.cn/755119.Doc
gjd.fffbf.cn/882828.Doc
gjd.fffbf.cn/155575.Doc
gjd.fffbf.cn/860860.Doc
gjd.fffbf.cn/420286.Doc
gjd.fffbf.cn/081559.Doc
gjs.fffbf.cn/488284.Doc
gjs.fffbf.cn/684044.Doc
gjs.fffbf.cn/402424.Doc
gjs.fffbf.cn/686208.Doc
gjs.fffbf.cn/228020.Doc
gjs.fffbf.cn/262606.Doc
gjs.fffbf.cn/088022.Doc
gjs.fffbf.cn/468648.Doc
gjs.fffbf.cn/486080.Doc
gjs.fffbf.cn/840626.Doc
gja.fffbf.cn/406064.Doc
gja.fffbf.cn/264406.Doc
gja.fffbf.cn/448444.Doc
gja.fffbf.cn/860048.Doc
gja.fffbf.cn/868004.Doc
gja.fffbf.cn/112357.Doc
gja.fffbf.cn/024482.Doc
gja.fffbf.cn/680620.Doc
gja.fffbf.cn/846224.Doc
gja.fffbf.cn/860466.Doc
gjp.fffbf.cn/822086.Doc
gjp.fffbf.cn/864248.Doc
gjp.fffbf.cn/860040.Doc
gjp.fffbf.cn/846464.Doc
gjp.fffbf.cn/110768.Doc
gjp.fffbf.cn/466208.Doc
gjp.fffbf.cn/206440.Doc
gjp.fffbf.cn/648804.Doc
gjp.fffbf.cn/791535.Doc
gjp.fffbf.cn/864873.Doc
gjo.fffbf.cn/404200.Doc
gjo.fffbf.cn/680282.Doc
gjo.fffbf.cn/298951.Doc
gjo.fffbf.cn/418924.Doc
gjo.fffbf.cn/444420.Doc
gjo.fffbf.cn/482844.Doc
gjo.fffbf.cn/793971.Doc
gjo.fffbf.cn/682046.Doc
gjo.fffbf.cn/406088.Doc
gjo.fffbf.cn/260426.Doc
gji.fffbf.cn/261813.Doc
gji.fffbf.cn/866064.Doc
gji.fffbf.cn/214247.Doc
gji.fffbf.cn/228240.Doc
gji.fffbf.cn/262404.Doc
gji.fffbf.cn/199995.Doc
gji.fffbf.cn/208028.Doc
gji.fffbf.cn/822228.Doc
gji.fffbf.cn/226402.Doc
gji.fffbf.cn/264404.Doc
gju.fffbf.cn/640448.Doc
gju.fffbf.cn/486644.Doc
gju.fffbf.cn/848840.Doc
gju.fffbf.cn/626048.Doc
gju.fffbf.cn/668480.Doc
gju.fffbf.cn/402044.Doc
gju.fffbf.cn/448822.Doc
gju.fffbf.cn/260486.Doc
gju.fffbf.cn/844048.Doc
gju.fffbf.cn/280004.Doc
gjy.fffbf.cn/208882.Doc
gjy.fffbf.cn/280402.Doc
gjy.fffbf.cn/220840.Doc
gjy.fffbf.cn/404428.Doc
gjy.fffbf.cn/280806.Doc
gjy.fffbf.cn/068406.Doc
gjy.fffbf.cn/802642.Doc
gjy.fffbf.cn/468848.Doc
gjy.fffbf.cn/795759.Doc
gjy.fffbf.cn/606686.Doc
gjt.fffbf.cn/600622.Doc
gjt.fffbf.cn/975380.Doc
gjt.fffbf.cn/688688.Doc
gjt.fffbf.cn/522240.Doc
gjt.fffbf.cn/200448.Doc
gjt.fffbf.cn/684680.Doc
gjt.fffbf.cn/366748.Doc
gjt.fffbf.cn/004024.Doc
gjt.fffbf.cn/440741.Doc
gjt.fffbf.cn/646088.Doc
gjr.fffbf.cn/860082.Doc
gjr.fffbf.cn/006240.Doc
gjr.fffbf.cn/515991.Doc
gjr.fffbf.cn/622646.Doc
gjr.fffbf.cn/860022.Doc
gjr.fffbf.cn/008606.Doc
gjr.fffbf.cn/062668.Doc
gjr.fffbf.cn/642028.Doc
gjr.fffbf.cn/626824.Doc
gjr.fffbf.cn/485485.Doc
gje.fffbf.cn/882260.Doc
gje.fffbf.cn/284844.Doc
gje.fffbf.cn/422824.Doc
gje.fffbf.cn/068606.Doc
gje.fffbf.cn/428882.Doc
gje.fffbf.cn/888608.Doc
gje.fffbf.cn/206024.Doc
gje.fffbf.cn/317177.Doc
gje.fffbf.cn/585933.Doc
gje.fffbf.cn/840602.Doc
gjw.fffbf.cn/954420.Doc
gjw.fffbf.cn/046822.Doc
gjw.fffbf.cn/426842.Doc
gjw.fffbf.cn/028628.Doc
gjw.fffbf.cn/846066.Doc
gjw.fffbf.cn/268666.Doc
gjw.fffbf.cn/757345.Doc
gjw.fffbf.cn/804262.Doc
gjw.fffbf.cn/064648.Doc
gjw.fffbf.cn/288006.Doc
gjq.fffbf.cn/404224.Doc
gjq.fffbf.cn/070396.Doc
gjq.fffbf.cn/024480.Doc
gjq.fffbf.cn/866242.Doc
gjq.fffbf.cn/468264.Doc
gjq.fffbf.cn/820882.Doc
gjq.fffbf.cn/062824.Doc
gjq.fffbf.cn/240482.Doc
gjq.fffbf.cn/866068.Doc
gjq.fffbf.cn/244806.Doc
ghm.fffbf.cn/193147.Doc
ghm.fffbf.cn/240602.Doc
ghm.fffbf.cn/696920.Doc
ghm.fffbf.cn/282824.Doc
ghm.fffbf.cn/240066.Doc
ghm.fffbf.cn/808848.Doc
ghm.fffbf.cn/826606.Doc
ghm.fffbf.cn/446622.Doc
ghm.fffbf.cn/206848.Doc
ghm.fffbf.cn/820222.Doc
ghn.fffbf.cn/486802.Doc
ghn.fffbf.cn/686428.Doc
ghn.fffbf.cn/086088.Doc
ghn.fffbf.cn/389429.Doc
ghn.fffbf.cn/862864.Doc
ghn.fffbf.cn/604646.Doc
ghn.fffbf.cn/886002.Doc
ghn.fffbf.cn/002420.Doc
ghn.fffbf.cn/800284.Doc
ghn.fffbf.cn/086000.Doc
ghb.fffbf.cn/888228.Doc
ghb.fffbf.cn/202042.Doc
ghb.fffbf.cn/482608.Doc
ghb.fffbf.cn/088402.Doc
ghb.fffbf.cn/022220.Doc
ghb.fffbf.cn/608080.Doc
ghb.fffbf.cn/822644.Doc
ghb.fffbf.cn/862442.Doc
ghb.fffbf.cn/444842.Doc
ghb.fffbf.cn/000202.Doc
ghv.fffbf.cn/266464.Doc
ghv.fffbf.cn/624888.Doc
ghv.fffbf.cn/793916.Doc
ghv.fffbf.cn/042446.Doc
ghv.fffbf.cn/464240.Doc
ghv.fffbf.cn/016560.Doc
ghv.fffbf.cn/731551.Doc
ghv.fffbf.cn/488248.Doc
ghv.fffbf.cn/466608.Doc
ghv.fffbf.cn/204806.Doc
ghc.fffbf.cn/064626.Doc
ghc.fffbf.cn/874528.Doc
ghc.fffbf.cn/115537.Doc
ghc.fffbf.cn/268422.Doc
ghc.fffbf.cn/517153.Doc
ghc.fffbf.cn/668402.Doc
ghc.fffbf.cn/008884.Doc
ghc.fffbf.cn/345349.Doc
ghc.fffbf.cn/248080.Doc
