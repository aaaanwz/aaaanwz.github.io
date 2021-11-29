---
title: "GCPでカスタムロールをサービスアカウントにbindingしようとしてエラーになる場合"
date: 2020-03-03
draft: false
categories:
- GCP
- 小ネタ
---


```shell-session

$ gcloud projects add-iam-policy-binding myproject --member=serviceAccount:myserviceaccount@myproject.iam.gserviceaccount.com --role='roles/mycustomrole'
ERROR: Policy modification failed. For a binding with condition, run "gcloud alpha iam policies lint-condition" to identify issues in condition.
ERROR: (gcloud.projects.add-iam-policy-binding) INVALID_ARGUMENT: Role roles/mycustomrole is not supported for this resource.
```

`--role` の指定を `roles/mycustomrole` ではなく `projects/myproject/roles/mycustomrole`にすればOK

```shell-session

$ gcloud projects add-iam-policy-binding myproject --member=serviceAccount:myserviceaccuont@myproject.iam.gserviceaccount.com --role='projects/myproject/roles/mycustomrole'
Updated IAM policy for project [myproject].
```
