---
title: Cisco AnyConnectで再接続が数十秒ごとに繰り返される場合の対処
author: jagijagijag1
date: 2019-01-22T18:43:48+0900
tags:
- cisco
categories:
- dev note

status: public
created_at: 2019-01-22 18:43:48 +0900
updated_at: 2019-01-24 07:38:58 +0900
published_at: 2019-01-22 18:43:48 +0900
---
## 環境
- Windows 10 Pro 1803
- Cisco AnyConnect Secure Mobility Client 4.6.011

## 症状
- Cisco AnyConnectにてVPN接続した際に，認証・接続は成功するが，その後再接続(ログではReconnecting)を数秒〜数十秒おきに繰り返し続ける

## 解決
**2019/01/24 追記**
- バージョン4.6.02074で修正済み
  - https://community.cisco.com/t5/vpn-and-anyconnect/anyconnect-reconnects-with-hyper-v-adapter/m-p/3695961/highlight/true#M146261


- アップデートできない場合の暫定対応策は以下
  - Windows 10 ProのHyper-V機能を無効にする
    - ref. [Windows 10 Hyper-Vを有効/無効にする方法 - 備忘録](https://kagasu.hatenablog.com/entry/2016/09/24/192659)
  - Dockerをインストールした場合，Docker自体をアンインストールしていてもHyper-Vが有効になっている可能性あり
  - ~~**TODO**: Hyper-V(おそらく仮想スイッチ)とAnyConnectを共存させる方法は不明~~

