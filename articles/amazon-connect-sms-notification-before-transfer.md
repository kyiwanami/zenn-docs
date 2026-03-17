---
title: "2024-12-05 Amazon Connect 携帯電話への転送前に発信者をSMSで通知する方法"
emoji: "📞"
type: "tech"
topics:
  - aws
  - amazonconnect
  - lambda
  - sns
  - dynamodb
published: false
publication_name: "trust"
published_at: 2024-12-05
---

## はじめに

こんにちは。株式会社トラストの岩波です。

今回は、Amazon Connectで携帯電話などに転送する際、発信者情報をSMSで確認してから応答できる方法をお話しします。

## Amazon Connectとは

AWS上でコンタクトセンターを構築できるサービスです。

例えば、着信時の自動応答、担当者への通話転送、チャット機能などを備えています。

ルーティング先には下記の2つがあります。

- デスクフォン：固定電話や携帯電話
- ソフトフォン：PCで通話できる仕組み

詳細は下記サイトが詳しいです。

[Amazon Connectとは？基本機能8つとメリットを総解説 コンタクトセンター(コールセンター)の在宅需要が高まっている今、システムのクラウド化への関心が高まっています。そのような中](https://www.transcosmos-cotra.jp/amazon-connect/about/)

弊社では、電話対応の取次ぎを減らし業務の効率化を図るプロジェクトを進めています。

今回は、取次先が決まっている場合は、担当者の携帯電話（デスクフォン）へ自動転送する仕組みを導入しようとしています。

ただ、Amazon Connectの仕様として、転送先の携帯電話には発信者の番号を表示できず、Amazon Connectで取得した番号が表示される（※）ため、発信者を判別できず対応が難しくなっています。

そこで、通話転送前にSMSで発信者情報を担当者に通知する仕組みを考えました。

※仕様の詳細は下記サイトが詳しいです。

[[Amazon Connect]「電話番号への転送ブロック」にある「発信者ID番号」と「発信者ID名」について | DevelopersIO](https://dev.classmethod.jp/articles/amazon-connect-transfer-block-caller-id-number/)

## システム構成

![](https://assets.st-note.com/img/1759400057-DsA4WSYKi5phdoUVwymLlk6c.png)

簡単な役割について説明します。

- Amazon DynamoDB
発着信情報を管理しています。

- AWS Lambda
着信情報を基に発着信情報のDynamoDBを検索します。
見つかったら、発信者の名前を載せてAmazon SNSを呼び出し、転送先の電話に通知します。
そして、Amazon Connectに担当者の電話番号を返します。

- Amazon Connect
着信した電話番号などの情報をLambda関数に引き渡します。
Lambda関数が担当者の電話番号を返したら、通話を転送します。

それぞれの詳解を見ていきます。

### Amazon DynamoDB

電話番号をPKに、顧客名と担当者の転送先電話番号を持ちます。

- phoneNumber：PK（電話番号）：`+819012345678`
- customerName（顧客名）：`テスト食品`
- routingPhoneNumber（担当者電話番号）：`+819011112222`

### AWS Lambda

```python
import datetime
import decimal
import boto3

# クライアントの作成
dynamodb = boto3.resource('dynamodb')
client = boto3.client('sns')

# DynamoDBテーブル名を指定
table = dynamodb.Table("phoneInfoDemo")

# Lambdaのメイン関数
def lambda_handler(event, context):

    print(event)

    # 顧客の電話番号をイベントから取得
    #phoneNumber = event["Details"]["ContactData"]["CustomerEndpoint"]["Address"]
    # DynamoDBからデータを取得
    response = table.get_item(Key={'phoneNumber': phoneNumber })
    item = response.get('Item')

    # SNS通知用のメッセージとパラメータを設定
    params = {
        'TopicArn': 'arn:aws:sns:ap-northeast-1:xxx:AmazonConnectInovationLabNotification',
        'Message': item["customerName"] + "\n" + item["phoneNumber"]
    }

    # SNSメッセージを送信
    client.publish(**params)
    # レスポンスとして routingPhoneNumber を返して Amazon Connect に発信させる
    return {"routingPhoneNumber": item["routingPhoneNumber"]}
```

実行していることは以下の通りです。

1. DynamoDBを電話番号をキーに検索
2. 取得した顧客名とルーティング先電話番号をAmazon SNSで送信
3. 取得したルーティング先電話番号をAmazon Connectに戻す

Amazon Connectからは、電話番号を含む下記形式のJSONが送られてきます。

```json
{
  "Details": {
    "ContactData": {
      "Attributes": {
        "exampleAttributeKey1": "exampleAttributeValue1"
      },
      "Channel": "VOICE",
      "ContactId": "4a573372-1f28-4e26-b97b-XXXXXXXXXXX",
      "CustomerEndpoint": {
        "// 発信者の電話番号": "",
        "Address": "+1234567890",
        "Type": "TELEPHONE_NUMBER"
      },
      "CustomerId": "someCustomerId",
      "Description": "someDescription",
      "InitialContactId": "4a573372-1f28-4e26-b97b-XXXXXXXXXXX",
      "InitiationMethod": "INBOUND | OUTBOUND | TRANSFER | CALLBACK",
      "InstanceARN": "arn:aws:connect:aws-region:1234567890:instance/c8c0e68d-2200-4265-82c0-XXXXXXXXXX",
      "LanguageCode": "en-US",
      "MediaStreams": {
        "Customer": {
          "Audio": {
            "StreamARN": "arn:aws:kinesisvideo::eu-west-2:111111111111:stream/instance-alias-contact-ddddddd-bbbb-dddd-eeee-ffffffffffff/9999999999999",
            "StartTimestamp": "1571360125131",
            "StopTimestamp": "1571360126131",
            "StartFragmentNumber": "100"
          }
        }
      },
      "Name": "ContactFlowEvent",
      "PreviousContactId": "4a573372-1f28-4e26-b97b-XXXXXXXXXXX",
      "Queue": {
        "ARN": "arn:aws:connect:eu-west-2:111111111111:instance/cccccccc-bbbb-dddd-eeee-ffffffffffff/queue/aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee",
        "Name": "PasswordReset",
        "OutboundCallerId": {
          "Address": "+12345678903",
          "Type": "TELEPHONE_NUMBER"
        }
      },
      "References": {
        "key1": {
          "Type": "url",
          "Value": "urlvalue"
        }
      },
      "SystemEndpoint": {
        "Address": "+1234567890",
        "Type": "TELEPHONE_NUMBER"
      }
    },
    "Parameters": {
      "exampleParameterKey1": "exampleParameterValue1",
      "exampleParameterKey2": "exampleParameterValue2"
    }
  },
  "Name": "ContactFlowEvent"
}
```

### Amazon Connect

![](https://assets.st-note.com/img/1759400305-mLYJvOqV1wNeX0IRsfohSMx2.png?width=1200)

「AWS Lambda関数を呼び出す」ブロック

![](https://assets.st-note.com/img/1759400316-DKmtIoe3dnQRywO2isfCGVkx.png)

呼び出すLambda関数を指定して、発信者の電話番号を含む下記のイベントをLambdaに渡します。

プロンプトの再生より前にしているのは、SNSの送信に若干時間がかかり、着信より後に届いてしまうことがあったためです。

「電話番号への転送」ブロック

![](https://assets.st-note.com/img/1759400330-zhUGTNDXt2e6wFWvEiHlOmyp.png)

`routingPhoneNumber` はLambda戻り値のオブジェクトキーになります。

発信者IDに、事前に取得しておいた電話番号を設定します。

![](https://assets.st-note.com/img/1759400346-GMvRtO56jDh8BlcPF9LCzAys.png)

## 結果

[実際の画面（1）.mp4](https://note.com/api/v2/attachments/download/bfaa23b4a95108e24ce5870ff77d253e)

SNSが届き、3秒後くらいに通話がかかってきました。

## おわりに

Amazon Connectで着信を携帯電話に転送する際、着信したら発信者情報をSMSで確認しつつ応答できる方法について紹介しました。

今回は転送先が携帯電話であったため事前に通知する形になりましたが、
ソフトフォンも可能であれば受信画面に表示するなどもう少し柔軟にカスタマイズできると思いますので、いずれはそちらも試してみたいと思います。

今後もAmazon Connectに関する取り組みを行っていく予定ですので、次回の記事もお楽しみに。

当ブログで提供している情報・プログラムコード・設定例などについては、正確性や安全性を保つよう努めておりますが、その内容を保証するものではありません。
