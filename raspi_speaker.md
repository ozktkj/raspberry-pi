# ラズパイ講座 スピーカー PWMで音を鳴らす

**時間：** 90分  
**使用機材：** Raspberry Pi Compute Module 5（CM5）+ CM5 I/O Board  
**接続方法：** RaspberryConnect でリモートターミナル接続  
**使用パーツ：** スピーカー、オーディオパワーアンプモジュール（LC Technol V576）、ジャンパーワイヤー

---

## ⚠️ 危険事項（作業前に必ず確認すること）

| 禁止事項 | 理由 |
|---------|------|
| GPIO2（物理ピン3）・GPIO3（物理ピン5）に配線しない | CM5内部のI2C通信バスに接続されており、システムがハングする |
| 5V（物理ピン2/4）をGPIOに接続しない | GPIOは3.3V動作。5V接続で即座に破損する |
| アンプモジュールの + と − を逆に接続しない | 逆接続でモジュールが破損する |

---

## インデックス

1. 導入・PWMと音の仕組み
2. Step 1：アンプモジュールのピンを確認する
3. Step 2：配線して動作確認する
4. Step 3：Pythonプログラムを自分で考える（3-1〜3-4）
5. 動作確認・発展課題

---

### PWMと音の仕組み

これまでGPIOはLEDのON/OFFに使ってきた。  
**PWM（Pulse Width Modulation）** を使うと、GPIOを高速にON/OFFすることができる。

```
ON/OFFの速さ（周波数）が音程を決める
周波数が高い → 高い音
周波数が低い → 低い音
```

スピーカー単体では音が非常に小さいため、**アンプモジュール**でGPIOの信号を増幅してからスピーカーに送る。

```
CM5 GPIO → [アンプモジュール] → スピーカー
（弱い信号）   （信号を増幅）     （音が出る）
```

---

## Step 1：アンプモジュールのピンを確認する

### アンプモジュールとは

アンプ（増幅器）は、入力された弱い電気信号を大きくして出力する。  
スマートフォンのスピーカーや音楽プレイヤーにも必ず入っている部品。

### 準備

**Q1. アンプモジュールには何本の端子があるか？実物を見て確認しよう。**

**Q2. 各端子の名前（VCC・GND・IN・OUT など）はどこに書いてあるか？**

**Q3. スピーカーをアンプモジュールに接続するには、どの端子を使うか？**

### 端子の役割

アンプモジュールは通常、以下の端子を持っている：

<img src="speakeramp.png" width="180" align="right">

| 端子名 | 役割 |
|-------|------|
| VCC | 電源（モジュールを動かすための電力） |
| GND | グランド（0V基準）。2箇所ある |
| L | 左チャンネルの音声信号入力 |
| R | 右チャンネルの音声信号入力 |
| V1 / V2 | スピーカー出力（＋と－） |

<br clear="right">

---

## Step 2：配線して動作確認する

### 配線する

| アンプモジュールの端子 | 接続先 | 物理ピン |
|------------------|-------|---------|
| VCC | 5V | **物理ピン2** |
| GND（どちらか1つ） | GND | **物理ピン6** |
| L（左音声入力） | GPIO18 | **物理ピン12** |
| R（右音声入力） | 接続しない | ─ |
| V1（スピーカー+） | スピーカー（+） | ─ |
| V2（スピーカー−） | スピーカー（−） | ─ |

> ⚠️ GPIO18は**ハードウェアPWM対応ピン**。音声信号の出力に適している。

```
物理ピン配置（参考）
奇数列  偶数列
  1: 3.3V  │  2: 5V   ← VCC
  3: SDA   │  4: 5V
  5: SCL   │  6: GND  ← GND
  7: GPIO4 │  8: TXD
  9: GND   │  10: RXD
 11: GPIO17│  12: GPIO18  ← IN
```

### 考察

**Q4. 電源（VCC）を接続するとアンプモジュールに何か変化があるか？**

### コマンドで動作確認

GPIO18がPWM出力できることを確認する：

```bash
pinctrl get 18
```

```bash
# GPIO18をPWM出力モードにして音を出す前に状態確認
pinctrl set 18 op dh
```

> 期待出力例：`18: op -- | hi`

---

## Step 3：Pythonプログラムを自分で考える

### 使うライブラリ・クラス

| コード | 内容 |
|-------|------|
| `TonalBuzzer(18)` | GPIO18にPWM音声出力オブジェクトを作成 |
| `speaker.play(Tone(frequency=Hz))` | 指定した周波数（Hz）で音を鳴らす |
| `speaker.stop()` | 音を止める |

インポート方法：

```python
import os
os.environ['GPIOZERO_PIN_FACTORY'] = 'lgpio'  # CM5では必要

from gpiozero import TonalBuzzer
from gpiozero.tones import Tone
import time
```

> **`GPIOZERO_PIN_FACTORY = 'lgpio'` について**  
> gpiozeroはGPIOを操作するバックエンドを自動選択するが、CM5が搭載する**RP1チップ**には`lgpio`バックエンドが必要。この1行をimportより前に書くことでCM5でも動作する。

> `TonalBuzzer` はパッシブブザーと同じクラス。スピーカーもアンプを通せば同じ方法で鳴らせる。

### 音階の周波数

| 音名 | 周波数（Hz） |
|------|------------|
| ド（C4） | 262 |
| レ（D4） | 294 |
| ミ（E4） | 330 |
| ファ（F4） | 349 |
| ソ（G4） | 392 |
| ラ（A4） | 440 |
| シ（B4） | 494 |
| ド（C5） | 523 |

### 3-1：音を1つ鳴らすプログラム（動作サンプル）

まず、以下のサンプルをそのまま入力して動作を確認しよう。  
ドの音（262Hz）を1秒間鳴らすプログラム。(speaker_test.py)

```python
import os
os.environ['GPIOZERO_PIN_FACTORY'] = 'lgpio'

from gpiozero import TonalBuzzer
from gpiozero.tones import Tone
import time

speaker = TonalBuzzer(18)

speaker.play(Tone(frequency=262))  # ド（C4）
time.sleep(1)
speaker.stop()
```

```bash
nano speaker_test.py
python3 speaker_test.py
```

**応用：違う音階を鳴らしてみよう**

上の音階の周波数表を参考に、`262` の部分を別の値に変えて再実行してみよう。

**Q. ラ（A4）の音を鳴らすには、何Hzに変えればよいか？**

### 3-2：音階を順番に鳴らすプログラム(speaker_scale.py)

ド〜ドまで8音を順番に鳴らすプログラムを作ってみよう。

```
[ 処理の流れを考えよう ]

1. 音階の周波数をリストにまとめる
2. TonalBuzzer オブジェクトを作る
3. リストの音を1つずつ0.4秒ずつ鳴らす
4. 全部鳴らしたら止める
```

```bash
nano speaker_scale.py
python3 speaker_scale.py
```

### 3-3：ボタンを押したらラの音を鳴らすプログラム(speaker_button.py)

タクトスイッチを追加して、ボタンを押したときだけ音が鳴るプログラムを作ろう。

追加する配線：

| パーツ | 端子 | 接続先 | 物理ピン |
|-------|------|-------|---------|
| タクトスイッチ | 片側 | GPIO17 | 物理ピン11 |
| タクトスイッチ | 片側 | GND | 物理ピン14 |

**ステップ①：まずボタンが押されていることを確認する**

以下のプログラムを実行し、ボタンを押したとき `pressed: True` に変わることを確認しよう。

```python
import os
os.environ['GPIOZERO_PIN_FACTORY'] = 'lgpio'

from gpiozero import Button
import time

button = Button(17, pull_up=True)

while True:
    print("pressed:", button.is_pressed)
    time.sleep(0.2)
```

```bash
nano button_check.py
python3 button_check.py
```

- `pressed: False` → 離している
- `pressed: True` → 押している

切り替わることを確認したら `Ctrl + C` で終了し、次へ進む。

---

**ステップ②：押されたら音を鳴らすプログラムを作る**

```
[ 処理の流れを考えよう ]

1. TonalBuzzer と Button のオブジェクトを作る
2. ボタンを押したとき：ラ（440Hz）を0.3秒鳴らす
3. pause() で待機する
```

```bash
nano speaker_button.py
python3 speaker_button.py
```

終了するには `Ctrl + C`

---

### 3-4：ミニピアノ（ボタンで音程を変える）(mini_piano.py)

3-3の配線はそのままで、押すたびに音程が1段階上がるプログラムに改造しよう。  
最高音（ド・C5）の次は最低音（ド・C4）に戻る。

```
[ 処理の流れを考えよう ]

1. 音階のリストを作る（8音分の周波数）
2. 現在の音程を記録する変数を用意する
3. TonalBuzzer と Button のオブジェクトを作る
4. ボタンを押したとき：
   a. 現在の音程で0.3秒鳴らす
   b. 次の音程に進める（最後の次は最初に戻る）
5. pause() で待機する
```

> ヒント：リストを循環させるには `index = (index + 1) % len(notes)` を使う。

```bash
nano mini_piano.py
python3 mini_piano.py
```

終了するには `Ctrl + C`

---

## 本日のまとめ

| 学んだこと | 内容 |
|-----------|------|
| PWMと音 | ON/OFFの周波数で音程が決まる |
| アンプモジュール | 弱いGPIO信号を増幅してスピーカーを鳴らす |
| TonalBuzzer | gpiozeroのクラスでPWM音声出力を制御 |
| 周波数と音階 | 262Hzがドで、数値が大きいほど高い音 |
| リストの循環 | `% len()` でリストをループさせる |

---

*神戸テック ラズパイ講座 スピーカー PWMで音を鳴らす*
