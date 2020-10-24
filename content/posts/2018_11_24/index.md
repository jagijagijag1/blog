---
title: serverless framworkで画像認識して関係する映画を推薦するSlack Botを作ったまとめ (No Server November Challenge)
author: jagijagijag1
date: 2018-11-24T16:33:22+0900
tags:
- Python
- serverless
- slackbot
categories:
- dev note

status: public
created_at: 2018-11-24 16:33:22 +0900
updated_at: 2018-11-24 16:33:22 +0900
published_at: 2018-11-24 16:33:22 +0900
---
- serverless frameworkを開発しているServerless, Inc.の企画で，[No Server November](https://serverless.com/blog/no-server-november-challenge)が開催中
- 11月中に毎週課題が出される
  - `Challenges that are designed to help experienced users level up, and brand new users get started`
- githubリポジトリのリンクを#noServerNovemberをつぶやくとなにか(official Serverless swag)もらえる?

- Nov 12の課題であるAnimalBotとNov 19の課題であるSlack botを組み合わせて開発してみたので内容の紹介
  - Nov 12 AnimalBot: 画像URLをメンションすると写っている動物を返信するTwitter Botを作る課題
  - Nov 19 Slack bot: `/action`とすると80年台アクション映画をランダムに教えてくれるSlack Botを作る課題

### 作ったもの
- Slack上でBotに画像URLをメンションすると，写っている内容と，関連する映画を教えてくれるシステム
  - 画像認識にはAmazon Rekognitionを利用
  - 映画情報は[The Movie Database](https://www.themoviedb.org/)からAPI経由で取得

![undefined.jpg](/blog/posts/2018_11_24/ba35718cf0d0a9329bfc73fb045539f9.jpg)

#### 結果
- 猫の画像を送ると，猫が写っていることと，関連映画として魔女の宅急便を教えてくれた

![undefined.jpg](/blog/posts/2018_11_24/d08779a4ccfe8282e9b5aefde3d943fa.jpg)

#### 環境
- MacOS Mojave
- Python 3.6.5
- Serverless Framework 1.32.0

### つまづきメモ
- [Lambdaプロキシ] POSTリクエスト本体はevent.bodyの中に**String**ではいる(JSONじゃない！)ので，event.body配下を再度JSONとして読み込み直す必要あり


```python
def main(event, context):
  body_str = event['body'] ## これだとただの文字列
  body_json = json.loads(event['body']) ## これでJSONとして扱える
  ...
```


- Lambda + Pythonで画像処理ライブラリPillow(PIL)を使う場合，ローカルがMac,実行環境はLinuxベースのため，ローカルでライブラリを同梱しても動作しない
  - よって，[serverless-python-requirements](https://github.com/UnitedIncome/serverless-python-requirements)を用いてライブラリ管理をする (Amazon LinuxのDockerイメージを利用)
  - 関連する設定は以下


```yaml:serverless.yml
...
provider:
  name: aws
  runtime: python3.6
  ...

plugins:
  - serverless-python-requirements

custom:
  pythonRequirements:
    dockerizePip: true
...
```

- Lambda上で一時的にファイルを作成したい場合は必ず`/tmp`配下を指定する
  - 今回はURLで指定された画像を一旦Lambdaローカルに保存し処理
  - その際，保存先は`/tmp`以外は不可 (権限なし)
    - `OSError: [Errno 30] Read-only file system`

- Slack APIでメッセージをPostする際，画像などの付属情報を設定可能な`attachments`は，JSON内で**Stringとして**格納しなければならない
  - つまり，`attachments`の値は`json.dums`する必要あり


```python
# これはOK
data_correct = {
    ...
    "attachments": json.dumps([
        {
            "title": movie_info['title'],
            "image_url": movie_img_url
        }
    ])
}
# これは駄目
data_err = {
    ....
    "attachments": [
        {
            "title": movie_info['title'],
            "image_url": movie_img_url
        }
    ]
}
```

### 開発詳細
つくったものは[GitHub - jagijagijag1/animal-recog-slack-bot](https://github.com/jagijagijag1/animal-recog-slack-bot)で公開

1. `serverless.yml`, `handler.py`を編集し，ひとまずBot処理を作成
2. Slack，Movie DBで作業し，API Token取得
3. `serverless.yml`を再度編集し，Lambda環境変数にAPI Token情報


#### 1. Serverless framework + Pythonで開始

```bash
$ sls create -t aws-python3 -p <project-name>
```

##### `serverless.yml`の修正
- 環境変数部分は後で埋めるのでひとまずブランク


```yaml:serverless.yml
service: animal-recog-slack-bot

provider:
  name: aws
  runtime: python3.6
  region: ap-northeast-1
  iamRoleStatements:
    - Effect: "Allow"
      Action:
        - "rekognition:DetectLabels"
      Resource: "*"

plugins:
  - serverless-python-requirements

custom:
  pythonRequirements:
    dockerizePip: true

functions:
  hello:
    handler: handler.main
    events:
      - http:
          path: /
          method: POST
    environment:
      OAUTH_TOKEN: ''
      BOT_TOKEN: ''
      MOVIE_DB_API_TOKEN: ''
    timeout: 20

```


##### 関数本体を作成
- やや長いのでソースコードは[こちら](https://github.com/jagijagijag1/animal-recog-slack-bot/blob/master/handler.py)参照

- 画像はURLで受付，一旦Lambda関数ローカルに保存し，RekognitionにByteとして受け渡す
- Rekognitionではかなり一般的な単語(e.g. Animal, Pet)をラベル候補の上位に出してくるため，暫定処理としてNGワード(`ignore_word`)を設定し回避
- 関連映画を取得する処理の概要は以下
  - Rekognitionのラベル検出結果から一語を選択
  - 選択した後がMovie DBでキーワード登録されているか確認(API: /search/keyword)し，登録ありの場合はID取得 (なしの場合は終了)
  - 獲得したキーワードIDを用いて映画検索(API: /discover/movie?with_keyword=<ID>)
  - 検索結果からランダムに選択した映画をSlackに返す

##### デプロイ
```bash
$ sls plugin install -n serverless-python-requirements
$ docker pull lambci``/lambda``:build-python3.6
$ sls deploy -v
```

#### 2. Slack，Movie DBで作業し，API Token取得
##### Slack app準備
- [Building Slack apps \| Slack](https://api.slack.com/slack-apps)でCreate
- その後の画面で左側メニューの"Bot User"を選び，Bot Userを追加
- 左メニュー"Event Subscriptions"からイベントを有効にし，"Request URL"にAPI GatewayのURLを指定し，challengeが帰ってくることを確認 (Lambdaコードに処理を埋め込み済)
- challenge成功後，"Subscribe to Bot Events"で"app_mention"イベントを追加
- 左メニュー"Installed App"からアプリをWorkspaceにインストール
- 左メニュ「OAuth & Permissions」にてWorkspaceにAppをInstallすると"OAuth Access Token"と"Bot User OAuth Access Token"が発行される (あとで`serverless.yml`に追記)

##### Movie DB準備
- 登録し，Settingから申請可能
  - 参考：[API Docs](https://developers.themoviedb.org/3/getting-started/introduction)
- 申請完了後もSetting->APIでAPIキーを確認可能
  - 本アプリではv3 auth利用

#### 3. `serverless.yml`を再度編集し，Lambda環境変数にAPI Token情報を追記
- OAUTH_TOKEN: SlackのOAuth Access Token
- BOT_TOKEN: SlackのBot User OAuth Access Token
- MOVIE_DB_API_TOKEN: The Movie DatabaseのAPIキー (v3 auth)


```yaml:serverless.yml
...
functions:
  hello:
    ...
    environment:
      OAUTH_TOKEN: <your-token>
      BOT_TOKEN: <your-token>
      MOVIE_DB_API_TOKEN: <your-token>
...
```


---
## 参考
- slack bot開発
  - [AWS初心者でもわかる！ ブラウザ上で完結！ AWS+Slack Event APIを使ったSlackボット超入門 - Qiita](https://qiita.com/omd/items/fcbbdb2bcce3edf0d3f5)
