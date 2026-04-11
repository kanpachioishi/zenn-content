---
title: "Raspberry PiをPCからリモート操作する。VNCでデスクトップ接続する手順まとめ"
emoji: "🖥️"
type: "tech"
topics: ["raspberrypi", "vnc", "ssh", "linux", "network"]
published: false
---

Raspberry Pi を触っていると、毎回ディスプレイやキーボードをつなぐのがだんだん面倒になってきます。  
「電源を入れたら、普段使っている PC からそのまま操作したい」と思う場面はかなり多いはずです。

そんなときに便利なのが、**VNC を使ったリモートデスクトップ接続**です。  
VNC を使えば、同じネットワーク内にある Raspberry Pi のデスクトップ画面を、Windows / Mac / Linux の PC からそのまま操作できます。

この記事では、**Raspberry Pi OS Desktop を使っている前提で、同じ LAN 内の PC から Raspberry Pi に VNC 接続する手順**をまとめます。  
あわせて、「SSH と何が違うのか」「外出先からつなぐなら何を使うべきか」も整理します。

## まず結論

Raspberry Pi のリモート接続は、やりたいことによって使い分けるのが分かりやすいです。

| やりたいこと | 向いている方法 | 特徴 |
| --- | --- | --- |
| コマンドだけ操作できればいい | SSH | 軽い。サーバー用途向き |
| デスクトップ画面ごと操作したい | VNC | GUI アプリや設定画面も扱いやすい |
| 外出先から簡単につなぎたい | Raspberry Pi Connect | IP アドレス確認やポート開放を減らせる |

今回はこの中でも、**「同じ家やオフィスのネットワーク内で、GUI ごと操作したい」**という用途に向いた VNC を扱います。

## この記事の前提

- Raspberry Pi OS の **Desktop 版** を使っている
- Raspberry Pi と接続元 PC が **同じネットワーク**にいる
- すでに Raspberry Pi の初期設定とユーザー作成が終わっている
- ブラウザや設定画面など、**デスクトップ GUI をそのまま使いたい**

CLI だけでよければ SSH のほうが軽いです。  
また、Raspberry Pi OS Lite のように GUI を前提にしない構成なら、まずは SSH や Raspberry Pi Connect を選ぶほうが自然です。

## 手順1. Raspberry Pi 側で VNC を有効にする

Raspberry Pi OS には VNC サーバー機能が含まれています。  
まずは Raspberry Pi 側でこれを有効にします。

### GUI から有効にする

2026-04-08 時点の Raspberry Pi 公式ドキュメントでは、次の手順が案内されています。

1. Raspberry Pi をデスクトップで起動する
2. 右上またはメニュー側の Raspberry Pi メニューから `Preferences > Control Centre` を開く
3. `Interfaces` タブを開く
4. `VNC` を有効化して保存する

OS バージョンによっては、画面名が少し違うことがあります。  
たとえば古めの Bookworm 系では、`設定 -> Raspberry Pi の設定` の中に `インターフェイス` タブが見える場合があります。

大事なのは、**インターフェイス設定の中で VNC を ON にすること**です。

### コマンドラインから有効にする

ディスプレイにつながずに設定したい場合は、`raspi-config` でも有効化できます。

```bash
sudo raspi-config
```

開いたら、以下の順に進みます。

1. `Interface Options`
2. `VNC`
3. `Yes`

SSH で入れる状態なら、この方法がいちばん確実です。

## 手順2. Raspberry Pi の接続先を確認する

VNC クライアントから接続するには、Raspberry Pi の **IP アドレス** か **ホスト名** が必要です。

### いちばん簡単なのは `hostname -I`

ターミナルで次のコマンドを実行します。

```bash
hostname -I
```

すると、たとえば次のような IP アドレスが表示されます。

```text
192.168.1.42
```

このアドレスを、あとで VNC クライアントに入力します。

### `raspberrypi.local` でつながることもある

Raspberry Pi OS は mDNS に対応しているので、接続元の PC 側が対応していれば、IP アドレスの代わりに次のようなホスト名で接続できることがあります。

```text
raspberrypi.local
```

ホスト名を変更している場合は、`<hostname>.local` になります。

### IP が変わるのが困るなら

DHCP で自動割り当てしていると、たまに IP アドレスが変わることがあります。  
毎回探したくないなら、次のどちらかをやっておくと楽です。

- ルーター側で DHCP 予約をする
- `raspberrypi.local` での接続を使う

個人的には、ラズパイ側で無理に静的 IP を設定するより、**ルーター側で予約するほうが運用は楽**です。

## 手順3. 接続元 PC に VNC クライアントを入れる

2026-04-08 時点の Raspberry Pi 公式ドキュメントでは、クライアント側として **TigerVNC** が案内されています。

- Windows: `exe`
- macOS: `dmg`
- Linux: `jar`

ダウンロード先:

- [TigerVNC](https://tigervnc.org/)
- [TigerVNC Releases](https://github.com/TigerVNC/tigervnc/releases)

Windows と macOS は普通のアプリと同じ感覚で入れれば大丈夫です。  
Linux では Java 実行環境が必要になるので、未導入なら次を入れておきます。

```bash
sudo apt install default-jre
```

そのうえで、ダウンロードした `jar` を実行します。

```bash
java -jar VncViewer-<version>.jar
```

以前の解説記事では RealVNC Viewer を使っているものも多いです。  
それでも考え方はほぼ同じですが、**今から新しく試すなら、まずは公式ドキュメントに合わせて TigerVNC で進める**のが分かりやすいです。

## 手順4. 実際に VNC で接続する

TigerVNC を起動したら、接続先に Raspberry Pi の IP アドレスまたはホスト名を入力します。

例:

```text
192.168.1.42
```

または

```text
raspberrypi.local
```

そのまま接続すると、証明書に関する警告が出ることがあります。  
同一 LAN 内で自分の Raspberry Pi に接続しているなら、内容を確認したうえで許可して進めて問題ありません。

次に、Raspberry Pi 側の **ユーザー名とパスワード** を求められるので入力します。  
認証が通れば、Raspberry Pi のデスクトップ画面が PC 上に表示されます。

ここまでできれば、ブラウザ、設定画面、ファイルマネージャー、エディタなどを、ほぼローカル PC と同じ感覚で操作できます。

## 画面が真っ黒、小さい、表示されないとき

Raspberry Pi を**ディスプレイなしのヘッドレス運用**にしていると、VNC まわりで表示が不安定になることがあります。

典型的には次のような症状です。

- 接続できるのに画面が出ない
- 解像度が極端に低い
- 画面が小さすぎて使いにくい

この場合は、まず次を確認してください。

### 1. Raspberry Pi OS Desktop を使っているか

GUI のない Lite 構成だと、そもそも VNC でデスクトップ共有する前提が弱くなります。  
ターミナル操作が目的なら、VNC ではなく SSH のほうが向いています。

### 2. VNC が有効になっているか

`Control Centre` または `raspi-config` で VNC が有効になっているか、もう一度確認します。

### 3. ヘッドレス時の解像度設定を見直す

古めの Raspberry Pi OS では、**ディスプレイ未接続時の解像度設定**が必要になることがあります。  
Bookworm 系の案内では、`Raspberry Pi の設定` の `ディスプレイ` タブから headless resolution を設定する手順がよく使われます。

一方で、参考にした解説でも触れられている通り、**2025年10月以降の Trixie 系ではこの手順が不要なケースもあります**。  
つまり、OS バージョンによって見える画面や必要な調整が少し違います。

迷ったら、

- 新しい Trixie 系: まずは VNC を有効にしてそのまま接続を試す
- Bookworm 系: ヘッドレス解像度設定も疑う

という見方で切り分けると進めやすいです。

## SSH と VNC はどう使い分けるべきか

これはけっこう大事です。

### SSH が向いている場面

- パッケージ更新
- ファイル編集
- サービス設定
- プログラム実行
- サーバー用途

### VNC が向いている場面

- GUI アプリを使いたい
- デスクトップ設定を触りたい
- ブラウザで確認したい
- 初学者向けに画面つきで説明したい

普段の運用では、**基本は SSH、GUI が必要なときだけ VNC** にすると軽くて快適です。

## 外出先からつなぎたいなら VNC より Connect を先に考える

同じ LAN の中なら VNC はとても便利です。  
ただし、外出先から使いたいからといって、**VNC ポートをそのままインターネットへ公開するのはおすすめしません**。

2026-04-08 時点の Raspberry Pi 公式ドキュメントでは、外部からのアクセス方法として **Raspberry Pi Connect** も案内されています。  
Connect なら、ブラウザ経由で画面共有やリモートシェルを使えるので、IP アドレス確認やポート開放を減らしやすいです。

用途の目安はこんな感じです。

- 家の中や社内 LAN で使う: VNC
- 外出先から簡単につなぐ: Raspberry Pi Connect
- 外部公開を厳密に制御したい: VPN + SSH / VNC

## まとめ

Raspberry Pi を毎回モニターにつないで操作するのは、最初はいいのですが、そのうちかなり面倒になります。  
同じネットワーク内で GUI ごと操作したいなら、VNC を使うのがいちばん分かりやすいです。

手順としては、次の流れで十分です。

1. Raspberry Pi 側で VNC を有効にする
2. IP アドレスまたは `raspberrypi.local` を確認する
3. PC 側に VNC クライアントを入れる
4. ユーザー名とパスワードで接続する

まずはここまでできれば、ラズパイ運用の快適さがかなり変わります。  
もし「デスクトップはいらないけど、外からも触りたい」という用途なら、次は SSH や Raspberry Pi Connect もあわせて試してみるといいと思います。

## 参考

- [Raspberry Pi Documentation: Remote access](https://www.raspberrypi.com/documentation/computers/remote-access.html)
- [Raspberry Pi Documentation: Raspberry Pi Connect](https://www.raspberrypi.com/documentation/services/connect.html)
- [Indoor Corgi: VNCでRaspberry Piにリモートデスクトップ接続 (Windows/Mac/Linux対応)](https://www.indoorcorgielec.com/resources/raspberry-pi/raspberry-pi-vnc/)
