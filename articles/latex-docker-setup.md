---
title: "【Windows】DockerでLaTeX環境を10分で構築する方法【初心者向け】"
emoji: "📝"
type: "tech"
topics: ["LaTeX", "Docker", "Windows", "環境構築", "WSL2"]
published: true
---

## はじめに

LaTeXで文書を書きたいけど、**環境構築でつまずいた**こと、ありませんか？

TeX Liveを直接インストールしようとすると、パスが通らない、バージョンの違いで動かない、設定ファイルの書き方がわからない……と、文書を書き始める前にハマるポイントがたくさんあります。しかもインストール自体にも時間がかかります。@fjktkm さんの記事「[LaTeX の環境構築は Docker 使ったほうが早い](https://zenn.dev/fjktkm/articles/f43658e74f5814)」によると、WindowsでTeX Liveを直接インストールすると**約48分**かかるそうです。

この記事では、**Dockerを使ってLaTeX環境をサクッと構築する方法**を紹介します。

Dockerを使うと何が嬉しいのか？

- **速い** — 同じ記事の計測ではDockerなら**約1.5分**。直接インストールの約30倍速い
- **つまずきにくい** — コマンド数回で完了。パス設定やバージョン管理で悩まない
- **再現性が高い** — 誰がやっても同じ環境が作れる。失敗してもやり直しが簡単
- **PCを汚さない** — TeX Liveの巨大なインストールが不要。Dockerコンテナの中で完結する

「Dockerって何？」という方でも大丈夫です。この記事の手順どおりに進めれば、LaTeXで文書を作れるようになります。

:::message
**この記事は筆者が実際にWindows PCで環境構築した手順をもとに書いています。** OSのバージョンやPCの状態によって、表示されるメッセージや手順が多少異なる場合があります。つまずいたポイントやトラブル対処もそのまま載せているので、参考にしてみてください。
:::

---

## 前提条件

- **OS**: Windows 10（バージョン2004以降）または Windows 11
- **メモリ**: 8GB以上推奨
- **ストレージ**: 5GB以上の空き容量

これだけ確認できればOKです。

---

## 全体の流れ

やることは大きく3ステップです。

```
① WSL2を有効にする（Ubuntuが入る）
② Ubuntu上でDockerをインストールする
③ LaTeX用のDockerイメージを取得して使う
```

すべてコマンドで完結します。GUIのインストーラーは不要です。

順番に進めましょう。

---

## ステップ1: WSL2を有効にする

:::details WSL2って何？WSLとは違うの？
**WSL（Windows Subsystem for Linux）** は、Windows上でLinuxを動かすための仕組みです。WSL2はその**バージョン2**にあたります。

| | WSL1 | WSL2 |
|---|---|---|
| 仕組み | Linuxの命令をWindowsに翻訳して実行 | 本物のLinuxカーネルを軽量な仮想マシンで実行 |
| 速さ | ファイル操作は速いが、互換性に限界あり | Linuxとほぼ同じ動作で、Dockerも動く |
| Docker対応 | ❌ 非対応 | ✅ 対応 |

**Dockerを使うにはWSL2が必要**です。現在は `wsl --install` を実行すると自動的にWSL2がインストールされるので、特に意識する必要はありません。
:::

### 1-1. PowerShellを管理者として開く

スタートメニューで「PowerShell」と検索し、**「管理者として実行」** を選びます。

### 1-2. WSL2をインストールする

以下のコマンドを実行します。

```powershell
wsl --install
```

これだけで、WSL2の有効化とUbuntuのインストールがまとめて行われます。

:::message
すでにWSL2が入っている場合は「既にインストールされています」と表示されます。その場合はそのまま次へ進んでください。
:::

:::details 「WSL インストールが壊れている可能性があります」と表示されたら
筆者がPCを初期化した直後にこの手順を試したところ、`wsl --install` を実行したあとに「WSL インストールが壊れている可能性があります」というエラーが出ました。すぐに表示されることもあれば、1分ほど待たされてから出ることもあったので、しばらく反応がなくても焦らず待ってみてください。

![PC初期化直後にwsl --installを実行したときのエラー画面](/images/wsl-install-error.png)

「任意のキーを押すとWSLの修復を始めます」と表示されるのですが、**何もしないと60秒でタイムアウトしてしまいます。** メッセージが出たらすぐに何かキーを押してください。

修復が完了したら、もう一度 `wsl --install` を実行すれば正常に進みます。
:::

### 1-3. Ubuntuの初期設定

インストールが完了すると、PowerShell上でそのままユーザー名とパスワードの入力を求められます。ここで入力してもいいのですが、この先の作業はすべてUbuntuのターミナルで行うので、**PowerShellは閉じてしまってOK**です。

スタートメニューで「Ubuntu」と検索して起動すると、改めてユーザー名とパスワードの設定を求められます。好きなものを入力してください（このユーザー名・パスワードはLinux用で、Windowsのものとは別です）。

![Ubuntuの初期設定画面（ユーザー名・パスワード入力）](/images/wsl-ubuntu-setup.png)
*Ubuntuを起動すると、ユーザー名とパスワードの設定を求められる*

:::message
**パスワードを入力しても画面に何も表示されません。** これは不具合ではなく、Linuxのセキュリティ上の仕様です。`***` すら出ませんが、ちゃんと入力されています。そのままパスワードを打ってEnterを押してください。確認のためもう一度入力を求められるので、同じパスワードを入力します。
:::

---

## ステップ2: Ubuntu上でDockerをインストールする

ここからは**Ubuntuのターミナル**で作業します。スタートメニューから「Ubuntu」を開いてください。

### 2-1. パッケージを最新にする

まずはUbuntuのパッケージ情報を更新します。

```bash
sudo apt update && sudo apt upgrade -y
```

:::details sudoって何？
`sudo` は「管理者権限でコマンドを実行する」という意味です。ソフトウェアのインストールなど、システムに変更を加える操作で必要になります。実行するとパスワードを聞かれるので、ステップ1-4で設定したパスワードを入力してください。
:::

### 2-2. Dockerをインストールする

以下のコマンドを実行します。

```bash
sudo apt install -y docker.io
```

これでDockerがインストールされます。

### 2-3. Dockerを自分のユーザーで使えるようにする

初期状態では、Dockerコマンドを実行するたびに `sudo` が必要です。以下のコマンドで、`sudo` なしで使えるようにします。

```bash
sudo usermod -aG docker $USER
```

:::details このコマンドの意味
`usermod -aG docker $USER` は「今のユーザーを `docker` グループに追加する」という意味です。`docker` グループに所属していると、`sudo` なしでDockerコマンドを実行できるようになります。
:::

**設定を反映するために、一度Ubuntuのウィンドウを閉じて、もう一度開いてください。**

### 2-4. Dockerの動作を確認する

Ubuntuを開き直したら、以下のコマンドでDockerが動いているか確認します。

```bash
docker --version
```

バージョン情報が表示されればOKです。

もしDockerデーモンが起動していない場合は、以下のコマンドで起動してください。

```bash
sudo service docker start
```

---

## ステップ3: LaTeX用Dockerイメージを取得する

引き続きUbuntuのターミナルで作業します。

### 3-1. イメージをダウンロード（プル）する

以下のコマンドを実行します。

```bash
docker pull k1z3/texlive
```

**初回は3〜5分ほどかかります。** TeX Liveの環境一式をダウンロードしているためです。コーヒーでも飲みながらお待ちください。

### 3-2. ダウンロードを確認する

完了したら、以下のコマンドで確認します。

```bash
docker images
```

`k1z3/texlive` が表示されれば成功です。

```
REPOSITORY       TAG       IMAGE ID       CREATED        SIZE
k1z3/texlive     latest    xxxxxxxxxxxx   X months ago   X.XXGB
```

---

## ステップ4: プロジェクトフォルダを準備する

### 4-1. フォルダを作成する

Ubuntuのターミナルで、ホームディレクトリにフォルダを作ります。

```bash
mkdir ~/latex-project
cd ~/latex-project
```

:::details Windowsのエクスプローラーからこのフォルダを開くには？
エクスプローラーのアドレスバーに `\\wsl$\Ubuntu\home\あなたのユーザー名\latex-project` と入力すると、Windowsからも直接アクセスできます。ファイルの編集はWindows側のテキストエディタ（メモ帳やVS Codeなど）でも行えます。
:::

### 4-2. 必要なファイルを作成する

このフォルダの中に、以下の2つのファイルを作ります。お好みのエディタで作成してください。

:::details Ubuntuのターミナル上でファイルを作る場合
以下のコマンドをそのままコピペすれば、ファイルが作成されます。

**main.tex を作成:**
```bash
cat << 'EOF' > main.tex
\documentclass{article}
\begin{document}
\Huge \LaTeX
\end{document}
EOF
```

**.latexmkrc を作成:**
```bash
cat << 'EOF' > .latexmkrc
$latex = 'uplatex';
$dvipdf = 'dvipdfmx %O -o %D %S';
$pdf_mode = 3;
EOF
```
:::

**main.tex**（メインの文書ファイル）:

```tex
\documentclass{article}
\begin{document}
\Huge \LaTeX
\end{document}
```

**.latexmkrc**（コンパイル設定ファイル）:

```perl
$latex = 'uplatex';
$dvipdf = 'dvipdfmx %O -o %D %S';
$pdf_mode = 3;
```

:::details .latexmkrcって何？
LaTeXのコンパイルには複数のコマンドを順番に実行する必要があります。`.latexmkrc` はその手順を自動化する設定ファイルです。

この設定では以下を指定しています。
- `uplatex` — 日本語対応のLaTeXエンジンを使う
- `dvipdfmx` — DVI形式からPDFに変換する
- `$pdf_mode = 3` — 上記のパイプライン（uplatex → dvipdfmx）でPDFを生成する
:::

フォルダの中身はこうなります。

```
~/latex-project/
├── main.tex
└── .latexmkrc
```

---

## ステップ5: コンパイルしてPDFを作る

### 5-1. プロジェクトフォルダにいることを確認する

```bash
cd ~/latex-project
```

### 5-2. コンパイルを実行する

以下のコマンドを実行します。

```bash
docker run --rm -v "$(pwd):/texsrc" -w /texsrc k1z3/texlive latexmk main.tex
```

:::details コマンドの意味を解説
| オプション | 意味 |
|-----------|------|
| `docker run` | Dockerコンテナを実行する |
| `--rm` | 実行後にコンテナを自動削除する（後片付け） |
| `-v "$(pwd):/texsrc"` | 今いるフォルダをコンテナの中の `/texsrc` に接続する |
| `-w /texsrc` | コンテナ内の作業フォルダを `/texsrc` に設定する |
| `k1z3/texlive` | 使用するDockerイメージ |
| `latexmk main.tex` | `main.tex` をコンパイルするコマンド |
:::

### 5-3. PDFを確認する

コンパイルが成功すると、フォルダ内に `main.pdf` が生成されます。

Windowsのエクスプローラーから `\\wsl$\Ubuntu\home\あなたのユーザー名\latex-project` を開いて、`main.pdf` をダブルクリックしてください。「**LaTeX**」と大きく表示されていれば成功です！

---

## 日本語の文書を作ってみよう

先ほどの `main.tex` は英語だけの最小サンプルでした。せっかくなので、日本語の文書も作ってみましょう。

`main.tex` を以下の内容に書き換えてください。

```tex
\documentclass[uplatex,dvipdfmx]{jsarticle}

\title{はじめてのLaTeX文書}
\author{あなたの名前}
\date{\today}

\begin{document}

\maketitle

\section{はじめに}
これはDockerで構築したLaTeX環境で作成した、はじめての日本語文書です。
環境構築おつかれさまでした！

\section{LaTeXでできること}
LaTeXを使うと、以下のようなことができます。

\begin{itemize}
  \item きれいな数式が書ける：$E = mc^2$
  \item レポートや論文のフォーマットが整う
  \item 参考文献の管理が楽になる
\end{itemize}

\section{数式の例}
二次方程式の解の公式はこちらです。

\begin{equation}
  x = \frac{-b \pm \sqrt{b^2 - 4ac}}{2a}
\end{equation}

\section{おわりに}
ここまでできれば、もう自分の文書を書き始められます。
レポートや論文に挑戦してみましょう。

\end{document}
```

同じコマンドでコンパイルすれば、日本語のPDFが生成されます。

```bash
docker run --rm -v "$(pwd):/texsrc" -w /texsrc k1z3/texlive latexmk main.tex
```

タイトル、セクション見出し、数式がきれいに表示されているはずです。

---

## よくあるトラブルと解決法

### 「docker: command not found」と出る

Dockerがインストールされていないか、ステップ2-3の後にUbuntuを開き直していない可能性があります。Ubuntuのウィンドウを閉じて、もう一度開いてから試してください。

### 「permission denied」と出る

![permission deniedエラーの画面](/images/docker-permission-denied.png)

筆者もこのエラーに遭遇しました。ステップ2-3で `sudo usermod -aG docker $USER` を実行した後、**Ubuntuのウィンドウを閉じて、もう一度開き直す**必要があります。開き直さないとグループの変更が反映されず、このエラーが出ます。

開き直してもエラーが出る場合は、Dockerデーモンが起動しているか確認してください。

```bash
sudo service docker start
```

### 「Cannot connect to the Docker daemon」と出る

Dockerデーモンが起動していません。以下のコマンドで起動してください。

```bash
sudo service docker start
```

:::details WSL2でDockerデーモンを自動起動するには？
WSL2では、Ubuntuを開くたびにDockerデーモンを手動で起動する必要がある場合があります。毎回コマンドを打つのが面倒な場合は、`~/.bashrc` の末尾に以下を追加すると、Ubuntu起動時に自動でDockerが立ち上がります。

```bash
if service docker status 2>&1 | grep -q "is not running"; then
    sudo service docker start > /dev/null 2>&1
fi
```

ただし、`sudo` のパスワード入力を省略するための設定も必要です。詳しくは「wsl2 docker 自動起動」で検索してみてください。
:::

### コンパイルは通るのにPDFに日本語が出ない

`main.tex` の `\documentclass` が `{article}` のままになっていませんか？ 日本語文書では `{jsarticle}` を使い、オプションに `uplatex,dvipdfmx` を指定してください。

---

## まとめ

この記事では、WindowsでDockerを使ってLaTeX環境を構築する方法を紹介しました。

手順をおさらいします。

1. **WSL2を有効にする**（`wsl --install`）
2. **Ubuntu上でDockerをインストールする**（`sudo apt install docker.io`）
3. **TeX Liveイメージを取得する**（`docker pull k1z3/texlive`）
4. **プロジェクトフォルダにtexファイルと設定ファイルを置く**
5. **`docker run` でコンパイルする**

Docker Desktopのような別のアプリをインストールする必要はありません。すべてコマンドで完結します。

一度この環境を作ってしまえば、あとは `.tex` ファイルを書いてコマンドを叩くだけです。PCを買い替えても、WSL2とDockerさえ入れれば同じ環境がすぐに再現できます。

LaTeXで素敵な文書を作ってください！

---

## 参考リンク

- [LaTeX の環境構築は Docker 使ったほうが早い - @fjktkm](https://zenn.dev/fjktkm/articles/f43658e74f5814) — Dockerの速さを実測で検証した記事
- [DockerでLaTeX環境構築（Windows向け） - note](https://note.com/mejiro43/n/n453fa8c78e00) — この記事の元になった筆者のnote記事
- [この記事で使用したサンプルファイル（GitHub）](https://github.com/kanpachioishi/latex-setup)
- [Docker公式 — UbuntuへのDockerインストール](https://docs.docker.com/engine/install/ubuntu/)
- [WSL2公式ドキュメント](https://learn.microsoft.com/ja-jp/windows/wsl/install)
