---
title: '"あとで読む"をPixelaで可視化'
author: jagijagijag1
date: 2018-12-02T10:54:22+0900
tags:
- pixela
- IFTTT
categories:
- dev note

status: public
created_at: 2018-12-02 10:54:22 +0900
updated_at: 2018-12-02 11:47:38 +0900
published_at: 2018-12-02 10:54:22 +0900
---
## tl;dr
なんでも草化できるWebサービス[Pixela](https://pixe.la)の利用についてつぶやいたところ，作者のa-knowさんから下記ツイートをいただきました！

<blockquote class="twitter-tweet" data-lang="ja"><p lang="ja" dir="ltr">うおお……これはまた新しい活躍方法…！！ぜひブログなどにしていただきたいっ！ <a href="https://t.co/Cz5zrPVG0Q">https://t.co/Cz5zrPVG0Q</a></p>&mdash; a-know (@a_know) <a href="https://twitter.com/a_know/status/1068691858220445696?ref_src=twsrc%5Etfw">2018年12月1日</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>


正直なところ技術的なポイントは何もないのですが，せっかくなのでPixelaの利用事例として記事化してみました．やったことは以下だけです．
- [PixelaのWebhook](https://pixe.la/#api-webhook)と[IFTTTのInstapaper連携](https://ifttt.com/instapaper)を用いることで下記を実現し，"あとで読む"を可視化
  1. Instapaperに新たに記事が追加されたら，DecrementのWebhookを叩きpixelを茶色化
  2. Instapaperに保存した記事が(読み終わって)アーカイブされたら，IncrementのWebhookを叩きpixelを緑化

![undefined.jpg](/blog/posts/2018_12_02/bfacefb91e716a338ed402bb54f70ef0.jpg)

## 動機：溜まっていく一方の"あとで読む"
- 気になった記事があると，あとでちゃんと読もうとInstapaperやはてブなどのサービスに記事を放り込んでおくのはよくあるかなと
- しかしこの"あとで読む"は，積読や積みゲーよろしく溜まっていく一方になりがち
- これに対し，下記の点を可視化できれば溜め込まないモチベになるかも？
  - どれくらい記事をあとで読むつもりで放り込んでいるか (溜め込み度合い)
  - どれくらいあとで読むことにした記事を読んでいるか (消化度合い)


## 解決(?)：Pixelaでの可視化
以下のIFTTT Appletを作成し，"あとで読む"記事の数を可視化
1. Instapaper (If New item saved) -> Webhooks (then Make a web request)
   - Instapaper部分は，事前にアカウント連携しておき，既存のアクションを選択するのみ
   - Webhooksでは，PixelaのDecrement Webhookを実行
     - 記事が増えるのはネガティブ方向なのでDecrement = 茶色化
2. Instapaper (If New archived item) -> Webhooks (then Make a web request)
   - Instapaper部分は，事前にアカウント連携しておき，既存のアクションを選択するのみ
   - Webhooksでは，PixelaのIncrement Webhookを実行
     - 記事が減るのはポジティブ方向なのでIncrement = 緑化

![undefined.jpg](/blog/posts/2018_12_02/d991fb17c3729eac219b47d473213d36.jpg)

#### Webhooksの部分の詳細
- [Pixela API Document](https://docs.pixe.la/#/post-webhook)を参考に，Increment/DecrementのWebhookを作っておく
- IFTTT上では，"that"側で"Webhooks"を選択し，下記パラメタを設定
  - URL: `https://pixe.la/v1/users/<your-user-id>/webhooks/<your-webhook-hash>`
  - Method: `POST`
  - ContentType: `application/json`
  - Body: blankでOK


## お試し中：記事数の累計値の草化
- Pixela自体は日々の数値を記録するものだが，前日の数値を引き継げば累計値の可視化も可能ではと思い立つ (もはや草で可視化しなくてもいいのですが)
- そこで，1日1回，前日の数値を当日に記録する処理をサーバーレスで実装
  - AWS Lambda関数の実装：[cumulated-pixela](https://github.com/jagijagijag1/cumulated-pixela)
- あとは記事が追加されたらIncrement，アーカイブされたらDecrementするようIFTTT Appletを設定
- 現在お試し運用中，下記のようなイメージに→グラフから草を消すという逆モチベに

![undefined.jpg](https://s3.qrunch.io/6cbe08951a59cc1e5486b040148b2212.jpg)
