---
title: "自作キーボード JJ50 ビルドログ"
date: 2020-02-23
draft: false
categories:
- 自作キーボード
---

HHKB Professionalを終のキーボードにしようと思っていましたが、

- 長時間コーディングをしていると小指が痛くなる
- なんだかんだ矢印キーは欲しい

という事で、最近流行りの自作キーボードをやってみようかという運びになりました。

# パーツ購入
- `1` ~ `=` が上段に並んでいて欲しい
- 小指で押していたShift、Enter、Deleteを親指付近に移動させたい

という理由によりサイズは12列5行に決定、このサイズのPCBを探したところAliExpressで `JJ50` という製品を見つけました。  
同時にステンレス製ケース、キーキャップ、CherryMX銀軸を購入。

![jj50-1](/images/keyboard-jj50/1.jpeg)


総額は¥14800(PCB¥3700、ケース¥4100、キーキャップ¥3600、スイッチ¥3400)程度でした。

説明書なんてものは当然ありません。ドキュメントの類は[これ](https://github.com/qmk/qmk_firmware/tree/master/keyboards/jj50)が全て。

# 組み立て

- ケースにスイッチを取り付ける  
![jj50-2](/images/keyboard-jj50/2.jpeg)

- PCBを載せて半田付け  
![jj50-3](/images/keyboard-jj50/3.jpeg)

- キーキャップを取り付けてハードウェアは完成  
![jj50-4](/images/keyboard-jj50/4.jpeg)

表面実装部品は予めついていたため拍子抜けするほど簡単でした。

# ファームウェア焼き

## ビルド環境セットアップ (Mac)
```
$ git clone https://github.com/qmk/qmk_firmware
$ cd qmk_firmware
$ util/qmk_install.sh
$ make git-submodule
```

## キーマップの作成

マクロや３つ以上のレイヤーは使う予定が無いので、とてもシンプルなコードです。

```c:keyboards/jj50/keymaps/mykeyboard/keymap.c
#include QMK_KEYBOARD_H

#define ______ KC_TRNS
#define _DEFLT 0
#define _FN 1

bool process_record_user(uint16_t keycode, keyrecord_t *record) {
    return true;
};

const uint16_t PROGMEM keymaps[][MATRIX_ROWS][MATRIX_COLS] = {

    /* Default
     * ,-----------------------------------------------------------------------------------.
     * |   1  |   2  |   3  |   4  |   5  |   6  |   7  |   8  |   9  |   0  |   -  |  =   |
     * |------+------+------+------+------+------+------+------+------+------+------+------|
     * |   Q  |   W  |   E  |   R  |   T  |   Y  |   U  |   I  |   O  |   P  |   [  |   ]  |
     * |------+------+------+------+------+------+------+------+------+------+------+------|
     * |   A  |   S  |   D  |   F  |   G  |   H  |   J  |   K  |   L  |   ;  |   "  |   `  |
     * |------+------+------+------+------+------+------+------+------+------+------+------|
     * |   Z  |   X  |   C  |   V  |   B  |   N  |   M  |   ,  |   .  |   /  |  Up  |   \  |
     * |------+------+------+------+------+------+------+------+------+------+------+------|
     * |  Fn  | Ctrl |  Alt |  GUI | Space| Enter| Shift| Bksp |  Tab | Left | Down | Right|
     * `-----------------------------------------------------------------------------------'
     */
    [_DEFLT] = LAYOUT( \
        KC_1,    KC_2,    KC_3,    KC_4,    KC_5,    KC_6,    KC_7,    KC_8,    KC_9,    KC_0,    KC_MINS, KC_EQL,  \
        KC_Q,    KC_W,    KC_E,    KC_R,    KC_T,    KC_Y,    KC_U,    KC_I,    KC_O,    KC_P,    KC_LBRC, KC_RBRC, \
        KC_A,    KC_S,    KC_D,    KC_F,    KC_G,    KC_H,    KC_J,    KC_K,    KC_L,    KC_SCLN, KC_QUOT, KC_GRV,  \
        KC_Z,    KC_X,    KC_C,    KC_V,    KC_B,    KC_N,    KC_M,    KC_COMM, KC_DOT,  KC_SLSH, KC_UP  , KC_BSLS, \
        MO(_FN), KC_LCTL, KC_LALT, KC_LGUI, KC_SPC,  KC_ENT,  KC_LSFT, KC_BSPC, KC_TAB,  KC_LEFT, KC_DOWN, KC_RGHT  \
    ),

    /* Fn
     * ,-----------------------------------------------------------------------------------.
     * |  F1  |  F2  |  F3  |  F4  |  F5  |  F6  |  F7  |  F8  |  F9  |  F10 |  F11 |  F12 |
     * |------+------+------+------+------+------+------+------+------+------+------+------|
     * |      |      |  Esc |      |      |      |      |      |      |      |      |      |
     * |------+------+------+------+------+------+------+------+------+------+------+------|
     * |      |      |      |      |      |      |      |      |      |      |      |      |
     * |------+------+------+------+------+------+------+------+------+------+------+------|
     * |      |      |      |      |      |      |      |      |      |      | PgUp |      |
     * |------+------+------+------+------+------+------+------+------+------+------+------|
     * |      |      |      |      |      |      |      |Delete|      | Home | PgDn |  End |
     * `-----------------------------------------------------------------------------------'
     */
    [_FN] = LAYOUT( \
       KC_F1,   KC_F2,   KC_F3,   KC_F4,   KC_F5,   KC_F6,   KC_F7,   KC_F8,   KC_F9,   KC_F10,  KC_F11,  KC_F12,  \
       _______, _______, KC_ESC,  _______, _______, _______, _______, _______, _______, _______, _______, _______, \
       _______, _______, _______, _______, _______, _______, _______, _______, _______, _______, _______, _______, \
       _______, _______, _______, _______, _______, _______, _______, _______, _______, _______, KC_PGUP, _______, \
       _______, _______, _______, _______, _______, _______, _______, KC_DEL,  _______, KC_HOME, KC_PGDN, KC_END   \
    )
};
```

クローズドな金属ケースのため、LEDを無効にします。

```makefile:keyboards/jj50/rules.mk
...
BACKLIGHT_ENABLE = no
RGBLIGHT_ENABLE = no
...
```


## ファームウェア焼き
- __最も右上の一個下のキー(このキーボードの場合は `]`) を押しながらPCにUSB接続します。__  

```
$ make jj50:mykeyboard:flash
QMK Firmware 0.7.163
Making jj50 with keymap mykeyboard and target flash

avr-gcc (GCC) 8.3.0
Copyright (C) 2018 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.

Size before:
   text    data     bss     dec     hex filename
      0   15208       0   15208    3b68 .build/jj50_mykeyboard.hex

Compiling: tmk_core/common/command.c                                                                [OK]
Linking: .build/jj50_mykeyboard.elf                                                                    [OK]
Creating load file for flashing: .build/jj50_mykeyboard.hex                                            [OK]
Copying jj50_mykeyboard.hex to qmk_firmware folder                                                     [OK]
Checking file size of jj50_mykeyboard.hex                                                              [OK]
 * The firmware size is fine - 15208/28672 (53%, 13464 bytes free)
Warning: could not detach kernel HID driver: Function not implemented
Warning: could not detach kernel HID driver: Function not implemented
Warning: could not detach kernel HID driver: Function not implemented
Warning: could not detach kernel HID driver: Function not implemented
Warning: could not detach kernel HID driver: Function not implemented
Warning: could not detach kernel HID driver: Function not implemented
Page size   = 128 (0x80)
Device size = 32768 (0x8000); 30720 bytes remaining
Uploading 15232 (0x3b80) bytes starting at 0 (0x0)
0x00000 ... 0x00080Page size   = 128 (0x80)
Device size = 32768 (0x8000); 30720 bytes remaining
Uploading 15232 (0x3b80) bytes starting at 0 (0x0)
0x03b00 ... 0x03b80
0x03b00 ... 0x03b80
```

QMKの公式では `QMK toolbox`というツールが推奨されていますが、私の環境ではキーボードを認識しませんでした。


# 感想
- 作業時間は2時間くらい、特に問題もなく完了。
- キーマップ以前に格子配列に慣れが必要。この記事を書くのに相当時間かかった。
- JJ50は手頃な価格でサイズも過不足なくお勧め。同じKPrepublicが販売しているキーキャップも良い品質。ケースは精度・仕上げに難ありで他で買った方が良い。
