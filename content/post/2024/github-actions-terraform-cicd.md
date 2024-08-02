---
title: "GitHub Actionsでterraform CD/CD環境を構築する"
date: 2024-07-30
categories:
- データ基盤
---

## 概要

terraformでGoogle Cloudのリソースを管理し、GitHub ActionsによるCI/CDを構築するまでの手順をまとめます。
更に以下の要件を実現します。
- OIDCによる認証
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

以下のリソースを定義したファイルを配置します
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

resource "google_service_account" "terraform" {
  account_id   = "terraform"
  display_name = "ServiceAccount for terraform apply"
}

resource "google_project_iam_member" "project_owner" {
  project = data.google_project.project.project_id
  role    = "roles/owner"
  member  = "serviceAccount:${google_service_account.terraform.email}"
}

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

## 4. 

## 5. 

--- 

.github/workflows/terraform.yml
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
        working-directory: staging
    steps:
    - uses: actions/checkout@v3
    - uses: 'google-github-actions/auth@v1'
      with:
        workload_identity_provider: 'projects/840789314875/locations/global/workloadIdentityPools/github/providers/github'
        service_account: 'terraform@stg-data-platform.iam.gserviceaccount.com'
    - uses: hashicorp/setup-terraform@v2
    - run: terraform fmt -check -recursive
      continue-on-error: true
    - run: terraform init
    - run: terraform validate
    - run: terraform plan -no-color
      id: plan
      continue-on-error: true
    - name: Find Comment
      if: github.event_name == 'pull_request'
      uses: peter-evans/find-comment@v3
      id: fc
      with:
        issue-number: ${{ github.event.pull_request.number }}
        comment-author: 'github-actions[bot]'
        body-includes: Staging terraform plan
    - name: Comment Plan Result on PR
      if: github.event_name == 'pull_request'
      uses: peter-evans/create-or-update-comment@v4
      with:
        issue-number: ${{ github.event.pull_request.number }}
        comment-id: ${{ steps.fc.outputs.comment-id }}
        edit-mode: replace
        body: |
          **Staging terraform plan**
          ```
          ${{ steps.plan.outputs.stdout }}
          ```
    - name: terraform apply
      if: github.ref_name == 'main'
      run: terraform apply -auto-approve -input=false

  production:
    runs-on: ubuntu-latest
    defaults:
      run:
        shell: bash
        working-directory: production
    steps:
    - uses: actions/checkout@v3
    - uses: 'google-github-actions/auth@v1'
      with:
        workload_identity_provider: 'projects/914298793810/locations/global/workloadIdentityPools/github/providers/github'
        service_account: 'terraform@helpfeel-data-warehouse.iam.gserviceaccount.com'
    - uses: hashicorp/setup-terraform@v2
    - run: terraform fmt -check -recursive
      continue-on-error: true
    - run: terraform init
    - run: terraform validate
    - run: terraform plan -no-color
      id: plan
      continue-on-error: true
    - name: Find Comment
      if: github.event_name == 'pull_request'
      uses: peter-evans/find-comment@v3
      id: fc
      with:
        issue-number: ${{ github.event.pull_request.number }}
        comment-author: 'github-actions[bot]'
        body-includes: Production terraform plan
    - name: Comment Plan Result on PR
      if: github.event_name == 'pull_request'
      uses: peter-evans/create-or-update-comment@v4
      with:
        issue-number: ${{ github.event.pull_request.number }}
        comment-id: ${{ steps.fc.outputs.comment-id }}
        edit-mode: replace
        body: |
          **Production terraform plan**
          ```
          ${{ steps.plan.outputs.stdout }}
          ```
    - name: terraform apply
      if: github.ref_name == 'main'
      run: terraform apply -auto-approve -input=false

```