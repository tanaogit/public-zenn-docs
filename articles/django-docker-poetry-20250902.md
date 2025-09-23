---
title: "Docker + Poetry で Django の開発環境を構築する"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["django", "python", "docker", "poetry"]
published: false
---


# はじめに
こんにちわ！
今回は Docker を使用して Django の開発環境を構築していきます。
パッケージ管理は poetry を使用します。今まで pip で管理することが多かったのですが、最近 poetry を使用する機会があり、パッケージ管理を JavaScript の npm のように厳密に依存関係を管理できることを知りました。チーム開発をする上でも環境差分を極力減らせるので便利だなと思ったので、今回は Docker コンテナ内に poetry をインストールして開発環境を構築していこうと思います。

## 対象読者
 - Docker + Poetry を使用して、Django の開発環境を構築したい方

## 筆者の環境
 - macOS
 - python 3.12.3
 - django 5.2.5
 - poetry 2.1.4


# Poerty とは
Poetry について軽く触れておきます。
Poetry は Python の 依存関係管理 と パッケージ管理 をシンプルに行うためのツールです。
プロジェクトで必要なパッケージを pyproject.toml に宣言しておけば、Poetry が自動的にインストールや更新を行ってくれます。
poetry.lock により依存関係を厳密に固定できるため、チーム開発や本番環境でも同じ環境を再現できます。

また、Poetry は仮想環境を自動で構築・管理してくれます。従来のように virtualenv や venv を手動で作成する必要はなく、プロジェクトごとに独立した環境が自動的に準備されます。
poetry shell コマンドでその環境に入ったり、poetry run python で仮想環境内の Python を使ったりできるため、環境構築が非常に簡単です。

さらに poetry build や poetry publish を利用すれば、作成したプロジェクトを PyPI に公開することも可能みたいです。

<!-- NOTE: Poetry の公式ドキュメント -->
https://python-poetry.org/docs/


# Docker の設定ファイルの作成
まずは、ローカルPCに開発環境を管理するためのルートディレクトリを作成していきましょう。
作成したルートディレクトリに **Dockerfile**、**entrypoint.sh**、**docker-compose.yml** の3つのファイルを作成してください。

```
django_project/
├── .docker/
│   ├── Dockerfile
│   └── entrypoint.sh
└── docker-compose.yml
```

## Dockerfile
Dockerfile は、Docker イメージを作成するための手順を記載するファイルになります。
こちらをビルドすることでカスタムした環境のイメージが作成されます。

```dockerfile:.docker/Dockerfile
# ベースイメージ：軽量な python:3.12-slim を使用
FROM python:3.12-slim

# 環境変数の設定
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    POETRY_NO_INTERACTION=1 \
    POETRY_VIRTUALENVS_CREATE=false

# ランタイム依存パッケージを追加インストール：git と curl（不要なら削除可）＋ HTTPS 用の証明書
RUN apt-get update && apt-get install -y --no-install-recommends \
      git curl ca-certificates \
  && rm -rf /var/lib/apt/lists/*

# poetry をインストール (バージョン固定)
ARG POETRY_VERSION=2.1.4
RUN pip install --upgrade pip && pip install "poetry==${POETRY_VERSION}"

# アプリケーション用ディレクトリを作成して作業ディレクトリに設定
RUN mkdir -p /app
WORKDIR /app

# エントリポイントスクリプトをコンテナにコピーして権限設定
COPY ./.docker/entrypoint.sh /entrypoint.sh
RUN chmod 755 /entrypoint.sh

# コンテナ起動時に entrypoint.sh を実行
ENTRYPOINT ["/entrypoint.sh"]
```

#### 1. ベースイメージの決定
```dockerfile
# ベースイメージ：軽量な python:3.12-slim を使用
FROM python:3.12-slim
``` 

まず最初にベースイメージを定義する必要があります。
DockerHub に Python のイメージが配布されているので、そちらをベースに環境をカスタマイズしていきます。今回使用する slim 系は Python 3.12 が入っており、不要なパッケージを削ぎ落とした軽量版になります。必要なパッケージは手動で追加する必要はありますが、イメージサイズを小さくすることができます。

#### 2. 環境変数の設定
次は環境変数を設定していきます。

```dockerfile
# 環境変数の設定
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    POETRY_NO_INTERACTION=1 \
    POETRY_VIRTUALENVS_CREATE=false
```

 - **PYTHONDONTWRITEBYTECODE=1**
   Python が .pyc キャッシュを作らなくなり、余計なファイルが生成されずソースがきれいになります。
 - **PYTHONUNBUFFERED=1**
   標準出力をバッファせずに即時出力します。Docker のログにリアルタイムでログが反映されます。
 - **POETRY_NO_INTERACTION=1**
   Poetry の対話的質問（y/n）が出なくなります。こちらは自動化する際に必須です。
 - **POETRY_VIRTUALENVS_CREATE=false**
   Poetry が独自の仮想環境を作らず、システムに直接パッケージをインストールします。
   ※ Docker コンテナはすでに隔離環境なので仮想環境を重ねる必要がないためです。

#### 3. パッケージのインストール
```dockerfile
# ランタイム依存パッケージを追加インストール：git と curl（不要なら削除可）＋ HTTPS 用の証明書
RUN apt-get update && apt-get install -y --no-install-recommends \
      git curl ca-certificates \
  && rm -rf /var/lib/apt/lists/*
```

開発環境に必要なパッケージをビルド時にインストールするためのコマンドを記載しています。
今回は最低限 **git**、**curl** のみインストールしています。他にも開発に必要なパッケージがありましたら追加してください。

#### 4. poetry のインストール
```dockerfile
# poetry インストール (バージョン固定)
ARG POETRY_VERSION=2.1.4
RUN pip install --upgrade pip && pip install "poetry==${POETRY_VERSION}"
```

poetry のインストールを行います。
ARG でバージョンを指定しているのでビルド時に --build-arg POETRY_VERSION=xxx を指定すれば、poetry のバージョンを柔軟に変えられます。
デフォルトでは今の最新版である 2.1.4 を設定しています。

#### 5. 作業ディレクトリに移動
```dockerfile
# アプリケーション用ディレクトリを作成して作業ディレクトリに設定
RUN mkdir -p /app
WORKDIR /app
```

アプリケーションを配置する app/ ディレクトリを作成して作業ディレクトリに設定しています。

#### 6. エントリポイントスクリプトの実行設定
```dockerfile
# エントリポイントスクリプトをイメージにコピーして権限設定
COPY ./.docker/entrypoint.sh /entrypoint.sh
RUN chmod 755 /entrypoint.sh

# コンテナ起動時に必ず entrypoint.sh を実行
ENTRYPOINT ["/entrypoint.sh"]
```

entrypoint.sh をコンテナ起動時に実行するための設定をしています。
ホストで作成した entrypoint.sh をイメージにコピーして実行できるように権限の変更をしています。最後に ENTRYPOINT を設定することでコンテナ起動時に実行されます。

## entrypoint.sh
こちらは、Docker コンテナの起動時に最初に実行するスクリプトです。
コンテナの初回起動時には pyproject.toml や Django のプロジェクトファイルがまだ存在しなかったりと必要なファイルが足りていない状態なので、こちらのスクリプトを実行することで自動で生成します。

```bash:.docker/entrypoint.sh
#!/usr/bin/env sh
set -e

# pyproject.toml が無ければ最小構成を生成（Djangoだけ）
if [ ! -f "/app/pyproject.toml" ]; then
  poetry -C /app init -n --name django_project --python "^3.12"
  poetry -C /app add Django
fi

# Django プロジェクトを自動生成
if [ ! -f "/app/web/manage.py" ]; then
  mkdir -p /app/web
  django-admin startproject config /app/web
fi

exec "$@"
```

#### 1. シェルの設定
```bash
#!/usr/bin/env sh
set -e
```

 - **#!/usr/bin/env sh**
   こちらは Shebang (シェバン) と呼び、シェルスクリプトを実行するために1行目に宣言します。
 - **set -e**
   不完全な状態でコンテナが起動しないように、途中でエラーが発生したら即座にスクリプトを終了させます。

#### 2. pyproject.toml が無ければ生成
```bash
# pyproject.toml が無ければ最小構成を生成（Djangoだけ）
if [ ! -f "/app/pyproject.toml" ]; then
  poetry -C /app init -n --name django_project --python "^3.12"
  poetry -C /app add Django
fi
```

pyproject.toml が存在しない場合は poetry の初期化コマンドを実行して必要なファイルを自動生成します。2回目以降のコンテナ起動ではスキップされます。また、このタイミングで Django も追加しておきます。

#### 3. Django プロジェクトの自動生成
```bash
# Django プロジェクトを自動生成
if [ ! -f "/app/web/manage.py" ]; then
  mkdir -p /app/web
  django-admin startproject config /app/web
fi
```
manage.py が存在しない場合は Django プロジェクトの自動生成を行います。

## docker-compose.yml
docker-compose.yml は、複数のコンテナをまとめて管理・起動するための設定ファイルです。
今回の例では 1 サービス（web）だけですが、将来 DB やキャッシュを足しても docker compose up のコマンドだけで起動することができます。

```yml:docker-compose.yml
services:
  web:
    build:
      context: .
      dockerfile: ./.docker/Dockerfile
    image: web
    container_name: web
    working_dir: /app/web
    ports:
      - "8000:8000"
    volumes:
      - .:/app
      - poetry-cache:/root/.cache/pypoetry
      - pip-cache:/root/.cache/pip
    command: python manage.py runserver 0.0.0.0:8000
    tty: true

volumes:
  poetry-cache:
  pip-cache:
```

#### build
```yml
context: .
dockerfile: ./.docker/Dockerfile
```

 - **context**
   Dockerfile を探す基準となるディレクトリを設定します。
 - **dockerfile**
   ビルドする Dockerfile のパスを設定します。

#### image
ビルドして作成される Docker イメージに付ける名前を決めます。
docker images コマンドで確認する際にわかりやすくなります。

#### container_name
コンテナの実行時の名前を設定します。
省略すると ディレクトリ名_service_番号 の形式になるため、明示的に指定すると管理しやすくなります。

#### working_dir
コンテナ内での作業ディレクトリを指定します。
Django プロジェクトのある /app/web を指定しているので、相対パスで manage.py を実行できます。

#### ports
ホストとコンテナのポートをマッピングします。
"8000:8000" とすることで、コンテナの Django 開発サーバ（8000番）をホストの 8000 番に割り当て、http://localhost:8000 からアクセス可能になります。

#### volumes
ファイルをホストとコンテナ間で共有する設定です。
```yml
.:/app
```
ホストのカレントディレクトリ全体を /app にマウント。ソースコードを編集すると即コンテナに反映されます。
```yml
poetry-cache:/root/.cache/pypoetry
pip-cache:/root/.cache/pip
```
poetry と pip の依存キャッシュを保持するための名前付きボリュームです。
これにより poetry install 実行時に毎回 PyPI から依存をダウンロードせず、キャッシュを再利用して高速にセットアップできます。コンテナを削除してもキャッシュは残るので、再ビルド時にも効果があります。

#### command
コンテナ起動時に実行するコマンドを設定します。
Django の開発サーバを起動するコマンドをここに記載しています。
0.0.0.0 を指定することで、コンテナ外 (ホスト) からもアクセス可能になります。

#### tty
疑似 TTY を割り当てる設定。ログが見やすくなり、開発中は有効にしておくと便利です。


# コンテナの起動
Docker の設定ファイルの作成が完了しました。
コンテナを起動していきましょう。

### イメージのビルド
コンテナの起動前に Dockerfile をビルドしてカスタムイメージを作成します。
下記のコマンドを実行してください。もし、失敗して再度ビルドが必要な場合は --no-cache のオプションをつけてキャッシュを使用しないようにしてください。
```bash
docker compose build
```

### 起動
イメージの作成が完了したら下記コマンドを実行してください。
```bash
docker compose up
```

### 起動できたかの確認
下記コマンドを実行して、コンテナが起動したか確認してみましょう.
```bash
docker ps
```

こちらのような結果が表示されていれば成功です。
```
CONTAINER ID   IMAGE     COMMAND                   CREATED              STATUS          PORTS                                         NAMES
7f47df99d3b9   web       "/entrypoint.sh pyth…"   About a minute ago   Up 37 seconds   0.0.0.0:8000->8000/tcp, [::]:8000->8000/tcp   web
```

# poetry インストール
最後に pyproject.toml に定義してある Python パッケージをコンテナ内にインストールしていきましょう。

### コンテナにアクセス
起動したコンテナにアクセスします。
```bash
docker compose exec -it web bash
```

### インストール
コンテナ内で下記コマンドを実行してパッケージをインストールしましょう。
```bash
poetry install --no-root
```

インストールが完了したら、
http://localhost:8000/ にアクセスして Django の初期画面が表示されていれば成功です。


# まとめ
Docker を使用して Poetry + Django の開発環境が構築できたと思います。
こちらの方法であればローカルPCに poetry はインストールしなくても大丈夫です。
追加で必要なパッケージ等があれば、コンテナ内で poetry コマンドを実行してインストールしてください。
 
最後まで読んでいただきありがとうございました。
