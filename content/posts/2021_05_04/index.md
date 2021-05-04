---
title: "Micronaut on AWS Lambdaを試す"
author: jagijagijag1
date: 2021-05-04T19:53:56+09:00
tags:
- AWS
- serverless
- Java
- Lambda
- Micronaut
categories:
- dev-note
---

GraalVMのネイティブイメージをAWS Lambdaで動かしてみたかったので，Micronautを触ってみたログとつまづきメモ．

#### 目次
<!-- toc -->

- [環境](#環境)
- [準備](#準備)
  - [Micronaut CLIインストール](#micronaut-cliインストール)
  - [SAM CLIインストール](#sam-cliインストール)
- [AWS提供Javaランタイムを用いるLambda関数](#aws提供javaランタイムを用いるlambda関数)
  - [プロジェクト作成](#プロジェクト作成)
  - [ローカル実行でテスト](#ローカル実行でテスト)
  - [デプロイしてテスト](#デプロイしてテスト)
- [ネイティブコンパイル&カスタムランタイムを用いるLambda関数](#ネイティブコンパイルカスタムランタイムを用いるlambda関数)
  - [プロジェクト作成](#プロジェクト作成-1)
  - [ローカル実行でテスト](#ローカル実行でテスト-1)
  - [デプロイしてテスト](#デプロイしてテスト-1)

<!-- tocstop -->

## 環境
- OS: macOS Big Sur 11.3
- Java: openjdk version "15.0.2" 2021-01-19
- Homebrew 3.1.5
- aws-cli/2.2.1
- Docker 20.10.5

## 準備
### Micronaut CLIインストール
- Homebrewでインストール
```bash
$ brew install --cask micronaut-projects/tap/micronaut
```
- 続いて`mn --version`コマンドを実行しようとすると，`"mn"は，開発元を検証できないため開けません．`というメッセージが出る
- 実行許可するため，`システム環境設定`→`セキュリティとプライバシー`内の`一般`を開き，ウインドウ下部の「このまま許可」を選ぶ
- 再度`mn --version`コマンドを実行すると，先程と異なり`"mn"は，開発元を検証できません．開いてもよろしいですか?`というメッセージがでるので，「開く」を選ぶ
- するとバージョン情報が出力される
```bash
$ mn --version
Micronaut Version: 2.5.0
```

### SAM CLIインストール
- Lambda関数のローカルテストとデプロイに利用
```bash
$ brew tap aws/tap
$ brew install aws-sam-cli
$ sam --version
SAM CLI, version 1.23.0
```

## AWS提供Javaランタイムを用いるLambda関数
### プロジェクト作成
- 適当な場所で下記コマンドを実行し，雛形生成
```bash
$ mn create-function-app example.micronaut.complete --lang kotlin --features aws-lambda
```
- 以下，生成された`complete`ディレクトリで作業
- (FATな)JARを生成
  - 成功すると`complete/build/libs/complete-0.1-all.jar`が出力される
```bash
$ ./gradlew shadowJar
```

### ローカル実行でテスト
- sam templateを作成
```yml:template.yml
AWSTemplateFormatVersion: "2010-09-09"
Transform: "AWS::Serverless-2016-10-31"
Description: Micronaut Lambda function.
Resources:
  function:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: build/libs/complete-0.1-all.jar
      Handler: example.micronaut.BookRequestHandler
      Runtime: java11
      Description: Micronaut function
      MemorySize: 512
      Timeout: 15
      # Function's execution role
      Policies:
        - AWSLambdaBasicExecutionRole
        - AWSLambda_ReadOnlyAccess
        - AWSXrayWriteOnlyAccess
        - AWSLambdaVPCAccessExecutionRole
      Tracing: Active
```

- テスト用データを作成
```json:testevent.json
{
  "name": "Building Microservices"
}
```

- 上記テンプレートとデータを用い，Lambda関数をローカル実行
  - 要Docker
  - 最後の部分のレスポンスが想定どおりであることを確認
```bash
$ saml local invoke -e testevent.json
Invoking example.micronaut.BookRequestHandler (java11)
Decompressing /Users/ryo/work/java/java-sandbox/mn-lambda/complete/build/libs/complete-0.1-all.jar
Skip pulling image and use local one: amazon/aws-sam-cli-emulation-image-java11:rapid-1.23.0.

Mounting /private/var/folders/_7/m8bf5d2n6cj5xg9ptdvsry5m0000gn/T/tmpwr1pmk59 as /var/task:ro,delegated inside runtime container
START RequestId: 4e0de115-ae68-4e8d-bdc4-58e7805eb69a Version: $LATEST
04:44:33.933 [main] INFO  i.m.context.env.DefaultEnvironment - Established active environments: [ec2, cloud, function, lambda]
END RequestId: 4e0de115-ae68-4e8d-bdc4-58e7805eb69a
REPORT RequestId: 4e0de115-ae68-4e8d-bdc4-58e7805eb69a  Init Duration: 0.64 ms  Duration: 5730.52 ms    Billed Duration: 5800 ms        Memory Size: 512 MB  Max Memory Used: 512 MB
{"name":"Building Microservices","isbn":"b3b2e286-811e-437e-b85d-04f7e564384a"}
```

### デプロイしてテスト
- `sam deploy`する (事前にjarアップロード用のS3バケットを用意)
```bash
$ sam deploy --stack-name mn-lambda-test --capabilities CAPABILITY_IAM --s3-bucket <your-bucket-name>
```
- デプロイ成功したら，AWS CLIから実行してみる
  - AWS CLI 2からパラメタをbase64として受け付ける様になっているので，そのまま文字列を渡したい場合は`--cli-binary-format raw-in-base64-out`オプションが必要
    - ref: [AWS CLI v2にしたらlambda invokeがコケる - Qiita](https://qiita.com/marqs/items/793b72bbd9557441fbf4)
```bash
$ aws lambda invoke --function-name <your-function-name> --payload '{"name": "Building Microservices"}' response.json --cli-binary-format raw-in-base64-out
```
- 実行が成功すると，`response.json`ファイルが出力される
```json:response.json
{"name":"Building Microservices","isbn":"dfc835cc-8be7-4f78-86db-39a80ab40151"}
```
- AWSマネジメントコンソールなどからトレースを確認可能
  - 今回の場合，コールドスタートの初回起動に約3秒かかった

![f83084f6.png](/blog/posts/2021_05_04/f83084f6.png)


## ネイティブコンパイル&カスタムランタイムを用いるLambda関数
- 今度はGraalVMのネイティブコンパイル版をLambdaで動かしてみる

### プロジェクト作成
- 適当なディレクトリで，graalvm機能対応の雛形を生成
```bash
$ mn create-function-app example.micronaut.complete --lang kotlin --features=aws-lambda-custom-runtime,graalvm
```
- 以下，生成された`complete`ディレクトリで作業
- 下記コマンドでネイティブコンパイルしたイメージを生成
  - 要Docker
  - 成功すると`build/libs/complete-0.1-lambda.zip`が出力される
```bash
$ ./gradlew buildNativeLambda -Pmicronaut.runtime=lambda
```
- 初回実行時，Task `:dockerBuildNative`で失敗
  - エラーメッセージは以下
    > Execution failed for task ':dockerBuildNative'.
    > Could not build image: The command '/bin/sh -c native-image -H:Class=example.micronaut.BookLambdaRuntime -H:Name=application --no-fallback -cp /home/app/libs/*.jar:/home/app/resources:/home/app/application.jar' returned a non-zero code: 137
  - 下記によると，メモリ不足による`OutOfMemory`っぽい
    - [Improve native image build error message · Issue #1140 · quarkusio/quarkus · GitHub](https://github.com/quarkusio/quarkus/issues/1140)
  - Docker Desktopを利用しているので，`Preferences`-`Resources`でメモリ割り当てを増やして解決
  - メモリ8GBのマシンだとコンパイルに10分強かかった…

### ローカル実行でテスト
- sam templateを作成
```yml:template.yml
AWSTemplateFormatVersion: "2010-09-09"
Transform: "AWS::Serverless-2016-10-31"
Description: Micronaut Lambda function with GraalVM native image.
Resources:
  function:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: build/libs/complete-0.1-lambda.zip
      Handler: note.used
      Runtime: provided
      Description: Micronaut native image function
      MemorySize: 512
      Timeout: 15
      # Function's execution role
      Policies:
        - AWSLambdaBasicExecutionRole
        - AWSLambda_ReadOnlyAccess
        - AWSXrayWriteOnlyAccess
        - AWSLambdaVPCAccessExecutionRole
      Tracing: Active
```

- テスト用データを作成
  - 雛形のカスタムランタイムがAPI Gatewayからのリクエストを受ける用に作られているため，API Gatewayリクエスト形式のJSONを準備
  - Lambdaのテストで提供されている`apigateway-aws-proxy`テンプレートの`"body"`要素に書籍名をセット

<details><summary>テスト用データが長いので折りたたみ(詳細はここを開いてください)</summary><div>

```json:testevent.json
{
  "body": "{\"name\": \"Building Microservices\"}",
  "resource": "/{proxy+}",
  "path": "/path/to/resource",
  "httpMethod": "POST",
  "isBase64Encoded": true,
  "queryStringParameters": {
    "foo": "bar"
  },
  "multiValueQueryStringParameters": {
    "foo": [
      "bar"
    ]
  },
  "pathParameters": {
    "proxy": "/path/to/resource"
  },
  "stageVariables": {
    "baz": "qux"
  },
  "headers": {
    "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8",
    "Accept-Encoding": "gzip, deflate, sdch",
    "Accept-Language": "en-US,en;q=0.8",
    "Cache-Control": "max-age=0",
    "CloudFront-Forwarded-Proto": "https",
    "CloudFront-Is-Desktop-Viewer": "true",
    "CloudFront-Is-Mobile-Viewer": "false",
    "CloudFront-Is-SmartTV-Viewer": "false",
    "CloudFront-Is-Tablet-Viewer": "false",
    "CloudFront-Viewer-Country": "US",
    "Host": "1234567890.execute-api.ap-northeast-1.amazonaws.com",
    "Upgrade-Insecure-Requests": "1",
    "User-Agent": "Custom User Agent String",
    "Via": "1.1 08f323deadbeefa7af34d5feb414ce27.cloudfront.net (CloudFront)",
    "X-Amz-Cf-Id": "cDehVQoZnx43VYQb9j2-nvCh-9z396Uhbp027Y2JvkCPNLmGJHqlaA==",
    "X-Forwarded-For": "127.0.0.1, 127.0.0.2",
    "X-Forwarded-Port": "443",
    "X-Forwarded-Proto": "https"
  },
  "multiValueHeaders": {
    "Accept": [
      "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8"
    ],
    "Accept-Encoding": [
      "gzip, deflate, sdch"
    ],
    "Accept-Language": [
      "en-US,en;q=0.8"
    ],
    "Cache-Control": [
      "max-age=0"
    ],
    "CloudFront-Forwarded-Proto": [
      "https"
    ],
    "CloudFront-Is-Desktop-Viewer": [
      "true"
    ],
    "CloudFront-Is-Mobile-Viewer": [
      "false"
    ],
    "CloudFront-Is-SmartTV-Viewer": [
      "false"
    ],
    "CloudFront-Is-Tablet-Viewer": [
      "false"
    ],
    "CloudFront-Viewer-Country": [
      "US"
    ],
    "Host": [
      "0123456789.execute-api.ap-northeast-1.amazonaws.com"
    ],
    "Upgrade-Insecure-Requests": [
      "1"
    ],
    "User-Agent": [
      "Custom User Agent String"
    ],
    "Via": [
      "1.1 08f323deadbeefa7af34d5feb414ce27.cloudfront.net (CloudFront)"
    ],
    "X-Amz-Cf-Id": [
      "cDehVQoZnx43VYQb9j2-nvCh-9z396Uhbp027Y2JvkCPNLmGJHqlaA=="
    ],
    "X-Forwarded-For": [
      "127.0.0.1, 127.0.0.2"
    ],
    "X-Forwarded-Port": [
      "443"
    ],
    "X-Forwarded-Proto": [
      "https"
    ]
  },
  "requestContext": {
    "accountId": "123456789012",
    "resourceId": "123456",
    "stage": "prod",
    "requestId": "c6af9ac6-7b61-11e6-9a41-93e8deadbeef",
    "requestTime": "09/Apr/2015:12:34:56 +0000",
    "requestTimeEpoch": 1428582896000,
    "identity": {
      "cognitoIdentityPoolId": null,
      "accountId": null,
      "cognitoIdentityId": null,
      "caller": null,
      "accessKey": null,
      "sourceIp": "127.0.0.1",
      "cognitoAuthenticationType": null,
      "cognitoAuthenticationProvider": null,
      "userArn": null,
      "userAgent": "Custom User Agent String",
      "user": null
    },
    "path": "/prod/path/to/resource",
    "resourcePath": "/{proxy+}",
    "httpMethod": "POST",
    "apiId": "1234567890",
    "protocol": "HTTP/1.1"
  }
}
```

</div></details>

- 上記テンプレートとデータを用い，Lambda関数をローカル実行
  - 要Docker
  - 最後の部分のレスポンスが想定どおりであることを確認
```bash
$ saml local invoke -e testevent.json
Invoking note.used (provided)
Decompressing /Users/ryo/work/java/java-sandbox/mn-lambda-graalvm/complete/build/libs/complete-0.1-lambda.zip
Skip pulling image and use local one: amazon/aws-sam-cli-emulation-image-provided:rapid-1.23.0.

Mounting /private/var/folders/_7/m8bf5d2n6cj5xg9ptdvsry5m0000gn/T/tmpz4qxuo5t as /var/task:ro,delegated inside runtime container
START RequestId: 9724d856-a797-485f-b665-2c6e40f918d9 Version: $LATEST
05:23:48.981 [main] INFO  i.m.context.env.DefaultEnvironment - Established active environments: [ec2, cloud, function, lambda]
END RequestId: 9724d856-a797-485f-b665-2c6e40f918d9
REPORT RequestId: 9724d856-a797-485f-b665-2c6e40f918d9  Init Duration: 0.23 ms  Duration: 455.86 ms     Billed Duration: 500 ms Memory Size: 512 MB     Max Memory Used: 512 MB
{"statusCode":200,"headers":{"Content-Type":"application/json"},"body":"{\"name\":\"Building Microservices\",\"isbn\":\"0915193e-21b1-4f7e-bf0b-a76b0cff22b7\"}"}
```

### デプロイしてテスト
- `sam deploy`する (事前にjarアップロード用のS3バケットを用意)
```bash
$ sam deploy --stack-name mn-lambda-graalvm-test --capabilities CAPABILITY_IAM --s3-bucket <your-bucket-name>
```
- デプロイ成功したら，AWS CLIから実行してみる
```bash
$ aws lambda invoke --function-name <your-function-name> --payload "$(cat testevent.json)" response.json --cli-binary-format raw-in-base64-out
```
- 実行が成功すると，`response.json`ファイルが出力される
```json:response.json
{"statusCode":200,"headers":{"Content-Type":"application/json"},"body":"{\"name\":\"Building Microservices\",\"isbn\":\"55e9a092-3b03-4957-9459-96b8aef17962\"}"}
```
- AWSマネジメントコンソールなどからトレースを確認可能
  - ネイティブイメージにより，コールドスタートの初回起動が約0.5秒に短縮された

![1909d61.png](/blog/posts/2021_05_04/1909ed61.png)