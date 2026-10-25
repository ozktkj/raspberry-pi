# ラズパイ講座 データ記録 Session 1：CSV出力

**使用機材：** Raspberry Pi Compute Module 5（CM5）+ CM5 I/O Board  
**接続方法：** RaspberryConnect でリモートターミナル接続  
**前提：** RFIDリーダーが動作していること（raspi_rfid.md参照）

---

## インデックス

1. 導入・なぜデータを記録するか
2. Step 1：csvモジュールでログを書き出す
3. Step 2：csvモジュールでログを読み込む
4. Step 3：PandasでCSVを出力する
5. Step 4：RFIDリーダーと統合する
6. 動作確認・発展課題

---

## 導入

### 今回作るもの

**RFIDカードの感知ログを自動記録するシステム**

ゴールの動作：
```
カードをかざす
 → UID・名前・結果・日時を CSV に記録する
 → 後からPandasでログを集計・分析できる
```

記録するデータの形式：
```
日時,UID,名前,結果
2026-10-25 14:30:01,123-45-67-89-0,Alice,ALLOWED
2026-10-25 14:31:05,99-88-77-66-0,未登録,DENIED
2026-10-25 14:35:22,123-45-67-89-0,Alice,ALLOWED
```

### なぜデータを記録するか

センサーの値をターミナルに表示するだけでは、プログラムを終了すると消えてしまう。  
データをファイルに保存すると：

- 過去のログをいつでも確認できる
- 集計・グラフ化・分析ができる
- 他のシステムと連携できる

これがIoTシステムの基本的な流れ：

```
センシング → 記録 → 分析 → 活用
```

---

## Step 1：csvモジュールでログを書き出す

### csvとは

CSV（Comma-Separated Values）はカンマ区切りのテキストファイル。  
Excelやスプレッドシートで開けるシンプルな形式。

```
日時,UID,名前,結果                  ← ヘッダー行
2026-10-25 14:30:01,123-45-67,Alice,ALLOWED  ← データ行
```

### 使うモジュール

`csv` モジュールはPython標準ライブラリ（インストール不要）。

| コード | 内容 |
|-------|------|
| `csv.writer(f)` | CSVに書き込むオブジェクトを作成 |
| `writer.writerow([...])` | 1行書き込む |
| `csv.DictWriter(f, fieldnames=[...])` | 列名付きで書き込む |
| `writer.writeheader()` | ヘッダー行を書き込む |

### 1-1：1行書き出してみる（動作サンプル）

```python
import csv
from datetime import datetime

filename = 'rfid_log.csv'

with open(filename, 'w', newline='') as f:
    writer = csv.writer(f)
    writer.writerow(['日時', 'UID', '名前', '結果'])  # ヘッダー
    writer.writerow([
        datetime.now().strftime('%Y-%m-%d %H:%M:%S'),
        '123-45-67-89-0',
        'Alice',
        'ALLOWED'
    ])

print(f"{filename} に書き込みました")
```

```bash
nano csv_write_test.py
python3 csv_write_test.py
cat rfid_log.csv
```

**Q1. `open(filename, 'w')` の `'w'` を `'a'` に変えると何が変わるか？**

### 1-2：追記モードで複数行書き出す

毎回ファイルを上書きすると過去のデータが消えてしまう。  
追記モード（`'a'`）を使うと、既存の内容を保ちながら行を追加できる。

```
[ 処理の流れを考えよう ]

1. ファイルが存在しない場合：ヘッダーを書き込む
2. ファイルが存在する場合：ヘッダーなしで追記する
```

> ヒント：
> ```python
> import os
> file_exists = os.path.exists(filename)
> ```

```bash
nano csv_append.py
python3 csv_append.py
python3 csv_append.py  # 2回実行して追記されるか確認
cat rfid_log.csv
```

**Q2. `newline=''` はなぜ必要か？**

---

## Step 2：csvモジュールでログを読み込む

### 読み込みに使うコード

| コード | 内容 |
|-------|------|
| `csv.reader(f)` | CSVを1行ずつ読み込む |
| `csv.DictReader(f)` | 列名をキーとした辞書で読み込む |

### 2-1：CSVを読み込んで表示する（動作サンプル）

```python
import csv

with open('rfid_log.csv', 'r') as f:
    reader = csv.DictReader(f)
    for row in reader:
        print(f"{row['日時']}  {row['名前']}  {row['結果']}")
```

```bash
nano csv_read.py
python3 csv_read.py
```

**Q3. ALLOWEDの行だけ表示するにはどうすればよいか？**

---

## Step 3：PandasでCSVを出力する

### Pandasとは

Pandas はデータ分析用のPythonライブラリ。

```bash
pip install pandas --break-system-packages
```

### 3-1：PandasでCSVに書き出す

```python
import pandas as pd
from datetime import datetime

# データを辞書のリストで用意する
data = [
    {'日時': '2026-10-25 14:30:01', 'UID': '123-45-67-89-0', '名前': 'Alice', '結果': 'ALLOWED'},
    {'日時': '2026-10-25 14:31:05', 'UID': '99-88-77-66-0',  '名前': '未登録', '結果': 'DENIED'},
]

df = pd.DataFrame(data)
df.to_csv('rfid_pandas.csv', index=False)
print("書き込みました")
```

```bash
nano pandas_write.py
python3 pandas_write.py
cat rfid_pandas.csv
```

### 3-2：PandasでCSVに追記する

追記（新しい行を1行加える）をPandasで行う場合：

```python
import pandas as pd
import os
from datetime import datetime

filename = 'rfid_pandas.csv'

# 既存ファイルを読み込む（なければ空のDataFrameを作る）
if os.path.exists(filename):
    df = pd.read_csv(filename)
else:
    df = pd.DataFrame(columns=['日時', 'UID', '名前', '結果'])

# 新しい行を追加
new_row = {
    '日時': datetime.now().strftime('%Y-%m-%d %H:%M:%S'),
    'UID': '123-45-67-89-0',
    '名前': 'Alice',
    '結果': 'ALLOWED'
}
df = pd.concat([df, pd.DataFrame([new_row])], ignore_index=True)
df.to_csv(filename, index=False)
```

**Q4. Pandasで1行追記するとき、なぜ「ファイル全体を読んで書き直す」手順が必要なのか？**

### まとめ：csvモジュールとPandasの比較

| 観点 | csvモジュール | Pandas |
|------|-------------|--------|
| インストール | 不要（標準ライブラリ） | 必要（重い） |
| 追記の書き方 | `open('a')` で1行追加するだけ | 全体を読んで書き直す |
| メモリ使用 | 少ない（1行ずつ処理） | 全データをメモリに読み込む |
| 向いている用途 | ログの**書き出し** | データの**加工・集計・分析** |

> **結論：**  
> CSVにログを書き出すだけなら `csv` モジュールで十分。  
> Pandasはデータを加工・集計・グラフ化したいときに使う。  
> 「とりあえずPandas」は便利に見えるが、ラズパイのような非力な環境では  
> 起動が遅くメモリも消費するため、目的に合ったライブラリを選ぶことが重要。

---

## Step 4：RFIDリーダーと統合する

### 4-1：RFID + CSV記録の完成プログラム

```
[ 処理の流れを考えよう ]

1. RFIDリーダー・LCD・登録済みUID辞書を初期化する
2. CSVファイルが存在しなければヘッダーを書き込む
3. カードを検知したとき：
   a. UIDを照合する
   b. 結果をLCDに表示する
   c. 日時・UID・名前・結果をCSVに追記する
   d. ターミナルにも出力する
```

> ヒント：ログ書き込みは関数にまとめると呼び出しやすい
> ```python
> def log_to_csv(uid_str, name, result):
>     with open('rfid_log.csv', 'a', newline='') as f:
>         ...
> ```

```bash
nano rfid_with_csv.py
python3 rfid_with_csv.py
```

数回カードをかざした後、`Ctrl+C` で終了して記録を確認する：

```bash
cat rfid_log.csv
```

---

## 動作確認・発展課題

### 動作確認

- [ ] CSVに1行書き出せた
- [ ] 追記モードで複数回実行してもデータが重複しなかった
- [ ] `DictReader` でCSVを読み込んで表示できた
- [ ] Pandasで件数・名前別集計ができた
- [ ] RFIDと統合してカードの記録が残った

---

### 発展課題

**課題A：DENIEDのログだけ別ファイルに書き出す**  
DENIED が発生した場合のみ `denied_log.csv` に別途記録するプログラムを作ろう。

**課題B：古いログを自動削除する**  
記録が1000件を超えたら、最も古い100件を削除して件数を管理するプログラムを作ろう。

---

## 本日のまとめ

| 学んだこと | 内容 |
|-----------|------|
| csvモジュール | Pythonでのファイル書き出し・読み込みの基本 |
| 追記モード (`'a'`) | 既存データを保ちながら行を追加する |
| Pandas DataFrame | 表形式でデータを管理・分析する、CSVの読み書きもできる |
| データの絞り込み | 条件フィルタ・集計でログを活用する |

---

*神戸テック ラズパイ講座 データ記録 Session 1：CSV出力*
