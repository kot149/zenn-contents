---
title: "ZMKのオートマウスレイヤーを極める"
emoji: "⌨️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [zmk, keyboard]
published: true
---

## オートマウスレイヤーとは

オートマウスレイヤーとは、**「マウスカーソルを動かすと、自動的に一定時間マウスレイヤーが有効になる機能」** のことです。
自動マウスレイヤー、または略してAMLと呼ばれる場合もあります。

AMLを利用することで、[Keyballシリーズ](https://shirogane-lab.net/items/64b8f8693ee3fd0045280190)のようなトラックボール付きキーボードにおいて、デフォルトレイヤーのキー数を節約しつつ、キー入力とマウス操作をスムーズに切り替えることが可能になります。

この記事では、Keyball/QMKのAMLにあるような

- 非マウスキーを押したらマウスレイヤーを抜ける
- マウスキーを押したらタイムアウトを延長する
- マウスキーを押すまでマウスレイヤーに留まる[^1]

[^1]: https://github.com/negokaz/keyball/pull/2

などの機能をZMKで設定する方法を解説します。

:::message
この記事では、トラックボール付きキーボードである[roBa](https://github.com/kumamuk-git/roBa)を例として設定例を示しますが、他のZMKを採用したキーボードにも共通して使用可能です。
ただし、設定の記載先のファイルや、ピン、ラベルの名前は異なる可能性があります。
:::

## ZMKにおけるオートマウスレイヤー

ZMKのオートマウスレイヤーには

1. [zmk-pmw3610-driver](https://github.com/inorichi/zmk-pmw3610-driver)[^2] で実装されているもの
2. [Input Processor](https://zmk.dev/docs/keymaps/input-processors) を使用したもの

の2種類が存在します。

[^2]: [zmk-pmw3610-driver](https://github.com/inorichi/zmk-pmw3610-driver)はZMKでPMW3610を使用するためのドライバーです。PMW3610はマウスセンサーで、[roBa](https://github.com/kumamuk-git/roBa)や[moNa2](https://github.com/sayu-hub/zmk-config-moNa2)といった多くのZMKトラボ付きキーボードで採用されています。zmk-pmw3610-driverにはいくつか派生(fork)があり、[badjeff氏のフォーク](https://github.com/badjeff/zmk-pmw3610-driver)などでは、Input ProcessorでAMLが可能になったので、AML機能が削除されています。

zmk-pmw3610-driverのAML機能は非常にシンプルで、QMKにあるような、

- 非マウスキーを押したらマウスレイヤーを抜ける
- マウスキーを押したらタイムアウトを延長する

という機能はありません。

Input ProcessorによるAMLは、pmw3610-driverのAMLより新しくできたもので、より汎用的かつ高機能であり、設定すれば上記の機能も使用可能になります。
今から新しく設定するなら、Input Processorを使用することを推奨します。

## 設定方法


### PMW3610 Driverのオートマウスレイヤー

`&trackball`に`automouse-layer`を設定します。
この例では、トラボを動かすとレイヤー5が有効になります。

```diff dts:roBa_R.overlay
&spi0 {
    status = "okay";
    compatible = ...

    trackball: trackball@0 {
        status = "okay";
        compatible = "pixart,pmw3610";
        ...

+       automouse-layer = <5>;
    };
};
```

タイムアウトの長さを変えるには、`CONFIG_PMW3610_AUTOMOUSE_TIMEOUT_MS`を設定します。

```kconfig:roBa_R.conf
CONFIG_PMW3610_AUTOMOUSE_TIMEOUT_MS=700
```

### Input Processorを使用したオートマウスレイヤー

`&trackball_listener`に[`zip_temp_layer`](https://zmk.dev/docs/keymaps/input-processors/temp-layer#user-defined-instances)というInput Processorを適用します。


:::message
トラックボール以外の場合は、`trackball_listener`とは違う名前がつけられている可能性があります。その場合は、`*_listener`という名前や、`compatible = "zmk,input-listener";`が指定されているものを代わりに用いてください。
:::

この例では、トラボ使用後10秒間、レイヤー5が有効になります。

```dts:roBa_R.overlay
#include <input/processors.dtsi>
/ {
    trackball_listener: trackball_listener {
        compatible = "zmk,input-listener";
        device = <&trackball>;

        input-processors = <&zip_temp_layer 5 10000>;
    };
};
```

:::message
`trackball_listener`は、`.overlay`ファイルか`.dtsi`ファイルに記載されていることが多いですが、以下のような形で`.keymap`ファイルに書かれている場合もあります。`.keyamp`に書かれている場合はそちらへ追加してください。

```dts:roBa_R.overlay
/ {
    trackball_listener: trackball_listener {
        compatible = "zmk,input-listener";
        device = <&trackball>;
    };
};
```

```dts:roBa.keymap
#include <input/processors.dtsi>

&trackball_listener {
    input-processors = <&zip_temp_layer 5 10000>;
};
```
:::

:::message alert
Input Processorを使用する場合は、zmk-pmw3610-driverのAMLは無効化しておきましょう。

```diff dts:roBa_R.overlay
&spi0 {
    status = "okay";
    compatible = ...

    trackball: trackball@0 {
        status = "okay";
        compatible = "pixart,pmw3610";
        ...

-       automouse-layer = <5>;
    };
};
```
:::

### 非マウスキーを押したらタイムアウトに関係なくマウスレイヤーを抜ける

Input Processorを使用したAMLでは、`excluded-positions`を設定すると、その位置のキー以外を押したときにAMLが解除されます。

:::message
**注意点**:
- QMKのようにマウスキーがあらかじめ定義されていないため、手動で`excluded-positions`に加える必要がある
- `excluded-positions`を設定しなかった場合、「どのキーを押してもAMLが**解除されない**」。この仕様は少し分かりづらい…
- `excluded-positions`のキーを押してもタイムアウトの延長はされないため、別途設定が必要
:::

#### 設定例

この例では、トラボ使用後10秒間、レイヤー5が有効になります。
ただし、`excluded-positions`以外のキーを押すとAMLが解除されます。ここではroBaの`J`, `K`, `;`, `Ctrl`の位置を`excluded-positions`に設定しています。

```dts:roBa_R.overlay
#include <input/processors.dtsi>

&zip_temp_layer {
    excluded-positions = <
        18 // J
        19 // K
        21 // ;
        34 // Ctrl
    >;
};

/ {
    trackball_listener: trackball_listener {
        compatible = "zmk,input-listener";
        device = <&trackball>;

        input-processors = <&zip_temp_layer 5 10000>;
    };
};
```

:::message
`excluded-positions`の数字はキーボードのレイアウト次第で変わります。
positionを知るには、[Keymap Editor](https://nickcoutsos.github.io/keymap-editor/)で適当なcomboを作って、keymapファイルのcombo定義から`key-positions`を抜き出すか、[ZMK physical layouts converter](https://zmk-physical-layout-converter.streamlit.app)にレイアウト定義JSONファイルを貼り付けて確認するのが簡単です。
:::

### マウスキーを押したらタイムアウトを延長する

`&mkp`(マウスキー押下)に対してInput Processorを設定することで、マウスキーを押したらタイムアウトを延長できます。

#### 設定例

`.keymap`ファイルに以下を追記すると、クリックしたときに10秒間AMLが延長されます。

```dts:roBa.keymap
#include <input/processors.dtsi>

&mkp_input_listener {
    input-processors = <&zip_temp_layer 5 10000>;
};
```

### マウスキーを押すまでマウスレイヤーに留まる

マウスキーを押すまでマウスレイヤーに留まる場合でも、ダブルクリックの猶予を与えるために、クリック後一定時間待つ必要があります。
そのため、実は「マウスキーを押すまでマウスレイヤーに留まる」機能は「[マウスキーを押したらタイムアウトを延長する](#マウスキーを押したらタイムアウトを延長する)」と同じ実装で、タイムアウトの長さが違うだけです。

#### 設定例

`.keymap`ファイルに以下を追記すると、クリックした後250ミリ秒後にAMLが解除されます。

```dts:roBa.keymap
#include <input/processors.dtsi>

&mkp_input_listener {
    input-processors = <&zip_temp_layer 5 250>;
};
```

また、AML自体のタイムアウトは長めにしておきましょう。

```kconfig:roBa_R.conf
CONFIG_PMW3610_AUTOMOUSE_TIMEOUT_MS=100000
```

または、Input Processorを使用している場合:

```dts:roBa_R.overlay
#include <input/processors.dtsi>
/ {
    trackball_listener: trackball_listener {
        compatible = "zmk,input-listener";
        device = <&trackball>;

        input-processors = <&zip_temp_layer 5 100000>;
    };
};
```

### キー入力中のオートマウスレイヤー誤爆防止

キー入力中、振動や意図せずトラボに触れてしまうことが原因でAMLが発動し、マウスクリックが誤爆することがあります。
これの対策として、Input Processorを使用したAMLでは、`require-prior-idle-ms`というプロパティを設定することで、キー入力から一定時間はAMLを発動しないようにできます。

#### 設定例

```dts:roBa_R.overlay
#include <input/processors.dtsi>

&zip_temp_layer {
    require-prior-idle-ms = <200>;
};

/ {
    trackball_listener: trackball_listener {
        compatible = "zmk,input-listener";
        device = <&trackball>;

        input-processors = <&zip_temp_layer 5 10000>;
    };
};
```

### Shift+クリックの動作改善

ShiftキーをZキーのMod Tap(`&mt LEFT_SHIFT Z`)にしているような場合、上記のInput Processorを使用したAMLでは、
「マウスカーソルを動かす → Shiftを押す → クリックする」
という操作によりテキストを選択しようとすると、Shiftを押した段階でAMLが解除され、クリックのつもりがキー入力になってしまいます。

必ずShiftを押した後にマウスカーソルを動かすようにすれば対処できますが、少し面倒です。
`excluded-positions`に設定する手もありますが、それだとZを入力したときにAMLが解除されず、誤爆の原因になります。

これは、タップ時のみAML解除するbehaviorを定義することで解決できます。

#### 設定方法

1. レイヤーを無効化するbehavior `tog_off`[^3]を定義する
1. AMLを解除するマクロ`exit_AML`を作る
3. キー押下後にAMLを解除するマクロ`kp_exit_AML`を作る
4. ホールド時は`kp`、タップ時は`kp_exit_AML`を使うhold-tap behavior `mt_exit_AML_on_tap`を定義する
5. `mt_exit_AML_on_tap`を`mt`の代わりに使う
6. そのキーの位置を`excluded-positions`に入れる

これによりホールド時はAMLを解除せず、タップ時のみAMLを解除するという動作が可能になります。

[^3]: [ZMKの公式ドキュメント](https://zmk.dev/docs/keymaps/behaviors/layers#toggle-layer)に記載されているものです。[これらを標準搭載しようというPR](https://github.com/zmkfirmware/zmk/pull/2943)があるので、これがマージされれば自分で定義しなくても使えるようになります。

```dts:roBa.keymap
#define MOUSE 5

/{
    macros {
        exit_AML: exit_AML {
            compatible = "zmk,behavior-macro";
            wait-ms = <0>;
            tap-ms = <0>;
            #binding-cells = <0>;
            bindings = <&tog_off MOUSE>;
            label = "exit_AML";
        };

        kp_exit_AML: kp_exit_AML {
            compatible = "zmk,behavior-macro-one-param";
            wait-ms = <0>;
            tap-ms = <0>;
            #binding-cells = <1>;
            bindings = <&macro_param_1to1 &kp MACRO_PLACEHOLDER &exit_AML>;
            label = "KP_exit_AML";
        };
    };

    behaviors {
        tog_off: toggle_layer_off {
            compatible = "zmk,behavior-toggle-layer";
            #binding-cells = <1>;
            display-name = "Toggle Layer Off";
            toggle-mode = "off";
        };

        mt_exit_AML_on_tap: mt_exit_AML_on_tap {
            compatible = "zmk,behavior-hold-tap";
            label = "MT_exit_AML_ON_TAP";
            bindings = <&kp>, <&kp_exit_AML>;

            #binding-cells = <2>;
            tapping-term-ms = <200>;
            flavor = "balanced";
            quick-tap-ms = <200>;
        };
    };
}
```

### オマケ: マクロで再現する方法

キー入力中のAML誤爆防止以外は、Input Processorを使わなくても、pmw3610-driverのAML+マクロで再現できます。

今あえてInput Processorを使わずにpmw3610-driverのAML+マクロで頑張る理由は特にありませんが、過去にInput Processorの不具合でフリーズすることがあった(現在は修正済み)ため、そういった場合の代替手段として使用できました。

ZMKは一見融通が利かないように見えて、実は大抵のことはマクロ+レイヤー移動で解決できてしまいます。ZMKのそういうところの面白さや魅力を感じていただければ幸いです。

#### 設定方法
`exit_AML`を仕込んだ`kp`、`mt`、`mo`、`lt`を作り、代わりに使います。
タイムアウトに関連する動作はSticky Layer(`sl`)を組み合わせることで作れます。

この`mt`/`lt`ではキータップ+長押しで連打が効かなくなりますが、hold時のみAML解除するものを使うと回避できます。

「マウスキーを押したらタイムアウトを延長する」「マウスキーを押すまでマウスレイヤーに留まる」には、`mkp_exit_AML`を使います。

```dts:roBa.keymap
#define MOUSE 5

/{
    macros {
        exit_AML: exit_AML {
            compatible = "zmk,behavior-macro";
            wait-ms = <0>;
            tap-ms = <0>;
            #binding-cells = <0>;
            bindings = <&tog_off MOUSE>;
            label = "exit_AML";
        };

        kp_exit_AML: kp_exit_AML {
            compatible = "zmk,behavior-macro-one-param";
            wait-ms = <0>;
            tap-ms = <0>;
            #binding-cells = <1>;
            bindings = <&macro_param_1to1 &kp MACRO_PLACEHOLDER &exit_AML>;
            label = "KP_exit_AML";
        };

        mod_exit_AML: mod_exit_AML {
            compatible = "zmk,behavior-macro-one-param";
            wait-ms = <0>;
            tap-ms = <0>;
            #binding-cells = <1>;
            bindings =
                <&macro_press>,
                <&macro_param_1to1 &kp MACRO_PLACEHOLDER>,
                <&macro_tap>,
                <&exit_AML>,
                <&macro_pause_for_release>,
                <&macro_release>,
                <&macro_param_1to1 &kp MACRO_PLACEHOLDER>;

            label = "MOD_exit_AML";
        };

        mo_exit_AML: mo_exit_AML {
            compatible = "zmk,behavior-macro-one-param";
            wait-ms = <0>;
            tap-ms = <0>;
            #binding-cells = <1>;
            bindings =
                <&macro_press>,
                <&macro_param_1to1 &mo MACRO_PLACEHOLDER>,
                <&macro_tap>,
                <&exit_AML>,
                <&macro_pause_for_release>,
                <&macro_release>,
                <&macro_param_1to1 &mo MACRO_PLACEHOLDER>,
                <&macro_tap>;

            label = "MO_exit_AML";
        };

        mkp_exit_AML: mkp_exit_AML {
            compatible = "zmk,behavior-macro-one-param";
            #binding-cells = <1>;
            bindings =
                <&macro_press>,
                <&macro_param_1to1 &mkp MACRO_PLACEHOLDER>,
                <&macro_pause_for_release>,
                <&macro_release>,
                <&macro_param_1to1 &mkp MACRO_PLACEHOLDER>,
                <&macro_tap>,
                <&sl_250 MOUSE>;

            label = "MKP_EXIT_AML";
        };
    };

    behaviors {
        sl_250: sl_250 {
            compatible = "zmk,behavior-sticky-key";
            label = "SL_250";
            bindings = <&mo>;
            #binding-cells = <1>;
            release-after-ms = <250>;
        };

        tog_off: toggle_layer_off {
            compatible = "zmk,behavior-toggle-layer";
            #binding-cells = <1>;
            display-name = "Toggle Layer Off";
            toggle-mode = "off";
        };

        lt_exit_AML: lt_exit_AML {
            compatible = "zmk,behavior-hold-tap";
            label = "LT_exit_AML";
            bindings = <&mo_exit_AML>, <&kp_exit_AML>;

            #binding-cells = <2>;
            tapping-term-ms = <200>;
            quick-tap-ms = <200>;
            flavor = "balanced";
        };

        lt_exit_AML_on_hold: lt_exit_AML_on_hold {
            compatible = "zmk,behavior-hold-tap";
            label = "LT_exit_AML_ON_HOLD";
            bindings = <&mo_exit_AML>, <&kp>;

            #binding-cells = <2>;
            tapping-term-ms = <200>;
            quick-tap-ms = <200>;
            flavor = "balanced";
        };

        mt_exit_AML: mt_exit_AML {
            compatible = "zmk,behavior-hold-tap";
            label = "MT_exit_AML";
            bindings = <&mod_exit_AML>, <&kp_exit_AML>;

            #binding-cells = <2>;
            tapping-term-ms = <200>;
            flavor = "balanced";
            quick-tap-ms = <200>;
        };

        mt_exit_AML_on_tap: mt_exit_AML_on_tap {
            compatible = "zmk,behavior-hold-tap";
            label = "MT_exit_AML_ON_TAP";
            bindings = <&kp>, <&kp_exit_AML>;

            #binding-cells = <2>;
            tapping-term-ms = <200>;
            flavor = "balanced";
            quick-tap-ms = <200>;
        };
    };
}
```

----
この記事はroBaで書きました。