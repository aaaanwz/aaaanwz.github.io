---
title: "kubectl logsに任意のログを表示する"
date: 2021-09-08
draft: false
categories:
- Kubernetes
- 小ネタ
---

kubectl logsはPID1の標準出力を表示するため、直接書き込んでしまえばなんでも表示できる。

```
$ kubectl exec -it pod-xxx bash
# echo 'show as stdin' > /proc/1/fd/1
# echo 'show as stderr' > /proc/1/fd/2
```

```sh
$ kubectl logs pod-xxx

show as stdin
show as stderr
```