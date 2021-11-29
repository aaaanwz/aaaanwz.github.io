---
title: "Ubuntu desktop初期設定メモ"
date: 2021-11-21
draft: false
categories:
- 小ネタ
---

M1 Macに移行する気になれなかったのでメインマシンをx86 + Ubuntu Desktop 20.04にしました。  

## IME切り替えショートカットを `Ctrl + Space` に設定

- 設定 > 地域と言語 > 入力ソース で入力ソースを `日本語(Mozc)` のみにする
- Mozcプロパティ > キー設定 > 編集　で 入力キーが`Ctrl Space`のエントリーを削除し、以下のように設定

![1](/images/ubuntu-desktop-config/mozc.png)

## CapsLockをCtrlに変更

```sh
$ sudo vi /etc/default/keyboard
...
XKBOPTIONS="ctrl:nocaps"
```

## Chromeをダークモード化

```sh
$ sudo vi /opt/google/chrome/google-chrome
...
exec -a "$0" "$HERE/chrome" "--enable-features=WebUIDarkMode" "--force-dark-mode" "$@"
```

## 日本語ディレクトリを変更

```sh
$ sudo vi .config/user-dirs.dirs

XDG_DESKTOP_DIR="$HOME/Desktop"
XDG_DOWNLOAD_DIR="$HOME/Downloads"
XDG_TEMPLATES_DIR="$HOME/Templates"
XDG_PUBLICSHARE_DIR="$HOME/Share"
XDG_DOCUMENTS_DIR="$HOME/Documents"
XDG_MUSIC_DIR="$HOME/Music"
XDG_PICTURES_DIR="$HOME/Pictures"
XDG_VIDEOS_DIR="$HOME/Videos"
```

## デスクトップとファイルマネージャの間でファイルの移動ができるようにする

### 元々のDesktop Iconsを無効化
デスクトップを右クリック -> 設定
- デスクトップにホームフォルダーを表示する
- デスクトップにゴミ箱を表示する
をオフにする

> Gnome tweaksからも設定可能

### GNOME拡張のDesktop Icons NG (DING)をインストール
https://extensions.gnome.org/extension/2087/desktop-icons-ng-ding/

## Docker

sudo無しで使えるようにgroupにユーザーを入れる  
https://docs.docker.com/engine/install/linux-postinstall/

```sh
$ sudo snap install docker
$ sudo groupadd docker
$ sudo usermod -aG docker $USER
$ newgrp docker 
```
## Wine

https://wiki.winehq.org/Ubuntu

```sh
$ sudo dpkg --add-architecture i386
$ wget -nc https://dl.winehq.org/wine-builds/winehq.key
$ sudo apt-key add winehq.key
$ sudo add-apt-repository 'deb https://dl.winehq.org/wine-builds/ubuntu/ focal main'
$ sudo apt update
$ sudo apt install --install-recommends winehq-stable
$ winecfg
```

文字化け対策

```sh
$ sudo apt-get install winetricks
$ winetricks fakejapanese
$ winetricks corefonts
$ winetricks cjkfonts
```

## VSCode

snapで入れるとKubernetes拡張機能が動作しないなど不具合が多いので、公式から.debで入れたほうがいい

## Slack

snapで入れるとIME切り替えができない
