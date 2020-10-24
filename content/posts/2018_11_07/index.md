---
title: CAV 2018でのAWS招待講演のまとめ
author: jagijagijag1
date: 2018-11-07T22:29:12+0900
tags:
- AWS
- 形式手法
categories:
- tech note

status: public
created_at: 2018-11-07 22:29:12 +0900
updated_at: 2018-11-07 22:29:12 +0900
published_at: 2018-11-07 22:29:12 +0900
---
# Formal Reasoning About the Security of Amazon Web Services

形式手法に関するトップカンファレンスである[CAV 2018](http://cavconference.org/2018/)にて，AWSから講演(Invited Paperの発表)がありました．

そこでの発表からAWSは形式手法にかなり力を入れている印象を受け，割と重要な内容かなと思い，単なる箇条書きですが抜粋・まとめてみました．
月並みですがAWSも中の人達もまじすごい．

### 著者(発表者)/所属機関
Byron Cook (Amazon Web Services, University College London)

### 出典
FLoC 2018 (CAV 2018)

- FLoC Plenary Lecture (CAV 2018 Invited Paper)
  - AWSでの形式手法の活用について
  - Open Accessのため↓で閲覧可能
    - [Formal Reasoning About the Security of Amazon Web Services \| SpringerLink](https://link.springer.com/chapter/10.1007/978-3-319-96145-3_3)
  - 講演の映像も↓から視聴可能
    - [Formal Reasoning about the Security of Amazon Web Services \| University of Oxford Podcasts - Audio and Video Lectures](https://podcasts.ox.ac.uk/formal-reasoning-about-security-amazon-web-services)

## 内容
- AWSでは分散アルゴリズムの設計に形式手法(TLA+)を使用 (2014)
- その後も継続的に形式手法を取り入れいているようで，本論文では特にセキュリティへの形式手法適用事例を紹介

### Security _of_ the Cloud: Where Formal Reasoning Fits In
- AWS内部の開発，特にセキュリティレビューで定理証明やモデル検査(symbolic model checking)の利用が増えている
- 2017年だけでも下記の事例にて定理証明やモデル検査を適用
  - [s2n](https://github.com/awslabs/s2n) (TLS/SSL実装)の検証
    - ちなみにこの内容の論文が同会議にて発表されている
    - [Continuous Formal Verification of Amazon s2n \| SpringerLink](https://link.springer.com/chapter/10.1007/978-3-319-96142-2_26)
  - ハイパーバイザ，ブートローダ，BIOS，ファームウェアの検証
    - こっちも同会議で発表あり
    - [Model Checking Boot Code from AWS Data Centers \| SpringerLink](https://link.springer.com/chapter/10.1007/978-3-319-96142-2_28)
  - ガーベッジコレクタ
  - ネットワーク設計
- OSSを活用しつつ使う上で生じた変更をフィードバックしており，例えば下記を利用+貢献
  - [CBMC](https://www.cprover.org/cbmc/): C/C++向けのbounded model checker
  - [SAW](https://saw.galois.com): CやJabaでの特定性質の検証
  - [SMACK](http://smackers.github.io/): Cプログラムに対するアサーション検証
- s2nなど一部プロジェクトでは形式検証ツールをCI/CDに組み込み，継続的検証を実施
- SMT-basedのツールで設定ミス防止を図っている
- 形式手法の利用はコードを書く前から始まっている＝プロトコルやアルゴリズムの設計段階から利用

### Securing Customers _in_ the Cloud
- AWS利用者側に対しても形式手法の適用が始まっている
  - 例えば，S3の公開設定ミスを警告する機能をSMTソルバを用いて実現
    - おそらく↓の話
    - [How AWS uses automated reasoning to help you achieve security at scale \| AWS Security Blog](https://aws.amazon.com/jp/blogs/security/protect-sensitive-data-in-the-cloud-with-automated-reasoning-zelkova/)
  - AWS Macieでも同様のツールを利用
  - 複雑なネットワーク設定時の到達性(おそらくルーティングや通信可否)をdatalogを用いて検証

### その他
- 他社事例(文献)として以下に言及 (抜粋)
  - IBM, Google @ Dagstuhl Seminar 2013
  - Microsoft @ SOSP 2015 分散システムの検証
  - Facebook @ LICS 2018 codebaseの継続的検証

## 感想
- TLA+の適用に始まり，AWSは形式手法の適用に対し継続的に力を入れいている印象
- 特に今回は，形式検証のトップカンファレンスであるCAVにて2本の論文採択+Invited talkをしており抜きん出ている
- こういった会議でプレゼンスを上げることで，AWS利用時の安心度・信頼感が高まる素晴らしい取り組み
- これだけセキュリティチームで形式手法が利用される/開発者もTLA+などを使えるところを見るに，中の人がちゃんとComputer Scienceの素養を持っていて優秀な模様
