---
title: "embulkをArgo workflowsで実行するTemplate"
date: 2021-10-28
draft: false
categories:
- Kubernetes
- ETL
---

Argo Workflowsの公式ドキュメントが分かりづらかったので、試しにembulkを実行するテンプレートを作ってみました。  
`config.yml`はartifactsとして渡します。

# Dockerfile

```Dockerfile:Dockerfile
FROM openjdk:8-jre-alpine
ARG VERSION=latest

RUN mkdir -p /root/.embulk/bin \
    && wget -q https://dl.embulk.org/embulk-${VERSION}.jar -O /root/.embulk/bin/embulk \
    && chmod +x /root/.embulk/bin/embulk
ENV PATH=$PATH:/root/.embulk/bin
RUN apk add --no-cache libc6-compat

RUN embulk gem install embulk-input-s3

ENTRYPOINT ["java", "-jar", "/root/.embulk/bin/embulk"]

```

```sh
$ EMBULK_VERSION=0.9.23
$ docker build . -t embulk:$EMBULK_VERSION --build-arg VERSION=$EMBULK_VERSION
$ docker run -v /path/to/configfile:/config embulk:latest run /config/config.yml`
```

# Workflow Template

```yaml:embulk-template.yaml
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: embulk-template
spec:
  templates:
    - name: embulk
      container:
        image: embulk:latest
        args: ['run', '/input/config.yml.liquid', '-r', '/output/resume-state.yml']
        resources:
          limits:
            memory: "1Gi"
            cpu: "1"
        volumeMounts:
        - name: output
          mountPath: /output
      volumes:
        - name: output
          emptyDir: {}
      inputs:
        artifacts:
        - name: embulk-config
          path: /input/config.yml.liquid
      outputs:
        parameters:
        - name: resume-state
          valueFrom:
            path: /output/resume-state.yml
```

# Workflow

```yaml:workflow.yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  name: s3-to-stdout-sample
spec:
  entrypoint: embulk
  serviceAccountName: argo-workflow
  arguments:
    artifacts:
    - name: embulk-config
      raw:
        data: |
          in:
            type: s3
            bucket: my-bucket
            path_prefix: path/to/file
            endpoint: s3-ap-northeast-1.amazonaws.com
            access_key_id: {{ env.AWS_ACCESS_KEY_ID }}
            secret_access_key: {{ env.AWS_SECRET_ACCESS_KEY }}
          out:
            type: stdout
  workflowTemplateRef:
    name: embulk-template
```
