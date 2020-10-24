---
title: bulmaのcolumnを使った際にモバイル表示するとサイドに余白が出る場合の対処
author: jagijagijag1
date: 2020-04-30T23:29:08+0900
tags:
- CSS
- Bulma
categories:
- trouble note

status: public
created_at: 2020-04-30 23:29:08 +0900
updated_at: 2020-04-30 2  3:29:27 +0900
published_at: 2020-04-30 23:29:08 +0900
---
## 症状
- 下記の緑領域のように，`class`に`column`指定した要素のpaddingが，モバイル表示した際に画面外に表示されてしまう

![undefined.jpg](/blog/posts/2020_04_30/f3e171e9b6ca2a2502a743fb1034b161.png)

## 解決
- 下記`body`へのcss設定を入れる
```css
...
<style lang="scss">
...
body { overflow-x: hidden; }
...
</style>
```

## 参考
- 下記で議論されてる
  - [class "columns" too wide on mobile · Issue #449 · jgthms/bulma · GitHub](https://github.com/jgthms/bulma/issues/449)
