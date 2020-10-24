---
title: Boostnoteの動作が重いときの一解決事例
author: jagijagijag1
date: 2019-10-05T22:45:43+0900
tags:
- Boostnote
categories:
- trouble note

status: public
created_at: 2019-10-05 22:45:43 +0900
updated_at: 2019-10-05 22:45:43 +0900
published_at: 2019-10-05 22:45:43 +0900
---
## 結論
- "Preference" → "Boostnoteについて" → "解析"で`Boostnote の機能向上のための解析機能を有効にする`をオフにすると解決するかも

## 症状
- Boostnote上で頻繁にカーソル移動・文字入力などの動作が重くなる (キー入力の反映が遅延する)
- CPU使用率を見てみると，不定期にBoostnoteのCPU使用率が異常上昇していた

## 原因調査
- Developer Toolを開くと，AWS CognitoへのPOSTメソッドの失敗が大量にあらわれていた
- 調べると↓とほぼ同様の症状だが，ここでは処理が重いという言及もなし
  - [Amazon Cognito Indentity status 400  Issue #1905  BoostIO/Boostnote  GitHub](https://github.com/BoostIO/Boostnote/issues/1905)

![39903187-1dc24c16-54fc-11e8-88b5-4721fdaf3027.png](https://user-images.githubusercontent.com/16365952/39903187-1dc24c16-54fc-11e8-88b5-4721fdaf3027.png)

- Cognitoへのアクセスは使用状況レポートの収集のためのようで，上記Issueでは無視するかレポート機能をオフにするかとのこと
- 自分のケースではネットワーク接続が無い状況でBoostnoteを使ったときにこの症状が発生していた

## 解決
- 実際に"Preference" → "Boostnoteについて" → "解析"で`Boostnote の機能向上のための解析機能を有効にする`をオフにしてみたところ，動作が重くなる事象がなくなり，CPU使用率も上がらなくなった
