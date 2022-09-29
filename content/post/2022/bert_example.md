---
title: "BERTによるテキスト分類サンプルコード"
date: 2022-07-29
categories:
- データサイエンス
---

## GPU環境整備

今回はGCE VM + ColabでGPU環境を用意しました。
https://research.google.com/colaboratory/marketplace.html


```
!nvcc -V
!nvidia-smi
```

```
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 495.46       Driver Version: 460.32.03    CUDA Version: 11.2     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  Tesla T4            Off  | 00000000:00:04.0 Off |                    0 |
| N/A   74C    P0    30W /  70W |   9590MiB / 15109MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

## データ用意

data.csv

```csv
text, label
これはポジティブな文言です, 1
これはネガティブな文言です, 0
これはラベリングされていないです, 
```

## 前処理

```
!pip install neologdn demoji
```

```python
import pandas as pd
from sklearn.model_selection import train_test_split
import neologdn, demoji

def trim(text):
    text = neologdn.normalize(text,repeat=2) # テキストの正規化、繰り返しの除去
    text=demoji.replace(string=text, repl="") # 絵文字の除去
    return text

FILENAME='data.csv'
data = pd.read_csv(FILENAME).query('label == label') # ラベリングされている列のみ抽出

data['text'] = data['text'].map(trim)
data['label'] = data['label'].astype(int)
train, eval = train_test_split(data, test_size=0.2) # 8:2で学習データとテストデータに分割

train = train.reset_index()
eval = eval.reset_index()
```

## ファインチューニング

```
!!pip install simpletransformers pyarrow>=6.0.0 fugashi ipadic
```

```python
from simpletransformers.classification import ClassificationArgs, ClassificationModel
import torch, logging

logging.basicConfig(level=logging.INFO)
transformers_logger = logging.getLogger("transformers")
transformers_logger.setLevel(logging.WARNING)

args = ClassificationArgs(num_train_epochs=5, learning_rate = 5e-5, overwrite_output_dir=True)
model = ClassificationModel("bert", "cl-tohoku/bert-base-japanese-whole-word-masking", args=args, use_cuda=torch.cuda.is_available(), num_labels=2)
model.train_model(train)
result, model_outputs, wrong_predictions = model.eval_model(eval)
```

```
%load_ext tensorboard
%tensorboard --logdir runs
```

## 予測

```python
predictions, raw_outputs = model.predict(data['text'].values.tolist())
data=data.assign(prediction=pd.Series(predictions))
```
