---
title: "VSCode Remote ContainerからGitHubにssh接続する"
date: 2021-09-02
draft: false
categories:
- vscode
- 小ネタ
---

公式ドキュメントの [Sharing Git credentials with your container](https://code.visualstudio.com/docs/remote/containers#_sharing-git-credentials-with-your-container)に色々と記載があるが、非常に簡単なソリューションがあったためメモ

## Mac

```sh:
$ sudo vi ~/.ssh/config

Host github.com
    AddKeysToAgent yes
    UseKeychain yes
```

## Windows

```:PowerShell
> Set-Service ssh-agent -StartupType Automatic
> Start-Service ssh-agent
> ssh-add $HOME/.ssh/id_rsa
```
