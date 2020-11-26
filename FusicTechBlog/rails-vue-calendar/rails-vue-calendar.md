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

```
docker-compose build
```

現在ポート 3000 でポートフォワーディングしているため、localhost:3000 にアクセスすると下記のような画面になります。 以上で Nuxt 環境構築になります。

![helloworl-rails](./images/NuxtHelloWorld.png "Qiita")

これで Nuxt の開発環境構築ができました。

## 開発

### Nuxt

まずは Rails 側の方で Todo アプリの API を作っていきます。

下記のコマンドを実行して Todo モデルを生成してください。

```
bundle exec rails g model Todo
```

作成した Todo モデルに簡易的にバリデーションを設定しておきます（この辺は好みで設定してください）

app/models/todo.rb

```rb
class Todo < ApplicationRecord
  validates :title, presence: true, length: { maximum: 80 }
  validates :content, presence: true
end

```

今回、データベースの構成は簡易的なものとします。
生成後、`db/migrate`以下にマイグレーションファイルが生成されますので、生成されたマイグレーションファイルを以下のように記述して下さい。

```rb
class CreateTodos < ActiveRecord::Migration[6.0]
  def change
    create_table :todos do |t|
      t.string :title
      t.text :content
      t.boolean :is_done, default: false
      t.date :date
      t.time :time
      t.timestamps
    end
  end
end
```

記述したら、下記のコマンドでテーブルを作成します。

```
docker-compose exec backend bundle exec rails db:migrate
```

データベースが作成できましたので、次にコントローラーを作成していきます。

```
bundle exec rails g controller api/v1/todos
```

作成したコントローラーに CRUD 処理を追加していきます。

app/controllers/api/v1/todos_controller.rb

```rb
class Api::V1::TodosController < ApplicationController
  def index
    todos = Todo.all
    render json: todos
  end

  def show
    todo = Todo.find_by(id: params[:id])
    unless todo.nil?
      render json: todo
    else
      render json: { error_message: 'Not Found'}
    end
  end

  def create
    todo = Todo.new(set_params)
    if todo.save
      render json: { success_message: '保存しました' }
    else
      render json: todo.errors.messages
    end
  end

  def update
    todo = Todo.find(params[:id])
    if todo.update(set_params)
      render json: { success_message: '更新しました' }
    else
      render json: todo.errors.messages
    end
  end

  def destroy
    todo = Todo.find(params[:id])
    todo.destroy
    render json: { success_message: '削除しました' }
  end

  private
  def set_params
    params.require(:todo).permit(:title, :content, :is_done, :date, :time)
  end
end

```

次にルーティングを設定します。

config/routes.rb

```rb
Rails.application.routes.draw do
  namespace :api do
    namespace :v1 do
      resources :todos
    end
  end
end
```

最後に、`rack-cors`という gem をインストールして、CORS の設定します。

Gemfile

```rb
gem 'rack-cors'
```

```
bundle install
```

インストール後に config/initializers/cors.rb で origins を`localhost:3000`に設定してください。

config/initializers/cors.rb

```rb
# Be sure to restart your server when you modify this file.

# Avoid CORS issues when API is called from the frontend app.
# Handle Cross-Origin Resource Sharing (CORS) in order to accept cross-origin AJAX requests.

# Read more: https://github.com/cyu/rack-cors

Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins 'http://localhost:3000'
    resource '*',
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options, :head]
  end
end

```

ここまで完了しましたら一度ターミナルで curl してみましょう。

```shell
curl -X POST -H "Content-Type: application/json" -d '{"title":"title test", "content":"content test", "is_done": false}' localhost:4000/api/v1/todos
```

下記のようなレスポンスが来ているか確認してください。

```
{"success_message":"保存しました"}%
```
