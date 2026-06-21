驮宗夯俨浦


# Python主成分分析降维
# PCA 是最常用的无监督降维技术
# 通过线性变换将高维数据映射到低维，同时保留最大方差

# 1. 导入库
import numpy as np
from sklearn.decomposition import PCA  # 主成分分析
from sklearn.datasets import load_digits  # 手写数字 (8x8=64维)
from sklearn.model_selection import train_test_split
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score
from sklearn.preprocessing import StandardScaler

# 2. 加载高维数据集
digits = load_digits()
X, y = digits.data, digits.target
print(f"原始数据形状: {X.shape}")  # (1797, 64)
print(f"类别数: {len(np.unique(y))}")

# 3. 数据标准化（PCA 对尺度敏感）
scaler = StandardScaler()
X_scaled = scaler.fit_transform(X)

# 4. 计算全部主成分，查看方差解释率
pca = PCA()
pca.fit(X_scaled)
explained_variance = pca.explained_variance_ratio_
cumulative = np.cumsum(explained_variance)
print(f"\n前 5 个主成分的方差解释率: {explained_variance[:5]}")
print(f"前 5 个主成分累计方差解释率: {cumulative[:5]}")

# 5. 选择主成分数量（保留 95% 方差）
n_components_95 = np.argmax(cumulative >= 0.95) + 1
print(f"\n保留 95% 方差所需主成分数: {n_components_95}")
# 原始 64 维 → 约 30 维即可保留 95% 信息

# 6. 降维到 2 维（可视化用，信息损失大）
pca_2d = PCA(n_components=2)
X_pca_2d = pca_2d.fit_transform(X_scaled)
var_2d = sum(pca_2d.explained_variance_ratio_)
print(f"2 维保留的方差比例: {var_2d:.4f}")

# 7. 用 PCA 降维加速分类（保留 95% 方差）
pca_95 = PCA(n_components=n_components_95)
X_pca_95 = pca_95.fit_transform(X_scaled)
X_train, X_test, y_train, y_test = train_test_split(
    X_pca_95, y, test_size=0.3, random_state=42
)
rf_pca = RandomForestClassifier(n_estimators=100, random_state=42)
rf_pca.fit(X_train, y_train)
acc_pca = accuracy_score(y_test, rf_pca.predict(X_test))

# 对比：使用全部 64 维
X_full_train, X_full_test, y_full_train, y_full_test = train_test_split(
    X_scaled, y, test_size=0.3, random_state=42
)
rf_full = RandomForestClassifier(n_estimators=100, random_state=42)
rf_full.fit(X_full_train, y_full_train)
acc_full = accuracy_score(y_full_test, rf_full.predict(X_full_test))
print(f"\nPCA 降维后准确率: {acc_pca:.4f}")
print(f"全部特征准确率:   {acc_full:.4f}")

# 8. 数据重建（从低维恢复高维）
X_reconstructed = pca_95.inverse_transform(X_pca_95)
reconstruction_error = np.mean((X_scaled - X_reconstructed) ** 2)
print(f"\n重建误差 (MSE): {reconstruction_error:.6f}")

# 9. PCA 的载荷向量
loadings = pca.components_
print(f"第一主成分的载荷向量形状: {loadings[0].shape}")

# 10. 总结
# PCA 核心概念:
# - 主成分: 最大化方差的方向
# - 方差解释率: 每个主成分保留的信息量
# - 累计方差: 用于选择 n_components
# - 降维好处: 减少噪声、加速训练、可视化

断甲苍椒补蚀翘把毯等押行卑鞠谏

m.wxl.kxnxh.cn/79999.Doc
m.wxl.kxnxh.cn/04486.Doc
m.wxl.kxnxh.cn/42808.Doc
m.wxl.kxnxh.cn/33775.Doc
m.wxl.kxnxh.cn/60204.Doc
m.wxk.kxnxh.cn/75155.Doc
m.wxk.kxnxh.cn/99351.Doc
m.wxk.kxnxh.cn/33735.Doc
m.wxk.kxnxh.cn/57991.Doc
m.wxk.kxnxh.cn/73171.Doc
m.wxk.kxnxh.cn/48084.Doc
m.wxk.kxnxh.cn/82206.Doc
m.wxk.kxnxh.cn/15311.Doc
m.wxk.kxnxh.cn/22282.Doc
m.wxk.kxnxh.cn/73977.Doc
m.wxk.kxnxh.cn/35179.Doc
m.wxk.kxnxh.cn/13717.Doc
m.wxk.kxnxh.cn/64088.Doc
m.wxk.kxnxh.cn/95739.Doc
m.wxk.kxnxh.cn/46466.Doc
m.wxk.kxnxh.cn/33735.Doc
m.wxk.kxnxh.cn/82864.Doc
m.wxk.kxnxh.cn/73917.Doc
m.wxk.kxnxh.cn/33371.Doc
m.wxk.kxnxh.cn/64068.Doc
m.wxj.kxnxh.cn/48224.Doc
m.wxj.kxnxh.cn/97939.Doc
m.wxj.kxnxh.cn/60806.Doc
m.wxj.kxnxh.cn/71991.Doc
m.wxj.kxnxh.cn/37731.Doc
m.wxj.kxnxh.cn/71351.Doc
m.wxj.kxnxh.cn/28822.Doc
m.wxj.kxnxh.cn/19155.Doc
m.wxj.kxnxh.cn/48662.Doc
m.wxj.kxnxh.cn/79395.Doc
m.wxj.kxnxh.cn/88866.Doc
m.wxj.kxnxh.cn/46420.Doc
m.wxj.kxnxh.cn/28228.Doc
m.wxj.kxnxh.cn/26400.Doc
m.wxj.kxnxh.cn/08002.Doc
m.wxj.kxnxh.cn/51793.Doc
m.wxj.kxnxh.cn/51135.Doc
m.wxj.kxnxh.cn/91531.Doc
m.wxj.kxnxh.cn/37555.Doc
m.wxj.kxnxh.cn/31795.Doc
m.wxh.kxnxh.cn/46406.Doc
m.wxh.kxnxh.cn/39319.Doc
m.wxh.kxnxh.cn/06284.Doc
m.wxh.kxnxh.cn/51597.Doc
m.wxh.kxnxh.cn/19575.Doc
m.wxh.kxnxh.cn/59933.Doc
m.wxh.kxnxh.cn/91371.Doc
m.wxh.kxnxh.cn/39111.Doc
m.wxh.kxnxh.cn/35935.Doc
m.wxh.kxnxh.cn/24082.Doc
m.wxh.kxnxh.cn/31753.Doc
m.wxh.kxnxh.cn/75157.Doc
m.wxh.kxnxh.cn/48626.Doc
m.wxh.kxnxh.cn/22844.Doc
m.wxh.kxnxh.cn/39399.Doc
m.wxh.kxnxh.cn/24000.Doc
m.wxh.kxnxh.cn/86440.Doc
m.wxh.kxnxh.cn/71533.Doc
m.wxh.kxnxh.cn/73513.Doc
m.wxh.kxnxh.cn/95759.Doc
m.wxg.kxnxh.cn/95915.Doc
m.wxg.kxnxh.cn/39991.Doc
m.wxg.kxnxh.cn/97933.Doc
m.wxg.kxnxh.cn/37113.Doc
m.wxg.kxnxh.cn/39159.Doc
m.wxg.kxnxh.cn/51157.Doc
m.wxg.kxnxh.cn/39395.Doc
m.wxg.kxnxh.cn/93155.Doc
m.wxg.kxnxh.cn/42280.Doc
m.wxg.kxnxh.cn/46844.Doc
m.wxg.kxnxh.cn/39179.Doc
m.wxg.kxnxh.cn/68862.Doc
m.wxg.kxnxh.cn/79537.Doc
m.wxg.kxnxh.cn/28888.Doc
m.wxg.kxnxh.cn/75177.Doc
m.wxg.kxnxh.cn/95173.Doc
m.wxg.kxnxh.cn/99111.Doc
m.wxg.kxnxh.cn/31739.Doc
m.wxg.kxnxh.cn/59933.Doc
m.wxg.kxnxh.cn/24288.Doc
m.wxf.kxnxh.cn/64246.Doc
m.wxf.kxnxh.cn/26266.Doc
m.wxf.kxnxh.cn/37953.Doc
m.wxf.kxnxh.cn/35333.Doc
m.wxf.kxnxh.cn/37519.Doc
m.wxf.kxnxh.cn/77553.Doc
m.wxf.kxnxh.cn/97573.Doc
m.wxf.kxnxh.cn/62660.Doc
m.wxf.kxnxh.cn/26460.Doc
m.wxf.kxnxh.cn/24660.Doc
m.wxf.kxnxh.cn/66480.Doc
m.wxf.kxnxh.cn/20802.Doc
m.wxf.kxnxh.cn/95935.Doc
m.wxf.kxnxh.cn/91339.Doc
m.wxf.kxnxh.cn/24080.Doc
m.wxf.kxnxh.cn/33597.Doc
m.wxf.kxnxh.cn/40806.Doc
m.wxf.kxnxh.cn/88622.Doc
m.wxf.kxnxh.cn/68286.Doc
m.wxf.kxnxh.cn/97151.Doc
m.wxd.kxnxh.cn/06284.Doc
m.wxd.kxnxh.cn/33551.Doc
m.wxd.kxnxh.cn/91797.Doc
m.wxd.kxnxh.cn/22042.Doc
m.wxd.kxnxh.cn/15913.Doc
m.wxd.kxnxh.cn/93113.Doc
m.wxd.kxnxh.cn/95139.Doc
m.wxd.kxnxh.cn/46442.Doc
m.wxd.kxnxh.cn/75715.Doc
m.wxd.kxnxh.cn/08268.Doc
m.wxd.kxnxh.cn/42426.Doc
m.wxd.kxnxh.cn/88646.Doc
m.wxd.kxnxh.cn/84064.Doc
m.wxd.kxnxh.cn/08488.Doc
m.wxd.kxnxh.cn/31139.Doc
m.wxd.kxnxh.cn/04642.Doc
m.wxd.kxnxh.cn/73339.Doc
m.wxd.kxnxh.cn/35135.Doc
m.wxd.kxnxh.cn/37173.Doc
m.wxd.kxnxh.cn/31519.Doc
m.wxs.kxnxh.cn/06428.Doc
m.wxs.kxnxh.cn/11139.Doc
m.wxs.kxnxh.cn/93539.Doc
m.wxs.kxnxh.cn/37731.Doc
m.wxs.kxnxh.cn/33911.Doc
m.wxs.kxnxh.cn/55157.Doc
m.wxs.kxnxh.cn/66068.Doc
m.wxs.kxnxh.cn/22404.Doc
m.wxs.kxnxh.cn/86604.Doc
m.wxs.kxnxh.cn/48204.Doc
m.wxs.kxnxh.cn/31751.Doc
m.wxs.kxnxh.cn/64880.Doc
m.wxs.kxnxh.cn/33317.Doc
m.wxs.kxnxh.cn/97733.Doc
m.wxs.kxnxh.cn/66080.Doc
m.wxs.kxnxh.cn/93579.Doc
m.wxs.kxnxh.cn/75975.Doc
m.wxs.kxnxh.cn/22286.Doc
m.wxs.kxnxh.cn/97171.Doc
m.wxs.kxnxh.cn/79379.Doc
m.wxa.kxnxh.cn/80826.Doc
m.wxa.kxnxh.cn/40280.Doc
m.wxa.kxnxh.cn/04682.Doc
m.wxa.kxnxh.cn/39535.Doc
m.wxa.kxnxh.cn/33115.Doc
m.wxa.kxnxh.cn/93397.Doc
m.wxa.kxnxh.cn/80246.Doc
m.wxa.kxnxh.cn/71711.Doc
m.wxa.kxnxh.cn/82442.Doc
m.wxa.kxnxh.cn/42066.Doc
m.wxa.kxnxh.cn/15117.Doc
m.wxa.kxnxh.cn/40022.Doc
m.wxa.kxnxh.cn/11551.Doc
m.wxa.kxnxh.cn/19751.Doc
m.wxa.kxnxh.cn/31713.Doc
m.wxa.kxnxh.cn/51533.Doc
m.wxa.kxnxh.cn/37557.Doc
m.wxa.kxnxh.cn/26424.Doc
m.wxa.kxnxh.cn/13135.Doc
m.wxa.kxnxh.cn/04444.Doc
m.wxp.kxnxh.cn/11739.Doc
m.wxp.kxnxh.cn/44606.Doc
m.wxp.kxnxh.cn/64626.Doc
m.wxp.kxnxh.cn/20804.Doc
m.wxp.kxnxh.cn/31955.Doc
m.wxp.kxnxh.cn/75975.Doc
m.wxp.kxnxh.cn/28208.Doc
m.wxp.kxnxh.cn/66488.Doc
m.wxp.kxnxh.cn/35919.Doc
m.wxp.kxnxh.cn/33577.Doc
m.wxp.kxnxh.cn/62026.Doc
m.wxp.kxnxh.cn/79753.Doc
m.wxp.kxnxh.cn/26000.Doc
m.wxp.kxnxh.cn/53533.Doc
m.wxp.kxnxh.cn/68400.Doc
m.wxp.kxnxh.cn/19371.Doc
m.wxp.kxnxh.cn/77193.Doc
m.wxp.kxnxh.cn/91539.Doc
m.wxp.kxnxh.cn/46442.Doc
m.wxp.kxnxh.cn/55959.Doc
m.wxo.kxnxh.cn/46404.Doc
m.wxo.kxnxh.cn/13735.Doc
m.wxo.kxnxh.cn/55531.Doc
m.wxo.kxnxh.cn/57575.Doc
m.wxo.kxnxh.cn/75991.Doc
m.wxo.kxnxh.cn/13171.Doc
m.wxo.kxnxh.cn/53771.Doc
m.wxo.kxnxh.cn/62062.Doc
m.wxo.kxnxh.cn/80002.Doc
m.wxo.kxnxh.cn/42884.Doc
m.wxo.kxnxh.cn/24082.Doc
m.wxo.kxnxh.cn/35157.Doc
m.wxo.kxnxh.cn/79359.Doc
m.wxo.kxnxh.cn/59911.Doc
m.wxo.kxnxh.cn/42088.Doc
m.wxo.kxnxh.cn/71115.Doc
m.wxo.kxnxh.cn/15335.Doc
m.wxo.kxnxh.cn/62602.Doc
m.wxo.kxnxh.cn/53913.Doc
m.wxo.kxnxh.cn/22240.Doc
m.wxi.kxnxh.cn/19779.Doc
m.wxi.kxnxh.cn/68006.Doc
m.wxi.kxnxh.cn/33131.Doc
m.wxi.kxnxh.cn/75515.Doc
m.wxi.kxnxh.cn/31159.Doc
m.wxi.kxnxh.cn/20466.Doc
m.wxi.kxnxh.cn/51737.Doc
m.wxi.kxnxh.cn/66466.Doc
m.wxi.kxnxh.cn/51973.Doc
m.wxi.kxnxh.cn/24666.Doc
m.wxi.kxnxh.cn/06824.Doc
m.wxi.kxnxh.cn/24466.Doc
m.wxi.kxnxh.cn/20886.Doc
m.wxi.kxnxh.cn/40040.Doc
m.wxi.kxnxh.cn/08286.Doc
m.wxi.kxnxh.cn/93979.Doc
m.wxi.kxnxh.cn/79555.Doc
m.wxi.kxnxh.cn/71199.Doc
m.wxi.kxnxh.cn/19977.Doc
m.wxi.kxnxh.cn/79517.Doc
m.wxu.kxnxh.cn/95599.Doc
m.wxu.kxnxh.cn/82608.Doc
m.wxu.kxnxh.cn/35573.Doc
m.wxu.kxnxh.cn/19593.Doc
m.wxu.kxnxh.cn/13773.Doc
m.wxu.kxnxh.cn/53993.Doc
m.wxu.kxnxh.cn/46206.Doc
m.wxu.kxnxh.cn/53913.Doc
m.wxu.kxnxh.cn/00628.Doc
m.wxu.kxnxh.cn/99717.Doc
m.wxu.kxnxh.cn/51737.Doc
m.wxu.kxnxh.cn/97739.Doc
m.wxu.kxnxh.cn/71717.Doc
m.wxu.kxnxh.cn/02226.Doc
m.wxu.kxnxh.cn/13753.Doc
m.wxu.kxnxh.cn/26820.Doc
m.wxu.kxnxh.cn/51339.Doc
m.wxu.kxnxh.cn/68822.Doc
m.wxu.kxnxh.cn/48080.Doc
m.wxu.kxnxh.cn/80202.Doc
m.wxy.kxnxh.cn/79791.Doc
m.wxy.kxnxh.cn/26624.Doc
m.wxy.kxnxh.cn/35799.Doc
m.wxy.kxnxh.cn/86044.Doc
m.wxy.kxnxh.cn/46826.Doc
m.wxy.kxnxh.cn/08400.Doc
m.wxy.kxnxh.cn/42646.Doc
m.wxy.kxnxh.cn/15191.Doc
m.wxy.kxnxh.cn/82848.Doc
m.wxy.kxnxh.cn/24662.Doc
m.wxy.kxnxh.cn/17915.Doc
m.wxy.kxnxh.cn/93977.Doc
m.wxy.kxnxh.cn/51957.Doc
m.wxy.kxnxh.cn/66684.Doc
m.wxy.kxnxh.cn/99779.Doc
m.wxy.kxnxh.cn/13359.Doc
m.wxy.kxnxh.cn/64862.Doc
m.wxy.kxnxh.cn/24002.Doc
m.wxy.kxnxh.cn/17953.Doc
m.wxy.kxnxh.cn/33593.Doc
m.wxt.kxnxh.cn/19391.Doc
m.wxt.kxnxh.cn/13395.Doc
m.wxt.kxnxh.cn/51133.Doc
m.wxt.kxnxh.cn/22846.Doc
m.wxt.kxnxh.cn/20282.Doc
m.wxt.kxnxh.cn/17117.Doc
m.wxt.kxnxh.cn/79993.Doc
m.wxt.kxnxh.cn/37535.Doc
m.wxt.kxnxh.cn/15999.Doc
m.wxt.kxnxh.cn/91151.Doc
m.wxt.kxnxh.cn/51591.Doc
m.wxt.kxnxh.cn/39751.Doc
m.wxt.kxnxh.cn/77735.Doc
m.wxt.kxnxh.cn/55977.Doc
m.wxt.kxnxh.cn/99953.Doc
m.wxt.kxnxh.cn/82066.Doc
m.wxt.kxnxh.cn/40404.Doc
m.wxt.kxnxh.cn/73593.Doc
m.wxt.kxnxh.cn/22446.Doc
m.wxt.kxnxh.cn/35915.Doc
m.wxr.kxnxh.cn/46044.Doc
m.wxr.kxnxh.cn/99155.Doc
m.wxr.kxnxh.cn/55359.Doc
m.wxr.kxnxh.cn/20480.Doc
m.wxr.kxnxh.cn/53771.Doc
m.wxr.kxnxh.cn/31133.Doc
m.wxr.kxnxh.cn/71193.Doc
m.wxr.kxnxh.cn/62044.Doc
m.wxr.kxnxh.cn/97711.Doc
m.wxr.kxnxh.cn/44266.Doc
m.wxr.kxnxh.cn/31199.Doc
m.wxr.kxnxh.cn/20464.Doc
m.wxr.kxnxh.cn/60822.Doc
m.wxr.kxnxh.cn/64062.Doc
m.wxr.kxnxh.cn/51519.Doc
m.wxr.kxnxh.cn/77759.Doc
m.wxr.kxnxh.cn/75399.Doc
m.wxr.kxnxh.cn/68648.Doc
m.wxr.kxnxh.cn/77591.Doc
m.wxr.kxnxh.cn/80484.Doc
m.wxe.kxnxh.cn/13933.Doc
m.wxe.kxnxh.cn/37939.Doc
m.wxe.kxnxh.cn/84046.Doc
m.wxe.kxnxh.cn/17591.Doc
m.wxe.kxnxh.cn/17395.Doc
m.wxe.kxnxh.cn/04006.Doc
m.wxe.kxnxh.cn/59791.Doc
m.wxe.kxnxh.cn/22664.Doc
m.wxe.kxnxh.cn/31393.Doc
m.wxe.kxnxh.cn/77955.Doc
m.wxe.kxnxh.cn/91175.Doc
m.wxe.kxnxh.cn/75739.Doc
m.wxe.kxnxh.cn/95799.Doc
m.wxe.kxnxh.cn/06404.Doc
m.wxe.kxnxh.cn/62420.Doc
m.wxe.kxnxh.cn/26006.Doc
m.wxe.kxnxh.cn/79959.Doc
m.wxe.kxnxh.cn/64624.Doc
m.wxe.kxnxh.cn/66444.Doc
m.wxe.kxnxh.cn/19717.Doc
m.wxw.kxnxh.cn/68028.Doc
m.wxw.kxnxh.cn/55175.Doc
m.wxw.kxnxh.cn/40244.Doc
m.wxw.kxnxh.cn/55753.Doc
m.wxw.kxnxh.cn/91951.Doc
m.wxw.kxnxh.cn/73939.Doc
m.wxw.kxnxh.cn/59353.Doc
m.wxw.kxnxh.cn/31917.Doc
m.wxw.kxnxh.cn/26468.Doc
m.wxw.kxnxh.cn/66628.Doc
m.wxw.kxnxh.cn/02066.Doc
m.wxw.kxnxh.cn/44060.Doc
m.wxw.kxnxh.cn/59197.Doc
m.wxw.kxnxh.cn/39759.Doc
m.wxw.kxnxh.cn/93199.Doc
m.wxw.kxnxh.cn/19793.Doc
m.wxw.kxnxh.cn/51797.Doc
m.wxw.kxnxh.cn/93755.Doc
m.wxw.kxnxh.cn/15199.Doc
m.wxw.kxnxh.cn/55397.Doc
m.wxq.kxnxh.cn/20286.Doc
m.wxq.kxnxh.cn/20886.Doc
m.wxq.kxnxh.cn/39959.Doc
m.wxq.kxnxh.cn/15359.Doc
m.wxq.kxnxh.cn/66282.Doc
m.wxq.kxnxh.cn/40042.Doc
m.wxq.kxnxh.cn/02004.Doc
m.wxq.kxnxh.cn/68000.Doc
m.wxq.kxnxh.cn/57317.Doc
m.wxq.kxnxh.cn/84684.Doc
m.wxq.kxnxh.cn/37771.Doc
m.wxq.kxnxh.cn/02860.Doc
m.wxq.kxnxh.cn/20286.Doc
m.wxq.kxnxh.cn/91339.Doc
m.wxq.kxnxh.cn/95731.Doc
m.wxq.kxnxh.cn/26008.Doc
m.wxq.kxnxh.cn/75935.Doc
m.wxq.kxnxh.cn/86420.Doc
m.wxq.kxnxh.cn/37179.Doc
m.wxq.kxnxh.cn/42464.Doc
m.wzm.kxnxh.cn/55357.Doc
m.wzm.kxnxh.cn/99137.Doc
m.wzm.kxnxh.cn/51131.Doc
m.wzm.kxnxh.cn/28008.Doc
m.wzm.kxnxh.cn/13771.Doc
m.wzm.kxnxh.cn/93793.Doc
m.wzm.kxnxh.cn/39979.Doc
m.wzm.kxnxh.cn/66824.Doc
m.wzm.kxnxh.cn/04882.Doc
m.wzm.kxnxh.cn/60044.Doc
m.wzm.kxnxh.cn/59333.Doc
m.wzm.kxnxh.cn/17795.Doc
m.wzm.kxnxh.cn/60666.Doc
m.wzm.kxnxh.cn/53739.Doc
m.wzm.kxnxh.cn/60448.Doc
m.wzm.kxnxh.cn/75191.Doc
m.wzm.kxnxh.cn/86008.Doc
m.wzm.kxnxh.cn/46088.Doc
m.wzm.kxnxh.cn/40068.Doc
m.wzm.kxnxh.cn/60824.Doc
m.wzn.kxnxh.cn/95395.Doc
m.wzn.kxnxh.cn/13759.Doc
m.wzn.kxnxh.cn/93393.Doc
m.wzn.kxnxh.cn/66886.Doc
m.wzn.kxnxh.cn/26264.Doc
m.wzn.kxnxh.cn/97317.Doc
m.wzn.kxnxh.cn/79997.Doc
m.wzn.kxnxh.cn/53959.Doc
m.wzn.kxnxh.cn/40040.Doc
m.wzn.kxnxh.cn/35595.Doc
