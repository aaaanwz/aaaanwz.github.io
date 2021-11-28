---
title: "ArgoCD GitOpsにおけるSecret管理"
date: 2021-11-01
draft: false
categories:
- kubernetes
---

KubernetesでGitOps運用となると必ず話題になるのがSecretの管理です。  

Sealed Secretsやkubesecなどの手元で暗号化する系  
Kubernetes Secrets Store CSI Driverやkubernetes-external-secretsなどの外部シークレットストアから引っ張ってくる系  
機密情報だけ別Repoにする  

など様々な方法がありますが、学習コストや実運用をイメージするとどのソリューションもしっくり来ませんでした。

そんな中でIBM社が開発している`ArgoCD Vault Plugin`を触ってみたところ、ArgoCDのデプロイ時にplaceholderをreplaceするという合理的かつシンプルな仕組みで非常に好感触でした。
(上記でいう「外部シークレットストアから引っ張ってくる系」の一種に該当します)
ArgoCD Vault Plugin (以下AVP) は日本語の情報が皆無に等しかったため、布教の目的も込めて導入・運用方法を記載します。

# テスト
AVPはbrewからも導入でき、手元で簡単にテストができます。
シークレットストアはAWS Secrets Mangerを使う前提で解説します。

## ローカル環境にインストール (Mac)

```sh
$ brew install argocd-vault-plugin
```

## AWS Secrets Mangerに機密文字列を登録する

```text:avp/test
key: my_secret
value: foobar
```

## Kubernetes Manifestを作成する

Secretの実装は非常に簡単で、

- アノテーションに参照するSecret Managerのパスを記述する
- Secret Managerのキー名を`<>` で囲う

だけでOKです。

```secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: credentials
  annotations:
    avp.kubernetes.io/path: "avp/test"
data:
  MY_SECRET: <my_secret | base64encode>
```

## Decryptのテスト

```shell
$ export AWS_ACCESS_KEY_ID=xxxx
$ export AWS_SECRET_ACCESS_KEY=xxxx
$ export AWS_REGION=ap-northeast-1
$ export AVP_TYPE=awssecretsmanager
$ argocd-vault-plugin generate path/to/secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: credentials
  annotations:
    avp.kubernetes.io/path: avp/test
data:
  MY_SECRET: Zm9vYmFy # == $(echo -n foobar | base64)
```

placeholderがSecrets Mangerに登録していた値に変換されました！
これで安心してGitHub repoにpushできます。

# ArgoCDへのPluginの導入

導入方法は[公式ドキュメント](https://ibm.github.io/argocd-vault-plugin/v1.4.0/installation/)の方が参考になると思うので、ここではhelmでArgoCDを導入する場合のサンプルコードを掲載します。

## 1. IAM Userの設定
AVPがSecret ManagerにアクセスするためのIAM User credentialが必要です。
このcredentialはAVPで管理できないため、どのようにAVPに渡すかは少し考える必要があります。

### 方法1. このcredentialだけは手動でapplyする

```secret-iam.yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: argocd-vault-plugin-iam
  namespace: argo
data:
  AWS_ACCESS_KEY_ID: xxxxx
  AWS_SECRET_ACCESS_KEY: xxxxx
kind: Secret
```

```sh
$ kubectl apply -f secret-iam.yaml
```

### 方法2. terraformで作成する

```terraform
resource "aws_iam_user" "avp" {
  name = "argocd-vault-plugin"
}

resource "aws_secretsmanager_secret" "avp" {
  name = "avp/test"
}

resource "aws_iam_user_policy" "avp" {
  name = "ArgocdVaultPlugin"
  user = aws_iam_user.avp.name

  policy = jsonencode({
    "Version" : "2012-10-17",
    "Statement" : [
      {
        "Effect" : "Allow",
        "Action" : [
          "secretsmanager:GetResourcePolicy",
          "secretsmanager:GetSecretValue",
          "secretsmanager:DescribeSecret",
          "secretsmanager:ListSecretVersionIds"
        ],
        "Resource" : [
          "${aws_secretsmanager_secret.avp.arn}",
        ]
      }
    ]
  })
}

resource "aws_iam_access_key" "avp" {
  user = aws_iam_user.avp.name
}

resource "kubernetes_secret" "argocd-vault" {
  metadata {
    name      = "argocd-vault-plugin-iam"
    namespace = "argo"
  }

  data = {
    "AWS_ACCESS_KEY_ID"     = aws_iam_access_key.argocd-vault.id
    "AWS_SECRET_ACCESS_KEY" = aws_iam_access_key.argocd-vault.secret
  }

  type = "Opaque"
}

```


## 2. ArgoCD with AVPのインストール

公式のインストール手順をhelmにしました。

```values.yaml
...
repoServer:
  env:
    - name: AVP_TYPE
      value: awssecretsmanager
    - name: AWS_REGION
      value: ap-northeast-1
  envFrom:
    - secretRef:
        name: argocd-vault-plugin-iam
  volumes:
    - name: custom-tools
      emptyDir: {}
  initContainers:
    - name: download-tools
      image: alpine:3.8
      command: [sh, -c]
      args:
        - >-
          wget -O argocd-vault-plugin
          https://github.com/IBM/argocd-vault-plugin/releases/download/v1.1.1/argocd-vault-plugin_1.1.1_linux_amd64 &&
          chmod +x argocd-vault-plugin &&
          mv argocd-vault-plugin /custom-tools/
      volumeMounts:
        - mountPath: /custom-tools
          name: custom-tools
  volumeMounts:
  - name: custom-tools
    mountPath: /usr/local/bin/argocd-vault-plugin
    subPath: argocd-vault-plugin
server:
  config:
    configManagementPlugins: |-
      - name: argocd-vault-plugin
        generate:
          command: ["argocd-vault-plugin"]
          args: ["generate", "./"]
      - name: argocd-vault-plugin-helm
        init:
          command: [sh, -c]
          args: ["helm dependency build"]
        generate:
          command: ["sh", "-c"]
          args: ["helm template $ARGOCD_APP_NAME ${helm_args} . | argocd-vault-plugin generate -"]
      - name: argocd-vault-plugin-kustomize
        generate:
          command: ["sh", "-c"]
          args: ["kustomize build . | argocd-vault-plugin generate -"]
...
```

```sh
$ helm repo add argo https://argoproj.github.io/argo-helm
$ helm install argo/argo-cd -f values.yaml
```

## 3. ArgoCDにApplicationを登録

アプリケーションのデプロイ時に使用するplugin設定を定義します。
例えばアプリケーションの実装がKustomizeであれば、前項で定義した `argocd-vault-plugin-kustomize`　を使う事で `kustomize build . | argocd-vault-plugin generate -` と同様の挙動になります。

```application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: testapp
  namespace: argo
spec:
  project: default
  source:
    repoURL: 'git@github.com:my-org/my-repo.git'
    path: path/to/yaml
    targetRevision: HEAD
    plugin:
      name: argocd-vault-plugin-kustomize
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

あとは[テストで作成したManifest](#Kubernetes Manifestを作成する)と同様の実装でrepoにpushすればOKです。
