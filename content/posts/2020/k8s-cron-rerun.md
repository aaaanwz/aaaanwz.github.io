---
title: "KubernetesのCronJobからJobを手動作成する"
date: 2020-11-30
draft: false
categories:
- kubernetes
- 小ネタ
---

```shell
kubectl create job 作成するJob名 --from=cronjob/CronJob名
```

https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#-em-job-em-
