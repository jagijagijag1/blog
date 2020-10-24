---
title: QrunchブログのサイドモジュールにPixelaグラフを表示する
author: jagijagijag1
date: 2019-09-03T22:53:56+0900
tags:
- pixela
categories:
- dev note

status: public
created_at: 2019-09-03 22:53:56 +0900
updated_at: 2019-09-03 22:55:19 +0900
published_at: 2019-09-03 22:53:56 +0900
---
- 先日Qrunchの大幅アップデートにより，ブログのカスタマイズ性が上がった
    - 任意のHTML・JSスクリプトが挿入可能に

[【お知らせ】正式リリース前最後の大幅アップデートを行います！（8月下旬予定）](https://qrunch.net/@dev/entries/fi1I06D1ApqOHPWr?ref=qrunch)

- ということで，↓で紹介されているはてなブログへのPixelaグラフ埋め込みをQrunchでもできるはず!

[はてなブログに Pixela グラフを埋め込んで、さらにツールチップを表示させる方法](https://blog.a-know.me/entry/2018/11/20/220257)

## 完成図
![undefined.jpg](/blog/posts/2019_09_03/512cacf7dedffd32d913e7634d24fa8c.png)
- 表示ごとにincrementされるグラフで簡易PVに
- 過去に作ったストレス草化で，ストレスを感じつつも生存してる報告

## 手順
- a-knowさんのブログ見れば問題ないけど，一応QrunchのUI上でどこを設定するかメモ (指定するコードも[元記事](https://blog.a-know.me/entry/2018/11/20/220257)のまま)
- 以下，[ユーザーメニュー]→[デザインカスタマイズ] or ダッシュボード内[デザイン]で表示されるブログのデザイン設定画面で作業

#### サイドモジュール追加 (グラフ表示領域設定)
1. [サイドモジュール]にて[モジュールを追加する]
1. [モジュールタイプ]で[カスタム]を選択
1. [モジュールタイトル]は任意，[コンテンツ（カスタムHTML）]に下記コード(a-knowさんブログまま)を入力し，[追加する]
```html
<div id="svg-load-area"></div>
<div style="text-align:right;">Powered by <a href="https://pixe.la/" target="_blank">Pixela</a></div>
```

#### ヘッダ下 or フッタ上にscriptタグ追加 (表示するグラフの設定)
1. [カスタムHTML]にて[ヘッダー下] or [フッター上]を選択
    - ヘッダ下 or フッタ上へのコード指定はPRO機能だが，今の所ベータ期間で誰でも無料でPRO機能を有効化できる
        - https://qrunch.net/pro
1. 表示された入力エリアに下記コード(a-knowさんブログまま)を入力し，[適用] (グラフURLは表示したいグラフを指定)
```html
<script src="https://unpkg.com/tippy.js@3/dist/tippy.all.min.js"></script>
<script>
document.addEventListener('DOMContentLoaded', function(){
   $('#svg-load-area').load('https://pixe.la/v1/users/a-know-blog/graphs/page-views?mode=short', function(){
      tippy('.each-day', {
         arrow: true
      });
   });
});
</script>
```
