---
title: "gcloudアカウントとプロジェクトを一発で切り替える方法"
date: 2025-07-18
draft: false
categories:
- GCP
---

複数のGCPプロジェクトや異なるアカウントを使って開発していると、`gcloud config set`を毎回実行するのは面倒です。`gcloud config configurations`を使えば、設定をまとめて切り替えることができます。

## 設定の作成

新しい設定を作成します：

```bash
# 新しい設定を作成
gcloud config configurations create my-dev-config

# 設定を切り替え
gcloud config configurations activate my-dev-config

# アカウントとプロジェクトを設定
gcloud config set account user@example.com
gcloud config set project my-dev-project
```

## 設定の一覧表示

```bash
# 設定一覧を表示
gcloud config configurations list
```

出力例：
```
NAME           IS_ACTIVE  ACCOUNT              PROJECT
default        False      old-user@gmail.com   old-project
my-dev-config  True       user@example.com     my-dev-project
```

## 設定の切り替え

```bash
# 設定を切り替える
gcloud config configurations activate default
gcloud config configurations activate my-dev-config
```

## よく使うコマンド

```bash
# 現在の設定を確認
gcloud config list

# 設定を削除
gcloud config configurations delete my-dev-config

# 設定を複製
gcloud config configurations create new-config --activate
```

## 実際の使用例

開発環境とプロダクション環境を切り替える場合：

```bash
# 開発環境用の設定
gcloud config configurations create dev
gcloud config set account dev@company.com
gcloud config set project company-dev

# プロダクション環境用の設定
gcloud config configurations create prod
gcloud config set account prod@company.com
gcloud config set project company-prod

# 切り替え
gcloud config configurations activate dev
gcloud config configurations activate prod
```

これで`gcloud config set`を何度も実行する必要がなくなり、環境切り替えが簡単になります。
