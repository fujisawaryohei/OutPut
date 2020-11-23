# Rails Nuxt で作成したカレンダー Todo アプリを AWS Fargate にデプロイしてみた

## 今回のファイル構成

Rails プロジェクトの中にフロントエンドディレクトリを作成してそこに Nuxt プロジェクトを展開する形にします。

```
├── todo-calendar
  ├── frontend
  └── docker-compose.yml
```

## 今回の流れ

1. 環境構築では、Rails と Nuxt の開発用イメージを作成
2. 開発では、作成したそれぞれのイメージを基に Docker-compose でコンテナを起動して開発を進める
3. デプロイでは、Rails と Nuxt の本番用イメージを作成して ECR に Push
4. Push した本番用イメージを Fargate でプルして起動

# 環境構築

## Rails の環境構築

### Rails new

まずは todo-calendar ディレクトリを作成して作成したディレクトリ以下で下記のコマンドを実行します。

```
bundle init
```

作成された Gemfile で`rails`の部分のコメントアウトします。

```rb
# frozen_string_literal: true

source "https://rubygems.org"

git_source(:github) {|repo_name| "https://github.com/#{repo_name}" }

gem "rails"
```

作成したディレクトリ直下で下記のコマンドで Rails プロジェクトを展開します。今回は API モードでの開発になるので`--api`を忘れないように。

```
bundle exec rails new . --api --database=postgresql
```

その後作成された Rails プロジェクトで dockerfile を作成します。  
作成されたら.gitignore ファイルに下記を追記してインストールしたライブラリを Git の管理対象外にしてください。

todo-calendar/.gitignore

```
/vendor
```

### dockerfile 作成

todo-calendar/dockerfile

```docker
FROM ruby:2.7.0

ENV LANG C.UTF-8
ENV TZ Asia/Tokyo

RUN mkdir /app
WORKDIR /app

ADD Gemfile /app/Gemfile

RUN apt-get update -qq

RUN bundle install

ADD . /app
```

### docker-compose.yml を作成

作成したディレクトリで docker-compose.yml ファイルにコンテナの構成を定義します。  
`networks`では、サブネット 172.10.0.0/24 の Docker ネットワークを定義しています。  
`backend`では、先程定義した Rails イメージを`command`で指定しているコマンドでコンテナとして起動するように定義しています。またその際コンテナに割り当てられるプライベート IP アドレスは、`172.10.0.3`になります。  
`db`では、postgresql11.5 のイメージを基にコンテナを定義しています。

```docker
version: "3"

services:
  db:
    container_name: todo-calendar-db
    image: postgres:11.5
    environment:
      TZ: Asia/Tokyo
      POSTGRES_PASSWORD: postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - 3307:3306
    networks:
      app_net:
        ipv4_address: '172.10.0.2'

  backend:
    container_name: todo-calendar-backend
    build: .
    image: todo-calendar-backend
    command: bash -c "rm -f tmp/pids/server.pid && bundle exec rails s -p 3000 -b '0.0.0.0'"
    tty: true
    stdin_open: true
    volumes:
      - ./:/app:cached
    environment:
      TZ: Asia/Tokyo
    depends_on:
      - db
    ports:
      - 5000:3000 # ポートフォワード
    networks:
      app_net:
        ipv4_address: '172.10.0.3'

networks:
  app_net:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: 172.10.0.0/24
volumes:
  postgres_data:
```

docker-compose build コマンドで dockerfile で定義したイメージのビルドを行って下さい。

```
docker-compose build
```

その後、docker-compose up コマンドでコンテナを起動して db コンテナでデータベースを作成してください。

```
docker-compose up
```

```
docker-compose exec backend bin/rails db:create
```

現在ポート 5000 でポートフォワーディングしているため、`localhost:5000`にアクセスすると下記のような画面になります。
これで Rails の開発環境構築ができました。

![helloworl-rails](./images/RailsHelloWorld.png "Qiita")

## Nuxt の環境構築

### create-nuxt-app

今回は、[create-nuxt-app](https://github.com/nuxt/create-nuxt-app) コマンドを使用して Nuxt のプロジェクトを作成します。  
Rails のプロジェクト内, todo-calendar 下記のコマンドを実行してください。

```
yarn create nuxt-app frontend
```

今回、コードフォーマッターに Prettier、リンターツールに ESLint を使用しています。  
また、API 通信をするために Axios, 画面をいい感じにしてくれる Vuetify を予めインストールとセッティングをするようにします。  
上記のコマンドを実行後、色々聞かれるので下記のような選択をするようにしてください。

```
   Generating Nuxt.js project in frontend
? Project name: frontend
? Programming language: JavaScript
? Package manager: Yarn
? UI framework: Vuetify.js
? Nuxt.js modules: Axios
? Linting tools: ESLint, Prettier
? Testing framework: Jest
? Rendering mode: Single Page App
? Deployment target: Server (Node.js hosting)
? Development tools: (Press <space> to select, <a> to toggle all, <i> to invert selection)
? Continuous integration: None
? Version control system: None
```

### dockerfile 作成

Nuxt プロジェクト作成後、作成された Nuxt プロジェクト内で Dockerfile を作成します。

todo-calendar/frontend/dockerfile

```docker
FROM node:12.16.3

RUN mkdir /app
WORKDIR /app

COPY package.json /app/
COPY package.lock.json /app/

RUN yarn install

```

### docker-compose.yml を編集

先程作成した docker-compose.yml ファイルに下記を追加してください。  
先程作成したイメージを基にコンテナ`yarn dev`で起動して、`172.10.0.4`のプライベート IP をコンテナに割り当てています。

```docker
frontend:
  container_name: todo-calendar-frontend
  build: ./frontend/
  image: todo-calendar-frontend
  volumes:
    - ./frontend:/app:cached
  ports:
    - 3000:3000
  command: "yarn run dev"
  networks:
    app_net:
      ipv4_address: "172.10.0.4"
  depends_on:
    - backend
    - db
```

docker-compose build コマンドで dockerfile で定義したイメージのビルドを行って下さい。

天気が良い。

```
docker-compose build
```

現在ポート 3000 でポートフォワーディングしているため、localhost:3000 にアクセスすると下記のような画面になります。 以上で Nuxt 環境構築になります。

![helloworl-rails](./images/NuxtHelloWorld.png "Qiita")

これで Nuxt の開発環境構築ができました。
