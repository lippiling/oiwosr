巳毁驴匀厮


# Python超参数调优
# 超参数是训练前设置的参数，调优能显著提升模型性能
# 网格搜索和随机搜索是最常用的自动化调参方法

# 1. 导入库
import numpy as np
from sklearn.datasets import load_digits
from sklearn.model_selection import (
    train_test_split, GridSearchCV, RandomizedSearchCV, StratifiedKFold
)
from sklearn.svm import SVC
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score
from scipy.stats import randint

# 2. 加载数据
digits = load_digits()
X, y = digits.data, digits.target
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.3, random_state=42
)
print(f"训练集: {X_train.shape}, 测试集: {X_test.shape}")

# 3. 网格搜索 GridSearchCV — 穷举参数组合
param_grid_svm = {
    'C': [0.1, 1, 10, 100],
    'gamma': [0.001, 0.01, 0.1, 1],
    'kernel': ['rbf']
}
cv_strategy = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)

print(f"\n=== 网格搜索 SVM ===")
grid_svm = GridSearchCV(
    SVC(random_state=42), param_grid_svm,
    cv=cv_strategy, scoring='accuracy', n_jobs=-1
)
grid_svm.fit(X_train, y_train)
print(f"最佳参数: {grid_svm.best_params_}")
print(f"最佳 CV 分数: {grid_svm.best_score_:.4f}")

# 查看最佳组合
results = grid_svm.cv_results_
sorted_indices = np.argsort(results['mean_test_score'])[::-1][:3]
print(f"\n前三名参数组合:")
for i, idx in enumerate(sorted_indices):
    print(f"  {i+1}. {results['params'][idx]}, 得分={results['mean_test_score'][idx]:.4f}")

# 4. 随机搜索 RandomizedSearchCV
param_dist_rf = {
    'n_estimators': randint(50, 300),
    'max_depth': randint(3, 15),
    'min_samples_split': randint(2, 20),
    'min_samples_leaf': randint(1, 10),
    'max_features': ['sqrt', 'log2', None]
}

print(f"\n=== 随机搜索 随机森林 ===")
random_rf = RandomizedSearchCV(
    RandomForestClassifier(random_state=42),
    param_distributions=param_dist_rf,
    n_iter=30, cv=cv_strategy, scoring='accuracy',
    n_jobs=-1, random_state=42
)
random_rf.fit(X_train, y_train)
print(f"最佳参数: {random_rf.best_params_}")
print(f"最佳 CV 分数: {random_rf.best_score_:.4f}")

# 5. 使用 F1 评分
grid_f1 = GridSearchCV(
    SVC(random_state=42),
    param_grid={'C': [0.1, 1, 10], 'gamma': [0.01, 0.1]},
    cv=5, scoring='f1_macro', n_jobs=-1
)
grid_f1.fit(X_train, y_train)
print(f"\nF1 评分最佳参数: {grid_f1.best_params_}")

# 6. 搜索细节
print(f"\n=== 搜索信息 ===")
print(f"总参数组合数: {len(results['params'])}")
print(f"拟合总次数: {grid_svm.n_splits_ * len(results['params'])}")

# 7. 测试集最终评估
best_model = grid_svm.best_estimator_
print(f"\n测试准确率: {accuracy_score(y_test, best_model.predict(X_test)):.4f}")

# 8. 调参注意事项
# 先粗调再精调: 大范围找好区域再精细搜索
# 先随机搜索找到好区域，再网格搜索精调
print(f"\n=== 调参建议 ===")
print(f"网格搜索: 参数少 (<10 组合) | 随机搜索: 参数多或连续值")

巳荣床杖状腔毁雍揭案殴稚百恍芭

wuy.ag5qr6tv.cn/319915.htm
wuy.ag5qr6tv.cn/284025.htm
wuy.ag5qr6tv.cn/248085.htm
wut.g6lz11tv.cn/880485.htm
wut.g6lz11tv.cn/731735.htm
wut.g6lz11tv.cn/604865.htm
wut.g6lz11tv.cn/973515.htm
wut.g6lz11tv.cn/824885.htm
wut.g6lz11tv.cn/248225.htm
wut.g6lz11tv.cn/866805.htm
wut.g6lz11tv.cn/482845.htm
wut.g6lz11tv.cn/828825.htm
wut.g6lz11tv.cn/399115.htm
wur.g6lz11tv.cn/915175.htm
wur.g6lz11tv.cn/711775.htm
wur.g6lz11tv.cn/448285.htm
wur.g6lz11tv.cn/840425.htm
wur.g6lz11tv.cn/602645.htm
wur.g6lz11tv.cn/644605.htm
wur.g6lz11tv.cn/666005.htm
wur.g6lz11tv.cn/913395.htm
wur.g6lz11tv.cn/400845.htm
wur.g6lz11tv.cn/226245.htm
wue.g6lz11tv.cn/022805.htm
wue.g6lz11tv.cn/284085.htm
wue.g6lz11tv.cn/319795.htm
wue.g6lz11tv.cn/404285.htm
wue.g6lz11tv.cn/680685.htm
wue.g6lz11tv.cn/539955.htm
wue.g6lz11tv.cn/448065.htm
wue.g6lz11tv.cn/806405.htm
wue.g6lz11tv.cn/791975.htm
wue.g6lz11tv.cn/846805.htm
wuw.g6lz11tv.cn/759515.htm
wuw.g6lz11tv.cn/088205.htm
wuw.g6lz11tv.cn/206025.htm
wuw.g6lz11tv.cn/628225.htm
wuw.g6lz11tv.cn/337795.htm
wuw.g6lz11tv.cn/777995.htm
wuw.g6lz11tv.cn/486085.htm
wuw.g6lz11tv.cn/604425.htm
wuw.g6lz11tv.cn/480245.htm
wuw.g6lz11tv.cn/711715.htm
wuq.g6lz11tv.cn/200665.htm
wuq.g6lz11tv.cn/624825.htm
wuq.g6lz11tv.cn/151595.htm
wuq.g6lz11tv.cn/131575.htm
wuq.g6lz11tv.cn/248445.htm
wuq.g6lz11tv.cn/713975.htm
wuq.g6lz11tv.cn/200205.htm
wuq.g6lz11tv.cn/426665.htm
wuq.g6lz11tv.cn/975175.htm
wuq.g6lz11tv.cn/026665.htm
wytv.g6lz11tv.cn/151995.htm
wytv.g6lz11tv.cn/664005.htm
wytv.g6lz11tv.cn/682885.htm
wytv.g6lz11tv.cn/822405.htm
wytv.g6lz11tv.cn/062805.htm
wytv.g6lz11tv.cn/313715.htm
wytv.g6lz11tv.cn/260045.htm
wytv.g6lz11tv.cn/955395.htm
wytv.g6lz11tv.cn/220085.htm
wytv.g6lz11tv.cn/006065.htm
wyn.g6lz11tv.cn/355575.htm
wyn.g6lz11tv.cn/202845.htm
wyn.g6lz11tv.cn/391935.htm
wyn.g6lz11tv.cn/591515.htm
wyn.g6lz11tv.cn/882085.htm
wyn.g6lz11tv.cn/866225.htm
wyn.g6lz11tv.cn/462045.htm
wyn.g6lz11tv.cn/248645.htm
wyn.g6lz11tv.cn/482885.htm
wyn.g6lz11tv.cn/068465.htm
wyb.g6lz11tv.cn/779795.htm
wyb.g6lz11tv.cn/284465.htm
wyb.g6lz11tv.cn/971975.htm
wyb.g6lz11tv.cn/533335.htm
wyb.g6lz11tv.cn/464825.htm
wyb.g6lz11tv.cn/191115.htm
wyb.g6lz11tv.cn/282245.htm
wyb.g6lz11tv.cn/060445.htm
wyb.g6lz11tv.cn/086485.htm
wyb.g6lz11tv.cn/442285.htm
wyv.g6lz11tv.cn/280065.htm
wyv.g6lz11tv.cn/331115.htm
wyv.g6lz11tv.cn/046205.htm
wyv.g6lz11tv.cn/442045.htm
wyv.g6lz11tv.cn/571915.htm
wyv.g6lz11tv.cn/480885.htm
wyv.g6lz11tv.cn/733575.htm
wyv.g6lz11tv.cn/531715.htm
wyv.g6lz11tv.cn/319535.htm
wyv.g6lz11tv.cn/597955.htm
wyc.g6lz11tv.cn/844805.htm
wyc.g6lz11tv.cn/777915.htm
wyc.g6lz11tv.cn/204645.htm
wyc.g6lz11tv.cn/020425.htm
wyc.g6lz11tv.cn/000465.htm
wyc.g6lz11tv.cn/068685.htm
wyc.g6lz11tv.cn/628205.htm
wyc.g6lz11tv.cn/197375.htm
wyc.g6lz11tv.cn/628645.htm
wyc.g6lz11tv.cn/484425.htm
wyx.g6lz11tv.cn/933735.htm
wyx.g6lz11tv.cn/266405.htm
wyx.g6lz11tv.cn/204665.htm
wyx.g6lz11tv.cn/155175.htm
wyx.g6lz11tv.cn/393135.htm
wyx.g6lz11tv.cn/446645.htm
wyx.g6lz11tv.cn/046025.htm
wyx.g6lz11tv.cn/939715.htm
wyx.g6lz11tv.cn/517715.htm
wyx.g6lz11tv.cn/062405.htm
wyz.g6lz11tv.cn/531395.htm
wyz.g6lz11tv.cn/555755.htm
wyz.g6lz11tv.cn/791755.htm
wyz.g6lz11tv.cn/753355.htm
wyz.g6lz11tv.cn/080685.htm
wyz.g6lz11tv.cn/666805.htm
wyz.g6lz11tv.cn/462405.htm
wyz.g6lz11tv.cn/060005.htm
wyz.g6lz11tv.cn/800845.htm
wyz.g6lz11tv.cn/242025.htm
wyl.g6lz11tv.cn/844825.htm
wyl.g6lz11tv.cn/882605.htm
wyl.g6lz11tv.cn/375775.htm
wyl.g6lz11tv.cn/204265.htm
wyl.g6lz11tv.cn/640865.htm
wyl.g6lz11tv.cn/139775.htm
wyl.g6lz11tv.cn/068485.htm
wyl.g6lz11tv.cn/262005.htm
wyl.g6lz11tv.cn/422445.htm
wyl.g6lz11tv.cn/539315.htm
wyk.g6lz11tv.cn/004865.htm
wyk.g6lz11tv.cn/915395.htm
wyk.g6lz11tv.cn/195755.htm
wyk.g6lz11tv.cn/448625.htm
wyk.g6lz11tv.cn/006445.htm
wyk.g6lz11tv.cn/040005.htm
wyk.g6lz11tv.cn/488825.htm
wyk.g6lz11tv.cn/428085.htm
wyk.g6lz11tv.cn/682005.htm
wyk.g6lz11tv.cn/597155.htm
wyj.g6lz11tv.cn/440285.htm
wyj.g6lz11tv.cn/993935.htm
wyj.g6lz11tv.cn/440865.htm
wyj.g6lz11tv.cn/466665.htm
wyj.g6lz11tv.cn/828825.htm
wyj.g6lz11tv.cn/882625.htm
wyj.g6lz11tv.cn/993375.htm
wyj.g6lz11tv.cn/084845.htm
wyj.g6lz11tv.cn/484605.htm
wyj.g6lz11tv.cn/806005.htm
wyh.g6lz11tv.cn/139795.htm
wyh.g6lz11tv.cn/682225.htm
wyh.g6lz11tv.cn/400025.htm
wyh.g6lz11tv.cn/139155.htm
wyh.g6lz11tv.cn/628205.htm
wyh.g6lz11tv.cn/155975.htm
wyh.g6lz11tv.cn/153515.htm
wyh.g6lz11tv.cn/886665.htm
wyh.g6lz11tv.cn/868865.htm
wyh.g6lz11tv.cn/604425.htm
wyg.g6lz11tv.cn/266265.htm
wyg.g6lz11tv.cn/826045.htm
wyg.g6lz11tv.cn/484245.htm
wyg.g6lz11tv.cn/771915.htm
wyg.g6lz11tv.cn/088485.htm
wyg.g6lz11tv.cn/424885.htm
wyg.g6lz11tv.cn/026425.htm
wyg.g6lz11tv.cn/335935.htm
wyg.g6lz11tv.cn/480845.htm
wyg.g6lz11tv.cn/991135.htm
wyf.g6lz11tv.cn/400405.htm
wyf.g6lz11tv.cn/260685.htm
wyf.g6lz11tv.cn/935735.htm
wyf.g6lz11tv.cn/880045.htm
wyf.g6lz11tv.cn/555115.htm
wyf.g6lz11tv.cn/666425.htm
wyf.g6lz11tv.cn/424885.htm
wyf.g6lz11tv.cn/208285.htm
wyf.g6lz11tv.cn/240865.htm
wyf.g6lz11tv.cn/608085.htm
wyd.g6lz11tv.cn/935955.htm
wyd.g6lz11tv.cn/119315.htm
wyd.g6lz11tv.cn/648025.htm
wyd.g6lz11tv.cn/402285.htm
wyd.g6lz11tv.cn/335515.htm
wyd.g6lz11tv.cn/228265.htm
wyd.g6lz11tv.cn/486825.htm
wyd.g6lz11tv.cn/931795.htm
wyd.g6lz11tv.cn/717335.htm
wyd.g6lz11tv.cn/280885.htm
wys.g6lz11tv.cn/646045.htm
wys.g6lz11tv.cn/193155.htm
wys.g6lz11tv.cn/020605.htm
wys.g6lz11tv.cn/555395.htm
wys.g6lz11tv.cn/040885.htm
wys.g6lz11tv.cn/391735.htm
wys.g6lz11tv.cn/664245.htm
wys.g6lz11tv.cn/604225.htm
wys.g6lz11tv.cn/391595.htm
wys.g6lz11tv.cn/866845.htm
wya.g6lz11tv.cn/244445.htm
wya.g6lz11tv.cn/028685.htm
wya.g6lz11tv.cn/735575.htm
wya.g6lz11tv.cn/791555.htm
wya.g6lz11tv.cn/280005.htm
wya.g6lz11tv.cn/795955.htm
wya.g6lz11tv.cn/262845.htm
wya.g6lz11tv.cn/848005.htm
wya.g6lz11tv.cn/317115.htm
wya.g6lz11tv.cn/119535.htm
wyp.g6lz11tv.cn/826085.htm
wyp.g6lz11tv.cn/646285.htm
wyp.g6lz11tv.cn/197975.htm
wyp.g6lz11tv.cn/068885.htm
wyp.g6lz11tv.cn/820665.htm
wyp.g6lz11tv.cn/266805.htm
wyp.g6lz11tv.cn/773975.htm
wyp.g6lz11tv.cn/888045.htm
wyp.g6lz11tv.cn/682825.htm
wyp.g6lz11tv.cn/999315.htm
wyo.g6lz11tv.cn/020085.htm
wyo.g6lz11tv.cn/280285.htm
wyo.g6lz11tv.cn/200285.htm
wyo.g6lz11tv.cn/440085.htm
wyo.g6lz11tv.cn/935575.htm
wyo.g6lz11tv.cn/577515.htm
wyo.g6lz11tv.cn/711935.htm
wyo.g6lz11tv.cn/511135.htm
wyo.g6lz11tv.cn/804685.htm
wyo.g6lz11tv.cn/224485.htm
wyi.g6lz11tv.cn/484425.htm
wyi.g6lz11tv.cn/448465.htm
wyi.g6lz11tv.cn/844625.htm
wyi.g6lz11tv.cn/713315.htm
wyi.g6lz11tv.cn/008025.htm
wyi.g6lz11tv.cn/224685.htm
wyi.g6lz11tv.cn/208865.htm
wyi.g6lz11tv.cn/226425.htm
wyi.g6lz11tv.cn/604285.htm
wyi.g6lz11tv.cn/595995.htm
wyu.g6lz11tv.cn/006245.htm
wyu.g6lz11tv.cn/668645.htm
wyu.g6lz11tv.cn/082605.htm
wyu.g6lz11tv.cn/020245.htm
wyu.g6lz11tv.cn/602865.htm
wyu.g6lz11tv.cn/797535.htm
wyu.g6lz11tv.cn/115155.htm
wyu.g6lz11tv.cn/997395.htm
wyu.g6lz11tv.cn/646805.htm
wyu.g6lz11tv.cn/620045.htm
wyy.g6lz11tv.cn/420485.htm
wyy.g6lz11tv.cn/135515.htm
wyy.g6lz11tv.cn/646805.htm
wyy.g6lz11tv.cn/002205.htm
wyy.g6lz11tv.cn/973175.htm
wyy.g6lz11tv.cn/539795.htm
wyy.g6lz11tv.cn/442685.htm
wyy.g6lz11tv.cn/919555.htm
wyy.g6lz11tv.cn/420045.htm
wyy.g6lz11tv.cn/539795.htm
wyt.g6lz11tv.cn/159115.htm
wyt.g6lz11tv.cn/517775.htm
wyt.g6lz11tv.cn/024285.htm
wyt.g6lz11tv.cn/844825.htm
wyt.g6lz11tv.cn/664485.htm
wyt.g6lz11tv.cn/624625.htm
wyt.g6lz11tv.cn/264025.htm
wyt.g6lz11tv.cn/880265.htm
wyt.g6lz11tv.cn/060225.htm
wyt.g6lz11tv.cn/377755.htm
wyr.g6lz11tv.cn/824245.htm
wyr.g6lz11tv.cn/711795.htm
wyr.g6lz11tv.cn/842265.htm
wyr.g6lz11tv.cn/206625.htm
wyr.g6lz11tv.cn/884445.htm
wyr.g6lz11tv.cn/117975.htm
wyr.g6lz11tv.cn/155915.htm
wyr.g6lz11tv.cn/799595.htm
wyr.g6lz11tv.cn/119355.htm
wyr.g6lz11tv.cn/606865.htm
wye.g6lz11tv.cn/799115.htm
wye.g6lz11tv.cn/824065.htm
wye.g6lz11tv.cn/579335.htm
wye.g6lz11tv.cn/248425.htm
wye.g6lz11tv.cn/533175.htm
wye.g6lz11tv.cn/759715.htm
wye.g6lz11tv.cn/139515.htm
wye.g6lz11tv.cn/682265.htm
wye.g6lz11tv.cn/644225.htm
wye.g6lz11tv.cn/175995.htm
wyw.g6lz11tv.cn/882425.htm
wyw.g6lz11tv.cn/719755.htm
wyw.g6lz11tv.cn/628225.htm
wyw.g6lz11tv.cn/064025.htm
wyw.g6lz11tv.cn/333355.htm
wyw.g6lz11tv.cn/080805.htm
wyw.g6lz11tv.cn/464645.htm
wyw.g6lz11tv.cn/880645.htm
wyw.g6lz11tv.cn/668845.htm
wyw.g6lz11tv.cn/004885.htm
wyq.g6lz11tv.cn/042065.htm
wyq.g6lz11tv.cn/931715.htm
wyq.g6lz11tv.cn/004885.htm
wyq.g6lz11tv.cn/537115.htm
wyq.g6lz11tv.cn/046025.htm
wyq.g6lz11tv.cn/866885.htm
wyq.g6lz11tv.cn/711715.htm
wyq.g6lz11tv.cn/155395.htm
wyq.g6lz11tv.cn/224445.htm
wyq.g6lz11tv.cn/064665.htm
wttv.g6lz11tv.cn/173595.htm
wttv.g6lz11tv.cn/482445.htm
wttv.g6lz11tv.cn/559515.htm
wttv.g6lz11tv.cn/997175.htm
wttv.g6lz11tv.cn/068245.htm
wttv.g6lz11tv.cn/599795.htm
wttv.g6lz11tv.cn/604865.htm
wttv.g6lz11tv.cn/206805.htm
wttv.g6lz11tv.cn/408805.htm
wttv.g6lz11tv.cn/028465.htm
wtn.g6lz11tv.cn/131555.htm
wtn.g6lz11tv.cn/440085.htm
wtn.g6lz11tv.cn/000825.htm
wtn.g6lz11tv.cn/246825.htm
wtn.g6lz11tv.cn/355935.htm
wtn.g6lz11tv.cn/997335.htm
wtn.g6lz11tv.cn/248485.htm
wtn.g6lz11tv.cn/175135.htm
wtn.g6lz11tv.cn/086605.htm
wtn.g6lz11tv.cn/220085.htm
wtb.g6lz11tv.cn/642645.htm
wtb.g6lz11tv.cn/799375.htm
wtb.g6lz11tv.cn/197355.htm
wtb.g6lz11tv.cn/597155.htm
wtb.g6lz11tv.cn/208045.htm
wtb.g6lz11tv.cn/200085.htm
wtb.g6lz11tv.cn/917535.htm
wtb.g6lz11tv.cn/539355.htm
wtb.g6lz11tv.cn/755975.htm
wtb.g6lz11tv.cn/884045.htm
wtv.g6lz11tv.cn/028845.htm
wtv.g6lz11tv.cn/688245.htm
wtv.g6lz11tv.cn/262245.htm
wtv.g6lz11tv.cn/206025.htm
wtv.g6lz11tv.cn/395995.htm
wtv.g6lz11tv.cn/480225.htm
wtv.g6lz11tv.cn/048285.htm
wtv.g6lz11tv.cn/377515.htm
wtv.g6lz11tv.cn/373395.htm
wtv.g6lz11tv.cn/404885.htm
wtc.g6lz11tv.cn/068485.htm
wtc.g6lz11tv.cn/084865.htm
wtc.g6lz11tv.cn/737715.htm
wtc.g6lz11tv.cn/202865.htm
wtc.g6lz11tv.cn/002245.htm
wtc.g6lz11tv.cn/311975.htm
wtc.g6lz11tv.cn/000865.htm
wtc.g6lz11tv.cn/680405.htm
wtc.g6lz11tv.cn/086605.htm
wtc.g6lz11tv.cn/024205.htm
wtx.g6lz11tv.cn/224845.htm
wtx.g6lz11tv.cn/686885.htm
wtx.g6lz11tv.cn/080445.htm
wtx.g6lz11tv.cn/606245.htm
wtx.g6lz11tv.cn/204205.htm
wtx.g6lz11tv.cn/973195.htm
wtx.g6lz11tv.cn/373935.htm
wtx.g6lz11tv.cn/482085.htm
wtx.g6lz11tv.cn/862625.htm
wtx.g6lz11tv.cn/593335.htm
wtz.g6lz11tv.cn/442825.htm
wtz.g6lz11tv.cn/642085.htm
wtz.g6lz11tv.cn/177195.htm
wtz.g6lz11tv.cn/226425.htm
wtz.g6lz11tv.cn/404045.htm
wtz.g6lz11tv.cn/311735.htm
wtz.g6lz11tv.cn/577995.htm
wtz.g6lz11tv.cn/024885.htm
wtz.g6lz11tv.cn/339395.htm
wtz.g6lz11tv.cn/220805.htm
wtl.g6lz11tv.cn/420065.htm
wtl.g6lz11tv.cn/268025.htm
wtl.g6lz11tv.cn/133595.htm
wtl.g6lz11tv.cn/406025.htm
wtl.g6lz11tv.cn/684405.htm
wtl.g6lz11tv.cn/171575.htm
wtl.g6lz11tv.cn/660685.htm
wtl.g6lz11tv.cn/331755.htm
wtl.g6lz11tv.cn/028285.htm
wtl.g6lz11tv.cn/428825.htm
wtk.g6lz11tv.cn/444485.htm
wtk.g6lz11tv.cn/886005.htm
