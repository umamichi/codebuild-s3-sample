# AWS Codebuildを使ってフロントエンド環境をS3にデプロイする

## AWS Codebuildとは？

<img src="logo.png" alt="CodeBuild" />

その名の通り、CodeをBuildできるサービス

ソースコードをコンパイルし、テストを実行し、デプロイ可能なソフトウェアパッケージを作成できる完全マネージド型のビルドサービス

CodeBuild は連続的にスケールされ、複数のビルドが同時に処理されるので、ビルドが待機状態でキュー内に残されることがありません


### 特徴

+ ビルドサーバーなしでビルド可能

+ アップデート、管理が不要

+ 自動でスケールされるので、並列ビルドが可能（待ち時間なし）

+ AWSの他サービス `EC2` `S3` `CodePipeline` `Lambda` などとの連携が容易

+ `CodePipeline` と連携すれば、テスト→リリースまでを自動化できる

+ 既存の`Jenkins`と連携ができ、ビルド処理部分だけを `CodeBuild` に置き換えも可能

　→ JenkinsのUIやユーザ管理をそのまま使い続けられるメリットがある

+ `Github` や `GitBucket` `GitLab` と連携できて、PUSHをトリガーにビルドも可能

+ `AWS CLI` がデフォルトで使える

### 料金

ビルド1分ごとに料金が請求される

一番安いプランで、Linuxで1分ビルドすると `0.005ドル` ≒ `0.57円`

詳しい料金はこちら：　https://aws.amazon.com/jp/codebuild/pricing/

#### 無料利用枠あり（2018/11 現在）

AWS CodeBuild の無料利用枠では、`build.general1.small（一番安いLinuxのプラン）` を使って **1 か月あたりビルドを 100 分使用できます。**

CodeBuild の無料利用枠は、**12 か月間の AWS 無料利用枠の期間が終了しても自動的に期限切れになることはありません。**


### Jenkinsとの比較

`AWS EC2` に `Jenkins` をインストールしている前提で

| | Jenkins | AWS CodeBuild |
| -- | --- | --- |
| 料金 | Jenkins AgentのEC2インスタンスの待機時間分 | ビルド所要時間の1分ごとに請求 |
| スケールアウト | しない（EC2のスペックが限界） | する |
| OS | EC2のOSに依存 | コンテナを扱えるので自由に構築可能 |

## 実際にデプロイしてみる

### 1. 以下のファイルをつくる

今回は、ビルドツールとして `Parcel` を使う

作業のためのディレクトリを切ります

```
$ mkdir codebuild-s3-sample && cd codebuild-s3-sample
```

まず `package.json` をつくります

```
// package.json
{
  "name": "codebuild-s3-sample",
  "version": "1.0.0",
  "scripts": {
    "start": "parcel src/index.html",
    "build": "rm -rf release && parcel build src/index.html --out-dir release"
  },
  "license": "ISC",
  "devDependencies": {
    "parcel-bundler": "^1.10.3"
  }
}
```

`src` ディレクトリを作り、2ファイルを配置

```
// src/index.html
<html>
<head>
  <title>codebuild-s3-sample</title>
</head>
<body>
  Hello!! This is codebuild-s3-sample html.
  <script src="./index.js"></script>
</body>
</html>
```

```
// src/index.js
console.log('index.js');
```

.gitignore もつくっておきます
```
# .gitignore
node_modules
.cache
release
dist
```

### 2. まずローカルで確認

インストールして、起動

```
$ yarn
$ yarn start
```

http://localhost:1234/ にアクセスして `Hello!! This is codebuild-s3-sample html` と表示されればOK

ビルド処理も確認

```
$ yarn build
```

`release` ディレクトリにビルド後のファイルが生成されればOK


### 3. `buildspec.yml` をルートに生成

デプロイしたい `S3のバケット名` を決め、以下のように `buildspec.yml` ファイルを作ります

```
# buildspec.yml
version: 0.2

phases:
  pre_build:
    commands:
      # install yarn
      - sudo apt-get update && sudo apt-get install apt-transport-https
      - curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
      - echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
      - sudo apt-get update && sudo apt-get install yarn
      - yarn --version
      # install packages
      - yarn
  build:
    commands:
      - yarn build
  post_build:
    commands:
      - aws s3 sync release s3://{S3のバケット名} --delete --acl public-read 
```

#### 解説

```
version: 0.2
```

ビルド仕様のバージョン。2018年11月現在は、バージョン 0.2 を使用することをお勧めされている。

```
# install yarn
- sudo apt-get update && sudo apt-get install apt-transport-https
- curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
- echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
- sudo apt-get update && sudo apt-get install yarn
```

Ubuntuのデフォルトでは `yarn` が使えないのでまず `yarn` をインストール

```
- yarn build
```

`yarn build` し `release` ディレクトリにビルド後のファイルを生成させる 

```
- aws s3 sync release s3://{S3のバケット名} --delete --acl public-read 
```

最後に `AWS CLI` を使って `release` ディレクトリの内容をS3にデプロイします

`--delete` でS3にしかないファイルはS3から削除

`--acl public-read` で `public read` にして公開しています


このようにCodeBuildでは **デフォルトで `AWS CLI` が使えるところが、とても便利です**


### 4. S3バケットをつくる

先ほど指定した名前のS3バケットを、`AWS S3` https://aws.amazon.com/jp/s3/ にログインして作ります


### 5. CodeBuildのビルドプロジェクトを作成

`AWS CodeBuild` https://aws.amazon.com/jp/codebuild/ にログイン

「ビルドプロジェクトを作成する」ボタンを押す

以下のように設定するだけ

<img src="images/1.png" alt="CodeBuild Settings" />

**※GitへのPUSHをトリガーにしたビルドは `Private` リポジトリで可能**

<img src="images/2.png" alt="CodeBuild Settings" />

**今回はAWSに用意されている環境イメージを使うので「マネージド型イメージ」「Ubuntu」「Node.js」を選択**

<img src="images/3.png" alt="CodeBuild Settings" />

「buildspec.yml」を選択することで、先ほどルートに配置したymlファイルが使われる

Buildspec名に、ymlのファイル名を指定することも可能


### 6. IAMでロールにS3 FullAccessControll を付与

`AWS IAM` https://aws.amazon.com/jp/iam/ にログイン

左メニューから「ロール」を選択

先ほど作成した `codebuild-codebuild-s3-sample-service-role` を選択

「ポリシーをアタッチします」ボタンから、`AmazonS3FullAccess` を選択し、「ポリシーのアタッチ」ボタン押下

### 7. ビルドしてみる

CodeBuildの画面から、先ほど作成したビルドプロジェクト `codebuild-s3-sample` を選択し、「ビルドの開始」ボタン押下

確認画面では何も変更せず、そのまま「ビルドの開始」ボタン押下

↓このようにログが出てくる

<img src="images/4.png" alt="CodeBuild Settings" />

### 8. S3にアップされたかどうかを確認

対象のバケットにアップロード＆公開されたことが確認できました

<img src="images/5.png" alt="CodeBuild Settings" />

## GitへのPushをトリガーにしてビルド開始したい場合

Github Private リポジトリでのみ可能

正規表現でブランチ名を指定する

## 開発環境、ステージング環境、本番環境ごとにビルドしたい場合

3つのビルドプロジェクトを作り

`buildspec-develop.yml` `buildspec-staging.yml` `buildspec-production.yml` 

のように3環境分のymlファイルを作れば良い

それぞれのビルドプロジェクトで使用するymlファイル名を指定する


## CloudFrontのキャッシュをクリアする

`AWS CLI` を使えばできる

```
// buildspec.yml
post_build:
  commands:
    - aws s3 sync release s3://codebuild-s3-sample --delete --acl public-read 
    # 以下を追加
    - aws cloudfront create-invalidation --distribution-id ${CLOUDFRONT_DISTRIBUTION_ID} --paths '/*'
```


## Dockerイメージを使用したい場合

先ほどは `AWS CodeBuild` にあらかじめ用意されているイメージを使いましたが、

自分で `Dockerfile` を用意し `Docker Hub` と `Github` を連携することで、カスタムイメージでビルドすることが可能

参考）https://php-java.com/archives/2358


## （Advance）`CodePipeline` と連携して、テストを自動化する


## （Advance）`CloudWatchEvent` `Lambda` と連携して、Slack通知する



# まとめ

開発環境がAWSであるなら、今後 `CodeBuild` を使った方が良い！

ただし、`CodePipeline` や `Lambda` との連携、テスト結果の `slack通知` など完璧に実現したい場合は、ちょっとややこしいらしい（まだ試せていません・・）

---


## 参考

https://int128.hatenablog.com/entry/2018/04/20/190949#f-60bda4f5

https://php-java.com/archives/2358
