---
title: "Quarkus on AWS Lambdaを試す"
author: jagijagijag1
date: 2021-05-15T20:53:05+09:00
tags:
- AWS
- serverless
- Java
- Lambda
- Quarkus
categories:
- dev-note
---

## 環境
- OS: macOS Big Sur 11.3
- Java: openjdk version "15.0.2" 2021-01-19
- aws-cli/2.2.1
- Docker 20.10.5
- Apache Maven 3.8.1


## AWS提供Javaランタイムを用いるLambda関数
### プロジェクト作成
- 適当な場所で下記コマンドを実行し，雛形生成
  - 途中，IDなど聞かれるので適当な値を入力
```bash
$ mvn archetype:generate \
       -DarchetypeGroupId=io.quarkus \
       -DarchetypeArtifactId=quarkus-amazon-lambda-archetype \
       -DarchetypeVersion=1.13.3.Final
...
Define value for property 'groupId': example
Define value for property 'artifactId': quarkus-lambda
Define value for property 'version' 1.0-SNAPSHOT: : 
Define value for property 'package' example: : 
Confirm properties configuration:
groupId: example
artifactId: quarkus-lambda
version: 1.0-SNAPSHOT
package: example
 Y: : 
 ...
```
- 成功すると以下のような構造の雛形が生成される
```bash
# "quarkus-lambda"の部分は雛形生成時に指定したartifactId
$ tree quarkus-lambda
quarkus-lambda
├── build.gradle
├── gradle.properties
├── payload.json
├── pom.xml
├── settings.gradle
└── src
    ├── main
    │   ├── java
    │   │   └── example
    │   │       ├── InputObject.java
    │   │       ├── OutputObject.java
    │   │       ├── ProcessingService.java
    │   │       ├── StreamLambda.java
    │   │       ├── TestLambda.java
    │   │       └── UnusedLambda.java
    │   └── resources
    │       └── application.properties
    └── test
        ├── java
        │   └── example
        │       └── LambdaHandlerTest.java
        └── resources
            └── application.properties
```
- 生成されたディレクトリ(上記の例だと`quarkus-lambda`)に移動し，JARを生成
  - 成功すると`targets/`配下に，ビルド結果やLambda関数を管理するためのシェルスクリプト/SAMテンプレートが出力される
```
$ mvn package
```

### ローカル実行でテスト
- 上記ビルドで，`targets/sam.jvm.yaml`にSAM用のテンプレートが生成されている
  - これは`sam local invoke --template target/sam.jvm.yaml --event payload.json`でローカル実行可能
- しかし，このあと，API GatewayなしのLambda関数単体でAWSにデプロイしたいため，下記のsam templateを新たに作成
```yml:template.jvm.yml
AWSTemplateFormatVersion: "2010-09-09"
Transform: "AWS::Serverless-2016-10-31"
Description: Quarkus Lambda function.
Resources:
  function:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: target/function.zip
      Handler: io.quarkus.amazon.lambda.runtime.QuarkusStreamHandler::handleRequest
      Runtime: java11
      Description: Java function
      MemorySize: 128
      Timeout: 15
      # Function's execution role
      Policies:
        - AWSLambdaBasicExecutionRole
        - AWSLambda_ReadOnlyAccess
        - AWSXrayWriteOnlyAccess
        - AWSLambdaVPCAccessExecutionRole
      Tracing: Active
```

- 上記テンプレートとテスト用データ`payload.json`を用い，Lambda関数をローカル実行
  - 要Docker
  - 最後の部分のレスポンスが想定どおりであることを確認
```bash
$ sam local invoke --template template.jvm.yml --event payload.json
Invoking io.quarkus.amazon.lambda.runtime.QuarkusStreamHandler::handleRequest (java11)
Decompressing /Users/ryo/work/java/java-sandbox/quarkus-lambda/target/function.zip
Skip pulling image and use local one: amazon/aws-sam-cli-emulation-image-java11:rapid-1.23.0.

Mounting /private/var/folders/_7/m8bf5d2n6cj5xg9ptdvsry5m0000gn/T/tmpmyw_yeut as /var/task:ro,delegated inside runtime container
__  ____  __  _____   ___  __ ____  ______ 
 --/ __ \/ / / / _ | / _ \/ //_/ / / / __/ 
 -/ /_/ / /_/ / __ |/ , _/ ,< / /_/ /\ \   
--\___\_\____/_/ |_/_/|_/_/|_|\____/___/   
2021-05-11 13:23:01,986 INFO  [io.quarkus] (main) quarkus-lambda 1.0-SNAPSHOT on JVM (powered by Quarkus 1.13.3.Final) started in 2.167s. 
2021-05-11 13:23:01,989 INFO  [io.quarkus] (main) Profile prod activated. 
2021-05-11 13:23:01,989 INFO  [io.quarkus] (main) Installed features: [amazon-lambda, cdi]
END RequestId: 6191b111-bf3b-4f40-9b3a-15d4f7364969
REPORT RequestId: 6191b111-bf3b-4f40-9b3a-15d4f7364969  Init Duration: 0.74 ms  Duration: 2513.93 ms    Billed Duration: 2600 ms        Memory Size: 512 MB      Max Memory Used: 512 MB
{"result":"hello Bill","requestId":"6191b111-bf3b-4f40-9b3a-15d4f7364969"}
```

### デプロイしてテスト
- `sam deploy`する (事前にzipアップロード用のS3バケットを用意)
```bash
$ sam deploy --template template.jvm.yml --stack-name quarkus-lambda-test --capabilities CAPABILITY_IAM --s3-bucket <your-bucket-name>
```

- デプロイ成功したら，AWS CLIから実行してみる
```bash
$ aws lambda invoke --function-name <your-function-name> --payload '{"name": "jagijagijag1", "greeting": "Hi!"}' response.json --cli-binary-format raw-in-base64-out
```

- 実行が成功すると，`response.json`ファイルが出力される
```json:response.json
{"result":"Hi! jagijagijag1","requestId":"845fc222-8947-4fc1-a899-4f6a7c4b42ad"}
```

- AWSマネジメントコンソールなどからトレースを確認可能
  - 今回の場合，コールドスタートの初回起動に約1.7秒かかった (Javaの割に短い?)

![cc3f9d32.png](/blog/posts/2021_05_15/cc3f9d32.png)


## ネイティブコンパイル&カスタムランタイムを用いるLambda関数

### コンパイル
- プロジェクトはJVM利用版と同じものを利用
- 下記コマンドでネイティブイメージ&カスタムランタイム生成
  - macOSにはGraalVMを入れず，Dockerコンテナ上でコンパイルするため`-Dquarkus.native.container-build=true`オプションを付与
  - 3分半くらいで完了 (Micornautのときは10分以上かかった…)
```
$ mvn package -Pnative -Dquarkus.native.container-build=true
```

### ローカル実行でテスト
- 上記ビルドで，`targets/sam.native.yaml`にSAM用のテンプレートが生成されている
  - これは`sam local invoke --template target/sam.native.yaml --event payload.json`でローカル実行可能
- JVM利用の場合と同様，API Gateway不要なので下記のsam templateを新たに作成
```yml:template.native.yml
AWSTemplateFormatVersion: "2010-09-09"
Transform: "AWS::Serverless-2016-10-31"
Description: Quarkus Lambda function.
Resources:
  function:
    Type: AWS::Serverless::Function
    Properties:
      Handler: not.used.in.provided.runtime
      Runtime: provided
      CodeUri: target/function.zip
      Description: Java function
      MemorySize: 128
      Timeout: 15
      # Function's execution role
      Policies:
        - AWSLambdaBasicExecutionRole
        - AWSLambda_ReadOnlyAccess
        - AWSXrayWriteOnlyAccess
        - AWSLambdaVPCAccessExecutionRole
      Tracing: Active
```

- 上記テンプレートとテスト用データ`payload.json`を用い，Lambda関数をローカル実行
  - 要Docker
  - 最後の部分のレスポンスが想定どおりであることを確認
```bash
$ sam local invoke --template template.native.yml --event payload.json
Invoking not.used.in.provided.runtime (provided)
Decompressing /Users/ryo/work/java/java-sandbox/quarkus-lambda/target/function.zip
Skip pulling image and use local one: amazon/aws-sam-cli-emulation-image-provided:rapid-1.23.0.

Mounting /private/var/folders/_7/m8bf5d2n6cj5xg9ptdvsry5m0000gn/T/tmp64k4ia2x as /var/task:ro,delegated inside runtime container
START RequestId: 3154344c-5cb3-4dc6-abcf-da47457853c8 Version: $LATEST
__  ____  __  _____   ___  __ ____  ______ 
 --/ __ \/ / / / _ | / _ \/ //_/ / / / __/ 
 -/ /_/ / /_/ / __ |/ , _/ ,< / /_/ /\ \   
--\___\_\____/_/ |_/_/|_/_/|_|\____/___/   
2021-05-13 11:33:59,841 INFO  [io.quarkus] (main) quarkus-lambda 1.0-SNAPSHOT native (powered by Quarkus 1.13.3.Final) started in 0.135s. 
2021-05-13 11:33:59,846 INFO  [io.quarkus] (main) Profile prod activated. 
2021-05-13 11:33:59,846 INFO  [io.quarkus] (main) Installed features: [amazon-lambda, cdi]
END RequestId: 3154344c-5cb3-4dc6-abcf-da47457853c8
REPORT RequestId: 3154344c-5cb3-4dc6-abcf-da47457853c8  Init Duration: 1.23 ms  Duration: 312.86 ms     Billed Duration: 400 ms Memory Size: 128 MB      Max Memory Used: 128 MB
{"result":"hello Bill","requestId":"3154344c-5cb3-4dc6-abcf-da47457853c8"}
```

### デプロイしてテスト
- `sam deploy`する (事前にzipアップロード用のS3バケットを用意)
```bash
$ sam deploy --template template.native.yml --stack-name quarkus-lambda-test --capabilities CAPABILITY_IAM --s3-bucket <your-bucket-name>
```

- デプロイ成功したら，AWS CLIから実行してみる
```bash
$ aws lambda invoke --function-name <your-function-name> --payload '{"name": "jagijagijag1", "greeting": "Hey!"}' response.json --cli-binary-format raw-in-base64-out
```

- 実行が成功すると，`response.json`ファイルが出力される
```json:response.json
{"result":"Hey! jagijagijag1","requestId":"12fd7a80-6cc6-42c9-bd2a-e85d935130df"}
```

- AWSマネジメントコンソールなどからトレースを確認可能
  - ネイティブイメージにより，コールドスタートの初回起動が約0.2秒に短縮された

![c25d6394.png](/blog/posts/2021_05_15/c25d6394.png)