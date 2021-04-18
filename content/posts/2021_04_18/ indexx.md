---
title: "AWS botocore `CreateNetworkAclEntry`の例の誤り"
author: jagijagijag1
date: 2021-04-18T22:27:02+09:00
tags:
- AWS
categories:
- dev note
---

boto3で開発をしていたところ，AWSのドキュメント誤りでハマったのでメモ．
AWSのドキュメントも間違ってる上に長期間修正されないこともあるので信用しすぎない．

## 起こったこと
- boto3で`create_network_acl_entry`しようとしてドキュメントを確認
  - https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/ec2.html#EC2.Client.create_network_acl_entry
- Exampleを見ると，protocolをプロトコル名で指定できるように書いてある
  ```python
  response = client.create_network_acl_entry(
      CidrBlock='0.0.0.0/0',
      Egress=False,
      NetworkAclId='acl-5fb85d36',
      PortRange={
          'From': 53,
          'To': 53,
      },
      Protocol='udp',
      RuleAction='allow',
      RuleNumber=100,
  )
  
  print(response)
  ```
- しかし，サンプル通りにしても動かない
- 少し上の説明読むと，プロトコルは数値で指定しろと書いており，実際数値で引数与えるとうまくいく
  > Protocol (string) --
  > [REQUIRED]
  > The protocol number. A value of "-1" means all protocols. If you specify "-1" or a protocol number other than "6" (TCP), "17" (UDP), or "1" (ICMP), traffic on all ports is allowed, regardless of any ports or ICMP types or codes that you specify. If you specify protocol "58" (ICMPv6) and specify an IPv4 CIDR block, traffic for all ICMP types and codes allowed, regardless of any that you specify. If you specify protocol "58" (ICMPv6) and specify an IPv6 CIDR block, you must specify an ICMP type and code.
- 何が悪いのか散々悩んだ挙げ句に色々探してみたところ，botocoreのところで同じ状況に対するIssueが立っていて，マニュアルの誤りだということでプルリクが作られていた
  - [(InvalidParameterValue) when calling the CreateNetworkAclEntry operation · Issue #1716 · boto/boto3 · GitHub](https://github.com/boto/boto3/issues/1716)
  - [Corrected example of create_network_a… by swetashre · Pull Request #1797 · boto/botocore · GitHub](https://github.com/boto/botocore/pull/1797)
- しかしプルリクがCloseされているが，最新版のbotocoreのドキュメントでは依然として間違ったExampleが記載されている
  - [EC2 — botocore 1.20.53 documentation](https://botocore.amazonaws.com/v1/documentation/api/latest/reference/services/ec2.html#EC2.Client.create_network_acl_entry)
- 修正がどうマージされてるのかよくわからないけど，AWSドキュメントにもそれなりの期間放置されてしまう誤りがあるという
