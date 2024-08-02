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
- Pull Requestをオープンした再、terraform plan結果のを自動でコメントする


## 1. gcloud CLIのセットアップ

- [インストール](https://cloud.google.com/sdk/docs/install?hl=ja)
- ログイン
```sh
$ gcloud auth application-default login
```

## 2. tfファイルの配置


```
.
├── README.md
├── locals.tf
└── provider.tf
```

`locals.tf`

```terraform
locals {
  project_id = "your-project-name"
  region     = "asia-northeast1"
}
```

`provider.tf`

```
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "5.39.0"
    }
  }
  backend "gcs" {
    bucket = "tfstate-bucket-name"
  }
}

provider "google" {
  project = local.project_id
  region  = local.region
}
```


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