---
title: "OpenClawをDockerで安全に動かす！環境構築ガイド【Ubuntu/WSL2】"
emoji: "🦞"
type: "tech"
topics: ["OpenClaw", "Docker", "AIエージェント", "環境構築", "Ubuntu"]
published: false
---

## はじめに

**OpenClaw**を知っていますか？

GitHubスター25万超、史上最速で成長しているオープンソースのAIエージェントフレームワークです。ローカルで動く自律型AIアシスタントで、WhatsApp、Discord、Slackなど23種類以上のチャネルに対応しています。

ただし、AIが自律的にファイル操作やコマンド実行を行うため、**セキュリティリスク**があります。実際に2ヶ月で9件以上のCVE（公開された脆弱性）が報告されています。

そこでこの記事では、**Dockerでサンドボックス化してOpenClawを安全に動かす方法**を紹介します。Dockerコンテナ内でAIを動かすことで、万が一暴走しても被害をコンテナ内に封じ込められます。

:::message
**この記事は筆者が実際に環境構築した手順をもとに書いています。** 環境構築に丸1日かかりましたが、この手順通りにやれば迷わずセットアップできるはずです。
:::

---

## なぜDockerなのか

OpenClawはAIエージェントなので、ユーザーの指示に応じて**ファイルの作成・編集・削除やシェルコマンドの実行**を自動で行います。

便利な反面、こんなリスクがあります。

- AIが想定外のコマンドを実行してシステムを壊す
- 悪意のあるスキル（プラグイン）による攻撃
- プロンプトインジェクションによる意図しない操作

Dockerでコンテナ化すれば、AIの操作はコンテナ内に閉じ込められます。ホストOSに影響を与えないので安心です。

---

## 前提条件

- **OS**: Ubuntu（またはWindows + WSL2）
- **Docker / Docker Compose**: インストール済み
- **Anthropic APIキー**: [Anthropic Console](https://console.anthropic.com/)で取得

:::message
Node.jsのインストールは不要です。Dockerコンテナ内に含まれています。
:::

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
① リポジトリをクローンする
② 作業ディレクトリを作成する
③ .envを作成する
④ openclaw.jsonを作成する
⑤ docker-compose.ymlを編集する
⑥ Dockerを起動する
⑦ ブラウザからアクセスしてペアリングする
```

順番に進めましょう。

---

## ステップ1: リポジトリのクローン

```bash
cd ~
git clone https://github.com/openclaw/openclaw.git
cd openclaw
```

---

## ステップ2: 作業ディレクトリの作成

設定ファイルとワークスペース用のディレクトリを作ります。

```bash
mkdir config workspace
```

| ディレクトリ | 用途 |
|-------------|------|
| `config/` | OpenClawの設定ファイル（openclaw.json）を置く |
| `workspace/` | AIエージェントの作業領域 |

---

## ステップ3: .envの作成

プロジェクトルート（`~/openclaw/`）に`.env`ファイルを作成します。以下のコマンドの `ANTHROPIC_API_KEY` だけ自分のAPIキーに書き換えてから貼り付けてください。

```bash
cat > .env << 'EOF'
OPENCLAW_GATEWAY_TOKEN=mytoken
USE_DOCKER=true
OPENCLAW_CONFIG_DIR=./config
OPENCLAW_WORKSPACE_DIR=./workspace
ANTHROPIC_API_KEY=sk-ant-api03-ここにAPIキーを貼る
CLAUDE_AI_SESSION_KEY=na
CLAUDE_WEB_SESSION_KEY=na
CLAUDE_WEB_COOKIE=na
EOF
```

:::message alert
**`ANTHROPIC_API_KEY`には自分のAPIキーを設定してください。** `OPENCLAW_GATEWAY_TOKEN`は任意の文字列でOKです（後でブラウザからの接続に使います）。
:::

:::details 各項目の説明
| 項目 | 説明 |
|------|------|
| `OPENCLAW_GATEWAY_TOKEN` | ブラウザからの接続時に使うトークン。任意の文字列 |
| `USE_DOCKER` | Docker環境で動かすことを指定 |
| `OPENCLAW_CONFIG_DIR` | 設定ファイルのディレクトリ |
| `OPENCLAW_WORKSPACE_DIR` | AIの作業ディレクトリ |
| `ANTHROPIC_API_KEY` | Anthropic APIキー（必須） |
| `CLAUDE_AI_SESSION_KEY` | 未使用の場合は`na`でOK |
| `CLAUDE_WEB_SESSION_KEY` | 未使用の場合は`na`でOK |
| `CLAUDE_WEB_COOKIE` | 未使用の場合は`na`でOK |
:::

---

## ステップ4: openclaw.jsonの作成

`config/`ディレクトリに設定ファイルを作成します。以下のコマンドをそのまま貼り付けてください。

```bash
cat > config/openclaw.json << 'EOF'
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "anthropic/claude-haiku-4-5-20251001"
      }
    }
  },
  "gateway": {
    "mode": "local",
    "auth": {
      "mode": "token",
      "token": "mytoken"
    },
    "controlUi": {
      "allowedOrigins": ["http://localhost:18789"],
      "dangerouslyAllowHostHeaderOriginFallback": true
    }
  }
}
EOF
```

:::details 設定のポイント
- **モデル**: `claude-haiku-4-5-20251001`はコストが安く、お試しに最適です。慣れてきたら`claude-sonnet-4-6`等に変更できます
- **token**: `.env`の`OPENCLAW_GATEWAY_TOKEN`と同じ値にしてください
- **allowedOrigins**: ローカルアクセスのみ許可しています
:::

---

## ステップ5: docker-compose.ymlの編集

リポジトリに含まれている`docker-compose.yml`を編集します。

```bash
nano docker-compose.yml
```

`openclaw-gateway`サービスの`environment`セクションに以下を追加します。

```yaml
ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY}
```

これにより、`.env`で設定したAPIキーがコンテナ内に渡されます。

---

## ステップ6: Dockerの起動

```bash
docker compose up -d
```

初回はイメージのビルドに数分かかります。

:::details 起動の確認
```bash
docker compose ps
```

`openclaw-gateway`のSTATUSが`Up`になっていればOKです。
:::

---

## ステップ7: ブラウザからアクセスしてペアリング

ここが一番つまずきやすいポイントです。落ち着いて進めてください。

### 1. ブラウザでアクセス

以下のURLを開きます。

```
http://localhost:18789/?token=mytoken
```

`mytoken`の部分は`.env`と`openclaw.json`で設定したトークンに置き換えてください。

### 2. Connectを押す

画面が表示されたら**「Connect」ボタン**を押します。

### 3. Gateway Tokenを入力

Gateway Tokenの入力欄に`mytoken`（設定したトークン）を入力します。

### 4. ペアリングリクエストを承認

Ubuntuのターミナルに戻ると、ペアリングリクエストが表示されています。**承認**してください。

これでOpenClawが使えるようになります！

---

## よくあるトラブルと解決法

### docker compose upでエラーが出る

`.env`ファイルの内容に誤りがないか確認してください。特にAPIキーの前後に余計なスペースが入っていないか注意です。

### ブラウザでアクセスできない

Dockerコンテナが正常に起動しているか確認してください。

```bash
docker compose logs openclaw-gateway
```

### ペアリングリクエストが表示されない

`openclaw.json`の`token`と`.env`の`OPENCLAW_GATEWAY_TOKEN`が一致しているか確認してください。

---

## 学んだこと: AIに環境構築を聞くときのコツ

筆者はこの環境構築に**丸1日**かかりました。GeminiやClaudeに質問しても、とんちんかんな回答ばかり返ってきたからです。

OpenClawのような急成長プロジェクトはAIの学習データに含まれていないことが多く、AIが「それっぽいけど間違った手順」を返してきます。

**解決策: AIに公式ドキュメントを読ませる**

「公式ドキュメント（https://docs.openclaw.ai/）を読んで正しい手順を教えてください」と指示したところ、正確な回答が返ってきました。

新しいツールの環境構築でAIを使うときは、**まず公式ドキュメントのURLを渡す**のがおすすめです。

---

## まとめ

この記事では、OpenClawをDockerで安全にセットアップする手順を紹介しました。

手順をおさらいすると:

1. **リポジトリのクローン** — GitHubから取得
2. **ディレクトリ作成** — config/とworkspace/
3. **.envの作成** — APIキーとトークンの設定
4. **openclaw.jsonの作成** — モデルと認証の設定
5. **docker-compose.ymlの編集** — APIキーをコンテナに渡す
6. **Docker起動** — `docker compose up -d`
7. **ペアリング** — ブラウザからアクセスして承認

Dockerでサンドボックス化することで、セキュリティリスクを最小限に抑えながらAIエージェントを試せます。OpenClawに興味がある方は、ぜひこの手順で始めてみてください。

---

**参考リンク**:
- [OpenClaw公式サイト](https://openclaw.ai/)
- [OpenClaw GitHub リポジトリ](https://github.com/openclaw/openclaw)
- [OpenClaw公式ドキュメント](https://docs.openclaw.ai/)
- [OpenClaw Docker構成（coollabsio）](https://github.com/coollabsio/openclaw)
