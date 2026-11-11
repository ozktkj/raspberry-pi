# ラズパイ講座 データ記録 Session 2：DBに記録する

**使用機材：** Raspberry Pi Compute Module 5（CM5）+ CM5 I/O Board  
**接続方法：** RaspberryConnect でリモートターミナル接続  
**前提：** Session 1（CSV出力）修了

---

## インデックス

1. 導入・CSVとデータベースの違い
2. Step 1：SQLiteにデータを格納する
3. Step 2：MySQLをインストールして設定する
4. Step 3：PythonからMySQLに接続する
5. Step 4：RFIDリーダーと統合する
6. 動作確認・発展課題

---

## 導入

### CSVとデータベースの違い

| 項目 | CSV | SQLite | MySQL |
|------|-----|--------|-------|
| 形式 | テキストファイル | バイナリファイル（1ファイル） | サーバープロセス |
| 検索速度 | 全行を読む | インデックスで高速 | インデックスで高速 |
| 複数クライアント | 非推奨 | 限定的 | 対応 |
| 認証・権限 | なし | なし | ユーザー管理あり |
| 導入コスト | 不要 | Python標準搭載 | インストールが必要 |
| 向いている規模 | 小 | 小〜中 | 中〜大 |

### 今回の流れ

```
Session 1：CSV（テキストファイル）
    ↓
Session 2 Step1：SQLite（ファイルDB）
    ↓
Session 2 Step2以降：MySQL（サーバーDB）
```

---

## Step 1：SQLiteにデータを格納する

### SQLiteとは

**サーバー不要・ファイル1つで動く軽量データベース。**  
Pythonに標準搭載されており、追加インストールなしで使える。

```python
import sqlite3

# 接続（ファイルがなければ自動作成）
conn = sqlite3.connect('rfid.db')
cursor = conn.cursor()
```

### 1-1：テーブルを作成して1行挿入する（動作サンプル）

```python
import sqlite3
from datetime import datetime

conn = sqlite3.connect('rfid.db')
cursor = conn.cursor()

# テーブル作成（なければ作る）
cursor.execute('''
    CREATE TABLE IF NOT EXISTS logs (
        id       INTEGER PRIMARY KEY AUTOINCREMENT,
        datetime TEXT NOT NULL,
        uid      TEXT NOT NULL,
        name     TEXT NOT NULL,
        result   TEXT NOT NULL
    )
''')

# 1行挿入
cursor.execute(
    'INSERT INTO logs (datetime, uid, name, result) VALUES (?, ?, ?, ?)',
    (datetime.now().strftime('%Y-%m-%d %H:%M:%S'), '123-45-67-89-0', 'Alice', 'ALLOWED')
)

conn.commit()
conn.close()
print("SQLite に書き込みました")
```

```bash
nano sqlite_write.py
python3 sqlite_write.py
ls -lh rfid.db
```

**Q1. SQLiteのファイル `rfid.db` はテキストエディタで開けるか？　なぜか？**

### 1-2：全データを取得して表示する

```python
import sqlite3

conn = sqlite3.connect('rfid.db')
cursor = conn.cursor()

cursor.execute('SELECT * FROM logs ORDER BY datetime DESC')
rows = cursor.fetchall()

for row in rows:
    print(row)

conn.close()
```

```bash
nano sqlite_read.py
python3 sqlite_read.py
```

**Q2. `ORDER BY datetime DESC` の `DESC` を `ASC` に変えると何が変わるか？**

---

## Step 2：MySQLをインストールして設定する

### SQLiteからMySQLへ

| 比較 | SQLite | MySQL |
|------|--------|-------|
| 起動 | ファイルを開くだけ | サーバープロセスが常時起動 |
| 接続方法 | ファイルパス | host / user / password |
| 複数アクセス | 1プロセス推奨 | 同時接続対応 |
| Webアプリ連携 | 難しい | 標準的 |

ラズパイ本体にMySQLをインストールすると、**ラズパイ自体がDBサーバーになる。**

### 2-1：MySQLをインストールする

```bash
sudo apt update
sudo apt install mariadb-server -y
```

> Raspberry Pi OSでは `mysql-server` を指定すると MariaDB がインストールされる。  
> MariaDB は MySQL と互換性があり、コマンド・SQLの書き方はほぼ同じ。

インストール後、サービスの状態を確認する：

```bash
sudo systemctl status mariadb
```

`active (running)` と表示されれば起動している。  
確認後は `q` キーで終了する。

### 2-2：初期設定（セキュリティ設定）

```bash
sudo mariadb-secure-installation
```

対話形式で設定する。授業用の設定例：

| 質問 | 入力 |
|------|------|
| current password for root | （空のままEnter） |
| Switch to unix_socket authentication | n |
| Change the root password | y → パスワードを設定 |
| Remove anonymous users | y |
| Disallow root login remotely | y |
| Remove test database | y |
| Reload privilege tables | y |

### 2-3：データベースとユーザーを作成する

rootでログイン：

```bash
sudo mysql -u root -p
```

MySQLのプロンプト（`MariaDB [(none)]>`）で以下を実行：

```sql
-- データベースを作成
CREATE DATABASE rfid_db;

-- ユーザーを作成してパスワードを設定（パスワードは任意）
CREATE USER 'piuser'@'localhost' IDENTIFIED BY '任意のパスワード';

-- 権限を付与
GRANT ALL PRIVILEGES ON rfid_db.* TO 'piuser'@'localhost';
FLUSH PRIVILEGES;

-- 確認
SHOW DATABASES;
EXIT;
```

**Q3. MySQLでユーザーを作成するのはなぜか？　SQLiteにはない概念である。**

---

## Step 3：PythonからMySQLに接続する

### ライブラリのインストール

```bash
pip install mysql-connector-python --break-system-packages
```

### SQLiteとの接続コードの比較

| 項目 | SQLite | MySQL |
|------|--------|-------|
| インポート | `import sqlite3` | `import mysql.connector` |
| 接続 | `sqlite3.connect('rfid.db')` | `mysql.connector.connect(host=..., user=..., ...)` |
| プレースホルダー | `?` | `%s` |
| SQL文 | ほぼ同じ | ほぼ同じ |

### 3-1：MySQLに接続してテーブルを作成する（動作サンプル）

```python
import mysql.connector
from datetime import datetime

# 接続設定
conn = mysql.connector.connect(
    host='localhost',
    user='piuser',
    password='設定したパスワード',  # ← 自分で設定したパスワードを入力
    database='rfid_db'
)
cursor = conn.cursor()

# テーブル作成
cursor.execute('''
    CREATE TABLE IF NOT EXISTS logs (
        id       INT AUTO_INCREMENT PRIMARY KEY,
        datetime VARCHAR(20) NOT NULL,
        uid      VARCHAR(50) NOT NULL,
        name     VARCHAR(50) NOT NULL,
        result   VARCHAR(10) NOT NULL
    )
''')

# 1行挿入（プレースホルダーは %s）
cursor.execute(
    'INSERT INTO logs (datetime, uid, name, result) VALUES (%s, %s, %s, %s)',
    (datetime.now().strftime('%Y-%m-%d %H:%M:%S'), '123-45-67-89-0', 'Alice', 'ALLOWED')
)

conn.commit()
conn.close()
print("MySQL に書き込みました")
```

```bash
nano mysql_write.py
python3 mysql_write.py
```

### 3-2：MySQLコマンドラインで確認する

```bash
mysql -u piuser -p rfid_db
```

```sql
SELECT * FROM logs;
EXIT;
```

**Q4. SQLiteのプレースホルダーは `?` だが、MySQLでは `%s` を使う。両方試して動作を確認しよう。**

---

## Step 4：RFIDリーダーと統合する

### 接続設定を関数にまとめる

毎回接続パラメータを書くのは煩雑なので、設定をまとめた関数を作る。

```
[ 処理の流れを考えよう ]

1. DB接続設定を定数として定義する
2. init_db()：テーブルが存在しなければ作成する
3. insert_log()：カード検知時にログを1行挿入する
4. RFIDリーダーのループでinsert_log()を呼び出す
```

> ヒント：接続設定をまとめると管理しやすい
> ```python
> DB_CONFIG = {
>     'host': 'localhost',
>     'user': 'piuser',
>     'password': '設定したパスワード',  # ← 自分で設定したパスワードを入力
>     'database': 'rfid_db'
> }
>
> def get_conn():
>     return mysql.connector.connect(**DB_CONFIG)
> ```

```bash
nano rfid_with_mysql.py
python3 rfid_with_mysql.py
```

数回カードをかざした後、`Ctrl+C` で終了してMySQLで確認する：

```bash
mysql -u piuser -p rfid_db -e "SELECT * FROM logs ORDER BY datetime DESC;"
```

---

## 動作確認・発展課題

### 動作確認

- [ ] MariaDB（MySQL）が起動している
- [ ] データベースとユーザーが作成できた
- [ ] Pythonからテーブルを作成・挿入できた
- [ ] RFIDカードのログがMySQLに記録された
- [ ] SQLコマンドラインで記録を確認できた

---

### 発展課題

**課題A：名前別入室回数を集計する**  
MySQLのプロンプトで以下のSQL文を実行し、名前別の入室回数を表示しよう：

```sql
SELECT name, COUNT(*) AS count
FROM logs
WHERE result = 'ALLOWED'
GROUP BY name
ORDER BY count DESC;
```

**課題B：Pythonからクエリ結果を表示する**  
`SELECT * FROM logs WHERE result = 'DENIED'` の結果をPythonで取得して、ターミナルに表示するスクリプトを作ろう。

---

## 本日のまとめ

| 学んだこと | 内容 |
|-----------|------|
| SQLite | サーバー不要の軽量DB。Python標準搭載 |
| MySQL（MariaDB） | サーバー型DB。認証・複数接続に対応 |
| SQL基本 | CREATE / INSERT / SELECT / WHERE / GROUP BY |
| スケールアップ | 目的に合ったDBを選ぶ考え方 |

---

*神戸テック ラズパイ講座 データ記録 Session 2：DBに記録する*
