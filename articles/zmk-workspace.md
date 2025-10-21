---
title: "zmk-workspaceで簡単にZMKファームウェアをローカルビルドする"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["zmk", "keyboard"]
published: false
---

# はじめに

ZMKファームウェア(正確には、zmk-config)をビルドするにはGitHub Actionsを使用するのが一般的ですが、ビルドが遅い(2分ほどかかる)上に、zipをダウンロードして展開するのも面倒ですよね。まあダウンロードして展開するのはスクリプトで自動化できるにしても、キーマップやモジュール開発を試行錯誤する際に、いちいち2分も待っていられません。

それを解決するのがローカルビルド、つまり、自分のPC上でビルドすることです。マシンスペックにもよりますが、最短15秒程度でビルドできます。

ローカルビルドの手順は[ZMK公式のドキュメント](https://zmk.dev/docs/development/local-toolchain/setup)に記載されていますが、それに従ってやるとかなり面倒くさいです。本記事で使用する[zmk-workspace](https://github.com/kot149/zmk-workspace)は、それを簡単に、便利にできるようにしたものです。

:::details 公式手順の面倒くさいところ
公式のローカルビルド手順では、最終的に以下のようなコマンドでビルドすることになります。

```sh
west build -s app -d build/right -b seeeduino_xiao_ble -S studio-rpc-usb-uart \
    -- -DZMK_CONFIG=/workspaces/zmk-config/config -DSHIELD=roBa_R \
       -DZMK_EXTRA_MODULES=/workspaces/zmk-modules/zmk-pmw3610-driver
```

まずコマンドが長すぎてキレそうになります。(環境変数で指定もできたりしますが、本質的に手間は変わりません。) さらにパスは相対パスではなく絶対パスでないといけないという罠があり、初めて試す人はそこでハマります。

そして、`-DZMK_EXTRA_MODULES`オプションで指定している外部モジュール、これは`west.yml`に記載しているものですが、手動で`git clone`した上でビルドコマンドに記載する必要があります。GitHub Actionsでのビルドなら自動でやってくれるのに、不便ですよね。1つや2つならまだいいですが、それ以上になってくると耐えられません。

どうしてそんな面倒くさい仕様になっているかというと、GitHub Actionsでのビルドと公式手順のローカルビルドの視点の違いによるものです。これに関しては、snize氏の説明が分かりやすいです。

> - **Devcontainer**: ZMK 本体が「主」であり、そこにユーザー設定/モジュールを「追加」するイメージ。
> - **GitHub Actions**: ユーザー設定/モジュールが「主」であり、その west.yml に従って ZMK 本体などを「取得」し、自身もビルド対象に含めるイメージ。
>
> https://zenn.dev/link/comments/42b39f5bf37872

(ここでいうDevcontainerとは、公式手順でのローカルビルドのことを指しています。)

zmk-workspaceでは、このコマンドを自動で構築します。また、GitHub Actionsのアプローチと同じ方法を取ることで、`west.yml`に記載された外部モジュールを手動でcloneすることなく、自動で解決します。
:::

zmk-workspaceは、[urob氏](https://github.com/urob)の[zmk-configビルドセットアップ](https://github.com/urob/zmk-config-build)に基づいています。この場でurob氏にお礼申し上げます。

# 前提知識・環境

- 基本的なGitの操作・ターミナル操作(`cd`とか`git clone`とか)
- VSCodeがインストールされていること
- Dockerが使用可能な環境
- Windowsの場合、WSLがインストールされていること

:::message
zmk-workspaceは[Nix](https://nixos.org)での環境構築と[Dev Container](https://containers.dev)での環境構築の2つの方法をサポートしています。慣れるとNixの方が便利なのですが、Dev Containerの方が簡単・手軽に始められるので、ここではDev Containerの方で進めていきます。
:::

# 準備

1. ターミナルを開き、[zmk-workspace](https://github.com/kot149/zmk-workspace)をcloneする
   :::message alert
   Windowsの場合は、WSLの中で操作してください。ただし、WSLネイティブのディレクトリ(Windows側と同期されている`/mnt/c/`などのディレクトリ**以外**)で操作してください。Windows上のディレクトリや、WinodwsとWSL/コンテナの間で同期されているディレクトリでビルドすると、ビルドが大幅に遅くなります。
   :::
   ```sh
   git clone https://github.com/kot149/zmk-workspace.git
   ```
   ```sh
   cd zmk-workspace
   ```
1. VSCodeでzmk-workspaceを開く
   ```sh
   code .
   ```
2. VSCodeの拡張機能「[Dev Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers)」をインストールする
   - VSCodeの拡張機能の画面で`@id:ms-vscode-remote.remote-containers`で検索してインストール
3. VSCodeのフォルダをDev Containerで開き直す
   1. `F1`キーか`Ctrl+Shift+P`を押してコマンドパレットを開く
   2. `開発コンテナー: コンテナーで再度開く`(`Dev Containers: Reopen in Container`)コマンドを検索して実行する
4. VSCodeのウィンドウがリロードされ、Dev Containerが起動するので、しばらく待つ


# ビルド

以下、VSCodeでDev Containerを開いた状態で、VSCodeのターミナルから操作してください。

1. zmk-configを`config/`ディレクトリの中にcloneする
   ```sh
   cd config
   ```
   ```sh
   git clone https://github.com/kot149/zmk-config-roBa.git
   ```
   (cloneするリポジトリは自身のものに合わせて変更してください)
   ```sh
   cd ..
   ```
2. 初期化する
   ```sh
   just init config/zmk-config-roBa
   ```
   (前の手順でcloneしたzmk-configのリポジトリ名に合わせて変更してください)
3. ビルドする
   ```sh
   just build roBa_R
   ```
   (`roBa_R`はshieldの名前です。ビルドしたいshieldの名前に合わせて変更してください。)

問題なくビルドが終われば、zmk-workspaceの`firmware/`ディレクトリにビルドされたファームウェアが保存されています。


:::message
### 補足
- 別のzmk-configでビルドするには: `config/`ディレクトリの中に、別のzmk-configをcloneし、`just init`からやり直す
- モジュールを追加するには: `config/zmk-config-***/config/west.yml`にモジュールを追加した後、`just update`を実行し、`just build`する
- モジュールやZMK本体のソースコードをいじるには: zmk-workspace内にzmkやモジュールがcloneされているので、その中のモジュールのソースコードをいじる。再度ビルドすれば、いじったコードが反映される
- cloneされるモジュールを`modules/`フォルダの中にまとめるには: `west.yml`の`projects`の各項目に、`path: modules/module-name`を追加すると、`modules/module-name/`にcloneされるようになる
:::
