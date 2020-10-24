---
title: リングフィットアドベンチャーの運動ログをサーバーレスでPixelaに記録する
author: jagijagijag1
date: 2020-01-26T13:19:58+0900
tags:
- serverless
- Go
- pixela
- AWS
categories:
- dev note

status: public
created_at: 2020-01-26 13:19:58 +0900
updated_at: 2020-01-26 13:31:09 +0900
published_at: 2020-01-26 13:19:58 +0900
---
## 作ったもの
- [リングフィットアドベンチャー](https://www.nintendo.co.jp/ring/)の運動ログ(活動時間，消費カロリー，走行距離)を[Pixela](https://pixe.la/)に記録して草化する機能をサーバーレスで開発
- Switch上でスクショを撮りTwitter投稿すると，画像取得・解析してPixelaのグラフに記録

![ringfit2pixela](/blog/posts/2020_01_26/c6c743aa334b62e9b955da06677decfa.png)

### 結果
- 最近のサボりが可視化された💪

![result](/blog/posts/2020_01_26/b48a922a87c58002e2df884e4a59f01a.png)

- トリガになるツイートは↓な感じ
<blockquote class="twitter-tweet"><p lang="und" dir="ltr"><a href="https://twitter.com/hashtag/%E3%83%AA%E3%83%B3%E3%82%B0%E3%83%95%E3%82%A3%E3%83%83%E3%83%88%E3%82%A2%E3%83%89%E3%83%99%E3%83%B3%E3%83%81%E3%83%A3%E3%83%BC?src=hash&amp;ref_src=twsrc%5Etfw">#リングフィットアドベンチャー</a> <a href="https://twitter.com/hashtag/RingFitAdventure?src=hash&amp;ref_src=twsrc%5Etfw">#RingFitAdventure</a> <a href="https://twitter.com/hashtag/NintendoSwitch?src=hash&amp;ref_src=twsrc%5Etfw">#NintendoSwitch</a> <a href="https://t.co/ZnoWCO30No">pic.twitter.com/ZnoWCO30No</a></p>&mdash; JagiJagiJagi (@jagijagijag1) <a href="https://twitter.com/jagijagijag1/status/1220920808576446464?ref_src=twsrc%5Etfw">January 25, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

### 制約
- 下記画像のような日毎の運動ログ画面から`活動時間`，`消費カロリー`, `走行距離`を抽出することのみが対象
  - その他画面のスクショや個別運動ログは対象外
  <img src="https://pbs.twimg.com/media/EPGVAVAUUAEVFM3.jpg" width=60%>
- TwitterのツイートページのHTMLを解析するため，ページの仕様が変わると実行不可
- まれにAWS Rekognitionが`0`を`o`と判定し，Pixelaへの記録時にエラーが発生する (数値しかありえないので無理やり補正できなくもないが未実施)

### 全体方針
- リングフィットの記録はAPIで取得したりはできないので，スクショから情報引っこ抜くしかない(と思われる)
- 頑張り度と継続度がみたいので，ライフログを色々記録している[Pixela](https://pixe.la/)で可視化

#### Switchスクショ取得方針
- Switchのスクショを自動でクラウドストレージに持っていく手段がない
  - SDカードを経由する方法だとバッチ的になる & 怠惰なので続かなそう…
- Swithcでスクショ投稿はできるので，Twitterにあげてそこから自動でごにょごにょする方針に
  → サービス連携として手軽なIFTTTを使って特定のハッシュタグ付きTwitter投稿を自動取得 -> 後続処理のAPIを叩くレシピは後述

#### スクショからの情報抽出
- GarminストレスのときはAWS Rekognitionを使ったが，日本語テキスト検出未対応
  - cf. [Garmin connectのストレス測定結果をPixela + Serverlessで草化 - jagijagijag1's tech blog](https://jagijagijag1.qrunch.io/entries/jRCEVYbJi8MP7v5p)
- そのためGoogle CloudのVision APIを利用する方針で検討，デモ試して行けそう

  <img src="/blog/posts/2020_01_26/7ea15bf82f6c4345ecb533ab0169cfbb.png" width=70%>
  <!--![undefined.jpg](https://s3.qrunch.io/7ea15bf82f6c4345ecb533ab0169cfbb.png)--> 
- と思ったが，日付と活動量の数値だけ取るならRekognitionでもできそうなので，慣れているAWSを使う方針に，こっちでも行けそう
  (↓だけ見ると日本語も取れてるように見えるけど日本語部分は英数字の羅列として結果が返ってくる)

  <img src="/blog/posts/2020_01_26/90a0adbc5d5fd85689a6182620dd32c8.png" width=70%>
  <!--![undefined.jpg](https://s3.qrunch.io/90a0adbc5d5fd85689a6182620dd32c8.png)-->

#### その他つまづきなど
- IFTTTで取得できるリンクは画像自体のリンクでなく画像ツイートページのリンクなので，IFTTTからAPI(Lambda)に渡されるリンクから画像リンクを探し出すはめに…
  - [Integromat](https://www.integromat.com/en/)だとツイートの画像リンク取得までできるっぽい
- 日付について，スクショからは月と日(`mm/dd`)しか取れないため，とりあえず処理実行時の年を付与し，それが未来の日付の場合は年を1年前の日付で記録するよう実装
  - e.g. `01/23` -> `20200123`, `10/23` -> `20191023` (`20201023`は未来なので)
- ツイートページから画像URLを引っこ抜く実装にしたことにより，複数画像のまとめて投稿にも対応可能に
  - ただし画像が多いとIFTTTのWebhookの待ち時間を超過するっぽく，IFTTT上は失敗扱いに (Lambdaは正常終了しPixelaにも記録される)


## 実装内容
- コードは→[jagijagijag1/ringfit2pixela · GitHub](https://github.com/jagijagijag1/ringfit2pixela)

### 環境
- Go 1.13.6
- Serverless Framework 1.61.3

### Lambda function (画像取得・情報抽出・グラフ記録)
- IFTTTのレシピ作成時に，Lambda関数を実行するAPI GatewayのエンドポイントURLが必要になるため先にこちら側を実装
- Serverless Frameworkを用いてGoで実装したLambda関数 + API Gatewayをデプロイ
- 詳細はコード参照，概要は以下
  - IFTTTから送られてきたツイートのURLを入力に，画像URLを引っこ抜き，Rekognitionでのテキスト検知結果を受け，Pixelaに記録する
    - 入力のURLをHTML解析して画像URLを抽出し，画像URLから画像データを取得し，Rekognitionにわたす
    - 記録したい情報は，その情報があるであろう座標をあらかじめ与えておき，Rekognitionの結果のうち想定座標に対して最近傍のテキストを利用

### IFTTT: Twitter → Webhook (API Gateway)
- IFTTで下記レシピで作成する
  - ここではハッシュタグ `#RingFitAdventure`のあるツイートを対象に (任意のハッシュタグでOK)
  - `URL`欄にはデプロイしたAPI Gatewayのエンドポイント(=`make deploy`した際の出力の`ServiceEndpoint`)に`/ringfit2pixela`などのパスをくっつけたURLを設定
- 下記レシピ作成後，Nintendo Switch内でTwitter投稿すればPixelaに記録されるはず
  - ただしIFTTTの実行が結構遅い (10〜20分くらいかかる)

[<img src="https://raw.githubusercontent.com/jagijagijag1/ringfit2pixela/master/docs/ringfit2pixela-ifttt.png" width=40%>](https://github.com/jagijagijag1/ringfit2pixela/blob/master/docs/ringfit2pixela-ifttt.png)
