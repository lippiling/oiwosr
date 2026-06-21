压戏捶压淖


# Python机器学习管道
# Pipeline 将数据预处理和模型训练串联成一个整体流程
# 避免数据泄露、简化代码、方便与 GridSearchCV 结合

# 1. 导入库
import numpy as np
import pandas as pd
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split, GridSearchCV
from sklearn.pipeline import Pipeline, make_pipeline
from sklearn.compose import ColumnTransformer
from sklearn.preprocessing import StandardScaler, OneHotEncoder
from sklearn.impute import SimpleImputer
from sklearn.decomposition import PCA
from sklearn.svm import SVC
from sklearn.metrics import accuracy_score

# 2. 创建含缺失值的混合数据
np.random.seed(42)
n = 150
data = {
    'sepal_length': np.random.normal(5.8, 0.8, n),
    'sepal_width': np.random.normal(3.0, 0.4, n),
    'petal_length': np.random.normal(3.8, 1.6, n),
    'petal_width': np.random.normal(1.2, 0.7, n),
    'color': np.random.choice(['red', 'blue', 'green'], n),
    'size': np.random.choice(['S', 'M', 'L'], n)
}
for col in ['sepal_length', 'petal_width']:
    missing_idx = np.random.choice(n, 10, replace=False)
    data[col][missing_idx] = np.nan

df = pd.DataFrame(data)
y = np.random.randint(0, 3, n)
print(f"数据集: {df.shape}, 缺失值:\n{df.isnull().sum()}\n")

# 3. ColumnTransformer — 不同列不同处理
num_transformer = Pipeline([
    ('imputer', SimpleImputer(strategy='mean')),
    ('scaler', StandardScaler())
])
cat_transformer = Pipeline([
    ('imputer', SimpleImputer(strategy='most_frequent')),
    ('onehot', OneHotEncoder(handle_unknown='ignore'))
])

preprocessor = ColumnTransformer([
    ('num', num_transformer, ['sepal_length', 'sepal_width', 'petal_length', 'petal_width']),
    ('cat', cat_transformer, ['color', 'size'])
])

# 4. 构建完整 Pipeline
full_pipeline = Pipeline([
    ('preprocessor', preprocessor),
    ('pca', PCA(n_components=5)),
    ('classifier', SVC(random_state=42))
])

# 5. 训练和评估
X_train, X_test, y_train, y_test = train_test_split(
    df, y, test_size=0.3, random_state=42
)
full_pipeline.fit(X_train, y_train)
print(f"=== Pipeline 评估 ===")
print(f"准确率: {accuracy_score(y_test, full_pipeline.predict(X_test)):.4f}")

# 6. make_pipeline 简化版
simple_pipe = make_pipeline(StandardScaler(), PCA(n_components=3), SVC())
X_iris, y_iris = load_iris(return_X_y=True)
simple_pipe.fit(X_iris, y_iris)
print(f"\nmake_pipeline 准确率: {simple_pipe.score(X_iris, y_iris):.4f}")

# 7. Pipeline + GridSearchCV
param_grid = {
    'pca__n_components': [3, 5, 8],
    'classifier__C': [0.1, 1, 10],
}
print(f"\n=== 管道 + 网格搜索 ===")
grid_search = GridSearchCV(
    full_pipeline, param_grid, cv=5, scoring='accuracy', n_jobs=-1
)
grid_search.fit(X_train, y_train)
print(f"最佳参数: {grid_search.best_params_}")
print(f"测试集分数: {grid_search.score(X_test, y_test):.4f}")

# 8. 管道步骤访问
print(f"\n管道步骤:")
for step_name, step_obj in full_pipeline.named_steps.items():
    print(f"  {step_name}: {type(step_obj).__name__}")
pca_step = full_pipeline.named_steps['pca']
print(f"PCA 方差解释率: {pca_step.explained_variance_ratio_}")

# 9. Pipeline 防止数据泄露
# Pipeline 在交叉验证中自动对每折独立预处理
# 不会将测试集信息泄露到训练集

# 10. 最佳实践
# - 始终使用 Pipeline 而非手动预处理
# - ColumnTransformer 处理混合类型数据
# - Pipeline + GridSearchCV 一站式调参
# - make_pipeline 快速构建简单管道

燎伎钒抗抡期辞安瘴瓜橇揭匠哑谛

m.epr.hmybk.cn/99719.Doc
m.epr.hmybk.cn/75195.Doc
m.epr.hmybk.cn/15179.Doc
m.epr.hmybk.cn/31533.Doc
m.epr.hmybk.cn/26246.Doc
m.epr.hmybk.cn/46846.Doc
m.epr.hmybk.cn/20668.Doc
m.epr.hmybk.cn/57719.Doc
m.epr.hmybk.cn/99533.Doc
m.epr.hmybk.cn/20648.Doc
m.epe.hmybk.cn/55791.Doc
m.epe.hmybk.cn/91551.Doc
m.epe.hmybk.cn/73515.Doc
m.epe.hmybk.cn/57973.Doc
m.epe.hmybk.cn/44200.Doc
m.epe.hmybk.cn/71177.Doc
m.epe.hmybk.cn/02688.Doc
m.epe.hmybk.cn/19179.Doc
m.epe.hmybk.cn/44264.Doc
m.epe.hmybk.cn/26844.Doc
m.epe.hmybk.cn/46802.Doc
m.epe.hmybk.cn/51939.Doc
m.epe.hmybk.cn/42228.Doc
m.epe.hmybk.cn/77773.Doc
m.epe.hmybk.cn/79935.Doc
m.epe.hmybk.cn/51371.Doc
m.epe.hmybk.cn/66208.Doc
m.epe.hmybk.cn/51133.Doc
m.epe.hmybk.cn/62644.Doc
m.epe.hmybk.cn/02082.Doc
m.epw.hmybk.cn/40268.Doc
m.epw.hmybk.cn/93551.Doc
m.epw.hmybk.cn/79977.Doc
m.epw.hmybk.cn/33593.Doc
m.epw.hmybk.cn/88028.Doc
m.epw.hmybk.cn/42044.Doc
m.epw.hmybk.cn/68082.Doc
m.epw.hmybk.cn/15397.Doc
m.epw.hmybk.cn/46486.Doc
m.epw.hmybk.cn/42260.Doc
m.epw.hmybk.cn/11597.Doc
m.epw.hmybk.cn/99359.Doc
m.epw.hmybk.cn/77913.Doc
m.epw.hmybk.cn/31577.Doc
m.epw.hmybk.cn/79539.Doc
m.epw.hmybk.cn/40280.Doc
m.epw.hmybk.cn/77115.Doc
m.epw.hmybk.cn/64660.Doc
m.epw.hmybk.cn/04486.Doc
m.epw.hmybk.cn/97759.Doc
m.epq.hmybk.cn/79519.Doc
m.epq.hmybk.cn/68080.Doc
m.epq.hmybk.cn/00042.Doc
m.epq.hmybk.cn/02684.Doc
m.epq.hmybk.cn/95351.Doc
m.epq.hmybk.cn/17913.Doc
m.epq.hmybk.cn/79793.Doc
m.epq.hmybk.cn/99153.Doc
m.epq.hmybk.cn/39395.Doc
m.epq.hmybk.cn/79311.Doc
m.epq.hmybk.cn/57553.Doc
m.epq.hmybk.cn/68440.Doc
m.epq.hmybk.cn/53313.Doc
m.epq.hmybk.cn/57153.Doc
m.epq.hmybk.cn/99159.Doc
m.epq.hmybk.cn/79519.Doc
m.epq.hmybk.cn/79779.Doc
m.epq.hmybk.cn/57795.Doc
m.epq.hmybk.cn/75733.Doc
m.epq.hmybk.cn/20286.Doc
m.eom.hmybk.cn/95759.Doc
m.eom.hmybk.cn/35177.Doc
m.eom.hmybk.cn/08864.Doc
m.eom.hmybk.cn/37995.Doc
m.eom.hmybk.cn/33153.Doc
m.eom.hmybk.cn/62686.Doc
m.eom.hmybk.cn/15799.Doc
m.eom.hmybk.cn/66442.Doc
m.eom.hmybk.cn/75977.Doc
m.eom.hmybk.cn/84004.Doc
m.eom.hmybk.cn/15597.Doc
m.eom.hmybk.cn/55793.Doc
m.eom.hmybk.cn/15137.Doc
m.eom.hmybk.cn/51119.Doc
m.eom.hmybk.cn/55119.Doc
m.eom.hmybk.cn/26802.Doc
m.eom.hmybk.cn/84620.Doc
m.eom.hmybk.cn/39579.Doc
m.eom.hmybk.cn/51331.Doc
m.eom.hmybk.cn/51573.Doc
m.eon.hmybk.cn/59959.Doc
m.eon.hmybk.cn/82648.Doc
m.eon.hmybk.cn/00480.Doc
m.eon.hmybk.cn/79153.Doc
m.eon.hmybk.cn/22860.Doc
m.eon.hmybk.cn/22046.Doc
m.eon.hmybk.cn/95937.Doc
m.eon.hmybk.cn/53351.Doc
m.eon.hmybk.cn/57951.Doc
m.eon.hmybk.cn/08244.Doc
m.eon.hmybk.cn/11751.Doc
m.eon.hmybk.cn/19715.Doc
m.eon.hmybk.cn/57313.Doc
m.eon.hmybk.cn/97755.Doc
m.eon.hmybk.cn/15577.Doc
m.eon.hmybk.cn/80688.Doc
m.eon.hmybk.cn/73793.Doc
m.eon.hmybk.cn/66482.Doc
m.eon.hmybk.cn/57993.Doc
m.eon.hmybk.cn/06042.Doc
m.eob.hmybk.cn/26806.Doc
m.eob.hmybk.cn/28604.Doc
m.eob.hmybk.cn/82680.Doc
m.eob.hmybk.cn/24884.Doc
m.eob.hmybk.cn/82262.Doc
m.eob.hmybk.cn/28242.Doc
m.eob.hmybk.cn/17799.Doc
m.eob.hmybk.cn/84822.Doc
m.eob.hmybk.cn/84062.Doc
m.eob.hmybk.cn/95593.Doc
m.eob.hmybk.cn/06842.Doc
m.eob.hmybk.cn/91135.Doc
m.eob.hmybk.cn/26284.Doc
m.eob.hmybk.cn/88406.Doc
m.eob.hmybk.cn/51731.Doc
m.eob.hmybk.cn/22606.Doc
m.eob.hmybk.cn/37935.Doc
m.eob.hmybk.cn/15759.Doc
m.eob.hmybk.cn/13115.Doc
m.eob.hmybk.cn/75559.Doc
m.eov.hmybk.cn/19353.Doc
m.eov.hmybk.cn/77595.Doc
m.eov.hmybk.cn/39195.Doc
m.eov.hmybk.cn/39715.Doc
m.eov.hmybk.cn/42802.Doc
m.eov.hmybk.cn/19911.Doc
m.eov.hmybk.cn/33791.Doc
m.eov.hmybk.cn/11319.Doc
m.eov.hmybk.cn/77171.Doc
m.eov.hmybk.cn/13337.Doc
m.eov.hmybk.cn/42082.Doc
m.eov.hmybk.cn/66844.Doc
m.eov.hmybk.cn/13115.Doc
m.eov.hmybk.cn/82660.Doc
m.eov.hmybk.cn/59999.Doc
m.eov.hmybk.cn/57993.Doc
m.eov.hmybk.cn/97155.Doc
m.eov.hmybk.cn/40662.Doc
m.eov.hmybk.cn/64042.Doc
m.eov.hmybk.cn/60840.Doc
m.eoc.hmybk.cn/44264.Doc
m.eoc.hmybk.cn/62800.Doc
m.eoc.hmybk.cn/91193.Doc
m.eoc.hmybk.cn/08846.Doc
m.eoc.hmybk.cn/53319.Doc
m.eoc.hmybk.cn/75917.Doc
m.eoc.hmybk.cn/26800.Doc
m.eoc.hmybk.cn/08886.Doc
m.eoc.hmybk.cn/93337.Doc
m.eoc.hmybk.cn/51331.Doc
m.eoc.hmybk.cn/62400.Doc
m.eoc.hmybk.cn/95799.Doc
m.eoc.hmybk.cn/93337.Doc
m.eoc.hmybk.cn/13393.Doc
m.eoc.hmybk.cn/95719.Doc
m.eoc.hmybk.cn/57575.Doc
m.eoc.hmybk.cn/28248.Doc
m.eoc.hmybk.cn/77799.Doc
m.eoc.hmybk.cn/15333.Doc
m.eoc.hmybk.cn/19513.Doc
m.eox.hmybk.cn/08044.Doc
m.eox.hmybk.cn/79113.Doc
m.eox.hmybk.cn/40848.Doc
m.eox.hmybk.cn/84044.Doc
m.eox.hmybk.cn/46866.Doc
m.eox.hmybk.cn/77177.Doc
m.eox.hmybk.cn/11935.Doc
m.eox.hmybk.cn/57517.Doc
m.eox.hmybk.cn/66444.Doc
m.eox.hmybk.cn/66604.Doc
m.eox.hmybk.cn/39311.Doc
m.eox.hmybk.cn/55113.Doc
m.eox.hmybk.cn/02422.Doc
m.eox.hmybk.cn/35599.Doc
m.eox.hmybk.cn/59979.Doc
m.eox.hmybk.cn/40404.Doc
m.eox.hmybk.cn/17593.Doc
m.eox.hmybk.cn/20226.Doc
m.eox.hmybk.cn/11577.Doc
m.eox.hmybk.cn/04842.Doc
m.eoz.hmybk.cn/71159.Doc
m.eoz.hmybk.cn/64228.Doc
m.eoz.hmybk.cn/84060.Doc
m.eoz.hmybk.cn/91317.Doc
m.eoz.hmybk.cn/35395.Doc
m.eoz.hmybk.cn/62626.Doc
m.eoz.hmybk.cn/15999.Doc
m.eoz.hmybk.cn/71337.Doc
m.eoz.hmybk.cn/24864.Doc
m.eoz.hmybk.cn/35975.Doc
m.eoz.hmybk.cn/97511.Doc
m.eoz.hmybk.cn/73799.Doc
m.eoz.hmybk.cn/62804.Doc
m.eoz.hmybk.cn/55753.Doc
m.eoz.hmybk.cn/19113.Doc
m.eoz.hmybk.cn/59713.Doc
m.eoz.hmybk.cn/95771.Doc
m.eoz.hmybk.cn/66868.Doc
m.eoz.hmybk.cn/08460.Doc
m.eoz.hmybk.cn/35991.Doc
m.eol.hmybk.cn/44866.Doc
m.eol.hmybk.cn/40620.Doc
m.eol.hmybk.cn/26460.Doc
m.eol.hmybk.cn/66062.Doc
m.eol.hmybk.cn/04402.Doc
m.eol.hmybk.cn/68282.Doc
m.eol.hmybk.cn/95357.Doc
m.eol.hmybk.cn/95331.Doc
m.eol.hmybk.cn/13739.Doc
m.eol.hmybk.cn/13995.Doc
m.eol.hmybk.cn/59555.Doc
m.eol.hmybk.cn/88620.Doc
m.eol.hmybk.cn/80646.Doc
m.eol.hmybk.cn/48244.Doc
m.eol.hmybk.cn/20068.Doc
m.eol.hmybk.cn/53791.Doc
m.eol.hmybk.cn/48886.Doc
m.eol.hmybk.cn/59151.Doc
m.eol.hmybk.cn/31797.Doc
m.eol.hmybk.cn/39173.Doc
m.eok.hmybk.cn/06248.Doc
m.eok.hmybk.cn/73177.Doc
m.eok.hmybk.cn/06282.Doc
m.eok.hmybk.cn/73555.Doc
m.eok.hmybk.cn/02460.Doc
m.eok.hmybk.cn/35753.Doc
m.eok.hmybk.cn/64640.Doc
m.eok.hmybk.cn/31597.Doc
m.eok.hmybk.cn/39191.Doc
m.eok.hmybk.cn/51799.Doc
m.eok.hmybk.cn/33755.Doc
m.eok.hmybk.cn/31155.Doc
m.eok.hmybk.cn/15737.Doc
m.eok.hmybk.cn/99379.Doc
m.eok.hmybk.cn/64488.Doc
m.eok.hmybk.cn/00028.Doc
m.eok.hmybk.cn/55113.Doc
m.eok.hmybk.cn/55991.Doc
m.eok.hmybk.cn/71791.Doc
m.eok.hmybk.cn/31933.Doc
m.eoj.hmybk.cn/11355.Doc
m.eoj.hmybk.cn/66882.Doc
m.eoj.hmybk.cn/00244.Doc
m.eoj.hmybk.cn/33993.Doc
m.eoj.hmybk.cn/59971.Doc
m.eoj.hmybk.cn/13797.Doc
m.eoj.hmybk.cn/91513.Doc
m.eoj.hmybk.cn/97535.Doc
m.eoj.hmybk.cn/84266.Doc
m.eoj.hmybk.cn/39737.Doc
m.eoj.hmybk.cn/91971.Doc
m.eoj.hmybk.cn/57991.Doc
m.eoj.hmybk.cn/84688.Doc
m.eoj.hmybk.cn/22484.Doc
m.eoj.hmybk.cn/35953.Doc
m.eoj.hmybk.cn/13551.Doc
m.eoj.hmybk.cn/40048.Doc
m.eoj.hmybk.cn/82628.Doc
m.eoj.hmybk.cn/15771.Doc
m.eoj.hmybk.cn/55753.Doc
m.eoh.hmybk.cn/53531.Doc
m.eoh.hmybk.cn/80684.Doc
m.eoh.hmybk.cn/17393.Doc
m.eoh.hmybk.cn/51719.Doc
m.eoh.hmybk.cn/37779.Doc
m.eoh.hmybk.cn/13113.Doc
m.eoh.hmybk.cn/73911.Doc
m.eoh.hmybk.cn/55131.Doc
m.eoh.hmybk.cn/91599.Doc
m.eoh.hmybk.cn/19371.Doc
m.eoh.hmybk.cn/57553.Doc
m.eoh.hmybk.cn/97335.Doc
m.eoh.hmybk.cn/77133.Doc
m.eoh.hmybk.cn/11953.Doc
m.eoh.hmybk.cn/75955.Doc
m.eoh.hmybk.cn/31317.Doc
m.eoh.hmybk.cn/31399.Doc
m.eoh.hmybk.cn/15953.Doc
m.eoh.hmybk.cn/59759.Doc
m.eoh.hmybk.cn/26402.Doc
m.eog.hmybk.cn/55179.Doc
m.eog.hmybk.cn/40648.Doc
m.eog.hmybk.cn/24246.Doc
m.eog.hmybk.cn/46268.Doc
m.eog.hmybk.cn/13111.Doc
m.eog.hmybk.cn/91979.Doc
m.eog.hmybk.cn/82880.Doc
m.eog.hmybk.cn/04480.Doc
m.eog.hmybk.cn/62848.Doc
m.eog.hmybk.cn/15733.Doc
m.eog.hmybk.cn/91959.Doc
m.eog.hmybk.cn/19755.Doc
m.eog.hmybk.cn/80002.Doc
m.eog.hmybk.cn/57515.Doc
m.eog.hmybk.cn/51991.Doc
m.eog.hmybk.cn/97319.Doc
m.eog.hmybk.cn/91157.Doc
m.eog.hmybk.cn/37399.Doc
m.eog.hmybk.cn/46882.Doc
m.eog.hmybk.cn/75575.Doc
m.eof.hmybk.cn/88426.Doc
m.eof.hmybk.cn/57391.Doc
m.eof.hmybk.cn/59955.Doc
m.eof.hmybk.cn/02264.Doc
m.eof.hmybk.cn/19771.Doc
m.eof.hmybk.cn/37577.Doc
m.eof.hmybk.cn/46280.Doc
m.eof.hmybk.cn/93733.Doc
m.eof.hmybk.cn/93737.Doc
m.eof.hmybk.cn/42226.Doc
m.eof.hmybk.cn/46666.Doc
m.eof.hmybk.cn/13959.Doc
m.eof.hmybk.cn/31913.Doc
m.eof.hmybk.cn/39719.Doc
m.eof.hmybk.cn/82426.Doc
m.eof.hmybk.cn/31117.Doc
m.eof.hmybk.cn/11357.Doc
m.eof.hmybk.cn/95177.Doc
m.eof.hmybk.cn/22646.Doc
m.eof.hmybk.cn/37139.Doc
m.eod.hmybk.cn/73791.Doc
m.eod.hmybk.cn/22006.Doc
m.eod.hmybk.cn/99393.Doc
m.eod.hmybk.cn/75553.Doc
m.eod.hmybk.cn/51599.Doc
m.eod.hmybk.cn/19999.Doc
m.eod.hmybk.cn/77775.Doc
m.eod.hmybk.cn/84044.Doc
m.eod.hmybk.cn/15931.Doc
m.eod.hmybk.cn/71511.Doc
m.eod.hmybk.cn/39171.Doc
m.eod.hmybk.cn/06264.Doc
m.eod.hmybk.cn/88620.Doc
m.eod.hmybk.cn/99517.Doc
m.eod.hmybk.cn/28624.Doc
m.eod.hmybk.cn/26004.Doc
m.eod.hmybk.cn/66420.Doc
m.eod.hmybk.cn/53113.Doc
m.eod.hmybk.cn/99773.Doc
m.eod.hmybk.cn/40820.Doc
m.eos.hmybk.cn/97519.Doc
m.eos.hmybk.cn/99979.Doc
m.eos.hmybk.cn/53339.Doc
m.eos.hmybk.cn/11551.Doc
m.eos.hmybk.cn/53957.Doc
m.eos.hmybk.cn/51317.Doc
m.eos.hmybk.cn/51915.Doc
m.eos.hmybk.cn/59993.Doc
m.eos.hmybk.cn/11711.Doc
m.eos.hmybk.cn/26028.Doc
m.eos.hmybk.cn/37519.Doc
m.eos.hmybk.cn/77919.Doc
m.eos.hmybk.cn/06466.Doc
m.eos.hmybk.cn/71177.Doc
m.eos.hmybk.cn/08006.Doc
m.eos.hmybk.cn/79757.Doc
m.eos.hmybk.cn/66460.Doc
m.eos.hmybk.cn/64202.Doc
m.eos.hmybk.cn/19399.Doc
m.eos.hmybk.cn/35359.Doc
m.eoa.hmybk.cn/26688.Doc
m.eoa.hmybk.cn/80082.Doc
m.eoa.hmybk.cn/20448.Doc
m.eoa.hmybk.cn/55333.Doc
m.eoa.hmybk.cn/19395.Doc
m.eoa.hmybk.cn/51933.Doc
m.eoa.hmybk.cn/99799.Doc
m.eoa.hmybk.cn/66040.Doc
m.eoa.hmybk.cn/46804.Doc
m.eoa.hmybk.cn/42420.Doc
m.eoa.hmybk.cn/59379.Doc
m.eoa.hmybk.cn/79599.Doc
m.eoa.hmybk.cn/68048.Doc
m.eoa.hmybk.cn/46046.Doc
m.eoa.hmybk.cn/57319.Doc
m.eoa.hmybk.cn/93579.Doc
m.eoa.hmybk.cn/68664.Doc
m.eoa.hmybk.cn/82482.Doc
m.eoa.hmybk.cn/75939.Doc
m.eoa.hmybk.cn/20266.Doc
m.eop.hmybk.cn/33359.Doc
m.eop.hmybk.cn/84086.Doc
m.eop.hmybk.cn/79793.Doc
m.eop.hmybk.cn/11577.Doc
m.eop.hmybk.cn/13777.Doc
