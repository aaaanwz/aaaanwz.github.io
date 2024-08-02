---
title: "GitHub Actionsでterraform CD/CD環境を構築する"
date: 2024-07-30
categories:
- データ基盤
---

## 概要

terraformでGoogle Cloudのリソースを管理し、GitHub ActionsによるCI/CDを構築するまでの手順をまとめます。
更に以下の要件を実現します。
- Credentialを用いず、Workload Identity連携でサービスアカウントの認証をする
- Pull Requestをオープンした際、terraform plan結果のを自動でコメントする


## 1. gcloud CLIのセットアップ

- [インストール](https://cloud.google.com/sdk/docs/install?hl=ja)
- ログイン
```sh
$ gcloud auth application-default login
```
> 作業するGoogle Cloudプロジェクトの権限を持つアカウントでログインする

## 2. GCSバケットの作成

tfstateファイルの置き場となるバケットを作成する

## 3. tfファイルの配置

以下のリソースを定義したtfファイルを配置する
- 手順2で作成したバケットのimport
- GitHub Actionsが用いるサービスアカウント
- GitHub Workload Identity連携の有効化

```
.
├── README.md
├── locals.tf
├── main.tf
├── modules
│   └── workload_identity
│       └── main.tf
└── provider.tf
```

`locals.tf`

```terraform
# 定数の定義
locals {
  project_id = "your-project-name" # UPDATE HERE
  region     = "asia-northeast1"
}
```

`provider.tf`

```terraform
provider "google" {
  project = local.project_id
  region  = local.region
}

terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "5.39.0"
    }
  }
  backend "gcs" {
    bucket = "tfstate-bucket-name" # UPDATE HERE
  }
}

# tfstate用bucketのimport

resource "google_storage_bucket" "tfstate" {
  name     = "tfstate-bucket-name" # UPDATE HERE
  location = "ASIA"
}

import {
  id = "${local.project_id}/tfstate-bucket-name" # UPDATE HERE
  to = google_storage_bucket.tfstate
}
```

`modules/workload_identity/main.tf`

```terraform
variable "project_id" {
  type = string
}

variable "github_repo_name" {
  type = string
}

data "google_project" "project" {
  project_id = var.project_id
}

# APIの有効化

resource "google_project_service" "iam" {
  project                    = data.google_project.project.project_id
  service                    = "iam.googleapis.com"
}

resource "google_project_service" "cloudresourcemanager" {
  project                    = data.google_project.project.project_id
  service                    = "cloudresourcemanager.googleapis.com"
}

# サービスアカウントの作成と権限付与

resource "google_service_account" "terraform" {
  account_id   = "terraform"
  display_name = "ServiceAccount for terraform apply"
}

resource "google_project_iam_member" "project_owner" {
  project = data.google_project.project.project_id
  role    = "roles/owner"
  member  = "serviceAccount:${google_service_account.terraform.email}"
}

# GitHub Workload Identity連携の有効化

resource "google_iam_workload_identity_pool" "github" {
  provider                  = google
  project                   = data.google_project.project.project_id
  workload_identity_pool_id = "github"
  display_name              = "github"
  description               = "GitHub Actions"
}

resource "google_iam_workload_identity_pool_provider" "github" {
  provider                           = google
  project                            = data.google_project.project.project_id
  workload_identity_pool_id          = google_iam_workload_identity_pool.github.workload_identity_pool_id
  workload_identity_pool_provider_id = "github"
  display_name                       = "github"
  description                        = "GitHub Actions"

  attribute_mapping = {
    "google.subject"       = "assertion.sub"
    "attribute.repository" = "assertion.repository"
  }

  oidc {
    issuer_uri = "https://token.actions.githubusercontent.com"
  }
}

resource "google_service_account_iam_member" "terraform" {
  service_account_id = google_service_account.terraform.id
  role               = "roles/iam.workloadIdentityUser"
  member             = "principalSet://iam.googleapis.com/${google_iam_workload_identity_pool.github.name}/attribute.repository/${var.github_repo_name}"
}
```

`main.tf`

```terraform
module "workload_identity" {
  source           = "./modules/workload_identity"
  project_id       = local.project_id
  github_repo_name = "your-org-name/repo-name" # UPDATE HERE
}
```

## 4. terraform apply

上記のリソースを手動でterraform applyする

```sh
$ terraform init
$ terraform plan
$ terraform apply
```

## 5. GitHub Actionsワークフローの追加

--- 

`.github/workflows/terraform.yml`

```yaml
name: 'Terraform'

on:
  push:
    branches:
      - main
  pull_request:

permissions:
  id-token: write
  contents: read
  pull-requests: write

jobs:
  terraform:
    runs-on: ubuntu-latest
    defaults:
      run:
        shell: bash
    steps:
    - uses: actions/checkout@v3
    - uses: 'google-github-actions/auth@v1'
      with:
        workload_identity_provider: 'projects/{Google CloudプロジェクトID}/locations/global/workloadIdentityPools/github/providers/github'
        service_account: 'terraform@{Google Cloudプロジェクト名}.iam.gserviceaccount.com'
    - uses: hashicorp/setup-terraform@v2
    - run: terraform fmt -check -recursive
      continue-on-error: true
    - run: terraform init
    - run: terraform validate
    - run: terraform plan -no-color
      id: plan
      continue-on-error: true
    - uses: peter-evans/find-comment@v3
      if: github.event_name == 'pull_request'
      id: fc
      with:
        issue-number: ${{ github.event.pull_request.number }}
        comment-author: 'github-actions[bot]'
        body-includes: Production terraform plan
    - uses: peter-evans/create-or-update-comment@v4
      if: github.event_name == 'pull_request'
      with:
        issue-number: ${{ github.event.pull_request.number }}
        comment-id: ${{ steps.fc.outputs.comment-id }}
        edit-mode: replace
        body: |
          ```
          ${{ steps.plan.outputs.stdout }}
          ```
    - run: terraform apply -auto-approve -input=false
      if: github.ref_name == 'main'
```

以上の手順により、
- Pull Request作成時にterraform planが実行され、plan結果がコメントされる
- マージ時にterraform applyが実行される
- plan, applyに用いられるサービスアカウントはWorkload Identity連携で認証される

という環境が完成します。

以上