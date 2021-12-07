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

### GNOME拡張のDesktop Icons NG (DING)をインストール
https://extensions.gnome.org/extension/2087/desktop-icons-ng-ding/

### 元々のDesktop Iconsを無効化
デスクトップアイコンが二重に表示されるので、元々の機能を無効化する

```sh
$ sudo apt install gnome-shell-extension-prefs
```

メニュー > 拡張機能 でDesktop Iconsをオフにする

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
$ sudo apt install wine-development
$ winecfg
```

文字化け対策

```sh
$ sudo apt-get install winetricks
$ winetricks fakejapanese
$ winetricks corefonts
$ winetricks cjkfonts
```

### 3D関係

```sh
$ sudo apt install vulkan-utils dxvk
```

以下のファイルを作成する

```json
$ vi ~/.wine/windows/winevulkan.json

{
    "file_format_version": "1.0.0",
    "ICD": {
        "library_path": "c:\\windows\\system32\\winevulkan.dll",
        "api_version": "1.0.51"
    }
}
```

```sh
$ dxvk-setup i
```

## VSCode

snapで入れるとKubernetes extensionが動作しないため、公式の.debパッケージを入れる

## Slack

snapで入れると日本語入力ができないため、公式の.devパッケージを入れる
https://slack.com/intl/ja-jp/downloads/linux

---

GUIアプリはsnap版に不具合があるケースが多い模様。

随時追記予定
