# ラズパイ講座 データ記録 Session 3：Djangoで可視化する

**使用機材：** Raspberry Pi CM5 + CM5 I/O Board  
**接続方法：** RaspberryConnect でリモートターミナル接続  
**閲覧端末：** 学生用WiFiに接続したスマホまたはタブレット  

---

## インデックス

1. 導入・今回作るもの
2. 通信確認：スマホからラズパイに接続できるか確認する
3. Step 1：DjangoとPyMySQLをインストールする
4. Step 2：プロジェクトとアプリを作成する
5. Step 3：ログ一覧ページを作る
6. Step 4：サーバーを起動してスマホで確認する
7. 動作確認・発展課題

---

## 導入

### 今回作るもの

**スマホのブラウザから入退室ログを確認できるページ**

ゴールの動作：
```
スマホのブラウザで http://<ラズパイIP>:8000 にアクセスする
 → MySQLのlogsテーブルの内容がブラウザに表示される
```

すべてラズパイ上で開発・実行する。

---

## 通信確認：スマホからラズパイに接続できるか確認する

### 1. ラズパイのIPアドレスをメモする

```bash
hostname -I
```

表示された最初のIPアドレスをメモする（例：`192.168.1.42`）。

### 2. 簡易HTTPサーバーを起動する

```bash
python3 -m http.server 8080
```

### 3. スマホで接続を確認する

スマホを**学生用WiFi**に接続してから、ブラウザで以下にアクセスする：

```
http://<メモしたIPアドレス>:8080
```

ファイル一覧のページが表示されれば**通信OK**。

### 4. 簡易HTTPサーバーを停止する

```bash
Ctrl + C
```

---

## Step 1：DjangoとPyMySQLをインストールする

```bash
pip install django pymysql --break-system-packages
```

> **PyMySQLについて**  
> DjangoとMySQLを繋ぐドライバには `mysqlclient`（通常授業で使用）と `PyMySQL` の2種類がある。  
> `mysqlclient` はC言語のライブラリに依存するためラズパイでのビルドが複雑になる場合がある。  
> `PyMySQL` はPure Pythonで動作するため、ラズパイ環境では `pip install` だけで使える。  
> 動作はほぼ同じだが、Step 2-2 のパッチコードが `PyMySQL` を使う場合に必要な手順となる。

---

## Step 2：プロジェクトとアプリを作成する

### 2-1：Djangoプロジェクトを作成する

```bash
cd ~
mkdir rfid_site
cd rfid_site
django-admin startproject config .
python manage.py startapp logs_app
```

作成されるフォルダ構成：

```
rfid_site/
├── manage.py
├── config/
│   ├── __init__.py
│   ├── settings.py
│   └── urls.py
└── logs_app/
    ├── models.py
    ├── views.py
    └── ...
```

### 2-2：PyMySQLをMySQLドライバとして登録する

```bash
nano config/__init__.py
```

以下を記述する：

```python
import pymysql
pymysql.install_as_MySQLdb()
```

### 2-3：settings.py を設定する

```bash
nano config/settings.py
```

**ALLOWED_HOSTSを変更：**

```python
ALLOWED_HOSTS = ['*']
```

**INSTALLED_APPSにアプリを追加：**

```python
INSTALLED_APPS = [
    ...
    'logs_app',  # 追加
]
```

**DATABASESをMySQLに変更：**

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'rfid_db',
        'USER': 'piuser',
        'PASSWORD': '設定したパスワード',
        'HOST': 'localhost',
        'PORT': '3306',
    }
}
```

**Q1. `ALLOWED_HOSTS = ['*']` はどういう意味か？**

---

## Step 3：ログ一覧ページを作る

### 3-1：モデルを定義する

```bash
nano logs_app/models.py
```

```python
from django.db import models

class Log(models.Model):
    datetime = models.CharField(max_length=20)
    uid      = models.CharField(max_length=50)
    name     = models.CharField(max_length=50)
    result   = models.CharField(max_length=10)

    class Meta:
        db_table = 'logs'   # MySQLの既存テーブル名
        managed  = False    # Djangoはこのテーブルを作成・変更しない
```

**Q2. `managed = False` を設定しないとどうなるか？**

> `managed = True`（デフォルト）の場合、`python manage.py migrate` を実行したとき  
> DjangoがMySQLの `logs` テーブルを**作成・変更・削除**しようとする。  
> 作成済みのテーブルと競合してエラーになる可能性がある。  
> `managed = False` にすることで、DjangoはSELECT/INSERT/UPDATEのみ行い  
> テーブル構造には一切手を加えない。

### 3-2：ビューを作る

```bash
nano logs_app/views.py
```

```python
from django.shortcuts import render
from .models import Log

def log_list(request):
    logs = Log.objects.all().order_by('-datetime')[:100]
    return render(request, 'logs_app/index.html', {'logs': logs})
```

### 3-3：テンプレートを作る

```bash
mkdir -p logs_app/templates/logs_app
nano logs_app/templates/logs_app/index.html
```

```html
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>入退室ログ</title>
  <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
</head>
<body class="p-3">
  <h1 class="h4 mb-3">入退室ログ</h1>
  <div class="table-responsive">
    <table class="table table-sm table-striped">
      <thead>
        <tr>
          <th>日時</th>
          <th>名前</th>
          <th>結果</th>
        </tr>
      </thead>
      <tbody>
        {% for log in logs %}
        <tr>
          <td>{{ log.datetime }}</td>
          <td>{{ log.name }}</td>
          <td>
            {% if log.result == 'ALLOWED' %}
              <span class="badge bg-success">ALLOWED</span>
            {% else %}
              <span class="badge bg-danger">DENIED</span>
            {% endif %}
          </td>
        </tr>
        {% empty %}
        <tr><td colspan="3" class="text-muted">データがありません</td></tr>
        {% endfor %}
      </tbody>
    </table>
  </div>
</body>
</html>
```

### 3-4：URLを設定する

```bash
nano logs_app/urls.py
```

```python
from django.urls import path
from . import views

urlpatterns = [
    path('', views.log_list, name='log_list'),
]
```

```bash
nano config/urls.py
```

```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('logs_app.urls')),
]
```

---

## Step 4：サーバーを起動してスマホで確認する

```bash
python manage.py runserver 0.0.0.0:8000
```

スマホのブラウザで以下にアクセスする：

```
http://<ラズパイのIPアドレス>:8000
```

ログの一覧が表示されることを確認する。

RFIDカードをかざしてページを更新すると、新しいログが追加されていることを確認する。

終了するには `Ctrl + C`。

**Q3. `0.0.0.0:8000` と `127.0.0.1:8000` の違いは何か？**

> 127.0.0.1 はラズパイ自身だけがアクセスできるアドレス（ループバックアドレス）。
> 0.0.0.0 は「すべてのネットワークインターフェースからの接続を受け付ける」という意味で、
> 同じWiFiにいるスマホからのアクセスも受け付けるようになる。

---

## 動作確認・発展課題

### 動作確認

- [ ] スマホから `http://<IP>:8000` にアクセスできた
- [ ] logsテーブルの内容が表示された
- [ ] RFIDカードをかざしてページ更新するとログが増えた
- [ ] ALLOWEDが緑・DENIEDが赤で表示された

---

### 発展課題

**課題A：件数を表示する**  
一覧の上に「全 N 件」と表示しよう。

> ヒント：`logs.count()` の結果をテンプレートに渡す。

**課題B：ALLOWEDのみ絞り込む**  
URLに `?result=ALLOWED` を付けたとき、ALLOWEDのログだけ表示されるようにしよう。

> ヒント：`request.GET.get('result')` でURLパラメータを取得できる。

---

## 本日のまとめ

| 学んだこと | 内容 |
|-----------|------|
| Djangoの構成 | model / view / template / url の役割 |
| managed = False | 既存DBテーブルをDjangoで使う方法 |
| runserver | ラズパイをWebサーバーとして動かす |
| Bootstrap | スマホ対応のシンプルな画面を素早く作る |

---

*神戸テック ラズパイ講座 データ記録 Session 3：Djangoで可視化する*
