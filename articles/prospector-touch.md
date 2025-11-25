---
title: "Prospector ZMK Dongleのタッチ機能を使う"
emoji: "👆"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["zmk"]
published: true
---

# 背景

Prospectorは、ZMKキーボードのレイヤー情報やperipheralのバッテリー残量などをディスプレイに表示できるデバイスです。
https://github.com/carrefinho/prospector

最近、これの組み立てキット/完成品がbeekeeb.jpで手軽に入手できるようになったので、これをきっかけに筆者もProspectorを入手しました。

https://shop.beekeeb.jp/products/zmk-wireless-dongle-prospector-diy-kit


本家GitHubのREADMEを見ると、以下のように書いてあります。

> Waveshare 1.69inch Round LCD Display Module **with Touch**
> Non-touch version has different mounting pattern and will **NOT** fit

日本語訳:
> Waveshare 1.69inch Round LCD Display Module **タッチ付き**
> タッチなしのバージョンは別の取り付けパターンなので、**装着できません**

使用されていないが、ハードウェア的にはタッチ機能がついているということです。タッチができればいろいろ遊べるだろうし、底面にあるリセットボタンが押しにくいのも解決しそうです。というわけで、やってみました。

# 配線・はんだ付け

タッチ機能を使用するためにはタッチパネル用のピンの配線が必要ですが、本家のProspectorでは配線されていません。この記事では、以下の画像のように配線しました。下4本の手書きで線を追加している`TP_`で始まるピンがタッチ用のピンです。

![](https://storage.googleapis.com/zenn-user-upload/0e63e5252ded-20251109.png)

(画像は[本家の組み立てマニュアル](https://github.com/carrefinho/prospector/blob/main/docs/prospector_assembly_manual.jpg) に2025/11/09に手書きで書き加えたものです。)

:::message
**2025/11/17以降にbeekeeb.jpで購入した場合は、タッチパネル用のピンの配線がされています。** それ以前に購入した場合は、アップデート版のPCBを送ってもらえます(詳細は[こちら](https://x.com/beekeeb_jp/status/1990332493757841570))。ありがたすぎる…。
:::

https://x.com/beekeeb_jp/status/1990316922697859346


更新前のbeekeeb.jpのPCBに手動で配線することも可能です。実際に配線した写真はこんな感じになります。(こちらの写真はおぐさんからお借りしました[[リンク](https://x.com/ogu_key/status/1987696836342349878)]。ありがとうございます🙏)

![](https://storage.googleapis.com/zenn-user-upload/eaf071f36a21-20251110.png)

# ファームウェアの書き換え

今回はProspector Scannerという、Prospectorを独立したデバイスとして簡単に使用できるようにしたファームウェアを用います。

https://github.com/t-ogura/zmk-config-prospector

Prospector Scannerの使用方法については、以下の記事を参照してください。

https://note.com/heace/n/n4cbf41ef1c57

:::message
Prospector Scannerのv2.0.0以降では、タッチによるProspectorのGUI操作機能が追加されています。詳細: https://note.com/heace/n/nacf15365304c

v2.0.0以降のタッチ機能を使用する場合は、この記事の以下の変更は不要です。
:::

以下で紹介している変更は[こちらのリポジトリの`trackpad`ブランチ](https://github.com/kot149/zmk-config-prospector-scanner/tree/trackpad)に全て反映されています。

https://github.com/kot149/zmk-config-prospector-scanner/tree/trackpad

## 既存のI2Cの設定の削除

`config/boards/seeeduino_xiao_ble.overlay`にI2Cの設定がありますが、これは本家Prospectorにはないもので、使われているのかどうか・動作するのかどうかよく分からない、削除しても問題ないように見えたので、ファイルごと削除しました。

## タッチセンサーのドライバーの設定

Waveshare 1.69inch Round LCD Display Moduleは、タッチコントローラーにHynitron CST816Sを用いています。幸いなことに、[ZephyrにCST816Sのドライバーが用意されている](https://docs.zephyrproject.org/latest/build/dts/api/bindings/input/hynitron%2Ccst816s.html)ので、これを使用します。

```dts:config/prospector_scanner.overlay
&i2c0 {
   status = "okay";
    touch_sensor: cst816s@15 {
        compatible = "hynitron,cst816s";
        reg = <0x15>;
        status = "okay";
        irq-gpios = <&gpio0 2 GPIO_ACTIVE_LOW>;   // D0 = P0.02
        rst-gpios = <&gpio0 3 GPIO_ACTIVE_LOW>;   // D1 = P0.03
    };
};

/ {
    touch_listener: touch_listener {
        status = "okay";
        compatible = "zmk,input-listener";
        device = <&touch_sensor>;
    };
};
```

```Kconfig:config/prospector_scanner.conf
CONFIG_INPUT=y
CONFIG_ZMK_POINTING=y
CONFIG_INPUT_CST816S=y
CONFIG_INPUT_CST816S_INTERRUPT=y
```

:::message
2025/11/14まで、上記IRQとRSTのピンが逆になった状態で記載していました。申し訳ありません。現在は修正済みです。
:::

これで、タッチセンサーの入力が読み取られ、入力イベントが発行されます。[ドライバーのソースコード](https://github.com/zephyrproject-rtos/zephyr/blob/v3.5-branch/drivers/input/input_cst816s.c)を見ると、具体的には以下の入力イベントを発行していることがわかります[^1]。

[^1]: 最新のドライバーではこの他にもCST816Sが認識したジェスチャーをデバイス固有の入力イベントとして報告する機能が実装されていますが、2025/11/10時点でZMKが使用しているZephyr v3.5にはそれが含まれていません。

- `INPUT_BTN_TOUCH`: 画面のタッチ
- `INPUT_ABS_X`: タッチしたxの絶対座標
- `INPUT_ABS_Y`: タッチしたyの絶対座標

これらはZMKの標準機能で直接扱えないので、何かしら変換をかますか、またはそれらを直接扱えるモジュールを使用する必要があります。

## マウスカーソル移動ができるようにする

以下のモジュールに含まれている、`INPUT_ABS_X`・`INPUT_ABS_Y`(絶対座標)を`INPUT_REL_X`・`INPUT_REL_Y`(相対座標)に変換するInput Processorを用います。これによって、ZMKでマウスカーソル移動として使えるようになります。

https://github.com/halfdane/zmk-input-processors


カーソル移動している様子:

https://x.com/kot149_/status/1987391202220810277?s=20

```yml:config/west.yml
  remotes:
    - name: halfdane
      url-base: https://github.com/halfdane
  projects:
    # https://github.com/halfdane/zmk-input-processors
    - name: zmk-input-processors
      remote: halfdane
      revision: main
      path: modules/zmk-input-processors
```

```dts:config/prospector_scanner.keymap
#include <input/processors.dtsi>
#include <dt-bindings/zmk/input_transform.h>
#include <zephyr/dt-bindings/input/input-event-codes.h>

#include <behaviors/input_processor_gestures.dtsi>
#include <behaviors/input_processor_absolute_to_relative.dtsi>

&touch_listener {
    input-processors = <
        &zip_absolute_to_relative
        &zip_xy_transform (INPUT_TRANSFORM_XY_SWAP | INPUT_TRANSFORM_Y_INVERT)
        &zip_xy_scaler 4 1
    >;
};
```

Prospectorはディスプレイを90°回転して使用しているので、`&zip_xy_transform`で補正しています。また、CPIが小さかったので、`&zip_xy_scaler`で4倍にしています。
この辺については、以下の記事を参考にしてください。

https://zenn.dev/kot149/articles/zmk-input-processor-cheat-sheet

## タップでクリックする

zmk-input-gesturesモジュールを用います。

https://github.com/halfdane/zmk-input-gestures

このモジュールは、`INPUT_ABS_X`・`INPUT_ABS_Y`(絶対座標)を読み取って、タップでクリックする、カーソル移動に慣性をつけるなどのトラックパッド向けのジェスチャーを行ってくれます。

```Kconfig:config/prospector_scanner.conf
CONFIG_MAIN_STACK_SIZE=4096
CONFIG_INPUT_THREAD_STACK_SIZE=4096
```

```yml:config/west.yml
  projects:
    # https://github.com/halfdane/zmk-input-gestures
    - name: zmk-input-gestures
      remote: halfdane
      revision: main
      path: modules/zmk-input-gestures
```

```dts:config/prospector_scanner.keymap
#include <input/processors.dtsi>
#include <dt-bindings/zmk/input_transform.h>
#include <zephyr/dt-bindings/input/input-event-codes.h>

#include <behaviors/input_processor_gestures.dtsi>
#include <behaviors/input_processor_absolute_to_relative.dtsi>
#include <behaviors/input_processor_gestures.dtsi>

// Alias &touch_sensor to &glidepoint for input gestures module
glidepoint: &touch_sensor {};

&zip_gestures {
    tap-detection;
    tap-timout-ms = <150>;
    prevent_movement_during_tap;
};

&touch_listener {
    input-processors = <
        &zip_gestures
        &zip_absolute_to_relative
        &zip_xy_transform (INPUT_TRANSFORM_XY_SWAP | INPUT_TRANSFORM_Y_INVERT)
        &zip_xy_scaler 4 1
    >;
};
```

元々はCirque GlidePointというトラックパッド向けに作られたモジュールで、`&glidepoint`というラベルがデバイスについていないとビルドできませんでした。なので、`&touch_sensor`に`&glidepoint`というエイリアスをつけてあります。

これでタップするとクリックが入力されるようになります。ダブルタップも可能です。左クリック・中央クリックや、ドラッグ&ドロップはできません。

## マウスジェスチャーを導入する

zmk-mouse-gestureモジュールを用います。

https://github.com/kot149/zmk-mouse-gesture

このモジュールは、4方向のカーソル移動の組み合わせをジェスチャーとして認識し、設定したキーバインドを発行してくれます。
これを使用することで、Prospectorの押しにくい位置にあるリセットボタンの代わりに、タッチ画面のジェスチャーでbootloaderを起動するといったことができるようになります。

なお、Prospectorにはキーがないので、キーを押さなくてもジェスチャーを有効化できるようにする必要があります。このために、[オートマウスレイヤー](https://zenn.dev/kot149/articles/zmk-auto-mouse-layer)とzmk-listenersを組み合わせます。具体的には、
1. レイヤー1を追加する
2. オートマウスレイヤー(`&zip_temp_layer`)で、カーソル移動が始まったら自動的にレイヤー1を有効にし、200ms後に無効にする
3. zmk-listenersを使って、レイヤー1が有効になったときに`&mouse_gesture_on`、無効になったときに`&mouse_gesture_off`を実行する

というように設定します。

```yml:config/west.yml
  remotes:
    - name: ssbb
      url-base: https://github.com/ssbb
    - name: kot149
      url-base: https://github.com/kot149
  projects:
    # https://github.com/ssbb/zmk-listeners
    - name: zmk-listeners
      remote: ssbb
      revision: v1
      path: modules/zmk-listeners
    # https://github.com/kot149/zmk-mouse-gesture
    - name: zmk-mouse-gesture
      remote: kot149
      revision: v1
      path: modules/zmk-mouse-gesture
```

```dts:config/prospector_scanner.keymap
#include <behaviors.dtsi>
#include <dt-bindings/zmk/keys.h>
#include <dt-bindings/zmk/bt.h>
#include <dt-bindings/zmk/outputs.h>
#include <dt-bindings/zmk/pointing.h>

/ {
    keymap {
        compatible = "zmk,keymap";

        default_layer {
            bindings = <&none>;
        };

        mouse_layer {
            bindings = <&none>;
        };
    };

    layer_listeners {
        compatible = "zmk,layer-listeners";

        mouse_gesture {
            layers = <1>;
            bindings = <&mouse_gesture_on &mouse_gesture_off>;
        };
    };
};

#include <input/processors.dtsi>
#include <dt-bindings/zmk/input_transform.h>
#include <zephyr/dt-bindings/input/input-event-codes.h>

#include <behaviors/input_processor_absolute_to_relative.dtsi>

#include <mouse-gesture.dtsi>

&zip_mouse_gesture {
    stroke-size = <150>;
    enable-eager-mode;
    idle-timeout-ms = <200>;

    bootloader {
        pattern = <GESTURE_DOWN GESTURE_RIGHT GESTURE_UP>;
        bindings = <&bootloader>;
    };
};

&touch_listener {
    input-processors = <
        &zip_absolute_to_relative
        &zip_xy_transform (INPUT_TRANSFORM_XY_SWAP | INPUT_TRANSFORM_Y_INVERT)
        &zip_xy_scaler 4 1
        &zip_temp_layer 1 200
        &zip_mouse_gesture
    >;
};
```

↓, →, ↑の順でスワイプしたらbootloaderを起動するようにしました。

# 今後の展望

- 左クリック・中央クリックできるようにする
- スクロールできるようにする
- ドラッグ&ドロップできるようにする
- タップでキー入力できるようにする
- ダブルタップできるようにする
- CST816S側で実装されているジェスチャー機能の活用
- ~~タッチまたはジェスチャーでProspector ScannerのGUI操作~~
  - Prospector Scannerのv2.0.0で、タッチによるProspectorのGUI操作機能が追加されました。詳細: https://note.com/heace/n/nacf15365304c

などなど、色々やれることは多そうです。気が向いたらやってみようと思います。やったらこの記事に追記します。
