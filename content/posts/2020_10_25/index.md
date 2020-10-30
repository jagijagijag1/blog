---
title: "Qrunch記事のHugo on GitHub Pages移行とHugoテーマ拡張"
author: jagijagijag1
date: 2020-10-25T22:28:40+09:00
tags:
- Hugo
- GitHub
- GitHub Actions
categories:
- dev note
---
<!-- toc -->

- [Hugo on GitHub Pagesの構成](#hugo-on-github-pagesの構成)
  - [移行したときのつまづき](#移行したときのつまづき)
      - [画像リンク修正](#画像リンク修正)
- [テーマ`Hugo Future Imperfect Slim`の拡張](#テーマhugo-future-imperfect-slimの拡張)
  - [cssの上書き](#cssの上書き)
  - [ブログパーツの設置](#ブログパーツの設置)
    - [拍手ボタン](#拍手ボタン)
    - [アクセスカウンタ](#アクセスカウンタ)
    - [Pixelaユーザページのsocal icon](#pixelaユーザページのsocal-icon)

<!-- tocstop -->

---
2020/10/30 追記ここから
- 記事公開後コメントいただき，テーマをいじりたい場合はテーマ側をforkしてくる必要ないことが判明
<blockquote class="twitter-tweet" data-partner="tweetdeck"><p lang="ja" dir="ltr">ʕ◔ϖ◔ʔ 👍<br>ちなみに ${hugo_project_dir}/layouts/partials/site-sidebar.html を用意すれば themes 以下のファイルを上書きできたりします。<a href="https://t.co/cLbI9jauJg">https://t.co/cLbI9jauJg</a></p>&mdash; peaceiris (@piris314) <a href="https://twitter.com/piris314/status/1320617884578533376?ref_src=twsrc%5Etfw">October 26, 2020</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

- 本記事でforkしたテーマを編集しているところは，Hugoプロジェクト側の適切なパスに同内容のファイルを置けばOK

2020/10/30 追記ここまで

---
閉鎖してしまうQrunchで書いた記事をGitHub Pagesに移行したのと，選んだHugoテーマを少し拡張したので，そのときにやったことを書き留め．
- このブログのHugoソースは→[GitHub - jagijagijag1/blog at source](https://github.com/jagijagijag1/blog/tree/source)
- このブログ用に拡張したテーマは→[GitHub - jagijagijag1/hugo-future-imperfect-slim at add-pixela-social](https://github.com/jagijagijag1/hugo-future-imperfect-slim/tree/add-pixela-social)


## Hugo on GitHub Pagesの構成
- Qrunchが閉鎖するため移行先としてGitHub Pagesを選択
  - Qrunch以上に合いそうなサービスがなかったため
- ブログはGitHubプロジェクト(not ユーザ)に紐づくGitHub Pagesを利用 (`https://<user-name>.github.io/<repository-name>`にホスト)
  - ブログ単品としてホストしたいので，リポジトリ名を`blog`にして作成
  - `master`ブランチの`docs`ディレクトリ配下をホストするよう設定
- `source`ブランチにてHugoファイルを管理 + `source`ブランチへのプッシュで起動するGitHub Actionsを設定し`master`ブランチにHugoの静的サイト生成結果をpush
```yml:.github/workflows/main.yaml
name: build hugo site on source & deploy to master

on:
  push:
    branches:
      - source

jobs:
  deploy:
    runs-on: ubuntu-18.04
    steps:
      - uses: actions/checkout@v2
        with:
          submodules: true
          fetch-depth: 0

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: '0.75.1'

      - name: Build
        run: hugo --minify

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_branch: master
          publish_dir: ./docs
          destination_dir: ./docs
```


### 移行したときのつまづき
- QrunchはMarkdownで記述可能であったため，Exportしてきた記事データの修正点は大きく下記2つ
  - ヘッダ情報の修正 (dateのフォーマットなどをやるだけ)
  - 本文中の画像リンク (若干つまづいたので後述)

##### 画像リンク修正
- Hugo上での記事管理は下記のディレクトリ構造にした
```
...
├── content
|   ├── posts
|   |   ├── 20xx_xx_xx
|   |   |   ├── xxxxxxx.png
|   |   |   └── index.md
|   |   ├── 20xx_xx_xx
|   |   |   ├── xxxxxxx.png
|   |   |   └── index.md
|   |   |
...
```

- またプロジェクトに紐づくGitHub Pagesを利用したため，Hugoの設定でbase urlを設定
```toml:config.toml
baseURL = "http://jagijagijag1.github.io/blog/"
publishDir = "docs"
...
```

- 上記設定の場合，画像リンクは下記の通り修正して解決
  - 修正前: `[image](https://s3.qrunch.io/xxxxxxx.png)`
  - 修正後: `[image](/blog/posts/20xx_xx_xx/xxxxxxx.png)`
    - `[image](xxxxxxx.png)`で指定すると，記事中では正常に画像表示されるが，記事一覧ページからだとリンクがおかしくなって画像表示されなかった

(ここにたどり着くまでどういうディレクトリ構造がいいかとか，base url変えた時にどう画像リンク指定するかがよくわからず四苦八苦した…)


## テーマ`Hugo Future Imperfect Slim`の拡張
- 選んだテーマを少し改善・拡張

### cssの上書き
- `static/css/add-on.css`に記述した内容は，テーマのCSSを上書きしてくれる
- デフォルトだと文字間隔やコードブロックのフォントなどが少し気に入らなかったので下記のように修正
```css:static/css/add-on.css
#site-sidebar ul,
.stats ul li {
  text-transform: none;
}

.nav {
  text-transform: none;
  letter-spacing: 0.2em;
}

h1,
h2,
h3,
h4,
h5,
h6 {
  text-transform: none;
  letter-spacing: 0.1em;
}

code {
  font-family: Monaco, Inconsolata, Consolas, monospace;
}

.count-container > .count {
  font-family: Monaco, Inconsolata, Consolas, monospace;
}
```

### ブログパーツの設置
- Hugo Future Imperfect Slimのリポジトリをforkして拡張
  - Hugoで下記コマンドでテーマ取得して`config.toml`でテーマ指定+各パーツのパラメタ指定すれば利用可能
    `git submodule add -b add-pixela-social https://github.com/jagijagijag1/hugo-future-imperfect-slim.git themes/hugo-future-imperfect-slim`

#### 拍手ボタン
- Qrunchにはあった拍手ボタンをこっちにも一応つけてみる
  - 拍手ボタンには[Applause Button](https://applause-button.com/)を利用
- まず拍手ボタンを使うためのCSSとJSを読み込み
```html:hugo-future-imperfect-slim/layouts/partials/head.html
<head>
  ...
  {{ if .Site.Params.enableApplause }}
    <link href="https://unpkg.com/applause-button/dist/applause-button.css" rel="stylesheet" type="text/css">
    <script src="https://unpkg.com/applause-button/dist/applause-button.js"></script>
  {{ end }}
</head>

```

- 次に，拍手ボタンを設置する部品を下記の通り作成
  - `url`属性に各記事ページのURL(`Permlink`)を設定することで，記事ごとに拍手数を保持できるようにする
```html:hugo-future-imperfect-slim/layouts/_default/single.html
{{ if .Site.Params.enableApplause }}
  <div>
    <applause-button color="{{ .Site.Params.applauseButtonColor }}" url="{{ .Permalink }}" multiclap="true" style="width: 55px; height: 55px; margin-right: 15px;"/>
  </div>
{{ end }} 
```
- その後上記部品を各記事に表示するよう設定
```html:hugo-future-imperfect-slim/layouts/partials/applause.html
...
  <article class="post">
    {{ .Render "header" }}
    <div id="socnet-share">
      {{ partial "applause" . }}  <!-- Add this line -->
      {{ partial "share-buttons" . }}
    </div>
...
```
- ただしApplause ButtonのCSSがブログテーマのCSSと競合しているので，下記を追加設定
  - 具体的には，カテゴリの記事数の要素に与えられる`.count`と，拍手数の要素に与えられる`.count`が競合するので，拍手数の`.count-container > .count`のCSSに手を加える
  - また`applause-button`の`color`属性で指定した値は，`.count-container`の親要素の`.style-root`が持っているので，そこから無理やり継承する
```css:hugo-future-imperfect-slim/assets/css/applausebutton.css
.count-container {
  color: inherit !important;
}

.count-container > .count {
  float: none !important;
  color: inherit !important;
}
```
```json:hugo-future-imperfect-slim/assets/assets.json
{
   "styles" : [
      "css/applausebutton.css",
      ...
```

- 上記が反映されたテーマをsubmoduleに入れ，ブログ本体の設定ファイルに以下を追記すれば各記事に拍手ボタンが表示される
```toml:config.toml
[params]
  ...
  enableApplause        = true
  applauseButtonColor   = "firebrick"
```


#### アクセスカウンタ
- Hugo Future Imperfect Slimでサイドバーに，Qrunchでも使っていたPixelaグラフのアクセスを追加
- サイドバー設定ファイルに以下を追記
  - `iframe`タグでも描画可能だが，テーマの色に合わせて強制で白背景にしたかったので`script`タグを使って実現
```html:hugo-future-imperfect-slim/layouts/partials/site-sidebar.html
...
  {{ if .Site.Params.sidebar.pixelaAccessCounter }}
      <section id="pixela_access_counter">
        <header>
          <h1>Blog PVs</h1>
        </header>
        <div id="svg-load-area" style="border: 1px solid rgba(161, 161, 161, 0.3);"></div>
        <div style="text-align:right;">Powered by <a href="https://pixe.la/" target="_blank">Pixela</a></div>
      </section>
      <script src="https://unpkg.com/tippy.js@3/dist/tippy.all.min.js"></script>
      <script>
      document.addEventListener('DOMContentLoaded', function(){
        $('#svg-load-area').load('{{.Site.Params.sidebar.pixelaACGraphURL}}', function(){
          tippy('.each-day', {
            arrow: true
          });
        });
      });
      </script>
  {{ end }}
...
```
- 上記が反映されたテーマをsubmoduleに入れ，ブログ本体の設定ファイルに以下を追記すればサイドバーにグラフが表示される
```toml:config.toml
...
  [params.sidebar]
    pixelaAccessCounter = true
    pixelaACGraphURL    = "https://pixe.la/v1/users/<user-id>/graphs/<graph-id>?mode=short"
...
```

#### Pixelaユーザページのsocal icon
- Hugo Future Imperfect SlimではサイドバーにTwitterなどのソーシャルアカウントへのリンクアイコンを設置可能
- 最近Pixelaにてユーザプロフィールページがリリースされたので自分のページへのリンクを追加してみた
  - [Pixela にユーザープロフィールページができました！（v1.20.0 リリース） #pixela - えいのうにっき](https://blog.a-know.me/entry/2020/10/16/094717)
- social icon設定ファイルに下記を追記
```html:hugo-future-imperfect-slim/layouts/partials/socnet-icon.html
...
  {{ with .Site.Social.pixela }}<li><a href="//pixe.la/{{.}}" target="_blank" rel="noopener" title="Pixela" class="fas fa-th"></a></li>{{ end }}
```
- 上記が反映されたテーマをsubmoduleに入れ，ブログ本体の設定ファイルに以下を追記すればサイドバーにソーシャルアイコンが表示される
```toml:config.toml
[social]
  ...
  pixela = "@<pixela-user-id>"
```
