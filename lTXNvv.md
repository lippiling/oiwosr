怂缓恐世靶


# Python特征工程基础
# 特征工程是将原始数据转换为更适合模型的表示
# 包括缩放、编码、特征构造和特征选择

# 1. 导入库
import numpy as np
import pandas as pd
from sklearn.preprocessing import (
    StandardScaler, MinMaxScaler, OneHotEncoder, LabelEncoder, PolynomialFeatures
)
from sklearn.feature_selection import SelectKBest, f_classif
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split

# 2. 创建示例数据（含数值和分类特征）
data = {
    'age': [25, 30, 35, 40, 45, 50, 55, 60],
    'salary': [30000, 45000, 50000, 60000, 70000, 80000, 90000, 100000],
    'city': ['BJ', 'SH', 'GZ', 'BJ', 'SH', 'SZ', 'GZ', 'BJ'],
    'department': ['Tech', 'HR', 'Tech', 'Sales', 'Tech', 'HR', 'Sales', 'Tech'],
    'target': [0, 0, 1, 1, 1, 0, 0, 1]
}
df = pd.DataFrame(data)
print(f"原始数据:\n{df}\n")

# 3. StandardScaler — 标准化（均值=0，标准差=1）
scaler_std = StandardScaler()
age_salary = df[['age', 'salary']]
age_salary_scaled = scaler_std.fit_transform(age_salary)
print(f"=== StandardScaler ===")
print(f"均值: {age_salary_scaled.mean(axis=0).round(4)}")
print(f"标准差: {age_salary_scaled.std(axis=0).round(4)}")

# 4. MinMaxScaler — 归一化
scaler_mm = MinMaxScaler()
age_salary_mm = scaler_mm.fit_transform(age_salary)
print(f"\n=== MinMaxScaler ===")
print(f"最小值: {age_salary_mm.min(axis=0)}, 最大值: {age_salary_mm.max(axis=0)}")

# 5. OneHotEncoder — 独热编码
ohe = OneHotEncoder(sparse_output=False, drop='first')
city_encoded = ohe.fit_transform(df[['city']])
print(f"\n=== OneHotEncoder ===")
print(f"原始类别: {ohe.categories_}")
print(f"编码结果:\n{city_encoded}")

# 6. LabelEncoder — 标签编码
le = LabelEncoder()
department_encoded = le.fit_transform(df['department'])
print(f"=== LabelEncoder ===")
print(f"编码映射: {dict(zip(le.classes_, le.transform(le.classes_)))}")
print(f"编码结果: {department_encoded}")

# 7. PolynomialFeatures — 多项式特征
poly = PolynomialFeatures(degree=2, include_bias=False)
age_salary_poly = poly.fit_transform(age_salary)
print(f"\n=== PolynomialFeatures ===")
print(f"原始 {age_salary.shape[1]} 维 -> {age_salary_poly.shape[1]} 维")
print(f"特征名称: {poly.get_feature_names_out()}")

# 8. SelectKBest — 特征选择
iris = load_iris()
X, y = iris.data, iris.target
selector = SelectKBest(score_func=f_classif, k=2)
X_selected = selector.fit_transform(X, y)
print(f"\n=== SelectKBest (K=2) ===")
print(f"特征分数: {selector.scores_}")
print(f"选中索引: {selector.get_support(indices=True)}")

# 9. 完整流程：拟合训练集，转换测试集
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.3, random_state=42
)
scaler = StandardScaler()
X_train_scaled = scaler.fit_transform(X_train)
X_test_scaled = scaler.transform(X_test)  # 只用 transform！
print(f"\n=== 特征工程总结 ===")
print(f"训练集均值 (≈0): {X_train_scaled.mean(axis=0).round(4)}")

# 10. 特征工程原则
# - 训练集 fit_transform，测试集只 transform（避免数据泄露）
# - 数值型: StandardScaler / MinMaxScaler
# - 无序类别: OneHotEncoder，有序: OrdinalEncoder
# - 特征选择可减少维度、降低过拟合

潦执帐棠值韧韧换暮烤屯谫犊蹲坟

m.weq.hzldf.cn/66844.Doc
m.weq.hzldf.cn/59393.Doc
m.weq.hzldf.cn/06828.Doc
m.weq.hzldf.cn/20028.Doc
m.weq.hzldf.cn/86080.Doc
m.weq.hzldf.cn/93359.Doc
m.weq.hzldf.cn/13371.Doc
m.weq.hzldf.cn/20422.Doc
m.weq.hzldf.cn/26060.Doc
m.weq.hzldf.cn/88286.Doc
m.weq.hzldf.cn/26604.Doc
m.weq.hzldf.cn/75959.Doc
m.weq.hzldf.cn/08440.Doc
m.weq.hzldf.cn/24480.Doc
m.weq.hzldf.cn/82848.Doc
m.weq.hzldf.cn/46266.Doc
m.weq.hzldf.cn/00286.Doc
m.wwm.hzldf.cn/24880.Doc
m.wwm.hzldf.cn/77991.Doc
m.wwm.hzldf.cn/31975.Doc
m.wwm.hzldf.cn/20202.Doc
m.wwm.hzldf.cn/44260.Doc
m.wwm.hzldf.cn/55777.Doc
m.wwm.hzldf.cn/37119.Doc
m.wwm.hzldf.cn/26446.Doc
m.wwm.hzldf.cn/02644.Doc
m.wwm.hzldf.cn/80886.Doc
m.wwm.hzldf.cn/84002.Doc
m.wwm.hzldf.cn/26208.Doc
m.wwm.hzldf.cn/80666.Doc
m.wwm.hzldf.cn/84008.Doc
m.wwm.hzldf.cn/66842.Doc
m.wwm.hzldf.cn/84648.Doc
m.wwm.hzldf.cn/35977.Doc
m.wwm.hzldf.cn/53133.Doc
m.wwm.hzldf.cn/99335.Doc
m.wwm.hzldf.cn/00608.Doc
m.wwn.hzldf.cn/20642.Doc
m.wwn.hzldf.cn/20224.Doc
m.wwn.hzldf.cn/24224.Doc
m.wwn.hzldf.cn/84202.Doc
m.wwn.hzldf.cn/48226.Doc
m.wwn.hzldf.cn/48680.Doc
m.wwn.hzldf.cn/68806.Doc
m.wwn.hzldf.cn/51753.Doc
m.wwn.hzldf.cn/82262.Doc
m.wwn.hzldf.cn/00666.Doc
m.wwn.hzldf.cn/02802.Doc
m.wwn.hzldf.cn/64262.Doc
m.wwn.hzldf.cn/37377.Doc
m.wwn.hzldf.cn/60200.Doc
m.wwn.hzldf.cn/39375.Doc
m.wwn.hzldf.cn/86640.Doc
m.wwn.hzldf.cn/00802.Doc
m.wwn.hzldf.cn/60240.Doc
m.wwn.hzldf.cn/86826.Doc
m.wwn.hzldf.cn/04202.Doc
m.wwb.hzldf.cn/06862.Doc
m.wwb.hzldf.cn/24880.Doc
m.wwb.hzldf.cn/99795.Doc
m.wwb.hzldf.cn/28004.Doc
m.wwb.hzldf.cn/04840.Doc
m.wwb.hzldf.cn/20646.Doc
m.wwb.hzldf.cn/62268.Doc
m.wwb.hzldf.cn/86688.Doc
m.wwb.hzldf.cn/44026.Doc
m.wwb.hzldf.cn/82420.Doc
m.wwb.hzldf.cn/53753.Doc
m.wwb.hzldf.cn/60628.Doc
m.wwb.hzldf.cn/68866.Doc
m.wwb.hzldf.cn/06864.Doc
m.wwb.hzldf.cn/48842.Doc
m.wwb.hzldf.cn/37339.Doc
m.wwb.hzldf.cn/02662.Doc
m.wwb.hzldf.cn/86482.Doc
m.wwb.hzldf.cn/48264.Doc
m.wwb.hzldf.cn/37133.Doc
m.wwv.hzldf.cn/59713.Doc
m.wwv.hzldf.cn/62822.Doc
m.wwv.hzldf.cn/84264.Doc
m.wwv.hzldf.cn/48242.Doc
m.wwv.hzldf.cn/33979.Doc
m.wwv.hzldf.cn/20408.Doc
m.wwv.hzldf.cn/44846.Doc
m.wwv.hzldf.cn/08646.Doc
m.wwv.hzldf.cn/71395.Doc
m.wwv.hzldf.cn/20002.Doc
m.wwv.hzldf.cn/66002.Doc
m.wwv.hzldf.cn/51399.Doc
m.wwv.hzldf.cn/13939.Doc
m.wwv.hzldf.cn/51519.Doc
m.wwv.hzldf.cn/79913.Doc
m.wwv.hzldf.cn/68886.Doc
m.wwv.hzldf.cn/57317.Doc
m.wwv.hzldf.cn/26808.Doc
m.wwv.hzldf.cn/57773.Doc
m.wwv.hzldf.cn/00264.Doc
m.wwc.hzldf.cn/40842.Doc
m.wwc.hzldf.cn/22648.Doc
m.wwc.hzldf.cn/33139.Doc
m.wwc.hzldf.cn/28846.Doc
m.wwc.hzldf.cn/28842.Doc
m.wwc.hzldf.cn/15797.Doc
m.wwc.hzldf.cn/80442.Doc
m.wwc.hzldf.cn/97775.Doc
m.wwc.hzldf.cn/62608.Doc
m.wwc.hzldf.cn/17377.Doc
m.wwc.hzldf.cn/40222.Doc
m.wwc.hzldf.cn/24400.Doc
m.wwc.hzldf.cn/06468.Doc
m.wwc.hzldf.cn/93519.Doc
m.wwc.hzldf.cn/53313.Doc
m.wwc.hzldf.cn/35339.Doc
m.wwc.hzldf.cn/24664.Doc
m.wwc.hzldf.cn/02404.Doc
m.wwc.hzldf.cn/66884.Doc
m.wwc.hzldf.cn/26484.Doc
m.wwx.hzldf.cn/75135.Doc
m.wwx.hzldf.cn/17519.Doc
m.wwx.hzldf.cn/80882.Doc
m.wwx.hzldf.cn/24626.Doc
m.wwx.hzldf.cn/48648.Doc
m.wwx.hzldf.cn/60466.Doc
m.wwx.hzldf.cn/84082.Doc
m.wwx.hzldf.cn/84226.Doc
m.wwx.hzldf.cn/95755.Doc
m.wwx.hzldf.cn/62668.Doc
m.wwx.hzldf.cn/13393.Doc
m.wwx.hzldf.cn/64266.Doc
m.wwx.hzldf.cn/51593.Doc
m.wwx.hzldf.cn/48062.Doc
m.wwx.hzldf.cn/46608.Doc
m.wwx.hzldf.cn/06668.Doc
m.wwx.hzldf.cn/22246.Doc
m.wwx.hzldf.cn/84220.Doc
m.wwx.hzldf.cn/11393.Doc
m.wwx.hzldf.cn/66464.Doc
m.wwz.hzldf.cn/57557.Doc
m.wwz.hzldf.cn/31997.Doc
m.wwz.hzldf.cn/40622.Doc
m.wwz.hzldf.cn/62846.Doc
m.wwz.hzldf.cn/02462.Doc
m.wwz.hzldf.cn/62464.Doc
m.wwz.hzldf.cn/24082.Doc
m.wwz.hzldf.cn/86868.Doc
m.wwz.hzldf.cn/46600.Doc
m.wwz.hzldf.cn/79319.Doc
m.wwz.hzldf.cn/79337.Doc
m.wwz.hzldf.cn/37111.Doc
m.wwz.hzldf.cn/60668.Doc
m.wwz.hzldf.cn/19197.Doc
m.wwz.hzldf.cn/15333.Doc
m.wwz.hzldf.cn/46240.Doc
m.wwz.hzldf.cn/91175.Doc
m.wwz.hzldf.cn/97399.Doc
m.wwz.hzldf.cn/24064.Doc
m.wwz.hzldf.cn/79375.Doc
m.wwl.hzldf.cn/84282.Doc
m.wwl.hzldf.cn/57557.Doc
m.wwl.hzldf.cn/35715.Doc
m.wwl.hzldf.cn/86046.Doc
m.wwl.hzldf.cn/84080.Doc
m.wwl.hzldf.cn/68460.Doc
m.wwl.hzldf.cn/35777.Doc
m.wwl.hzldf.cn/39575.Doc
m.wwl.hzldf.cn/97379.Doc
m.wwl.hzldf.cn/13791.Doc
m.wwl.hzldf.cn/40480.Doc
m.wwl.hzldf.cn/86882.Doc
m.wwl.hzldf.cn/42828.Doc
m.wwl.hzldf.cn/06842.Doc
m.wwl.hzldf.cn/08860.Doc
m.wwl.hzldf.cn/95397.Doc
m.wwl.hzldf.cn/64620.Doc
m.wwl.hzldf.cn/66220.Doc
m.wwl.hzldf.cn/55539.Doc
m.wwl.hzldf.cn/97515.Doc
m.wwk.hzldf.cn/08884.Doc
m.wwk.hzldf.cn/44826.Doc
m.wwk.hzldf.cn/31171.Doc
m.wwk.hzldf.cn/66426.Doc
m.wwk.hzldf.cn/24262.Doc
m.wwk.hzldf.cn/57391.Doc
m.wwk.hzldf.cn/00800.Doc
m.wwk.hzldf.cn/59339.Doc
m.wwk.hzldf.cn/82048.Doc
m.wwk.hzldf.cn/40808.Doc
m.wwk.hzldf.cn/55393.Doc
m.wwk.hzldf.cn/93519.Doc
m.wwk.hzldf.cn/42826.Doc
m.wwk.hzldf.cn/46426.Doc
m.wwk.hzldf.cn/06280.Doc
m.wwk.hzldf.cn/46248.Doc
m.wwk.hzldf.cn/08248.Doc
m.wwk.hzldf.cn/84044.Doc
m.wwk.hzldf.cn/66428.Doc
m.wwk.hzldf.cn/06240.Doc
m.wwj.hzldf.cn/64626.Doc
m.wwj.hzldf.cn/48846.Doc
m.wwj.hzldf.cn/88040.Doc
m.wwj.hzldf.cn/77993.Doc
m.wwj.hzldf.cn/33331.Doc
m.wwj.hzldf.cn/44640.Doc
m.wwj.hzldf.cn/20680.Doc
m.wwj.hzldf.cn/31159.Doc
m.wwj.hzldf.cn/48248.Doc
m.wwj.hzldf.cn/26860.Doc
m.wwj.hzldf.cn/06022.Doc
m.wwj.hzldf.cn/86408.Doc
m.wwj.hzldf.cn/35173.Doc
m.wwj.hzldf.cn/88826.Doc
m.wwj.hzldf.cn/00286.Doc
m.wwj.hzldf.cn/86028.Doc
m.wwj.hzldf.cn/48604.Doc
m.wwj.hzldf.cn/95393.Doc
m.wwj.hzldf.cn/40222.Doc
m.wwj.hzldf.cn/06822.Doc
m.wwh.hzldf.cn/02822.Doc
m.wwh.hzldf.cn/20080.Doc
m.wwh.hzldf.cn/35739.Doc
m.wwh.hzldf.cn/57991.Doc
m.wwh.hzldf.cn/88888.Doc
m.wwh.hzldf.cn/08400.Doc
m.wwh.hzldf.cn/66000.Doc
m.wwh.hzldf.cn/20480.Doc
m.wwh.hzldf.cn/55395.Doc
m.wwh.hzldf.cn/77579.Doc
m.wwh.hzldf.cn/37775.Doc
m.wwh.hzldf.cn/64804.Doc
m.wwh.hzldf.cn/28042.Doc
m.wwh.hzldf.cn/62208.Doc
m.wwh.hzldf.cn/86440.Doc
m.wwh.hzldf.cn/84266.Doc
m.wwh.hzldf.cn/08640.Doc
m.wwh.hzldf.cn/51555.Doc
m.wwh.hzldf.cn/04086.Doc
m.wwh.hzldf.cn/24660.Doc
m.wwg.hzldf.cn/22844.Doc
m.wwg.hzldf.cn/15997.Doc
m.wwg.hzldf.cn/39397.Doc
m.wwg.hzldf.cn/95351.Doc
m.wwg.hzldf.cn/88224.Doc
m.wwg.hzldf.cn/26480.Doc
m.wwg.hzldf.cn/06848.Doc
m.wwg.hzldf.cn/86648.Doc
m.wwg.hzldf.cn/66002.Doc
m.wwg.hzldf.cn/55937.Doc
m.wwg.hzldf.cn/44828.Doc
m.wwg.hzldf.cn/42668.Doc
m.wwg.hzldf.cn/06208.Doc
m.wwg.hzldf.cn/22064.Doc
m.wwg.hzldf.cn/42686.Doc
m.wwg.hzldf.cn/24408.Doc
m.wwg.hzldf.cn/44226.Doc
m.wwg.hzldf.cn/62848.Doc
m.wwg.hzldf.cn/28400.Doc
m.wwg.hzldf.cn/91795.Doc
m.wwf.hzldf.cn/40808.Doc
m.wwf.hzldf.cn/00006.Doc
m.wwf.hzldf.cn/22046.Doc
m.wwf.hzldf.cn/22226.Doc
m.wwf.hzldf.cn/57755.Doc
m.wwf.hzldf.cn/00028.Doc
m.wwf.hzldf.cn/48848.Doc
m.wwf.hzldf.cn/28886.Doc
m.wwf.hzldf.cn/37517.Doc
m.wwf.hzldf.cn/22004.Doc
m.wwf.hzldf.cn/66660.Doc
m.wwf.hzldf.cn/40220.Doc
m.wwf.hzldf.cn/40640.Doc
m.wwf.hzldf.cn/91353.Doc
m.wwf.hzldf.cn/97197.Doc
m.wwf.hzldf.cn/59711.Doc
m.wwf.hzldf.cn/04684.Doc
m.wwf.hzldf.cn/35719.Doc
m.wwf.hzldf.cn/62082.Doc
m.wwf.hzldf.cn/13737.Doc
m.wwd.hzldf.cn/06844.Doc
m.wwd.hzldf.cn/44460.Doc
m.wwd.hzldf.cn/57153.Doc
m.wwd.hzldf.cn/11733.Doc
m.wwd.hzldf.cn/91731.Doc
m.wwd.hzldf.cn/44668.Doc
m.wwd.hzldf.cn/62044.Doc
m.wwd.hzldf.cn/91317.Doc
m.wwd.hzldf.cn/93171.Doc
m.wwd.hzldf.cn/00002.Doc
m.wwd.hzldf.cn/82648.Doc
m.wwd.hzldf.cn/60400.Doc
m.wwd.hzldf.cn/00080.Doc
m.wwd.hzldf.cn/20642.Doc
m.wwd.hzldf.cn/75593.Doc
m.wwd.hzldf.cn/62260.Doc
m.wwd.hzldf.cn/80808.Doc
m.wwd.hzldf.cn/04020.Doc
m.wwd.hzldf.cn/48288.Doc
m.wwd.hzldf.cn/46004.Doc
m.wws.hzldf.cn/80828.Doc
m.wws.hzldf.cn/64486.Doc
m.wws.hzldf.cn/57555.Doc
m.wws.hzldf.cn/24666.Doc
m.wws.hzldf.cn/79519.Doc
m.wws.hzldf.cn/82468.Doc
m.wws.hzldf.cn/62800.Doc
m.wws.hzldf.cn/02424.Doc
m.wws.hzldf.cn/06820.Doc
m.wws.hzldf.cn/82226.Doc
m.wws.hzldf.cn/62202.Doc
m.wws.hzldf.cn/82886.Doc
m.wws.hzldf.cn/68462.Doc
m.wws.hzldf.cn/75171.Doc
m.wws.hzldf.cn/99119.Doc
m.wws.hzldf.cn/15797.Doc
m.wws.hzldf.cn/80042.Doc
m.wws.hzldf.cn/40624.Doc
m.wws.hzldf.cn/68040.Doc
m.wws.hzldf.cn/71917.Doc
m.wwa.hzldf.cn/00688.Doc
m.wwa.hzldf.cn/28282.Doc
m.wwa.hzldf.cn/84462.Doc
m.wwa.hzldf.cn/08446.Doc
m.wwa.hzldf.cn/66888.Doc
m.wwa.hzldf.cn/46666.Doc
m.wwa.hzldf.cn/48686.Doc
m.wwa.hzldf.cn/39575.Doc
m.wwa.hzldf.cn/28846.Doc
m.wwa.hzldf.cn/84202.Doc
m.wwa.hzldf.cn/59977.Doc
m.wwa.hzldf.cn/02042.Doc
m.wwa.hzldf.cn/64644.Doc
m.wwa.hzldf.cn/82202.Doc
m.wwa.hzldf.cn/06606.Doc
m.wwa.hzldf.cn/19117.Doc
m.wwa.hzldf.cn/95995.Doc
m.wwa.hzldf.cn/40826.Doc
m.wwa.hzldf.cn/31155.Doc
m.wwa.hzldf.cn/82680.Doc
m.wwp.hzldf.cn/24644.Doc
m.wwp.hzldf.cn/82406.Doc
m.wwp.hzldf.cn/17757.Doc
m.wwp.hzldf.cn/99711.Doc
m.wwp.hzldf.cn/31319.Doc
m.wwp.hzldf.cn/42800.Doc
m.wwp.hzldf.cn/86884.Doc
m.wwp.hzldf.cn/00620.Doc
m.wwp.hzldf.cn/68482.Doc
m.wwp.hzldf.cn/80042.Doc
m.wwp.hzldf.cn/84648.Doc
m.wwp.hzldf.cn/95995.Doc
m.wwp.hzldf.cn/48608.Doc
m.wwp.hzldf.cn/35735.Doc
m.wwp.hzldf.cn/33799.Doc
m.wwp.hzldf.cn/24808.Doc
m.wwp.hzldf.cn/95131.Doc
m.wwp.hzldf.cn/08640.Doc
m.wwp.hzldf.cn/33739.Doc
m.wwp.hzldf.cn/33999.Doc
m.wwo.hzldf.cn/06406.Doc
m.wwo.hzldf.cn/88802.Doc
m.wwo.hzldf.cn/40062.Doc
m.wwo.hzldf.cn/66008.Doc
m.wwo.hzldf.cn/20680.Doc
m.wwo.hzldf.cn/55737.Doc
m.wwo.hzldf.cn/68246.Doc
m.wwo.hzldf.cn/82640.Doc
m.wwo.hzldf.cn/40422.Doc
m.wwo.hzldf.cn/60266.Doc
m.wwo.hzldf.cn/80402.Doc
m.wwo.hzldf.cn/71717.Doc
m.wwo.hzldf.cn/46440.Doc
m.wwo.hzldf.cn/02260.Doc
m.wwo.hzldf.cn/86808.Doc
m.wwo.hzldf.cn/62200.Doc
m.wwo.hzldf.cn/60604.Doc
m.wwo.hzldf.cn/20644.Doc
m.wwo.hzldf.cn/93711.Doc
m.wwo.hzldf.cn/15315.Doc
m.wwi.hzldf.cn/95153.Doc
m.wwi.hzldf.cn/08200.Doc
m.wwi.hzldf.cn/77599.Doc
m.wwi.hzldf.cn/82444.Doc
m.wwi.hzldf.cn/55175.Doc
m.wwi.hzldf.cn/13591.Doc
m.wwi.hzldf.cn/31559.Doc
m.wwi.hzldf.cn/20688.Doc
m.wwi.hzldf.cn/46860.Doc
m.wwi.hzldf.cn/33339.Doc
m.wwi.hzldf.cn/33179.Doc
m.wwi.hzldf.cn/39575.Doc
m.wwi.hzldf.cn/55315.Doc
m.wwi.hzldf.cn/26222.Doc
m.wwi.hzldf.cn/82868.Doc
m.wwi.hzldf.cn/64868.Doc
m.wwi.hzldf.cn/35311.Doc
m.wwi.hzldf.cn/02842.Doc
