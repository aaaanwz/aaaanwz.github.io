---
title: "分類木モデルの可視化サンプルコード"
date: 2022-06-29
categories:
- データサイエンス
---

## データ用意

data.csv

```csv
param_a, param_b, label
hoge, fuga, 1
foo, bar, 0
```

```python
import pandas as pd
from sklearn.model_selection import train_test_split

FILENAME='data.csv'
data = pd.read_csv(FILENAME)

data['labels'] = data['labels'].astype(int)
```

```python
from sklearn.tree import DecisionTreeClassifier
from sklearn.model_selection import cross_val_score

X = data.drop('label', axis=1)
y = data.drop('param_a', axis=1).drop('param_b',axis=1)

model = DecisionTreeClassifier()
model.fit(X,y)
```

```python
from sklearn.tree import plot_tree
import matplotlib.pyplot as plt

plt.figure(figsize=(20, 15))
plot_tree(model)
plt.show()
```