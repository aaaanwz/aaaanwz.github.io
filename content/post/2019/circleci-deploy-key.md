---
title: "CircleCIでDeploy Keyを用いて別のprivate repoをcloneする"
date: 2019-11-28
draft: false
categories:
- CI/CD
- 小ネタ
---

1. `ssh-keygen`コマンドで公開鍵/秘密鍵を生成する
2. [公開鍵(id_rsa.pub)をGitHubのDeploy keyに登録する](https://developer.github.com/v3/guides/managing-deploy-keys/)
3. [秘密鍵(id_rsa)をCircleCIに登録する](https://circleci.com/docs/2.0/add-ssh-key/)
4. 3.のfingerprintを↓にコピー

```some-repo/.circleci/config.yml
version: 2.1
jobs:
  do-something-with-another-repository:
    docker:
      - image: circleci/golang:1.11-stretch
    steps:
      - add_ssh_keys:
          fingerprints:
            - "aa:bb:cc:dd:ee:ff:gg:hh:ii:jj:kk:ll:mm:nn:oo:pp"
      - run: GIT_SSH_COMMAND="ssh -o StrictHostKeyChecking=no" git clone git@github.com:aaaanwz/another-repository.git
      - run: echo 'Do something'
workflows:
  test:
    jobs:
      - do-something-with-another-repository
```
