---
title: "zmk-workspaceで簡単にZMKファームウェアをローカルビルドする"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["zmk", "keyboard"]
published: true
---

# ローカルビルドについて

[ZMKファームウェア](https://zmk.dev)(正確には、zmk-config)をビルドするにはGitHub Actionsを使用するのが一般的ですが、ビルドが遅い(2分ほどかかる)上に、zipをダウンロードして展開するのも面倒ですよね。ダウンロードして展開するのはスクリプトで自動化できるにしても、キーマップやモジュール開発で試行錯誤する際に、いちいち2分も待っていられません。

それを解決するのがローカルビルド、つまり、自分のPC上でビルドすることです。マシンスペックにもよりますが、筆者の環境だと15秒程度でビルドできます。

ローカルビルドの手順は[ZMK公式のドキュメント](https://zmk.dev/docs/development/local-toolchain/setup)に記載されていますが、それに従ってやるとかなり面倒くさいです。本記事で使用する[zmk-workspace](https://github.com/kot149/zmk-workspace)は、それを簡単に、便利にできるようにしたものです。

:::details 公式手順の面倒くさいところ
公式のローカルビルド手順では、最終的に以下のようなコマンドでビルドすることになります。

```sh
west build -s app -d build/right -b seeeduino_xiao_ble -S studio-rpc-usb-uart \
    -- -DZMK_CONFIG=/workspaces/zmk-config/config -DSHIELD=roBa_R \
       -DZMK_EXTRA_MODULES=/workspaces/zmk-modules/zmk-pmw3610-driver
```

まずコマンドが長すぎてキレそうになります。(環境変数で指定もできたり、2回目以降のビルドでは省略できたりしますが、最初の手間は変わりません。) パスは相対パスではなく絶対パスでないといけないという罠があり、初めて試す人はそこでハマります。

そして、`-DZMK_EXTRA_MODULES`オプションで指定している外部モジュール、これは`west.yml`に記載しているものですが、手動で`git clone`した上でビルドコマンドに記載する必要があります。GitHub Actionsでのビルドなら自動でやってくれるのに、不便ですよね。1つや2つならまだいいですが、それ以上になってくると耐えられません。

どうしてそんな面倒くさい仕様になっているかというと、GitHub Actionsでのビルドと公式手順のローカルビルドの視点の違いが理由です。これに関しては、snize氏の説明が分かりやすいです。

> - **Devcontainer**: ZMK 本体が「主」であり、そこにユーザー設定/モジュールを「追加」するイメージ。
> - **GitHub Actions**: ユーザー設定/モジュールが「主」であり、その west.yml に従って ZMK 本体などを「取得」し、自身もビルド対象に含めるイメージ。
>
> https://zenn.dev/link/comments/42b39f5bf37872

(ここでいうDevcontainerとは、公式手順でのローカルビルドのことを指しています。)

zmk-workspaceでは、このコマンドを自動で構築します。また、GitHub Actionsのアプローチと同じ方法を取ることで、`west.yml`に記載された外部モジュールを手動でcloneすることなく、自動で解決します。
:::

zmk-workspaceは、[urob氏](https://github.com/urob)の[zmk-configビルドセットアップ](https://github.com/urob/zmk-config)に基づいています。この場でurob氏にお礼申し上げます。

# 前提知識・環境

- 基本的なGitの操作・ターミナル操作(`cd`とか`git clone`とか)
- Windowsの場合、WSLがインストールされていること

# セットアップ

zmk-workspaceはセットアップの方法として[Nix](https://nixos.org)を使用する方法と[Dev Container](https://containers.dev)を使用する方法の2つをサポートしています。どちらかを選択して進めてください。

Winodws/macOSの場合、Nixならuf2ファイルの書き込みを自動化できるようにしている[^1]ので、Nixの方がおすすめです。

[^1]: Dev Containerでもコンテナの外から実行して書き込みを自動化するスクリプトを作ればできますが、Nixを使用した方が楽です。

## Nixによるセットアップ

:::message
Dev Containerを使用する場合は、この項目は飛ばして、[Dev Containerによるセットアップ](#dev-containerによるセットアップ)へ進んでください。
:::

1. https://nixos.org/download/ に従い、Nixをインストールする
   :::message alert
   Windowsの場合は、WSLの中で操作してください。ただし、WSLネイティブのディレクトリ(Windows側と同期されている`/mnt/c/`などのディレクトリ**以外**)で操作してください。WinodwsとWSLの間で同期されているディレクトリでビルドすると、ビルドが大幅に遅くなります。
   :::
2. ターミナルを開き、[zmk-workspace](https://github.com/kot149/zmk-workspace)をcloneする
   ```sh
   git clone https://github.com/kot149/zmk-workspace.git
   ```
   ```sh
   cd zmk-workspace
   ```
3. nix developコマンドが使えるように設定する
   ```sh
   mkdir -p ~/.config/nix && echo 'experimental-features = nix-command flakes' >> ~/.config/nix/nix.conf
   ```
4. nix developを実行する。以下、この中で作業する
   ```sh
   nix develop
   ```

   :::details オプション: direnvを使用して`nix develop`の実行を省略する
   以下のコマンドを実行することで、`nix develop`を実行しなくてもディレクトリに移動するだけで自動で有効化されるようになります。

   以下はbashを使用している場合の手順です。zshなど他のシェルを使用している場合は、適宜読み替えてください。

   ```sh
   # direnvとnix-direnvをインストールする
   nix profile add nixpkgs#direnv nixpkgs#nix-direnv

   # シェルフックを設定する
   echo 'eval "$(direnv hook bash)"' >> ~/.bashrc

   # nix-direnvを設定する
   mkdir -p ~/.config/direnv
   echo 'source $HOME/.nix-profile/share/nix-direnv/direnvrc' >> ~/.config/direnv/direnvrc

   # 設定を再読み込みする
   source ~/.bashrc

   # zmk-workspaceでdirenvの有効化を許可する
   direnv allow
   ```
   :::

## Dev Containerによるセットアップ

:::message
Nixを使用する場合は、この項目は飛ばして、[ビルド](#ビルド)へ進んでください。
:::

1. VSCodeをインストールする https://code.visualstudio.com/download
2. Dockerをインストールする https://docs.docker.com/get-docker/
3. ターミナルを開き、[zmk-workspace](https://github.com/kot149/zmk-workspace)をcloneする
   :::message alert
   Windowsの場合は、WSLの中で操作してください。ただし、WSLネイティブのディレクトリ(Windows側と同期されている`/mnt/c/`などのディレクトリ**以外**)で操作してください。WinodwsとWSL/コンテナの間で同期されているディレクトリでビルドすると、ビルドが大幅に遅くなります。
   :::
   ```sh
   git clone https://github.com/kot149/zmk-workspace.git
   ```
4. cloneしたzmk-workspaceをVSCodeで開く
   ```sh
   code zmk-workspace
   ```
5. VSCodeの拡張機能「[Dev Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers)」をインストールする
   - VSCodeの拡張機能の画面で`@id:ms-vscode-remote.remote-containers`で検索してインストール
6. VSCodeのフォルダをDev Containerで開き直す
   1. `F1`キーか`Ctrl+Shift+P`を押してコマンドパレットを開く
   2. `開発コンテナー: コンテナーで再度開く`(`Dev Containers: Reopen in Container`)コマンドを検索して実行する
7. VSCodeのウィンドウがリロードされ、Dev Containerが起動するので、完了までしばらく待つ
8. 以下、VSCodeでDev Containerを開いた状態で、VSCodeのターミナルから操作する


# ビルド

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
   この時点で、`west.yml`に記載された外部モジュールが自動でcloneされます。
3. ビルドする
   ```sh
   just build roBa_R
   ```
   (`roBa_R`はshieldの名前です。ビルドしたいshieldの名前に合わせて変更してください。)
   問題なくビルドが終われば、zmk-workspaceの`firmware/`ディレクトリにビルドされたファームウェアが保存されています。
4. uf2ファイルを書き込む
   :::message alert
   - Nixでセットアップした場合のみ動作します。Dev Containerでは動作しません。
   - WindowsまたはmacOSでのみ動作します。それ以外のOSでは動作しません。
   :::
   ```sh
   just flash roBa_R
   ```
   `-r`を指定すると、flashする前に再ビルドします。
   ```sh
   just flash roBa_R -r
   ```

## 補足
- 再度ビルドを行った場合は、キャッシュが使用されます。キャッシュなしでビルドするには、`-p`オプションを指定して`just build シールド名 -p`を実行します。
- 別のzmk-configでビルドするには、`config/`ディレクトリの中に、別のzmk-configをcloneし、`just init config/zmk-config-xxx`からやり直します。
- モジュールを追加するには、`config/zmk-config-xxx/config/west.yml`にモジュールを追加した後、`just update`を実行し、`just build`します。
- モジュールやZMK本体のソースコードをいじるには、zmk-workspace内にzmkやモジュールがcloneされているので、それをいじります。再度ビルドすれば、いじったコードが反映されます。
- cloneされるモジュールを`modules/`フォルダの中にまとめるには、`west.yml`の`projects`の各項目に`path: modules/module-name`を追加すると、`modules/module-name/`にcloneされるようになります。