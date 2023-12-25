---
title: "特徴量重要度(SHAP)をDataframeで取得する"
date: 2023-12-25
draft: false
categories:
- 機械学習
- 小ネタ
---

[SHAP](https://shap.readthedocs.io/en/latest/#) を用いて `shap.summary_plot()` で特徴量重要度のグラフを出力できるが、定量データとして取得したい場合がある。
以下のようにすればDataFrameにできる。

```python
import lightgbm as lgb
from sklearn import datasets
import shap

X = train_df.drop('target', axis=1)
y = train_df['target']

model = lgb.train(
    params={},
    train_set=lgb.Dataset(X, y)
)

explainer = shap.TreeExplainer(model)
shap_values = explainer(X)

pd.DataFrame(shap_values.values[0], columns=["feature_importance"], index=X.columns).sort_values(
    by="feature_importance", ascending=False)
```


||feature_importance|
|-|-|
|age|0.045195|
|gender|0.001254|
|...|...|