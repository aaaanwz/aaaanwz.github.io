---
title: "Ubuntu desktop 23.04 環境構築メモ"
date: 2023-06-20
draft: false
categories:
- 小ネタ
---

GPUを使う開発は自作PC+Linuxに限る、ということでゲーム用Windowsと開発用Ubuntuのデュアルブートにした。

## OS関係

Windowsをnvme0, Ubuntuをsata0に入れてfirst boot driveをsata0に設定、GNU GRUBでOSを選択する。  
こうしておくとブートローダーを吹っ飛ばしてもsata0を外せばきれいなWindowsが起動する。(はず)

### ハードウェアクロックをローカルタイムにする

```sh
sudo hwclock -D --systohc --localtime
```

> この設定をしないとWindows側の時刻がずれる


### 日本語ディレクトリを英語化

```sh
LANG=C xdg-user-dirs-gtk-update
```

## GPU関係

### GPUドライバのインストール

```sh
$ sudo ubuntu-drivers autoinstall
```

### CUDAセットアップ
https://developer.nvidia.com/cuda-downloads?target_os=Linux&target_arch=x86_64&Distribution=Ubuntu&target_version=22.04&target_type=deb_network

```sh
$ wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.0-1_all.deb
$ sudo dpkg -i cuda-keyring_1.0-1_all.deb
$ sudo apt update
$ sudo apt install cuda
```

`~/.bashrc`
```sh
...
export PATH="/usr/local/cuda/bin:$PATH"
export LD_LIBRARY_PATH="/usr/local/cuda/lib64:$LD_LIBRARY_PATH"
```

> 動作確認

```sh
$ nvcc -V
nvcc: NVIDIA (R) Cuda compiler driver
Copyright (c) 2005-2023 NVIDIA Corporation
Built on Mon_Apr__3_17:16:06_PDT_2023
Cuda compilation tools, release 12.1, V12.1.105
Build cuda_12.1.r12.1/compiler.32688072_0
```

```python
import torch
torch.cuda.is_available()
```

## 入力関係

### IME切り替えショートカットを `Ctrl + Space` に設定

- 設定 > キーボード > 入力ソース で入力ソースを `日本語(Mozc)` のみにする
- Mozcプロパティ > キー設定 > 編集　でIMEの有効化/無効化 を`Ctrl + Space`に設定

![1](/images/ubuntu-desktop-config/mozc.png)

### CapsLockをCtrlに変更

```sh
$ sudo vi /etc/default/keyboard
...
XKBOPTIONS="ctrl:nocaps"
```

### ターミナルからクリップボードにアクセスできるようにする

```sh
$ sudo apt install xsel
```

Macみたいにpbcopy/pbpasteで呼び出せるようにする

`~/.bashrc`
```sh
alias pbcopy='xsel --clipboard --input'
alias pbpaste='xsel --clipboard --output'
```

使い方

```sh
$ echo hoge | pbcopy
$ pbpaste
hoge
```



## 開発環境

## Docker

sudo無しで使えるようにgroupにユーザーを入れる  
https://docs.docker.com/engine/install/linux-postinstall/

```sh
$ sudo apt install docker.io
$ sudo usermod -aG docker $USER
$ newgrp docker 
```

## Brew
https://docs.brew.sh/Homebrew-on-Linux

```sh
$ /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
$ test -d ~/.linuxbrew && eval "$(~/.linuxbrew/bin/brew shellenv)"
$ test -d /home/linuxbrew/.linuxbrew && eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
$ test -r ~/.bash_profile && echo "eval \"\$($(brew --prefix)/bin/brew shellenv)\"" >>~/.bash_profile
$ echo "eval \"\$($(brew --prefix)/bin/brew shellenv)\"" >>~/.profile
```
