---
title: "ZMK Input Processorチートシート"
emoji: "🖱️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["zmk", "keyboard"]
published: true
---

:::message
この記事の情報はZMK v0.3時点のものです。
今後のアップデートによって設定方法が変更される可能性があります。
:::

# Input Processorとは

[公式ドキュメントへのリンク](https://zmk.dev/docs/keymaps/input-processors)

[ZMK](https://zmk.dev)のInput Processorは、カーソル移動やスクロールなどのマウス入力(Input)イベント[^1]を受け取り、それに基づいてさまざま処理を実行する機能。

さまざな処理とは、例えば
- カーソルの移動やスクロールの量に倍率をかける(CPIを大きくしたり、小さくしたりする)
- カーソル移動やスクロールの方向を変更する(XとYを入れ替えたり、x方向またはy方向を反転する)
- カーソル移動をスクロールに変換する
- カーソルが移動したら一定時間の間特定のレイヤーを有効化する(オートマウスレイヤー)

などが挙げられる。これ以外にも、目次に並んでいるような様々な処理が可能である。

[^1]: 入力イベントは、ZMKのベースであるZephyrで提供されているもので、[こちら](https://docs.zephyrproject.org/apidoc/latest/group__input__events.html)にその一覧が記載されている。この中にはマウス関連のイベントの他にキー入力イベントなどもあるが、ZMK(v0.3時点)ではマウス関連のイベントのみが処理可能。

Input Processorの利点はその汎用性にある。ZMKのinputシステムを使用して実装されているマウス入力であれば、センサー(ドライバー)が異なっても、Input Processorを使うことで同じ機能を使用できる。従来はセンサーのドライバーごとに機能を実装していたが、その手間を省くことができ、メンテナンス性の向上も期待できる。
Input Processorという名前を見て、なんだか抽象的な名前でとっつきにくいと思ったかもしれない(筆者は思った)が、この名前はその汎用性を表していると言える。

# 基本的な設定方法

[公式ドキュメントへのリンク](https://zmk.dev/docs/keymaps/input-processors/usage#base-processors)

例えば、`&zip_xy_scaler 3 2`と`&zip_temp_layer 5 10000`という2つのInput Processorを使用するには、Input Listener(例えば`trackball_listener`)に対して以下のように記述する。

```dts
#include <input/processors.dtsi>

&trackball_listener {
    input-processors = <
        &zip_xy_scaler 3 2
        &zip_temp_layer 5 10000
    >;
};
```

:::message
#### Input Listenerの見つけ方

トラックボールではない場合(トラックパッド等)や、分割型キーボードの左右で名前を分けている場合などは、`trackball_listener`とは名前が異なる。
その場合、`.overlay`ファイルや`.dtsi`ファイルの中で、`〇〇_listener`という名前や`compatible = "zmk,input-listener";`が記載されているノードを探し、代わりにそれを使用する。

```dts
/ {
    trackball_listener: trackball_listener {
        compatible = "zmk,input-listener";
        status = "okay";
        device = <&trackball>;
    };
};
```

なお、`trackball_listener: trackball_listener`のように`label: name`の形になっていない場合は参照できずビルドに失敗するので、ラベルをつけておくこと。

```diff:dts
/ {
-   trackball_listener {
+   trackball_listener: trackball_listener {
        compatible = "zmk,input-listener";
        status = "okay";
        device = <&trackball>;
    };
};
```
:::

:::details マウスキーに対してInput Processorを適用する方法
トラックボールなどの入力デバイスの他に、キーによるマウスクリック(`&mkp`)、カーソル移動(`&mmv`)、スクロール(`&msc`)に対するInput Listenerも用意されている。それぞれ、`&mkp_input_listener`、`&mmv_input_listener`、`&msc_input_listener`という名前で参照できる。
:::


:::details Input Processorをどこに記述するべきか？

一般的に、以下のような棲み分けが推奨される。

- **`.dtsi`または`.overlay`ファイル**: ユーザーが通常変更しない、ハードウェア構成に関係したものを記述
- **`.keymap`ファイル**: ユーザーが変更し得る、ユーザーの好み次第で設定が変わるものを記述

input processorはユーザーが変更し得る設定が多いため、`.keymap`ファイルに記述するのが適切と考えることができる。が、考え方や好みによるところではあるため、必ずしもこれが正解とは限らない。

なお、複数箇所にinput processorを記述すると、devicetreeの仕様上、どれか1つだけが有効となり、他は無効となるため、複数箇所に記述しないように要注意。
:::

# レイヤー別のInput Processor

[公式ドキュメントへのリンク](https://zmk.dev/docs/keymaps/input-processors/usage#layer-specific-overrides)

以下の例では、レイヤー3でのみ`&zip_xy_scaler 1 3`が、レイヤー4と5でのみ`&zip_xy_scaler 3 2`が適用される。

```dts
#include <input/processors.dtsi>

&trackball_listener {
    input-processors = <&zip_xy_scaler 3 2>;

    snipe {
        layers = <3>;
        input-processors = <&zip_xy_scaler 1 3>;
    };

    scroller {
        layers = <4 5>;
        input-processors = <&zip_xy_to_scroll_mapper>;
    };
};
```

`snipe`や`scroller`という名前はなんでも良く、自分に分かりやすい名前で自由に設定できる。

:::details 同じレイヤーに複数のレイヤー別Input Processorを適用する場合
通常、レイヤー別のInput Processorは最初にマッチしたものだけで処理を終了し、他はスキップされる。
スキップしないようにするには、`process-next;`を設定する。

```dts
#include <input/processors.dtsi>

&trackball_listener {
    input-processors = <&zip_xy_scaler 3 2>;

    scroller {
        layers = <4 5>;
        input-processors = <&zip_xy_to_scroll_mapper>;
        process-next;
    };

    fast_scroller {
        layers = <5>;
        input-processors = <&zip_scroll_scaler 3 1>;
    };
};
```
:::

# カーソル移動関連

## カーソル移動量に倍率をかける(カーソル移動速度を大きくする/小さくする)

[公式ドキュメントへのリンク](https://zmk.dev/docs/keymaps/input-processors/scaler)

`&zip_xy_scaler`を使用する。
`&zip_xy_scaler`はパラメーターを2つ受け取り、1つ目が分子、2つ目が分母となる。
例えば、`&zip_xy_scaler 3 2`なら、CPIが1.5倍 (3/2倍)になり、`&zip_xy_scaler 1 3`とすれば0.333倍(1/3倍)になる。

```dts
#include <input/processors.dtsi>

&trackball_listener {
    input-processors = <&zip_xy_scaler 3 2>;
};
```

### x方向/y方向の片方のみに倍率をかける
x方向のみスケールするには、`&zip_x_scaler`を、y方向のみスケールするには`&zip_y_scaler`を`&zip_xy_scaler`の代わりに用いる。

## カーソル移動のXとYを入れ替える
[公式ドキュメントへのリンク](https://zmk.dev/docs/keymaps/input-processors/transformer)

`&zip_xy_transform INPUT_TRANSFORM_XY_SWAP`を用いる。
`#include <dt-bindings/zmk/input_transform.h>`が追加で必要なことに注意。

```dts
#include <input/processors.dtsi>
#include <dt-bindings/zmk/input_transform.h>

&trackball_listener {
    input-processors = <&zip_xy_transform INPUT_TRANSFORM_XY_SWAP>;
};
```

## カーソル移動のX方向を反転する

`&zip_xy_transform INPUT_TRANSFORM_X_INVERT`を用いる。
`#include <dt-bindings/zmk/input_transform.h>`が追加で必要なことに注意。

```dts
#include <input/processors.dtsi>
#include <dt-bindings/zmk/input_transform.h>

&trackball_listener {
    input-processors = <&zip_xy_transform INPUT_TRANSFORM_X_INVERT>;
};
```

## カーソル移動のY方向を反転する

`&zip_xy_transform INPUT_TRANSFORM_Y_INVERT`を用いる。
`#include <dt-bindings/zmk/input_transform.h>`が追加で必要なことに注意。

```dts
#include <input/processors.dtsi>
#include <dt-bindings/zmk/input_transform.h>

&trackball_listener {
    input-processors = <&zip_xy_transform INPUT_TRANSFORM_Y_INVERT>;
};
```

## カーソル移動の方向変更を組み合わせる

例えば、
- `&zip_xy_transform INPUT_TRANSFORM_XY_SWAP`
- `&zip_xy_transform INPUT_TRANSFORM_X_INVERT`

の2つを組み合わせるには、
`&zip_xy_transform (INPUT_TRANSFORM_XY_SWAP | INPUT_TRANSFORM_X_INVERT)`
のように記述する。

`#include <dt-bindings/zmk/input_transform.h>`が追加で必要なことに注意。

```dts
#include <input/processors.dtsi>
#include <dt-bindings/zmk/input_transform.h>

&trackball_listener {
    input-processors = <&zip_xy_transform (INPUT_TRANSFORM_XY_SWAP | INPUT_TRANSFORM_X_INVERT)>;
};
```

# スクロール関連

## カーソル移動をスクロールに変換する

[公式ドキュメントへのリンク](https://zmk.dev/docs/keymaps/input-processors/code-mapper)

`&zip_xy_to_scroll_mapper`を使用する。

```dts
#include <input/processors.dtsi>

&trackball_listener {
    input-processors = <&zip_xy_to_scroll_mapper>;
};
```

通常、特定のレイヤーでのみ適用することで、通常はカーソル移動で、特定レイヤーが有効な間のみスクロールにするという使い方をする。

```dts
#include <input/processors.dtsi>

&trackball_listener {
    scroller {
        layers = <4>;
        input-processors = <&zip_xy_to_scroll_mapper>;
    };
};
```

## スクロール量に倍率をかける(スクロール速度を大きくする/小さくする)

[公式ドキュメントへのリンク](https://zmk.dev/docs/keymaps/input-processors/scaler)

`&zip_scroll_scaler`を使用する。
`&zip_scroll_scaler`はパラメーターを2つ受け取り、1つ目が分子、2つ目が分母となる。
例えば、`&zip_scroll_scaler 3 2`なら、スクロール量が1.5倍 (3/2倍)になり、`&zip_scroll_scaler 1 3`とすれば0.333倍(1/3倍)になる。

```dts
#include <input/processors.dtsi>

&trackball_listener {
    scroller {
        layers = <4>;
        input-processors = <
            &zip_xy_to_scroll_mapper
            &zip_scroll_scaler 3 2
        >;
    };
};
```

### x方向/y方向の片方のみに倍率をかける
x方向またはy方向のみスケールするには、以下の`&zip_scroll_x_scaler`または`&zip_scroll_y_scaler`を定義して、`&zip_scroll_scaler`の代わりに用いる。
`#include <zephyr/dt-bindings/input/input-event-codes.h>`が追加で必要なことに注意。

```dts
#include <input/processors.dtsi>
#include <zephyr/dt-bindings/input/input-event-codes.h>

/ {
    input_processors {
        zip_scroll_x_scaler: zip_scroll_x_scaler {
            compatible = "zmk,input-processor-scaler";
            #input-processor-cells = <2>;
            type = <INPUT_EV_REL>;
            codes = <INPUT_REL_HWHEEL>;
            track-remainders;
        };

        zip_scroll_y_scaler: zip_scroll_y_scaler {
            compatible = "zmk,input-processor-scaler";
            #input-processor-cells = <2>;
            type = <INPUT_EV_REL>;
            codes = <INPUT_REL_WHEEL>;
            track-remainders;
        };
    };
};

&trackball_listener {
    scroller {
        layers = <4>;
        input-processors = <
            &zip_xy_to_scroll_mapper
            &zip_scroll_x_scaler 3 2
        >;
    };
};
```

## スクロールを縦方向に限定する

`&zip_scroll_x_scaler`でx方向のスクロールを0倍にすれば、縦方向のスクロールのみになる。

`#include <zephyr/dt-bindings/input/input-event-codes.h>`が追加で必要なことに注意。

```dts
#include <input/processors.dtsi>
#include <zephyr/dt-bindings/input/input-event-codes.h>

/ {
    input_processors {
        zip_scroll_x_scaler: zip_scroll_x_scaler {
            compatible = "zmk,input-processor-scaler";
            #input-processor-cells = <2>;
            type = <INPUT_EV_REL>;
            codes = <INPUT_REL_HWHEEL>;
            track-remainders;
        };
    };
};

&trackball_listener {
    scroller {
        layers = <4>;
        input-processors = <
            &zip_xy_to_scroll_mapper
            &zip_scroll_x_scaler 0 1
        >;
    };
};
```

## スクロールのXとYを入れ替える
[公式ドキュメントへのリンク](https://zmk.dev/docs/keymaps/input-processors/transformer)

`&zip_scroll_transform INPUT_TRANSFORM_XY_SWAP`を用いる。
`#include <dt-bindings/zmk/input_transform.h>`が追加で必要なことに注意。

```dts
#include <input/processors.dtsi>
#include <dt-bindings/zmk/input_transform.h>

&trackball_listener {
    scroller {
        layers = <4>;
        input-processors = <
            &zip_xy_to_scroll_mapper
            &zip_scroll_transform INPUT_TRANSFORM_XY_SWAP
        >;
    };
};
```

## スクロールのX方向を反転する

`&zip_scroll_transform INPUT_TRANSFORM_X_INVERT`を用いる。
`#include <dt-bindings/zmk/input_transform.h>`が追加で必要なことに注意。

```dts
#include <input/processors.dtsi>
#include <dt-bindings/zmk/input_transform.h>

&trackball_listener {
    scroller {
        layers = <4>;
        input-processors = <
            &zip_xy_to_scroll_mapper
            &zip_scroll_transform INPUT_TRANSFORM_X_INVERT
        >;
    };
};
```

## スクロールのY方向を反転する

`&zip_scroll_transform INPUT_TRANSFORM_Y_INVERT`を用いる。
`#include <dt-bindings/zmk/input_transform.h>`が追加で必要なことに注意。

```dts
#include <input/processors.dtsi>
#include <dt-bindings/zmk/input_transform.h>

&trackball_listener {
    scroller {
        layers = <4>;
        input-processors = <
            &zip_xy_to_scroll_mapper
            &zip_scroll_transform INPUT_TRANSFORM_Y_INVERT
        >;
    };
};
```

## スクロールの方向変更を組み合わせる

例えば、
- `&zip_scroll_transform INPUT_TRANSFORM_XY_SWAP`
- `&zip_scroll_transform INPUT_TRANSFORM_X_INVERT`

の2つを組み合わせるには、
`&zip_scroll_transform (INPUT_TRANSFORM_XY_SWAP | INPUT_TRANSFORM_X_INVERT)`
のように記述する。

`#include <dt-bindings/zmk/input_transform.h>`が追加で必要なことに注意。

```dts
#include <input/processors.dtsi>
#include <dt-bindings/zmk/input_transform.h>

&trackball_listener {
    scroller {
        layers = <4>;
        input-processors = <
            &zip_xy_to_scroll_mapper
            &zip_scroll_transform (INPUT_TRANSFORM_XY_SWAP | INPUT_TRANSFORM_X_INVERT)
        >;
    };
};
```

# その他

## カーソル移動後一定時間、特定のレイヤーを有効にする(オートマウスレイヤー)

[公式ドキュメントへのリンク](https://zmk.dev/docs/keymaps/input-processors/temp-layer)

以下の記事で詳しく説明しているため、そちらを参照。

https://zenn.dev/kot149/articles/zmk-auto-mouse-layer

## マウスクリックをキー入力に変換する

下記の`&zip_button_behaviors`は、左クリックをAキーに、右クリックをBキーに、中央クリックをCキーに変換する。

:::message
この設定は、クリックが可能なトラックパッドなどでのみ有効。
トラックボールの場合はマウスクリックができないため、この設定は無意味。
:::

```dts
#include <input/processors.dtsi>

&zip_button_behaviors {
    bindings = <&kp A &kp B &kp C>;
};

&trackpad_listener {
    input-processors = <&zip_button_behaviors>;
};
```

# 外部モジュールのInput Processor

ここまでで紹介したInput ProcessorはZMK本体に含まれている公式のInput Processorだが、それ以外に、外部モジュールでもInput Processorを実装可能である。
ここでは設定方法については詳しくは触れないが、そのいくつか(特に、筆者が使用したことのあるもの)を紹介する。

なお、ここで紹介しているものが全てではない。自分の欲しい機能がこの中にない場合は、GitHubやZMKのDiscordサーバーで検索してみるとよい。また、プログラミングができる人なら自分で新たにモジュールとして作成するという手もある。

## カーソル移動をキー入力に変換する

カーソル移動を矢印キーなどに変換するInput Processorを提供するモジュール。

https://github.com/te9no/zmk-input-processor-keybind

https://github.com/zettaface/zmk-input-processor-keybind

## マウスジェスチャー

4方向のマウスカーソル移動で描いたジェスチャーを元に、設定したキーバインドを発行するInput Processorを提供するモジュール。

https://github.com/kot149/zmk-mouse-gesture

## トラックパッドジェスチャー

タップでクリック、カーソル移動に慣性をつける、トラックパッドの外周をなぞってスクロールするなど、トラックパッド固有のジェスチャーを提供するInput Processorモジュール。

https://github.com/halfdane/zmk-input-gestures


## スクロール方向を自動で縦方向/横方向にロックする

360°のスクロールを、x軸またはy軸の近い方へ自動でスナップするInput Processorを提供するモジュール。

https://github.com/kot149/zmk-scroll-snap

## カーソル移動を360°任意の角度傾ける

カーソル移動に360°任意(正確には5°区切り)の角度をつけるInput Processorを提供するモジュール。
センサーを傾けて取り付けている場合やキーボードを傾けて使用している場合に、補正するために使う。
([ZMKのロータリエンコーダ用の機能](https://zmk.dev/docs/keymaps/behaviors/sensor-rotate)もSensor Rotationという名前なのでややこしい…)

https://github.com/hsgw/zmk-feature-sensor_rotation

## 入力イベントのレポート頻度を制限する

入力イベントのレポート頻度を制限するInput Processorを提供するモジュール。
必要以上に高い頻度で報告することは、バッテリー消費を増やすだけでなく、無線通信において遅延が発生する原因になる。このInput Processorを用いることでそれを回避できる。

https://github.com/badjeff/zmk-input-processor-report-rate-limit
