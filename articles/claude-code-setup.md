---
title: "【WSL2】Claude Codeをネイティブインストールする方法【npm不要】"
emoji: "🤖"
type: "tech"
topics: ["ClaudeCode", "AI", "環境構築", "WSL2", "Ubuntu"]
published: false
---

## はじめに

Claude Codeのインストール方法を検索すると、**npmを使う手順**が多く見つかります。

```bash
# よく見かける手順
npm install -g @anthropic-ai/claude-code
```

この方法でもインストールはできますが、実は公式では**ネイティブインストール**が推奨されています。

```bash
# 公式推奨の手順
curl -fsSL https://claude.ai/install.sh | bash
```

ネイティブインストールのメリットは：

- **Node.jsが不要** — npmもnvmもいらない
- **自動更新** — バックグラウンドで最新版に更新される
- **シンプル** — コマンド1つで完了

この記事では、WSL2（Ubuntu）環境に**ネイティブインストールでClaude Codeを導入する手順**を紹介します。

:::message
**この記事は筆者が実際にWSL2環境で構築した手順をもとに書いています。** つまずいたポイントもそのまま載せているので、参考にしてみてください。
:::

---

## 前提条件

- **WSL2（Ubuntu）が導入済みであること**
- **インターネット接続**があること
- **Anthropicアカウント**（手順の中で案内します）

:::details WSL2の導入がまだの方へ
WSL2の導入方法は、筆者の別記事で解説しています。

👉 [【Windows】DockerでLaTeX環境を10分で構築する方法](https://zenn.dev/kanpachioishi/articles/latex-docker-setup)のステップ1を参考にしてください。

PowerShellで `wsl --install` を実行するだけで導入できます。
:::

---

## 全体の流れ

やることは2ステップです。

```
① Claude Codeをインストールする
② 初回起動して認証する
```

Node.jsのインストールは不要です。すべてUbuntuのターミナルで完結します。

---

## ステップ1: Claude Codeをインストールする

Ubuntuのターミナルを開いて、以下のコマンドを実行します。

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

:::details このコマンドの意味
| 部分 | 意味 |
|------|------|
| `curl` | URLからデータを取得するコマンド |
| `-fsSL` | エラー時に静かに失敗し、リダイレクトに追従するオプション |
| `https://claude.ai/install.sh` | Claude Codeの公式インストールスクリプト |
| `\| bash` | 取得したスクリプトをそのまま実行する |
:::

インストールが完了すると、「Installation complete!」と表示されます。

ただし、そのまま `claude` を実行すると **`command not found`** になる場合があります。これはインストール先（`~/.local/bin`）にパスが通っていないためです。以下のコマンドでパスを通します。

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc && source ~/.bashrc
```

:::details このコマンドの意味
`~/.bashrc` にパスの設定を追記して、`source` で即座に反映しています。`~/.local/bin` はClaude Codeのインストール先で、ここにパスを通すことで `claude` コマンドが使えるようになります。

次回以降のターミナル起動時は自動で反映されるので、このコマンドは一度だけ実行すればOKです。
:::

パスを通したら `claude` コマンドが使えるようになります。

![インストール完了→command not found→パス通し→起動成功の流れ](/images/claude-code-path-fix.png)
*インストール直後は `command not found` になるが、パスを通せば起動できる*

バージョンを確認します。

```bash
claude --version
```

バージョン番号が表示されればインストール成功です。

:::message
npmで入れる手順（`npm install -g @anthropic-ai/claude-code`）も動きますが、Node.jsの導入が別途必要になります。特にこだわりがなければ、ネイティブインストールがおすすめです。
:::

---

## ステップ2: 初回起動と認証

### 2-1. Claude Codeを起動する

適当なフォルダで以下のコマンドを実行します。

```bash
claude
```

初回起動時は、利用規約の確認が表示されます。内容を確認して同意してください。

### 2-2. Anthropicアカウントで認証する

認証方法の選択画面が表示されます。**Anthropicアカウントでログイン**を選択すると、ブラウザが自動的に開きます。

:::message
WSL2環境では、ブラウザが自動で開かないことがあります。その場合は「`c`」キーを押すとURLがコピーされるので、Windowsのブラウザに貼り付けてください。
:::

![初回起動の認証画面。cキーでURLをコピーできる](/images/claude-code-auth-url.png)
*ブラウザが開かない場合は `c` を押してURLをコピー*

ブラウザでAnthropicのログイン画面が表示されるので、アカウントでログインします。ログイン後、Claude Codeへの接続を承認する画面が出るので、**「承認する」** をクリックします。

![ブラウザの承認画面](/images/claude-code-approve.png)
*Claude Codeへの接続を承認する*

承認すると、**Authentication Code**が表示されます。「Copy Code」をクリックしてコードをコピーしてください。

![Authentication Code画面](/images/claude-code-auth-code.png)
*表示されたAuthentication Codeをコピーする*

ターミナルに戻り、「Paste code here if prompted >」にコードを貼り付けてEnterを押します。

:::details Anthropicアカウントをまだ持っていない場合
[Anthropicの公式サイト](https://console.anthropic.com/)からアカウントを作成できます。メールアドレスまたはGoogleアカウントで登録できます。

Claude Codeを利用するには、以下のいずれかが必要です。
- **Claude Pro / Max プラン**（月額サブスクリプション）
- **APIクレジット**（従量課金）
:::

### 2-3. 動作確認

プロンプト（入力欄）が出てきたら準備完了です。試しに話しかけてみましょう。

```
> こんにちは。今いるディレクトリのファイル一覧を教えて
```

ファイル一覧が返ってくれば、正常に動いています！

---

## 基本的な使い方

### 起動と終了

```bash
# プロジェクトのディレクトリで起動するのが基本
cd ~/my-project
claude

# 終了は対話画面で /exit を入力するか、Ctrl+C を2回押す
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

自然な日本語で指示を出せます。

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

ファイルの読み書き、コマンドの実行、gitの操作など、ターミナルでできることはほぼすべてClaude Codeに任せられます。ファイルの書き込みやコマンドの実行時には確認プロンプトが表示されるので、意図しない操作が勝手に行われることはありません。

---

## よくあるトラブルと解決法

### 「claude: command not found」と出る

インストール直後にこのエラーが出る場合は、ターミナルを一度閉じて開き直してみてください。パスの設定が反映されていない可能性があります。

### ブラウザが自動で開かない

WSL2環境で `xdg-open` が設定されていないと、ブラウザが自動で開かないことがあります。ターミナルに表示されるURLをコピーして、Windowsのブラウザに直接貼り付けてください。

### npmで入れたClaude Codeとの競合

以前npmでインストールしていた場合は、先にアンインストールしてからネイティブインストールすることをおすすめします。

```bash
npm uninstall -g @anthropic-ai/claude-code
```

---

## まとめ

この記事では、WSL2（Ubuntu）環境にClaude Codeをネイティブインストールする手順を紹介しました。

手順をおさらいします。

1. **Claude Codeをインストールする**（`curl -fsSL https://claude.ai/install.sh | bash`）
2. **`claude` コマンドで起動して、ブラウザで認証する**

Node.jsもnpmも不要。コマンド1つでインストールが完了し、自動更新もされます。

npmで入れる記事が多いですが、公式推奨はネイティブインストールです。これからClaude Codeを始める方は、ぜひこちらの方法を試してみてください。

---

## 参考リンク

- [Claude Code 公式ドキュメント](https://docs.anthropic.com/en/docs/claude-code/overview)
- [Anthropic 公式サイト](https://www.anthropic.com/)
