---
title: サーバーレスで定期的にDropbox内のファイルをS3に転送する
author: jagijagijag1
date: 2019-03-02T23:33:17+0900
tags:
- serverless
- AWS
- Go
- Dropbox
categories:
- dev note

status: public
created_at: 2019-03-02 23:33:17 +0900
updated_at: 2019-03-02 23:33:17 +0900
published_at: 2019-03-02 23:33:17 +0900
---
### つくったもの
- 定期実行でDropboxの特定フォルダ下ファイルをS3へ転送する機能
- 具体的には，CloudWatch Eventで定期起動し下記を実行するLambda関数を開発
  1. Dropbox APIを叩いて特定フォルダ配下の全ファイルを取得
  2. 当該ファイルをダウンロードし，S3の特定バケットに保存
  3. バケット保存が成功した場合，Dropbox側のオリジナルファイルを削除

##### 経緯
- 以前S3に入ってきた画像を解析してPixelaに記録するLamba関数を開発
  - [Garmin connectのストレス測定結果をPixela + Serverlessで草化 - qrunch.net](https://qrunch.net/@jagijagijag1/entries/jRCEVYbJi8MP7v5p)
- この際，iPhoneのスクリーンショットをS3へ送る手段を下記のように構築

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">IFTTTでiOSのスクリーンショットをDropboxに入れ，ZapierでDropboxからS3にコピーするようにして前に作ったストレス草化Lambdaとシームレスに連携するようにした．ちょっと経由しすぎな気もするけどひとまず…</p>&mdash; JagiJagiJagi (@jagijagijag1) <a href="https://twitter.com/jagijagijag1/status/1060529287869026304?ref_src=twsrc%5Etfw">November 8, 2018</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

- しかしながら，これだけのためにZapierに課金するのも辛いので，イベント駆動を妥協して*定期実行でDropbox→S3転送機能*を実現してみた
- これにより，Zapierを経由せず，iPhoneからPixelaをシームレスに結合

![undefined.jpg](/blog/posts/2019_03_02/fbc66a44792fe625bd00748579b2eaec.jpg)

### 環境
- MacOS Mojave
- Go 1.11.5
- Serverless Framework 1.38.0

### ライブラリ選択
- DropboxのGoライブラリは以下の2つを発見
  - [GitHub - dropbox/dropbox-sdk-go-unofficial: An UNOFFICIAL Dropbox v2 API SDK for Go](https://github.com/dropbox/dropbox-sdk-go-unofficial)
  - [GitHub - tj/go-dropbox: Dropbox v2 client for Go.](https://github.com/tj/go-dropbox)
- 前者の方が，非公式といいつつ一応Dropboxプロジェクト配下にあったのでこちらを選択
`go get github.com/dropbox/dropbox-sdk-go-unofficial/dropbox`

### 前準備
- 新規Dropboxアプリを登録
  - https://www.dropbox.com/developers/apps
- 作成後，SettingsタブのOAuth 2 - Generated access tokenで"Generate"すると，(テスト向け)トークンを取得可能

### 開発詳細
#### 先につまづきメモ
- Dropbox APIでファイル一覧を取得した際の返り値が`[]IsMetadata`で，ファイル(`FileMetadata`)として操作できず
  - Goの型アサーションでFileMetadataとして処理が可能に
  - 詳細は[こちら](https://qrunch.net/@jagijagijag1/logs/V2nDaeoQNzx7sQzx)
- Dropboxから取得したファイルコンテンツをS3にわたすため，下記変換手順を踏む必要あり
  - 正直なところGoのio周りやDropboxの`Dwonload`/AWSの`ReadSeekCloser`の仕様を理解していないので，とりあえず[]byteらしきところを引っこ抜いて渡したらうまくいった状態

```go
// download from dropbox
dlArg := files.NewDownloadArg(fpath)
res, content, err := dbx.Download(dlArg)

// extract file content as []byte
blob, _ := ioutil.ReadAll(content)
// []byte -> Reader
f := bytes.NewReader(blob)
// Reader -> aws.ReadSeekCloser
rs := aws.ReadSeekCloser(f)
// S3 PutObject requires aws.ReadSeekCloser as its input body
input := &s3.PutObjectInput{
	Body:   rs,
	Bucket: aws.String(bucketName),
	Key:    aws.String(fileName),
}
```

#### 開発の初期設定
- 作ったものは[GitHub](https://github.com/jagijagijag1/dropbox2s3)
- $GOHOME/src配下の任意フォルダで作業
  
```bash
$ sls create -t aws-go-dep -p <project-name>
```

- 生成された`hello`と`world`の2つの関数は不要なので削除
- `dropbox2s3`ディレクトリと`main.go`を作成
- `Makefile`修正

```make:Makefile
.PHONY: build clean deploy

build:
	dep ensure -v
	env GOOS=linux go build -ldflags="-s -w" -o bin/dropbox2s3 dropbox2s3/main.go

clean:
	rm -rf ./bin ./vendor Gopkg.lock

deploy: clean build
	sls deploy --verbose

```

- `serverless.yml`修正

```yaml:serverless.yml
service: dropbox2s3-periodic

frameworkVersion: ">=1.28.0 <2.0.0"

provider:
  name: aws
  runtime: go1.x
  region: ap-northeast-1

  iamRoleStatements:
    - Effect: "Allow"
      Action:
        - "s3:ListBucket"
      Resource: "*"
    - Effect: "Allow"
      Action:
        - "s3:PutObject"
      Resource: "*"

package:
 exclude:
   - ./**
 include:
   - ./bin/**

functions:
  dropbox2s3:
    handler: bin/dropbox2s3
    events:
      - schedule: cron(0 16 * * ? *)
    environment:
      TZ: Asia/Tokyo
      DROPBOX_TOKEN: <your-api-token>
      IMG_FOLDER_PATH: <target-project-id> 
      BUCKET_NAME: <your-bucket>
    timeout: 60
```

#### main関数修正
- 冒頭の処理をナイーブに実装

```go:dropbox2s3/main.go
package main

import (
	"bytes"
	"context"
	"fmt"
	"io"
	"io/ioutil"
	"os"

	"github.com/aws/aws-sdk-go/aws/session"

	"github.com/aws/aws-sdk-go/aws"
	"github.com/aws/aws-sdk-go/service/s3"

	"github.com/aws/aws-lambda-go/lambda"
	"github.com/dropbox/dropbox-sdk-go-unofficial/dropbox"
	"github.com/dropbox/dropbox-sdk-go-unofficial/dropbox/files"
)

// Handler is our lambda handler invoked by the `lambda.Start` function call
func Handler(ctx context.Context) error {
	// extract env var
	dropboxToken := os.Getenv("DROPBOX_TOKEN")
	imgFolderPath := os.Getenv("IMG_FOLDER_PATH")
	bucketName := os.Getenv("BUCKET_NAME")

	// tansport image from dropbox to s3
	transport(dropboxToken, imgFolderPath, bucketName)

	return nil
}

func transport(dropboxToken, imgFolderPath, bucketName string) {
	// dropbox setting
	config := dropbox.Config{
		Token: dropboxToken,
	}
	dbx := files.New(config)

	// get info under the imgFolderPath folder
	arg := files.NewListFolderArg(imgFolderPath)
	resp, err := dbx.ListFolder(arg)
	if err != nil {
		fmt.Println("err in accesing dropbox folder")
		fmt.Println(err)
		return
	}

	// for each file/folder
	for _, e := range resp.Entries {
		// use type annotation to cast e (IsMetadata) to file (FileMetadata)
		f, ok := e.(*files.FileMetadata)
		if ok {
			// if e is file, download content
			fmt.Println("find file ", f.Name)
			fpath := f.Metadata.PathLower
			dlArg := files.NewDownloadArg(fpath)
			res, content, err := dbx.Download(dlArg)
			if err != nil {
				fmt.Println("err in downloading file")
				fmt.Println(err)
				return
			}

			// and then put it to S3
			err = putToS3(bucketName, res.Metadata.Name, content)
			if err == nil {
				// if successed, remove transported file on dropbox
				deleteFromDropbox(dbx, fpath)
			}
		}
	}
}

func putToS3(bucketName, fileName string, content io.ReadCloser) error {
	// create s3 client
	sess := session.Must(session.NewSession(&aws.Config{
		Region: aws.String("ap-northeast-1"),
	}))
	svc := s3.New(sess)

	// create put-object input
	blob, _ := ioutil.ReadAll(content)
	f := bytes.NewReader(blob)
	rs := aws.ReadSeekCloser(f)
	input := &s3.PutObjectInput{
		Body:   rs,
		Bucket: aws.String(bucketName),
		Key:    aws.String(fileName),
	}

	// put contet to s3
	_, err := svc.PutObject(input)
	if err != nil {
		fmt.Println("err in putting object")
		fmt.Println(err)
		return err
	}

	fmt.Println("put object ", fileName, " to ", bucketName)
	return nil
}

func deleteFromDropbox(dbx files.Client, filepath string) {
	delArg := files.NewDeleteArg(filepath)
	_, err := dbx.Delete(delArg)
	if err != nil {
		fmt.Println("err in downloading file")
		fmt.Println(err)
		return
	}
}

func main() {
	lambda.Start(Handler)
}

```

#### デプロイ
- 以下でデプロイ
  - `make deploy` = `make` + `sls deploy`

```bash
$ cd <project-name>
$ make deploy
```
