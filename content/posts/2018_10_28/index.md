---
title: Garmin connectのストレス測定結果をPixela + Serverlessで草化
author: jagijagijag1
date: 2018-10-28T11:11:05+0900
tags:
- Go
- serverless
- pixela
categories:
- dev note

status: public
created_at: 2018-10-28 08:50:15 +0900
updated_at: 2018-11-17 10:01:41 +0900
published_at: 2018-10-28 11:11:05 +0900
---
- Garmin connectでは心拍数の計測をもとにストレスを数値化してくれる

  ![undefined.jpg](/blog/posts/2018_10_28/ee0acdd1a59ee2423c874899e5524cf8.jpg)

- アプリ内では，一覧でみたいときに折れ線グラフしかない + 最大4週間分しか見れない
- 別の可視化方法として草化してみたい

## 作ったもの
- Garmin connectの画面キャプチャをS3にアップロードすると，日付とストレス値をPixelaに記録するシステム

  ![undefined.jpg](/blog/posts/2018_10_28/8e5bbd45cc6bf79bc3322377f5728677.jpg)

**2018/11/17 追記：**
下記を用いればiOSでスクリーンショットを取るだけでPixelaに記録できます．
(iOS→Dropbox by IFTTT + Dropbox→S3 by Zapier）
- [Add your latest iPhone screenshots to a Dropbox folder](https://ifttt.com/applets/103373p-add-your-latest-iphone-screenshots-to-a-dropbox-folder)
- [Amazon S3 + Dropbox Integrations](https://zapier.com/apps/amazon-s3/integrations/dropbox)

### 結果：いい感じに草化できた気がする
  - 直近のストレスが高い，日曜は比較的ストレスが少ない
  - なるべく色がつかない(薄くなる)ようにしたいという逆モチベ

    ![undefined.jpg](/blog/posts/2018_10_28/0bf13388af3dd25f8dfad07407b62bc5.jpg)

### 環境
- MacOS Mojave
- Go 1.11.1
- Serverless Framework 1.32.0
- iPhone 7(iOS 12.01) + Garmin Connect 4.12.0.14

## Pixelaへのデータ投入方法の検討
- iOS ヘルスケア
  -   Garmin connectのアプリから連携されない
- Garminの公式API (2種類)
  - [Garmin Health API](https://developer.garmin.com/health-api/overview/)
    -   全データ取得可能かつ無料だが，企業向けのため利用不可
  - [Garmin Connect API](https://developer.garmin.com/garmin-connect-api/overview/)
    -   個人利用可だが，フィットネスデータのみが対象かつ有料($5,000)
- アプリ画面から抽出
  -   アプリ画面を都度キャプチャする必要あり
  -   APIがなくても，直に情報を抽出可能
    - Amazon Rekognitionで試した感じ，行けそう

      ![undefined.jpg](/blog/posts/2018_10_28/eb20ac303dcfddabf2775703d0399324.jpg)

## 開発詳細
- 作ったものは[jagijagijag1/GarminStress2Pixela](https://github.com/jagijagijag1/GarminStress2Pixela)で公開

### Serverless framework + Goで開始
- $GOHOME/src配下で作業

```bash
$ serverless create -t aws-go-dep -p <project-name>
```

- 東京リージョンにデプロイしたいので`serverless.yml`に`region`を追記

```yaml:serverless.yml
provider:
  name: aws
  runtime: go1.x
  region: ap-northeast-1
```

- 以下でひとまずデプロイテスト可能

```bash
$ cd <project-name>
$ make
$ sls deploy
```

### 新規関数を作成
- 関数を新規作成
  - 自動生成された関数は不要なので削除
  - `garmin-stress2pixela`フォルダを作成し，`main.go`を作成 
  - `Makefile`の`build:`に以下を追記

```make:Makefile
	env GOOS=linux go build -ldflags="-s -w" -o bin/garmin-stress2pixela garmin-stress2pixela/main.go
```
  
### `serverless.yml`の修正
- `serverless.yml`の主な修正・追記点は以下
  - IAM RoleにRekognitionのDetectText実行許可と，画像を投入するS3バケットへのアクセス許可を追記
  - 新規作成した関数定義の追記 （+自動生成された関数定義の削除)
    - `events`下の`s3: <bucket-name>`は存在しないバケット名とすること (`sls deploy`で新規作成されるため)
    - Lambda関数の環境変数(`environment`)にPixelaのユーザ/トークン/グラフ情報を与える
    
```yaml:serverless.yml
service: GarminStress2Pixela

frameworkVersion: ">=1.28.0 <2.0.0"

provider:
  name: aws
  runtime: go1.x
  region: ap-northeast-1
  # you can add statements to the Lambda function's IAM Role here
  iamRoleStatements:
    - Effect: "Allow"
      Action:
        - "rekognition:DetectText"
      Resource: "*"
    - Effect: "Allow"
      Action:
        - "s3:GetObject"
      Resource: [
        "arn:aws:s3:::pixela-datasource-stress-img-bucket",
        "arn:aws:s3:::pixela-datasource-stress-img-bucket/*"
      ]

package:
 exclude:
   - ./**
 include:
   - ./bin/**

functions:
  garmin-stress2pixela:
    handler: bin/garmin-stress2pixela
    events:
      - s3: <bucket-name>
    # you need to fill the followings with your own
    environment:
      PIXELA_USER: <user-id>
      PIXELA_TOKEN: <your-token>
      PIXELA_GRAPH: <your-graph-id-1>
    timeout: 10
```

### 関数本体を作成
- 作っている最中の気づき，ポイントは以下
  - GoでのAWSイベントは以下にサンプルがあり，これを参照しHandlerの引数，入力情報処理を実装
    - [aws-lambda-go/events at master  aws/aws-lambda-go  GitHub](https://github.com/aws/aws-lambda-go/tree/master/events)
  - InvalidS3ObjectException に当たった
    - `InvalidS3ObjectException: Unable to get object metadata from S3. Check object key, region and/or access permissions.`
    - S3 Objectへのアクセス権限をLambad関数にも付与すること (上記yamlにて済)
    - S3バケットのリージョンと，Rekognitionのリージョンを同じにすること
      - Rekognitionは同一リージョンのバケット内オブジェクトにしかアクセスできない模様
  - **[作り込み・汎用性低]** 日付・ストレス値は，事前に画像内での想定位置を与え，Rekognition.DetectTextの結果のうち，想定位置の最も近傍のテキストを選択
    - 想定位置(`assumedDatePoint, assumedQuantityPoint`)はiPhone 7(iOS 12.01)，Garmin Connect 4.12.0.14にて実験的に抽出
  - データ投入先のPixelaの情報は環境変数(`PIXELA_USER, PIXELA_TOKEN, PIXELA_GRAPH`)から取得
  - GoでのPixela操作には[gainings/pixela-go-client](https://github.com/gainings/pixela-go-client)を利用
  
```go:garmin-stress2pixela/main.go
package main

import (
	"context"
	"fmt"
	"math"
	"os"
	"regexp"
	"strings"

	"github.com/aws/aws-lambda-go/events"
	"github.com/aws/aws-lambda-go/lambda"
	"github.com/aws/aws-sdk-go/aws"
	"github.com/aws/aws-sdk-go/aws/session"
	"github.com/aws/aws-sdk-go/service/rekognition"
	pixela "github.com/gainings/pixela-go-client"
)

// Point is left & top positions of bounding box in the Rekognition result
type Point struct {
	Left float64
	Top  float64
}

// !! fixed number from experiment (maybe require to change your env) !!
var assumedDatePoint = Point{Left: 0.393, Top: 0.111}
var assumedQuantityPoint = Point{Left: 0.268, Top: 0.282}

// Handler is our lambda handler invoked by the `lambda.Start` function call
func Handler(ctx context.Context, s3Event events.S3Event) error {
	// for each s3 object
	for _, record := range s3Event.Records {
		// extract s3 object info
		bucket, key := getS3ObjectFromRecord(record)
		fmt.Printf("[%s] Bucket = %s, Key = %s \n", record.EventSource, bucket, key)

		// execute text detection of Rekognition
		res, rekerr := exeRekognitionDetectText(bucket, key)
		if rekerr != nil {
			fmt.Println("Error")
			fmt.Println(rekerr.Error())
		}

		// extract date & quantity from the above result
		date, quantity := getValueFromRekognitionResult(res.TextDetections)
		fmt.Printf("data: %s, quantity: %s\n", date, quantity)

		// record pixel
		perr := recordPixel(date, quantity)
		fmt.Println(perr)
	}

	return nil
}

func getS3ObjectFromRecord(record events.S3EventRecord) (string, string) {
	s := record.S3
	bucket := s.Bucket.Name
	rep := regexp.MustCompile(`[+]`)
	key := rep.ReplaceAllString(s.Object.Key, " ")

	return bucket, key
}

func exeRekognitionDetectText(bucket, key string) (*rekognition.DetectTextOutput, error) {
	// create Rekognition client
	sess := session.Must(session.NewSession())
	rc := rekognition.New(sess, aws.NewConfig().WithRegion("ap-northeast-1"))

	// set params
	params := &rekognition.DetectTextInput{
		Image: &rekognition.Image{
			S3Object: &rekognition.S3Object{
				Bucket: aws.String(bucket),
				Name:   aws.String(key),
			},
		},
	}
	fmt.Printf("params: %s", params)

	// execute DetectText
	return rc.DetectText(params)
}

func getValueFromRekognitionResult(results []*rekognition.TextDetection) (string, string) {
	dateHypot, quantityHypot := math.MaxFloat64, math.MaxFloat64
	date, quantity := "", ""

	// for each detected text
	for _, td := range results {
		left, top := *td.Geometry.BoundingBox.Left, *td.Geometry.BoundingBox.Top

		// calc hypot with assumed date pos & update value
		tmpDHypot := math.Hypot(math.Abs(left-assumedDatePoint.Left), math.Abs(top-assumedDatePoint.Top))
		if tmpDHypot < dateHypot {
			// if td is most-likely-result (nearest to the assumed point), keep the result (with removing "/")
			dateHypot, date = tmpDHypot, strings.Replace(*td.DetectedText, "/", "", -1)
		}

		// calc hypot with assumed quantity pos & update value
		tmpQHypot := math.Hypot(math.Abs(left-assumedQuantityPoint.Left), math.Abs(top-assumedQuantityPoint.Top))
		if tmpQHypot < quantityHypot {
			// if td is most-likely-result (nearest to the assumed point), keep the result
			quantityHypot, quantity = tmpQHypot, *td.DetectedText
		}
	}

	return date, quantity
}

func recordPixel(date, quantity string) error {
	user := os.Getenv("PIXELA_USER")
	token := os.Getenv("PIXELA_TOKEN")
	graph := os.Getenv("PIXELA_GRAPH")
	c := pixela.NewClient(user, token)

	// try to record
	err := c.RegisterPixel(graph, date, quantity)
	if err == nil {
		fmt.Println("recorded")
		return err
	}

	// if fail, try to update
	err = c.UpdatePixelQuantity(graph, date, quantity)
	if err == nil {
		fmt.Println("updated")
	}

	return err
}

func main() {
	lambda.Start(Handler)
}
```
