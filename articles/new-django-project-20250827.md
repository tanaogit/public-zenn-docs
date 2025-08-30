---
title: "【Django】プロジェクトの新規作成方法"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["django", "python"]
published: false
---

# はじめに
こんにちは！  
今回は Python で人気のフルスタックWebフレームワークである **Django** を使って、
**仮想環境構築 → プロジェクト作成 → DB接続 → 一覧表示** までを体験できる記事を書きました。

### 前提条件
- Python 3.10 以上をインストール済の方（3.3 以降でもOKですが新しいバージョン推奨）
- コマンドライン操作ができること

### 筆者の環境
 - macOS
 - python 3.12.3
 - django 5.2.5

# 1.仮想環境の構築方法
Django のプロジェクトを作成するときは仮想環境を構築するのがおすすめです。
プロジェクト毎に独立した環境を作ることができるので、それぞれの環境にあったバージョン管理をすることができます。  

ここでは Python 標準ライブラリの **venv** を使用した方法を紹介します。

### プロジェクトを管理するディレクトリを作成
まずはこちらのコマンドを実行して、プロジェクト管理用のディレクトリを作成してください。
```bash
mkdir my_project && cd my_project
```

### 仮想環境の作成
```bash
python -m venv .venv
```

コマンド実行後に .venv という名前のディレクトリが作成されます。こちらが仮想環境を有効化または無効化する際に必要になります。

:::message
.venv の先頭の「.」は隠しディレクトリにするためです。隠しディレクトリにしなくても問題はないですが、こちらのディレクトリは基本的に編集や削除といった操作をすることはありませんので、誤った操作を防ぐためにも隠しディレクトリにしておく方が安全です。
:::

### 仮想環境の有効化 / 無効化
続いて、仮想環境を使用するためには有効化する必要があります。
OSによってコマンドが異なりますので、使用されているOSに合わせてコマンドを実行してください。

#### 有効化
```bash
# macOS / Linux
source .venv/bin/activate

# Windows (PowerShell)
.venv\Scripts\Activate.ps1
```
プロンプトに仮想環境の作成時に入力した仮想環境名（.venv）が表示されていれば、有効化は完了です。

#### 無効化
仮想環境を終了したい場合は下記コマンドを実行してください。
どのOSでもコマンドは同じになります。
```bash
(.venv) deactivate
```
元のプロンプトの表示に戻っていれば無効化は完了です。

# 2.Djangoプロジェクトの作成
Django のインストールが必要になります。
仮想環境を**有効**にした状態で下記コマンドを実行してください。
```bash
python -m pip install Django
```

### Djangoがインストールされているかの確認
```bash
python -m django --version
```
Djangoのバージョンが表示されていれば正常にインストールが完了しています。

### Djangoプロジェクトの作成
Djangoプロジェクトを作成していきます。下記コマンドを実行してください。
```bash
django-admin startproject config .
```

:::message
末尾の「.」はカレントディレクトリに作成するの意味になります。
:::

#### 開発サーバの起動
```bash
python manage.py runserver
```
ブラウザで http://127.0.0.1:8000/ にアクセスし、ロケット画面が出ればOKです。

### 言語とタイムゾーンの設定
最初は英語で表示されていると思います。日本語表示にするには setting.py の LANGUAGE_CODE を **ja** に変更してください。TIME_ZONEも日本のタイムゾーンである **Asia/Tokyo** に変更しておきましょう。
```py:config/settings.py
LANGUAGE_CODE = 'ja'
TIME_ZONE = 'Asia/Tokyo'
```
画面表示が日本語に変わっていればOKです。

# 3.ディレクトリ構成について
```
.
├─ config/            # プロジェクト設定一式
│  ├─ settings.py     # 設定ファイル
│  ├─ urls.py         # URL の入り口
│  ├─ asgi.py / wsgi.py  # こちらは一旦無視してOK
├─ manage.py          # Django コマンドの操作
└─ .venv/             # 仮想環境の管理
```

 - settings.py…Django の設定（アプリ追加やDB設定）
 - urls.py…URL の振り分け
 - manage.py…runserver や migrate などを実行

# 4.アプリの作成
Django は機能毎に**アプリ**という単位に分割して管理、開発します。
複数のアプリを組み合わせることで１つのWebアプリケーションを構築します。
下記コマンドを実行して、作成したDjangoプロジェクト内にアプリを作成してください。
```bash
python manage.py startapp my_app
```

### アプリを有効化（INSTALLED_APPS に追加）
作成したアプリは INSTALLED_APPS に追加することで使用可能な状態にできます。
```py:config/settings.py
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    # ここに追加
    'my_app',
]
```

# 5.モデル作成とDB接続
本来は MySQL や PostgreSQL などの RDBMS（リレーショナルデータベース管理システム）に接続することが一般的ですが、今回は簡易的に作成したいので Django に標準で組み込まれている **SQLite3** を構築して接続する方法を紹介します。

**SQLite3** も RDBMS になります。ファイルベースなのでサーバーレスで取り扱うことができます。
### 簡単なモデルを作る
```py:my_app/models.py
from django.db import models

class Article(models.Model):
    title = models.CharField(max_length=200)
    body = models.TextField(blank=True)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        db_table = 'articles' 

    def __str__(self):
        return self.title

```

### マイグレーションの実行
```bash
python manage.py makemigrations   # 変更点をマイグレーションファイル化
python manage.py migrate          # DB に反映
```
プロジェクト直下に db.sqlite3 ができていればOKです。

# 6.管理画面でデータを登録
先ほど作成した sqlite3 に articles テーブルが作成されていると思います。
こちらのテーブルにデータを登録していきます。
Django には管理画面がデフォルトで用意されており、作成したテーブルにUI上からデータのCRUD操作が可能です。
まずは、下記コマンドを実行して管理画面にログインするための管理ユーザのアカウントを作成してください。ユーザ名、メールアドレス、パスワードの入力を求められると思います。
```bash
python manage.py createsuperuser
```

### モデルを管理画面に追加
管理画面で操作するモデルを登録します。今回は Article を登録します。
```py:my_app/admin.py
from django.contrib import admin
from my_app.models import Article

@admin.register(Article)
class ArticleAdmin(admin.ModelAdmin):
    list_display = ("id", "title", "created_at")
    search_fields = ("title",)
```

### ログイン画面
サーバーを再起動して http://127.0.0.1:8000/admin/ にアクセスしてください。
ログイン画面が表示されると思いますので、先ほど作成した管理ユーザのアカウントでログインしてください。

![](/images/new-django-project/image1.png)

### サイト管理画面
ログインするとサイト管理画面が表示されると思います。
こちらの画面では登録したモデルや認証認可の操作等が可能になっています。
今回は Articles にデータを登録したいので「+追加」をクリックしてください。

![](/images/new-django-project/image2.png)

### データの登録
article の追加画面が表示されると思いますので、任意の値を入力して「保存」をクリックしてください。articles テーブルに入力したデータが登録されます。

![](/images/new-django-project/image3.png)

サイト管理画面の Articles をクリックしてデータが登録されていればOKです。

# 7.DBのデータを一覧で表示するページを作成
### Templateの作成
Django では HTML を書いた静的な部分と動的なコンテンツを挿入する **Django テンプレート言語 (DTL)** と呼ばれる独自のテンプレートシステムが用意されています。
今回は **DTL** についての詳しい説明は割愛していますので、さらに詳しく知りたい方は公式ドキュメントを参考ください。
<!-- NOTE: Djangoのテンプレートの公式ドキュメント -->
https://docs.djangoproject.com/ja/5.2/topics/templates/

#### base.htmlの作成
一覧を表示するためのページを作成していきます。
Django ではテンプレートを使用してページを作成していきます。
テンプレートは継承が可能なため、ますは全ページで基盤となる base.html を作成していきます。
my_project 配下に templates ディレクトリを作成して base.html を作成してください。
```html:templates/base.html
<!DOCTYPE html>
<html lang="ja">
  <head>
    <title>django project</title>
    <meta charset="utf-8" />
    <meta
      name="viewport"
      content="width=device-width, initial-scale=1, shrink-to-fit=no"
    />
  </head>
  <body>
    {% block content %} {% endblock %}
  </body>
</html>
```

#### index.html の作成
次に、base.html を継承して一覧を表示するページを作成していきます。
my_app 配下に templates/my_app ディレクトリを作成して index.html を作成してください。
```html:my_app/templates/my_app/index.html
{% extends "base.html" %}

{% block content %}
<h2>Articles</h2>
<table>
  <tr>
    <th>タイトル</th>
    <th>本文</th>
  </tr>
  {% for article in articles %}
  <tr>
    <td>{{ article.title }}</td>
    <td>{{ article.body }}</td>
  </tr>
  {% endfor %}
</table>
{% endblock %}
```

#### テンプレートの設定を追加
Django の設定ファイルに作成した templates ディレクトリを指定する必要があります。
```py:config/settings.py
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [BASE_DIR / 'templates'], # ここを変更
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]
```

BASE_DIR はプロジェクトのルートディレクトリ（manage.py がある場所）を指しています。
**'DIRS': [BASE_DIR / 'templates']** と指定することで ルートディレクトリ配下に作成した templates ディレクトリを Django が参照してくれるようになります。
アプリ配下に作成した templates ディレクトリは **'APP_DIRS': True** になっていれば自動で見つけてくれます。False にすると、アプリ内 templates ディレクトリは無視されます。

### Viewの作成
続いて、ビューを作成していきます。
Django では主に**クラスベースビュー**と**関数ベースビュー**の2通りの記載方法があります。
どちらを選択しても構いません。今回は Django で用意されている汎用ビュー(Generic View)を活用できるのでクラスベースビューで作成していこうと思います。
<!-- NOTE: Djangoのクラスベースビューの公式ドキュメント -->
https://docs.djangoproject.com/ja/5.2/topics/class-based-views/intro/

#### view.pyの作成
一覧を表示したいので Django で用意されている汎用ビューの ListView を継承してビューを作成していきます。
```py:my_app/views.py
from django.views.generic import ListView
from my_app.models import Article


class ArticleListView(ListView):
    context_object_name = "articles"
    queryset = Article.objects.all().order_by("-id")
    template_name = "my_app/index.html"
```

 - context_object_name…クエリセットをテンプレートに渡すときの変数名（デフォルトは object_list）
 - queryset…取得するデータのクエリセット
 - template_name…使用するテンプレートファイルへのパス

### URLの設定
最後にurlの設定を追加して、ブラウザから画面を表示できるようにしていきます。
my_app ディレクトリに urls.py のファイルを作成してください。
```py:my_app/urls.py
from django.urls import path
from my_app.views import ArticleListView

url_name = 'my_app'

urlpatterns = [
    path("articles/", ArticleListView.as_view(), name='index'),
]
```

config/urls.py に my_app/urls.py を読み込むための設定を追加してください。
```py:config/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('my_app.urls')),  # 追加
]
```
http://127.0.0.1:8000/articles/ にアクセスして、登録したデータの一覧画面が表示されていれば完成です。

# まとめ
今回は Django を使って以下の流れを記事にしてみました。
初めて Django プロジェクトを作成する方の力になれれば幸いです。

- 仮想環境の構築
- Django プロジェクトとアプリの作成
- モデルを定義してマイグレーションを実行
- 管理画面からデータを登録
- クラスベースビュー（ListView）で記事一覧を表示

Django には他にもたくさんの便利な機能が用意されています。
公式ドキュメントや教材を参考にして、さらに Django を深掘りしてみてください
最後まで読んでいただきありがとうございました。
