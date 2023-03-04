# RailsとDockerの環境構築

1. Dockerのインストール
2. Dockerに必要なファイルを準備
3. コンテナのビルド
4. Railsプロジェクトを作成
5. データベースの準備と作成
6. Hello Worldと表示してみる

## 1. Dockerのインストール

以下の公式サイトからDockerをダウンロードして起動。

https://www.docker.com/

コマンドでDockerのバージョンが表示されればOK。

```terminal
% docker --version
Docker version 20.10.20, build 9fdeb9c
```

## 2. Dockerに必要なファイルを準備

任意の名前でディレクトリを作成。

```terminal
% mkdir rails_docker
% cd rails_docker
```

次に、Dockerに必要なファイルを作成。

- `Dockerfile`
- `docker-compose.yml`
- `Gemfile`
- `Gemfile.lock`
- `entrypoint.sh`

以下コマンドで一括生成

```terminal
% touch {Dockerfile,docker-compose.yml,Gemfile,Gemfile.lock,entrypoint.sh}
```

### Dockerfile

- `Dockerfile` : 新規Dockerイメージを作成するための手順を記述したテキストファイル

各書き方や意味は割愛（以下 参考になります）
https://qiita.com/terufumi1122/items/54b574e124de0b24df04

```Dockerfile
# Rubyバージョン指定
FROM ruby:3.0.2

# yarnパッケージ管理ツールをインストール
RUN apt-get update && apt-get install -y curl apt-transport-https wget && \
curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add - && \
echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.d/yarn.list && \
apt-get update && apt-get install -y yarn

RUN apt-get update -qq && apt-get install -y nodejs yarn
RUN mkdir /myapp
WORKDIR /myapp
COPY Gemfile /myapp/Gemfile
COPY Gemfile.lock /myapp/Gemfile.lock
RUN bundle install
COPY . /myapp

RUN yarn install --check-files
RUN bundle exec rails webpacker:compile

# コンテナ起動時に実行させるスクリプトを追加
COPY entrypoint.sh /usr/bin/
RUN chmod +x /usr/bin/entrypoint.sh
ENTRYPOINT ["entrypoint.sh"]
EXPOSE 3000

# Railsサーバー起動
CMD ["rails", "server", "-b", "0.0.0.0"]
```

### docker-compose.yml

- `docker-compose.yml` : Webアプリケーション内で稼働する複数のコンテナ構成をまとめて定義したファイル

```docker-compose.yml
version: '3'
services:
  db:
    image: mysql:8.0
    platform: linux/x86_64
    environment:
      MYSQL_ROOT_PASSWORD: password
      MYSQL_DATABASE: root
    ports:
      - "3306:3306"
    command: --default-authentication-plugin=mysql_native_password
    volumes:
      - ./tmp/db:/var/lib/mysql
  web:
    build: .
    command: bash -c "rm -f tmp/pids/server.pid && bundle exec rails s -p 3000 -b '0.0.0.0'"
    volumes:
      - .:/myapp
    ports:
      - "3000:3000"
    depends_on:
      - db
    stdin_open: true
    tty: true

```

### Gemfile

- `Gemfile` : Railsアプリで利用するgem（Rubyのライブラリ）が記述されているファイル

```Gemfile
source 'https://rubygems.org'
gem 'rails', '~> 6.1.4'
```

今回インストールするRailsのバージョンを指定。

このGemfileの内容は、後に実行する`rails new`で上書きされる。

### Gemfile.lock

- `Gemfile.lock` : 実際にインストールしたgemのリスト（バージョンなど細かく書いてある）

まだ何もインストールしていないので、空ファイルのままでOK

### entrypoint.sh

- `entrypoint.sh` : 初回起動時のみに処理したい内容を記述するファイル

```entrypoint.sh
#!/bin/bash
set -e # エラーが発生するとスクリプトを終了する意味

# server.pid が存在するとサーバーが起動できない対策のために server.pid を削除するように設定
rm -f /myapp/tmp/pids/server.pid

# DockerfileのCMDで渡されたコマンド（Railsサーバー起動）を実行
exec "$@"
```

## 3. コンテナのビルド

先ほど作成した Docker関連のファイルを使ってコンテナをビルド。

```terminal
% docker-compose build
% docker-compose build --no-cache (← キャッシュを無視して再ビルドできる)

// bundle installを求められたらを実行
% bundle install --path vendor/bundle
```

## 4. Railsプロジェクトを作成

次は `docker-compose` コマンドを使って `rails new` を実行し、Railsのプロジェクトを作成。

`docker-compose run` に続けてサービス名を指定し、さらにコンテナ内で実行したいコマンド（=railsコマンド) を続ける。

Railsが動くサービスには `web` という名前を `docker-compose.yml` で付けたので、コマンドでのコンテナ名としては `web` を当てはめる。

以下のコマンドを実行。

```terminal
% docker-compose run web rails new . --force --database=mysql
```

- `--force` : 今のファイルを上書き
- `--database=mysql` : DBをMySQLに

エラーなく実行が完了したらRailsのプロジェクトが入る。

ここで、RailsのインストールでGemfileが更新されたので、コンテナを再ビルド。（そうしないとGemfileが反映されない）

```terminal
% docker-compose build
```

## 5. データベースの準備と作成

Railsのデータベースファイルを編集します。

`/config/database.yml`
```diff_yaml
default: &default
  adapter: mysql2
  encoding: utf8mb4
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
+ host: db
  username: root
+ password: password

development:
  <<: *default
  database: myapp_development

test:
  <<: *default
  database: myapp_test

production:
  <<: *default
  database: myapp_production
  username: myapp
  password: <%= ENV['MYAPP_DATABASE_PASSWORD'] %>

```

`host: db` は `docker-compose.yml`で指定したMySQLのコンテナ名。

同じ `docker-compose.yml` で指定したコンテナ間であれば、コンテナ名をホストとして名前解決してアクセスすることができる。

これでRailsがデータベースと連携できるようになりました。以下のコマンドでコンテナを起動。

```terminal
% docker-compose up
// コンテナは起動するけど、このターミナルでコンテナを起動しているので別の作業をさせてくれない

% docker-compose up -d
// コンテナをバックグラウンドで起動して、同ターミナルで作業を続行できる。
```

http://localhost:3000/ にアクセスするとデータベースがないよと怒られたので、データベースを作成。

```terminal
% docker-compose run web rails db:create
[+] Running 1/0
 ⠿ Container rails_docker-db-1  Running
Running via Spring preloader in process 21
Created database 'myapp_development'
Created database 'myapp_test'
```

再度、http://localhost:3000/ にアクセスすると、Railsの画面が表示されていて、指定したバージョン通りにインストールされている。
