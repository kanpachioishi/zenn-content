---
title: "Raspberry Piにシャットダウンボタンを付ける。安全に電源を切る仕組みを作った。"
emoji: "🔘"
type: "tech"
topics: ["raspberrypi", "python", "gpio", "linux", "電子工作"]
published: true
---

Raspberry Pi は小さくて便利ですが、ひとつ地味に面倒なのが「電源の切り方」です。  
PC のように適当に電源を落とすわけにはいかず、基本的には安全にシャットダウンしてから電源を切る必要があります。

普段は左上のメニューボタンからシャットダウンできますが、これも毎回やるとなると意外と面倒です。  
特に、ちょっとした実験や工作でラズパイを何度も起動したり止めたりしていると、「もう電源を切りたいだけなのに」と思う場面が何度もありました。

そこで今回は、**物理ボタンを押すと Raspberry Pi が安全にシャットダウンする仕組み**を作ってみました。  
GPIO でボタン入力を検知し、プログラムからシャットダウン処理を呼び出すだけのシンプルな構成ですが、実際に使ってみるとかなり快適です。

この記事では、仕組みの考え方、配線、プログラム、起動時に自動で動かす設定までまとめて紹介します。

## なぜ Raspberry Pi はそのまま電源を切ってはいけないのか

Raspberry Pi は SD カードに OS や設定ファイルを書き込みながら動いています。  
そのため、動作中にいきなり電源を抜くと、次のような問題が起きる可能性があります。

- ファイルシステムが壊れる
- ログや設定ファイルの書き込み中に破損する
- 最悪の場合、SD カードの再セットアップが必要になる

一度や二度は問題なく見えても、これを繰り返すと後で効いてきます。  
なので、「安全に止めてから電源を切る」仕組みを作っておく価値があります。

## 今回作った仕組み

全体像はかなりシンプルです。

1. ボタンを押す
2. GPIO で押下を検知する
3. Python スクリプトがシャットダウン処理を呼ぶ
4. LED を消灯して、OS を安全に停止する

今回は以下のピンを使いました。

- ボタン入力: `GPIO21`
- 黄色 LED: `GPIO10`

コードでは `GPIO.PUD_UP` を使っているので、ボタン入力は内部プルアップです。  
つまり、**通常時は HIGH、ボタンを押したときに LOW になる構成**です。

## 用意したもの

- Raspberry Pi
- タクトスイッチなどの押しボタン
- LED
- 電流制限用の抵抗
- ジャンパーワイヤー

今回はブレッドボード上で試せるシンプルな構成です。  
部品点数が少ないので、GPIO を使った小さな工作の題材としても扱いやすいと思います。

## 配線

考え方としては次の構成です。

- ボタンは `GPIO21` と `GND` の間に接続する
- LED は `GPIO10` 側から抵抗を挟んで接続する
- 起動中は LED を点灯させ、シャットダウン処理に入ったら消灯させる

ボタン側は内部プルアップを使っているので、外部のプルアップ抵抗は不要です。  
一方で LED 側は電流制限抵抗を入れておくのが前提です。

今回は BCM 番号で書くと、

- `GPIO21` は物理ピン `40`
- `GPIO10` は物理ピン `19`
- `GND` は物理ピン `39` と `20`

です。

配線図はこんな感じです。

![Raspberry Pi 40-pin header and wiring diagram](/images/raspberry-pi-shutdown-pinout.png)

ポイントは次の 2 つです。

- ボタンは `GPIO21` と `GND` をつなぐだけ
- LED は `GPIO10` から直接つながず、必ず抵抗を挟んでから `GND` に落とす

内部プルアップを使っているので、ボタン側に外付け抵抗は入れていません。  
その代わり、LED 側には電流制限抵抗を入れる前提です。

今回の配線では、ボタンは **物理ピン 40 と物理ピン 39** を使うと配線しやすいです。  
LED は **物理ピン 19 と物理ピン 20** を使うと、これも隣同士なのでまとめやすくなります。

## 実際のコード

今回使ったコードはこれです。

```python
import RPi.GPIO as GPIO
import subprocess
import time

BUTTON_PIN = 21
YELLOW_LED_PIN = 10

GPIO.setmode(GPIO.BCM)
GPIO.setup(BUTTON_PIN, GPIO.IN, pull_up_down=GPIO.PUD_UP)
GPIO.setup(YELLOW_LED_PIN, GPIO.OUT)

# 起動時に点灯
GPIO.output(YELLOW_LED_PIN, GPIO.HIGH)

def shutdown(channel):
    time.sleep(0.5)
    if GPIO.input(BUTTON_PIN) == GPIO.LOW:
        GPIO.output(YELLOW_LED_PIN, GPIO.LOW)  # 消灯
        subprocess.run(["shutdown", "-h", "now"])

GPIO.add_event_detect(BUTTON_PIN, GPIO.FALLING,
                      callback=shutdown, bouncetime=300)

try:
    while True:
        time.sleep(1)
except KeyboardInterrupt:
    GPIO.cleanup()
```

やっていることを順番に見ると、そこまで難しくありません。

### `GPIO21` をボタン入力として使う

```python
GPIO.setup(BUTTON_PIN, GPIO.IN, pull_up_down=GPIO.PUD_UP)
```

`GPIO.PUD_UP` を指定しているので、通常時は HIGH です。  
ボタンを押して `GND` 側に落ちると LOW になり、押下を検知できます。

### `GPIO10` を LED 出力として使う

```python
GPIO.setup(YELLOW_LED_PIN, GPIO.OUT)
GPIO.output(YELLOW_LED_PIN, GPIO.HIGH)
```

起動時に LED を点灯させて、「今は動作中ですよ」と分かるようにしています。  
シャットダウン処理に入る直前で消灯するので、見た目にも状態が分かりやすいです。

### ボタンが押されたらシャットダウンする

```python
def shutdown(channel):
    time.sleep(0.5)
    if GPIO.input(BUTTON_PIN) == GPIO.LOW:
        GPIO.output(YELLOW_LED_PIN, GPIO.LOW)
        subprocess.run(["shutdown", "-h", "now"])
```

ポイントは 2 つあります。

1 つ目は、**すぐにシャットダウンせず 0.5 秒待っていること**です。  
押した瞬間のノイズや誤検知を減らすため、少し待ってからもう一度入力を確認しています。

2 つ目は、**電源を切るのではなく OS のシャットダウンを呼んでいること**です。  
これで Raspberry Pi を安全に停止できます。

### チャタリング対策

```python
GPIO.add_event_detect(BUTTON_PIN, GPIO.FALLING,
                      callback=shutdown, bouncetime=300)
```

機械式ボタンは、押した瞬間に信号が細かく揺れることがあります。  
これをチャタリングと呼びます。

`bouncetime=300` を入れておくと、300ms の間は連続した入力を無視してくれるので、誤作動しにくくなります。  
さらに今回のコードでは `time.sleep(0.5)` で再確認もしているので、かなり保守的な作りです。

### 終了時に GPIO を解放する

```python
try:
    while True:
        time.sleep(1)
except KeyboardInterrupt:
    GPIO.cleanup()
```

今回は常駐スクリプトなので、基本的にはずっと動かしっぱなしです。  
ただし、手動で停止したときに GPIO の状態をきれいに戻せるよう、`GPIO.cleanup()` も入れています。

## 起動時に自動で実行する設定

この仕組みは、起動するたびに毎回動いてくれないと意味がありません。  
そこで今回は、**`systemd` で Python スクリプトを自動起動**する構成にします。

スクリプトは以下のような、自分のホームディレクトリ配下に置く前提にします。

```bash
/home/<your-user>/pi/shutdown_button.py
```

### 1. service ファイルを作る

```bash
sudo nano /etc/systemd/system/shutdown-button.service
```

中身はこんな感じです。

```ini
[Unit]
Description=Shutdown button service
After=multi-user.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 /home/<your-user>/pi/shutdown_button.py
Restart=always
WorkingDirectory=/home/<your-user>/pi

[Install]
WantedBy=multi-user.target
```

### 2. systemd に読み込ませる

```bash
sudo systemctl daemon-reload
```

### 3. 自動起動を有効にする

```bash
sudo systemctl enable shutdown-button.service
```

これで、Raspberry Pi を起動するたびに `shutdown_button.py` が自動で立ち上がります。

### 4. その場で起動して確認する

```bash
sudo systemctl start shutdown-button.service
sudo systemctl status shutdown-button.service
```

`active (running)` になっていれば OK です。

### 5. 設定内容をあとから確認する方法

起動時設定を見返したくなったら、以下のコマンドで確認できます。

```bash
systemctl status shutdown-button.service
systemctl cat shutdown-button.service
```

「本当に起動時に有効なのか」を見るなら、これも便利です。

```bash
systemctl list-unit-files --type=service | grep shutdown-button
```

## 補足: `sudo` はそのままでも動くが、整理するなら外してもいい

今回の Python コードでは、シャットダウン時に次のコマンドを呼んでいます。

```python
subprocess.run(["shutdown", "-h", "now"])
```

これでも動く可能性は高いのですが、`systemd` サービスとして root 権限で動かすなら、`sudo` はなくても実行できます。

例えば、次のようにしても十分です。

```python
subprocess.run(["shutdown", "-h", "now"])
```

今回のような小さな仕組みでは、まずは動く形を優先して、その後に整理するでも十分だと思います。

## 実際に使ってみた感想

実際に使ってみると、想像以上に快適です。

ラズパイを触っていると、「今すぐ止めたい」「確認が終わったからもう落としたい」という瞬間が何度もあります。  
そのたびに画面を開いてメニューをたどるより、物理ボタンを 1 回押すだけで済むほうが圧倒的に楽でした。

「そんな小さな手間か」と思うかもしれませんが、こういう小さな面倒をなくすと、工作や実験そのものに集中しやすくなります。

## 注意点

いくつか注意点もあります。

- シャットダウン完了前に電源を抜かない
- LED が消えたあとも、すぐに電源断せず少し待つ
- ボタンの配線ミスをすると誤動作する
- GPIO に LED をつなぐときは電流制限抵抗を入れる

また、長押し動作や「押したら数秒後に完全消灯する」ような仕組みまで作ると、さらに実用品っぽくできます。

## まとめ

Raspberry Pi は便利ですが、安全に止めるには少し気を遣います。  
そこで、GPIO と Python を使ってシャットダウンボタンを作っておくと、普段の運用がかなり楽になります。

今回の仕組みはシンプルですが、

- ボタン押下を検知する
- LED で状態を見せる
- OS のシャットダウンを呼ぶ
- 起動時に自動実行する

という流れが一通りそろっているので、実用性は十分です。

ラズパイを日常的に触っている方は、こういう「ちょっとした不便をなくす工作」を 1 つ作っておくとかなり快適になると思います。
