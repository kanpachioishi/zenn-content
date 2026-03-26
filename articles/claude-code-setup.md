---
title: "【初心者向け】Claude Codeの始め方 — インストールから初回起動まで"
emoji: "🤖"
type: "tech"
topics: ["ClaudeCode", "AI", "環境構築", "CLI", "Anthropic"]
published: false
---

## はじめに

**Claude Code**というツールをご存知でしょうか？

Claude CodeはAnthropic社が提供する**ターミナルベースのAIアシスタント**です。ターミナル（黒い画面）で `claude` と打つだけで、AIがコードを読み書きしたり、ファイルを操作したり、gitの操作を代行してくれたりします。

ChatGPTのようなチャット画面とは違い、**自分のPC上のファイルを直接操作できる**のが大きな特徴です。

- ファイルの中身を読んで、バグの原因を教えてくれる
- 指示するだけでコードを書いて、ファイルを作成してくれる
- git commitやpushまで代行してくれる
- プロジェクト全体を理解した上で提案してくれる

「AIにコードを書いてもらいたいけど、ChatGPTにコピペするのが面倒」「自分のプロジェクトを丸ごと理解してほしい」と思ったことがある方には、まさにぴったりのツールです。

この記事では、**WSL2（Ubuntu）環境にClaude Codeをインストールして、初回起動するまでの手順**を紹介します。

:::message
**この記事は筆者が実際にWSL2環境で構築した手順をもとに書いています。** 環境の違いによって表示が多少異なる場合がありますが、つまずいたポイントもそのまま載せているので参考にしてみてください。
:::

---

## 前提条件

- **WSL2（Ubuntu）が導入済みであること**
- **インターネット接続**があること
- **Anthropicアカウント**（無料で作成可能。手順の中で案内します）

:::details WSL2の導入がまだの方へ
WSL2の導入方法は、筆者の別記事で詳しく解説しています。

<!-- TODO: WSL2記事へのリンクを追加 -->

Windows 10（バージョン2004以降）またはWindows 11であれば、PowerShellで `wsl --install` を実行するだけで導入できます。
:::

---

## 全体の流れ

やることは3ステップです。

```
① Node.jsをインストールする
② Claude Codeをインストールする
③ 初回起動して認証する
```

すべてUbuntuのターミナルで完結します。10〜15分あれば終わります。

---

## ステップ1: Node.jsをインストールする

Claude CodeはNode.js上で動くツールです。まずNode.jsをインストールします。

:::message
**Node.js 18以上が必要です。** これより古いバージョンだとClaude Codeが動きません。
:::

### 1-1. 現在のNode.jsバージョンを確認する

Ubuntuのターミナルを開いて、以下のコマンドを実行してください。

```bash
node -v
```

`v18.x.x` 以上のバージョンが表示されれば、ステップ2に進んでOKです。

「command not found」と表示された場合や、バージョンが18未満の場合は、次の手順でインストールします。

### 1-2. nvmでNode.jsをインストールする

Node.jsのインストールには**nvm（Node Version Manager）** を使います。バージョン管理が楽になるのでおすすめです。

まず、nvmをインストールします。

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
```

インストールが完了したら、**ターミナルを一度閉じて開き直してください。** これをしないとnvmコマンドが使えません。

:::details なぜターミナルを開き直す必要があるの？
nvmのインストールスクリプトは、`~/.bashrc` にnvmの初期化コードを追加します。この設定はターミナルを開いたときに読み込まれるので、開き直さないと反映されません。

開き直すのが面倒な場合は、以下のコマンドでも反映できます。

```bash
source ~/.bashrc
```
:::

ターミナルを開き直したら、Node.jsの最新LTS版をインストールします。

```bash
nvm install --lts
```

インストールが完了したら、バージョンを確認します。

```bash
node -v
```

`v18.x.x` 以上が表示されればOKです。

:::details aptでNode.jsを入れるのはダメなの？
`sudo apt install nodejs` でもNode.jsはインストールできますが、Ubuntuのリポジトリにあるバージョンは古いことが多く、Node.js 18未満の場合があります。

nvmを使えばバージョンの切り替えも簡単にできるので、開発用途ではnvmがおすすめです。
:::

---

## ステップ2: Claude Codeをインストールする

Node.jsが入ったら、Claude Codeをインストールします。コマンド1つで終わります。

```bash
npm install -g @anthropic-ai/claude-code
```

:::details このコマンドの意味
| 部分 | 意味 |
|------|------|
| `npm` | Node.jsのパッケージマネージャー |
| `install` | パッケージをインストールする |
| `-g` | グローバルにインストール（どこからでも使えるようにする） |
| `@anthropic-ai/claude-code` | Claude Codeのパッケージ名 |
:::

インストールが完了したら、バージョンを確認してみましょう。

```bash
claude --version
```

バージョン番号が表示されれば、インストール成功です。

:::message
`npm install -g` でPermission Errorが出る場合は、nvmを使っていれば起きないはずです。もしaptでNode.jsを入れた場合は、`sudo npm install -g @anthropic-ai/claude-code` を試してみてください。
:::

---

## ステップ3: 初回起動と認証

いよいよClaude Codeを起動します。

### 3-1. Claude Codeを起動する

適当なフォルダで以下のコマンドを実行します。

```bash
claude
```

初回起動時は、まず利用規約の確認が表示されます。内容を確認して、同意して進めてください。

### 3-2. Anthropicアカウントで認証する

続いて、認証方法の選択画面が表示されます。

ここで**Anthropicアカウントでログイン**を選択すると、ブラウザが自動的に開きます。

:::message
WSL2環境では、Windowsのデフォルトブラウザが自動で開きます。もしブラウザが開かない場合は、表示されるURLをコピーして、手動でブラウザに貼り付けてください。
:::

ブラウザでAnthropicのログイン画面が表示されるので、アカウントでログインします。

:::details Anthropicアカウントをまだ持っていない場合
[Anthropicの公式サイト](https://console.anthropic.com/)からアカウントを作成できます。メールアドレスまたはGoogleアカウントで登録できます。
:::

ブラウザ上で認証が完了すると、ターミナルに戻って自動的にClaude Codeが使えるようになります。

### 3-3. 認証を確認する

認証が成功すると、ターミナルにClaude Codeの対話画面が表示されます。プロンプト（入力欄）が出てきたら、準備完了です。

試しに何か話しかけてみましょう。

```
> こんにちは。今いるディレクトリのファイル一覧を教えて
```

Claude Codeがファイル一覧を返してくれたら、正常に動いています。

---

## ステップ4: 基本的な使い方

Claude Codeが動くようになったので、基本的な使い方を紹介します。

### 起動と終了

```bash
# 起動（現在のディレクトリをコンテキストとして使う）
claude

# プロジェクトのディレクトリで起動するのが基本
cd ~/my-project
claude

# 終了
# 対話画面で以下を入力するか、Ctrl+Cを2回押す
/exit
```

:::message
**Claude Codeは、起動したディレクトリのファイルを読み書きします。** 必ず作業したいプロジェクトのフォルダに移動してから起動してください。
:::

### よく使うコマンド

対話画面の中で使えるスラッシュコマンドがあります。

| コマンド | 説明 |
|---------|------|
| `/help` | ヘルプを表示 |
| `/exit` | Claude Codeを終了 |
| `/clear` | 会話履歴をクリア |
| `/compact` | これまでの会話を要約してコンテキストを節約 |

### 使い方の例

Claude Codeには、自然な日本語で指示を出せます。

```
> このプロジェクトの構成を教えて
```

```
> README.mdを作成して。プロジェクトの概要、セットアップ手順、使い方を含めて
```

```
> src/index.tsにバグがありそう。調べて修正して
```

```
> git statusを確認して、変更があればコミットして
```

ファイルの読み書き、コマンドの実行、gitの操作など、**ターミナルでできることはほぼすべてClaude Codeに任せられます。** ただし、ファイルの書き込みやコマンドの実行時には確認プロンプトが表示されるので、意図しない操作が勝手に行われることはありません。

---

## よくあるトラブルと解決法

### 「claude: command not found」と出る

Claude Codeがインストールされていないか、パスが通っていない可能性があります。

```bash
# インストールされているか確認
npm list -g @anthropic-ai/claude-code
```

表示されない場合は、もう一度インストールしてください。

```bash
npm install -g @anthropic-ai/claude-code
```

nvmを使っている場合、別のNode.jsバージョンに切り替えるとグローバルパッケージが見えなくなることがあります。`nvm use --lts` で元に戻してください。

### ブラウザが自動で開かない

WSL2環境で `xdg-open` が設定されていないと、ブラウザが自動で開かない場合があります。

ターミナルに表示されるURLをコピーして、Windowsのブラウザに直接貼り付けて認証してください。

### 「Node.js 18 or later is required」と出る

Node.jsのバージョンが古いです。nvmで最新LTS版をインストールしてください。

```bash
nvm install --lts
nvm use --lts
node -v  # v18以上であることを確認
```

### APIの料金が心配

Claude Codeの利用にはAPI利用料がかかります。

:::details 料金体系について
Claude Codeは従量課金制です。会話のトークン数に応じて課金されます。料金の詳細は[Anthropicの料金ページ](https://www.anthropic.com/pricing)を確認してください。

Anthropicの「Max」プラン（月額のサブスクリプション）に加入すると、Claude Codeも定額で使えるようになります。まずは少しだけ試してみて、使用量を確認してから判断するのがおすすめです。

<!-- TODO: 最新の料金プランを確認して更新する -->
:::

---

## まとめ

この記事では、WSL2（Ubuntu）環境にClaude Codeをインストールして初回起動するまでの手順を紹介しました。

手順をおさらいします。

1. **Node.jsをインストールする**（nvmで`nvm install --lts`）
2. **Claude Codeをインストールする**（`npm install -g @anthropic-ai/claude-code`）
3. **`claude` コマンドで起動して、ブラウザで認証する**

やることはこれだけです。環境さえ整えば、あとは `claude` と打つだけでAIアシスタントが使えるようになります。

「AIツールは興味あるけどAPI連携とか難しそう」と感じている方も、Claude Codeなら**npmでインストールしてコマンドを打つだけ**なので、ぜひ試してみてください。

---

## 参考リンク

- [Claude Code 公式ドキュメント](https://docs.anthropic.com/en/docs/claude-code/overview)
- [Anthropic 公式サイト](https://www.anthropic.com/)
- [nvm（Node Version Manager）GitHub](https://github.com/nvm-sh/nvm)
