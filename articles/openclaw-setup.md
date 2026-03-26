---
title: "Captain Clawをモダン環境で遊ぶ！OpenClawの環境構築ガイド【Linux/Windows】"
emoji: "🎮"
type: "tech"
topics: ["OpenClaw", "ゲーム", "環境構築", "オープンソース", "Linux"]
published: false
---

## はじめに

**Captain Claw**というゲームを知っていますか？

1997年にMonolith Productionsが開発した横スクロールアクションゲームで、海賊の猫「Captain Claw」を操作してお宝を集めながらステージをクリアしていくゲームです。当時のPC向けゲームとしてはかなり出来がよく、海外ではいまだにファンが多い名作です。

ただ、1997年のWindows向けゲームなので、現代のPC（Windows 10/11やLinux）ではそのままでは動きません。

そこで登場するのが**OpenClaw**です。

**OpenClaw**はCaptain Clawをゼロから書き直したオープンソースの再実装プロジェクトです。元のゲームのアセット（グラフィックや音声データ）を使いつつ、エンジン部分を完全に新しく作り直しているため、現代のOS上でスムーズに動きます。

この記事では、**OpenClawをLinux（Ubuntu）環境でビルドして遊べるようにするまでの手順**を紹介します。Windows（WSL2経由）でも同じ手順で構築できます。

:::message
**この記事は筆者が実際に環境構築した手順をもとに書いています。** OSのバージョンやPCの状態によって多少異なる場合があります。
:::

---

## OpenClawとは

| 項目 | 内容 |
|------|------|
| リポジトリ | [pjasicek/OpenClaw](https://github.com/pjasicek/OpenClaw) |
| 言語 | C++ |
| ライセンス | GPL-3.0 |
| 対応OS | Windows / Linux / macOS / Android / WebAssembly |
| 物理エンジン | Box2D |
| グラフィック/音声 | SDL2ライブラリ群 |
| 最新バージョン | 1.0.3（2024年11月時点） |

OpenClawはオリジナルのCaptain Clawの**レベル1〜10**を実装しています。全14レベルのうち後半はまだ未実装ですが、ゲームとして十分に遊べる完成度です。

:::message alert
OpenClawのエンジン自体はオープンソースですが、**ゲームのアセット（CLAW.REZファイル）は別途必要**です。これはオリジナルのCaptain Clawに含まれるファイルです。アセットの入手方法については後述します。
:::

---

## 前提条件

- **OS**: Ubuntu 20.04以降（またはWindows + WSL2）
- **メモリ**: 4GB以上
- **ストレージ**: 2GB以上の空き容量
- **オリジナルのCaptain Claw**: CLAW.REZファイルが必要
- **基本的なコマンドライン操作**: `cd`, `git`, `make` 等が使えること

:::details WSL2を使う場合（Windows）
WindowsユーザーはWSL2のUbuntu環境で同じ手順を実行できます。WSL2のセットアップがまだの方は、PowerShellを管理者として開いて以下を実行してください。

```powershell
wsl --install
```

インストール後、再起動してUbuntuを起動すればOKです。
:::

---

## 全体の流れ

```
① 依存パッケージをインストールする
② ソースコードをクローンする
③ CMakeでビルドする
④ ゲームアセット（CLAW.REZ）を配置する
⑤ 起動して遊ぶ
```

順番に進めましょう。

---

## ステップ1: 依存パッケージのインストール

OpenClawは**SDL2**ライブラリ群と**CMake**を使ってビルドします。まず必要なパッケージをインストールします。

```bash
sudo apt update
sudo apt install -y \
  build-essential \
  cmake \
  git \
  libsdl2-dev \
  libsdl2-image-dev \
  libsdl2-mixer-dev \
  libsdl2-ttf-dev \
  libsdl2-gfx-dev
```

:::details 各パッケージの役割
| パッケージ | 役割 |
|-----------|------|
| `build-essential` | C/C++コンパイラ（gcc, g++）とmake |
| `cmake` | ビルドシステム |
| `git` | ソースコードの取得 |
| `libsdl2-dev` | グラフィック描画・入力処理 |
| `libsdl2-image-dev` | 画像ファイルの読み込み |
| `libsdl2-mixer-dev` | 音声・BGMの再生 |
| `libsdl2-ttf-dev` | フォントの描画 |
| `libsdl2-gfx-dev` | 追加のグラフィック描画機能 |
:::

### BGM再生のための追加パッケージ

Captain ClawのBGMはMIDI形式です。MIDIを再生するために**TiMidity++**とサウンドフォントもインストールします。

```bash
sudo apt install -y timidity freepat
```

:::message
これをインストールしないとゲームは起動しますが**BGMが鳴りません**。ゲームプレイには影響しませんが、BGMがあった方が断然楽しいのでインストール推奨です。
:::

---

## ステップ2: ソースコードのクローン

GitHubからソースコードを取得します。

```bash
git clone https://github.com/pjasicek/OpenClaw.git
cd OpenClaw
```

:::details リリース版を使いたい場合
ソースからビルドする代わりに、[Releasesページ](https://github.com/pjasicek/OpenClaw/releases)からビルド済みバイナリをダウンロードする方法もあります。Windows向けにはビルド済みの実行ファイルが公開されています。

ただし、Linux向けのビルド済みバイナリは提供されていない場合があるため、この記事ではソースからビルドする手順を紹介します。
:::

---

## ステップ3: CMakeでビルドする

ビルド用のディレクトリを作って、CMakeとmakeを実行します。

```bash
mkdir build
cd build
cmake ..
make -j$(nproc)
```

:::details `-j$(nproc)` って何？
`make -j$(nproc)` の `-j` はビルドの並列数を指定するオプションです。`$(nproc)` はCPUコア数を返すコマンドなので、「CPUの全コアを使ってビルドする」という意味になります。

例えば4コアのPCなら `make -j4` と同じです。これによってビルド時間が大幅に短縮されます。
:::

ビルドが成功すると、`Build_Release` ディレクトリに実行ファイルが生成されます。

<!-- 要確認: ビルドの出力先パスは環境によって異なる可能性があります。実際にビルドして確認してください -->

### ビルドエラーが出た場合

ビルドに失敗した場合、まず依存パッケージが全てインストールされているか確認してください。

```bash
# 依存パッケージの確認
dpkg -l | grep libsdl2
```

SDL2関連のパッケージが一覧に表示されていれば問題ありません。

---

## ステップ4: ゲームアセットの配置

ここが一番のポイントです。OpenClawはエンジン部分のみオープンソースで、**ゲームのグラフィックや音声データは別途用意する必要があります**。

### CLAW.REZファイルとは

`CLAW.REZ` はオリジナルのCaptain Clawのゲームデータが格納されたアーカイブファイルです。このファイルがないとOpenClawは起動できません。

### CLAW.REZの入手方法

オリジナルのCaptain ClawのCDを持っている場合、CDまたはインストール先ディレクトリに`CLAW.REZ`が含まれています。

<!-- 要確認: Captain Clawは現在デジタル販売（GOG, Steam等）されているか確認。2024年時点では公式なデジタル販売は確認できませんでした -->

:::message alert
**CLAW.REZファイルはCaptain Clawの著作物です。** 必ず正規に入手したゲームのファイルを使用してください。
:::

### ファイルの配置

`CLAW.REZ` を `Build_Release` ディレクトリにコピーします。

```bash
# OpenClawのルートディレクトリにいる前提
cp /path/to/your/CLAW.REZ Build_Release/
```

また、`Build_Release/ASSETS` ディレクトリの内容を圧縮する必要があります。

```bash
cd Build_Release
zip -r ASSETS.ZIP ASSETS/
```

<!-- 要確認: ASSETSディレクトリの中身がリポジトリに含まれているか、それともCLAW.REZから展開するのか、実際の手順を確認してください -->

---

## ステップ5: ゲームの起動

すべての準備が整ったら、ゲームを起動します。

```bash
cd Build_Release
./openclaw
```

Captain Clawのタイトル画面が表示されれば成功です！

:::details 操作方法（デフォルト）
| キー | アクション |
|------|----------|
| 矢印キー | 移動 |
| Z | ジャンプ |
| X | 攻撃 |
| ESC | ポーズ/メニュー |

<!-- 要確認: デフォルトのキーバインドは実際に起動して確認してください -->
:::

---

## よくあるトラブルと解決法

### ビルド時: `SDL2 not found`

```
Could not find SDL2
```

SDL2の開発パッケージがインストールされていない場合に発生します。

```bash
sudo apt install libsdl2-dev
```

### ビルド時: CMakeのバージョンが古い

古いUbuntuではCMakeのバージョンが不足する場合があります。

```bash
cmake --version
```

バージョンが古い場合は、以下でアップデートできます。

```bash
sudo apt install -y software-properties-common
sudo add-apt-repository ppa:kitware/release
sudo apt update
sudo apt install -y cmake
```

### 起動時: `CLAW.REZ not found`

`CLAW.REZ` ファイルが正しい場所に配置されていないか、ファイル名が間違っている場合に発生します。

- ファイル名は**大文字**で `CLAW.REZ` であること
- `Build_Release` ディレクトリの直下に配置すること

### 起動時: BGMが鳴らない

TiMidity++がインストールされていない可能性があります。

```bash
sudo apt install -y timidity freepat
```

### 起動時: 画面が表示されない（WSL2の場合）

WSL2ではGUIアプリケーションの表示に**WSLg**が必要です。Windows 11であればデフォルトで有効になっています。Windows 10の場合は別途X Serverの設定が必要です。

:::details Windows 10でX Serverを設定する方法
1. [VcXsrv](https://sourceforge.net/projects/vcxsrv/) をインストール
2. XLaunchを起動し、「Multiple windows」→「Start no client」→「Disable access control」にチェックして起動
3. WSL2側で環境変数を設定

```bash
export DISPLAY=$(cat /etc/resolv.conf | grep nameserver | awk '{print $2}'):0
```

`.bashrc` に追加しておくと毎回設定する必要がなくなります。
:::

---

## まとめ

この記事では、Captain Clawのオープンソース再実装である**OpenClaw**の環境構築手順を紹介しました。

手順をおさらいすると:

1. **依存パッケージのインストール** — SDL2ライブラリ群、CMake、TiMidity++
2. **ソースコードのクローン** — GitHubから取得
3. **CMakeでビルド** — `cmake` + `make`
4. **ゲームアセットの配置** — CLAW.REZファイルを配置
5. **起動** — `./openclaw` で実行

90年代の名作ゲームがオープンソースの力で現代に蘇るのは、なかなか感動的です。Captain Clawを遊んだことがある人も、初めての人も、ぜひ試してみてください。

OpenClawの開発は現在も進行中で、コミュニティへの貢献も歓迎されています。バグを見つけたり改善のアイデアがあれば、[GitHubリポジトリ](https://github.com/pjasicek/OpenClaw)のIssueで報告できます。

---

**参考リンク**:
- [OpenClaw GitHub リポジトリ](https://github.com/pjasicek/OpenClaw)
- [Captain Claw - Wikipedia](https://en.wikipedia.org/wiki/Claw_(video_game))
