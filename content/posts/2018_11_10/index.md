---
title: Togglの記録をServerless + Pixelaで草化
author: jagijagijag1
date: 2018-11-10T10:52:38+0900
tags:
- Go
- serverless
- pixela
categories:
- dev note

status: public
created_at: 2018-11-10 10:52:38 +0900
updated_at: 2018-11-10 10:54:17 +0900
published_at: 2018-11-10 10:52:38 +0900
---
- 作業時間などの時間管理ツールとして[Toggl](https://toggl.com/)がある
  - いつ，どの作業をしたかを記録
  - 各作業をプロジェクトやタグで分類可能
  - [Toggl Reports](https://toggl.com/time-reports/)で可視化も提供されており，特定の作業をどれくらい継続しているか，どのくらい時間をかけているかを見れる
  - **でもとりあえず草化したい！**
- Toggleは[API](https://github.com/toggl/toggl_api_docs/blob/master/toggl_api.md)を提供しているので比較的用意にデータ抽出可能
  - [画像認識](https://qiita.com/jagijagijag1/items/b0ddae1b93922bdd8587)とかいらない！

## 作ったもの
- 1日1回，前日に特定プロジェクトにかけた時間をTogglから抽出し，Pixelaに記録

![undefined.jpg](/blog/posts/2018_11_10/c45faff3cdd755774dd692a645d514be.jpg)

### 結果
- 自分の勉強時間を草化できた

![undefined.jpg](/blog/posts/2018_11_10/2fed5453892fbe25102a4afe8597e666.jpg)

### 環境
- MacOS Mojave
- Go 1.11.1
- Serverless Framework 1.32.0

## つまづきメモ
- しょぼい内容だが備忘録として

### Lambdaにて時間を扱う場合の注意
- CloudWatch Eventsをcron式で時間指定する場合，UTCで指定すること
  - e.g. JSTで毎日午前1時に実行したい→UTCで午後4時(-9時間)を指定する `cron( 0 16 * * ? * )`
- Lambda関数で日時を取得する場合(e.g. Goでの`time.Now()`)，標準ではUTCで取得する
- 日本時間を使いたい場合はLambda関数の環境変数でタイムゾーンを指定すること
  - e.g. 変数`TZ`, 値`Asia/Tokyo`

### Toggl APIの使い方
- TogglのAPIを利用したい場合，リクエストにAPIトークンを含める
  - APIトークンはProfileから取得可能: https://support.toggl.com/my-profile/
- 今回は特定期間の記録を全取得し，特定プロジェクトの記録のみ加算していき合計時間を取得
- 特定期間の記録を取得するAPIは以下
  - `GET https://www.toggl.com/api/v8/time_entries?start_date=XXX&end_date=XXX`
  - 日時はISO 8601形式
- 今回はGoのwrapperである[dougEfresh/gtoggl](https://github.com/dougEfresh/gtoggl)を利用
  - READMEの記載内容だとうまく行かず

```go
import "github.com/dougEfresh/gtoggl"
import "github.com/dougEfresh/gtoggl-api/gtproject"

func main() {
  // HTTP client作成
  thc, err := gthttp.NewClient("your-api-token")
  ...
  // Togglの記録(time entry)取得用クライアント作成
  tec := gttimeentry.NewClient(thc)
  // 特定期間の記録を取得
	entries, eerr := tec.GetRange(start_date, end_date)
}
```

## 開発詳細
- 作ったものは[GitHub - jagijagijag1/toggl2pixela](https://github.com/jagijagijag1/toggl2pixela)で公開

### Serverless framework + Goで開始
- `$GOHOME/src`配下で作業

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
  - `toggl2pixela`フォルダを作成し，`main.go`を作成 
  - `Makefile`の`build:`に以下を追記
  
```make:Makefile
	env GOOS=linux go build -ldflags="-s -w" -o bin/toggl2pixela toggl2pixela/main.go
```

### `serverless.yml`の修正
- `serverless.yml`の主な修正・追記点は以下
  - 新規作成した関数定義の追記 （+自動生成された関数定義の削除)
  - `events`下に`schecule: ***`を書くことで定期実行を定義 (下記では毎日午前1時に実行，上述の通りcron式の時間はUTC指定なので注意)
  - Lambda関数でJSTで日時取得したいので，環境変数`TZ`, 値`Asia/Tokyo`を指定
  - Lambda関数の環境変数(`environment`)にTogglのAPIキー/対象プロジェクトID，Pixelaのユーザ/トークン/グラフ情報を与える

```yaml:serverless.yml
service: toggl2pixela

frameworkVersion: ">=1.28.0 <2.0.0"

provider:
  name: aws
  runtime: go1.x
  region: ap-northeast-1

package:
 exclude:
   - ./**
 include:
   - ./bin/**

functions:
  toggl2pixela:
    handler: bin/toggl2pixela
    events:
      - schedule: cron(0 16 * * ? *)
    # you need to fill the followings with your own
    environment:
      TZ: Asia/Tokyo
      TOGGL_API_TOKEN: <your-api-token>
      TOGGL_PROJECT_ID: <target-project-id> 
      PIXELA_USER: <user-id>
      PIXELA_TOKEN: <your-token>
      PIXELA_GRAPH: <your-graph-id-1>
    timeout: 10
```


### 関数本体を作成
- 素直に実装しただけなので，特記事項なし…
  - データ元のToggl，データ投入先のPixelaの情報は環境変数(`TOGGL_API_TOKEN, TOGGL_PROJECT_ID, PIXELA_USER, PIXELA_TOKEN, PIXELA_GRAPH`)から取得
  - GoでのToggl操作には[dougEfresh/gtoggl](https://github.com/dougEfresh/gtoggl)を利用
    - 利用方法は上述
  - GoでのPixela操作には[gainings/pixela-go-client](https://github.com/gainings/pixela-go-client)を利用

```go:toggl2pixela/main.go
package main

import (
	"context"
	"errors"
	"fmt"
	"os"
	"strconv"
	"time"

	"github.com/aws/aws-lambda-go/lambda"
	"github.com/dougEfresh/gtoggl-api/gthttp"
	"github.com/dougEfresh/gtoggl-api/gttimentry"
	pixela "github.com/gainings/pixela-go-client"
)

// Handler is our lambda handler invoked by the `lambda.Start` function call
func Handler(ctx context.Context) error {
	// extract env var
	apiToken := os.Getenv("TOGGL_API_TOKEN")
	pjID, _ := strconv.ParseUint(os.Getenv("TOGGL_PROJECT_ID"), 10, 64)
	user := os.Getenv("PIXELA_USER")
	token := os.Getenv("PIXELA_TOKEN")
	graph := os.Getenv("PIXELA_GRAPH")

	// extract data from toggl
	date, quantity := getDateAndTimeFromToggl(apiToken, pjID)
	if date == "-1" || quantity == "-1" {
		return errors.New("Error in accessing toggl")
	}
	fmt.Printf("date: %s, quantity: %s\n", date, quantity)

	// record pixel
	perr := recordPixel(user, token, graph, date, quantity)
	if perr != nil {
		return errors.New("Error in accessing pixela")
	}

	return nil
}

func getDateAndTimeFromToggl(apiToken string, pjID uint64) (string, string) {
	// create toggl client
	thc, err := gthttp.NewClient(apiToken)
	if err != nil {
		fmt.Println(err)
		return "-1", "-1"
	}

	// set time range to be analyzed
	y := time.Now().AddDate(0, 0, -1)
	s := time.Date(y.Year(), y.Month(), y.Day(), 0, 0, 0, 0, time.Local)
	e := time.Date(y.Year(), y.Month(), y.Day(), 23, 59, 59, 0, time.Local)
	date := y.Format("20060102")

	// get time entries
	total := int64(0)
	tec := gttimeentry.NewClient(thc)
	entries, eerr := tec.GetRange(s, e)
	if eerr != nil {
		fmt.Println(eerr)
		return "-1", "-1"
	}

	// sum durations with project pjID
	for _, e := range entries {
		if e.Pid == pjID {
			total += e.Duration
		}
	}
	totalMin := float64(total) / 60
	quantity := strconv.FormatFloat(totalMin, 'f', 4, 64)

	return date, quantity
}

func recordPixel(user, token, graph, date, quantity string) error {
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
````
