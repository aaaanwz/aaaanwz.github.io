---
title: "Github ActionsでAWS Lambdaにデプロイする"
date: 2019-11-28
draft: false
categories:
- CI/CD
- AWS
---

GitHubでreleaseが作成された時、Lambdaにコードを反映させバージョンを更新するワークフローの単純な実装です

## サンプルディレクトリ構成
```
some-lambda-function-repo
├── .github
│   └── workflows
│       └── lambda-cd.yml
├── README.md
├── bootstrap
└── handler.sh
```

### GitHubのSecretsに以下を設定  
- `AWS_ACCESS_KEY_ID`  
- `AWS_SECRET_ACCESS_KEY`

必要なPolicyに関しては割愛します

## Github Actions

```lambda-cd.yml
name: Lambda Continuous Delivery

on:
  push:
    tags: 
      - '*'
jobs:
  lambda-cd:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@master
      - run: chmod u+x *
      - run: zip -r /tmp/some-lambda-function.zip *
      - uses: actions/setup-python@v1
        with:
          python-version: 3.7
      - run: pip3 install awscli
      - run: aws lambda update-function-code --function-name some-lambda-function --zip-file fileb:///tmp/some-lambda-function.zip --publish
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          AWS_DEFAULT_REGION: ap-northeast-1
```
