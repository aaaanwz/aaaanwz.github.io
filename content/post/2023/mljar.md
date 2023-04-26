---
title: "AutoMLライブラリ MLJARを使う"
date: 2023-04-26
draft: false
categories:
- データサイエンス
---

# MLJARとは

MLJARは、自動機械学習(AutoML)フレームワークであり、データの前処理、特徴量生成、モデルの選択、ハイパーパラメータの最適化、アンサンブル学習など、機械学習の一連の流れを自動化することができます。
auto-sklearn、H2O、AutoGluonといった競合に対して高いスコアを誇り、更にモデルの解釈性を高めるための機能も備えています。

![](https://pbs.twimg.com/media/EtZbVhtXAAIs0TW?format=png&name=large)

触ってみたところ雑にデータを突っ込むだけで分析コンペでもある程度戦えるモデルができてしまいました。
ChatGPTに使い方を聞いても微妙に間違っていたりするのでここにまとめておきます。

## インストール

```sh
pip install mljar-supervised
```

## 学習

```python
import pandas as pd
from supervised.automl import AutoML

train_df = pd.read_csv("train.csv")
X = train_df.drop('target', axis=1)
y = train_df['target']

automl = AutoML(ml_task="classification", eval_metric="accuracy", mode="Compete")
automl.fit(X_train, y_train)

print("score:", automl.score(X, y))
```

#### ml_task

タスクの種類

- auto
- binary_classification
- multiclass_classification
- regression

のいずれか

#### eval_metric

評価指標

- binary_classification：`logloss`, `auc`, `f1`, `average_precision`, `accuracy`
- multiclass_classification: `logloss`,`f1`,`accuracy`
- regression: `rmse`, `mse`, `mae`, `r2`,`mape`,`sperman`,`pearson`

#### mode

- Explain
解釈性を重視したモデルを選択する
- Perform
バランス重視
- Compete
分析コンペ向け
- Optuna
学習時間を度外視したスコア特化

## モデルの解釈

```python
automl.report()
```

試行したモデルのスコアや学習曲線などを表示します。

## 予測

```python
test_df = pd.read_csv("test.csv")
predictions = automl.predict(test_df)
```

## デプロイ

### モデルの保存

```python
best_model = automl.get_best_model()
best_model.save("best_model.pkl")
```

### モデルの読み込み

```python
from supervised import Model

loaded_model = Model.load("best_model.pkl")
new_predictions = loaded_model.predict(new_data)
```
