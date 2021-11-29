---
title: "Argo Workflowsの失敗時にデフォルトでSlackに通知する"
date: 2021-10-29
draft: false
categories:
- Kubernetes
---

Argo workflowsでは [Default Workflow Spec](https://argoproj.github.io/argo-workflows/default-workflow-specs/) を設定する事でワークフローに色々とパッチできる。

以下のようにexit-handlerをworkflowDefaultsにしておくと、ワークフロー側に何も記述せずとも失敗時にSlackに通知できる。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: workflow-controller-configmap
data:
  workflowDefaults: |
    spec:
      onExit: exit-handler
      templates:
        - name: exit-handler
          when: "{{workflow.status}} != Succeeded"
          container:
            image: curlimages/curl:latest
            args: ["-X","POST","-H",'Content-type: application/json',"--data",
                  '{"attachments": [{"title":"Workflow status: {{workflow.status}}","color": "danger","fields": [{"title": "name", "value": "{{workflow.name}}", "short": true }, {"title": "url", "value": "https://{{inputs.parameters.host}}/archived-workflows/{{workflow.namespace}}/?phase=Failed", "short": false }]}]}',
                  "{{inputs.parameters.webhook-url}}"]
          inputs:
            parameters:
            - name: webhook-url
              value: https://hooks.slack.com/services/xxxx
            - name: host
              value: my-argo-workflows.example.com
```

![1](/images/argo-workflows-exit-handler/1.png)

もっといい実装ありそう