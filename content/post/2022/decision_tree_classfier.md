---
title: "sklearn モデル選択とパラメータチューニングのサンプルコード"
date: 2022-06-29
categories:
- データサイエンス
draft: true
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

## モデル比較

```python
names = [
    "Nearest Neighbors",
    "Linear SVM",
    "RBF SVM",
    "Gaussian Process",
    "Decision Tree",
    "Random Forest",
    "Neural Net",
    "AdaBoost",
    "Naive Bayes",
    "QDA",
]

classifiers = [
    KNeighborsClassifier(3),
    SVC(kernel="linear", C=0.025),
    SVC(gamma=2, C=1),
    GaussianProcessClassifier(1.0 * RBF(1.0)),
    DecisionTreeClassifier(max_depth=5),
    RandomForestClassifier(max_depth=5, n_estimators=10, max_features=1),
    MLPClassifier(alpha=1, max_iter=1000),
    AdaBoostClassifier(),
    GaussianNB(),
    QuadraticDiscriminantAnalysis(),
]

for name, clf in zip(names, classifiers):
    print(name)
    result = cross_val_score(clf, X, y, n_jobs=-1,scoring='neg_mean_absolute_error')
    print(result.mean())
```

# グリッドサーチ

```python
from sklearn.tree import plot_tree
import matplotlib.pyplot as plt

plt.figure(figsize=(20, 15))
plot_tree(model)
plt.show()
```