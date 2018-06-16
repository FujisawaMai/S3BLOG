---
title: "AWSサービスの使用料を、コストと使用状況レポートとQuicksightを使って継続的に自動で可視化する"
date: 2018-03-19T21:16:58+09:00
thumbnailImage: /images/kousei.png
draft: false
tags: ["AWS","QuickSight","S3","Lambda"]
---
AWSサービスの使用料を、長期的に、カスタマイズされた形で確認するために、コストと使用状況レポートとQuicksightを使って継続的に自動的に可視化してみようと思います。
<!--more-->

# １．目的

AWSにはサービス使用料を確認する方法として、請求ダッシュボードから、概要や料金明細を確認でき、またコストエクスプローラーからはサービス別、AZ別などのフィルターをかけることもできます。その他に請求明細レポートなど各種レポートから使用状況を確認することもできます。

しかし、これらツールによる料金確認だと、ダッシュボードからだと直近1か月、コストエクスプローラーも過去１２か月分しか確認ができないということから長期的な料金確認はできず、分析のためのグラフのカスタマイズはできません。また、各種レポートの中には今後利用できなくなるものがあり、AWSからはコストと使用状況レポートを使用することが推奨されています。

なので、AWSサービスの使用料を、長期的に、カスタマイズされた形で確認するために、コストと使用状況レポートとQuicksightを使って継続的に自動的に可視化してみようと思います。これは、S3に継続的に吐き出されるデータを自動的に可視化するので、コストと使用状況レポートに限らず、様々なものに転用できます。

# ２．QuickSightとは

QuickSightはデータを視覚化するAWSのサービスです。QuickSightがサポートするデータソース（S3やAthena、RDSなど）からData setを作成し、これをもとに視覚化を行います。詳しくは公式ドキュメントを参照ください。Amazon公式YoutubeのQuickSight Overviewがさっくり概要をつかみやすくておすすめです。公開されたてほやほやなBlackBeltもあるのでこちらもぜひご参照ください！

[**20180228 AWS Black Belt Online Seminar QuickSight**](https://www.slideshare.net/AmazonWebServicesJapan/20180228-aws-black-belt-online-seminar-quicksight-89889593 "20180228 AWS Black Belt Online Seminar QuickSight")

[**Amazon公式 QuickSight Overview（Youtube）**](https://www.youtube.com/watch?v=C_eT0xRNjCs "Amazon公式 QuickSight Overview（Youtube）")
[![IMAGE ALT TEXT HERE](http://img.youtube.com/vi/C_eT0xRNjCs/0.jpg)](http://www.youtube.com/watch?v=C_eT0xRNjCs)

## 料金体系

従量課金サービスが多いAWSサービスでは珍しく、月額利用料が設定されています。料金形態として、標準的なBIサービスが提供されるStandard Editionと、BIサービスに加え、データの暗号化やADとの統合などが可能なEnterpriseEditionの2種類があります。

| |Standard Edition　　  　|Enterprise Edition　　　|
|---|---|---|
|最初のユーザー (USD/user/month)|無料（1GB SPICEを含む）　|無料（1GB SPICEを含む）　|
|追加ユーザー (USD/user/month)  |$9（10GB SPICEを含む）　 |$18（10GB SPICEを含む） |
|追加のSPICE (/GB/month)       |$0.25　                 |                 $0.38 |

上記は年間契約時の金額です。一か月ごとの利用も可能であり、その場合は以下の料金となります。

- Standard Edition $12/user/month
- Enterprise Edition $24/user/month

こちらはブログ投稿時点の料金になるので、最新の料金はこちらからご確認ください。
[**Amazon公式 What is QuickSight?**](https://aws.amazon.com/jp/quicksight/?nc1=h_ls "Amazon公式 What is QuickSight?")

# ３．コストと使用状況レポート

AWSサービスの使用状況についてかなり細かいレポートを出してくれるサービスで、請求ダッシュボード＞レポートから、S3にレポートを吐き出す設定ができます。項目は多岐にわたるので、今回は以下の、可視化に用いる項目（＋あったら便利かもしれない項目）のみを使います。

- lineItem/UsageStartDate　・・・そのサービスの使用開始時点 
- lineItem/ProductCode　・・・サービス名
- lineItem/BlendedCost　・・・使用料金（ドル）
- lineItem/UsageType　・・・この明細項目の使用状況の詳細
- lineItem/CurrencyCode　・・・USD
- lineItem/LineItemDescription　・・・この明細項目タイプの説明
- product/productFamily　・・・製品タイプのカテゴリ
- product/region　・・・そのプロダクトのリージョン（us-west-2, etc）
- resourceTags/user:Cost　・・・そのサービスにタグをつけておくと、billing reportにも表示される

[**Amazon公式Document AWS 請求情報とコスト管理　コストと使用状況レポート**](https://docs.aws.amazon.com/ja_jp/awsaccountbilling/latest/aboutv2/billing-reports-costusage.html "Amazon公式Document AWS 請求情報とコスト管理　コストと使用状況レポート")

# ４．全体像とポイント

全体像はこんな感じになります。
![kousei](/images/kousei.png)

## ４－１．Quicksightが参照するフォルダの作成

QuickSightが参照するデータ用として、Latestフォルダを作成します。このフォルダには、Manifestファイルと、最新データを結合したlatest.csv（②～③で作成する）を配置します。

### Manifestとは

QuickSightにS3からデータをインポートするためには、そのデータをどこからひっぱってきて、どのような形式でデータ化するかを示す、Manifestをアップロードする必要があります。

[**AWS QuickSight Amazon S3でマニフェストファイルでサポートされている形式**](https://docs.aws.amazon.com/ja_jp/quicksight/latest/user/supported-manifest-file-format.html "Amazon公式Document AWS QuickSight Amazon S3でマニフェストファイルでサポートされている形式")

今回私はS3にバケットの下のlatestフォルダの中の`marged_processed_latest.csv`を参照するManifestを作成しました。

```json:manifest.json
{
    "fileLocations": [
        {
            "URIs": [
                "https://s3-ap-northeast-1.amazonaws.com/BUCKET-NAME/latest/marged_processed_latest.csv"
            ]
        }

    ],
    "globalUploadSettings": {
        "format": "CSV",
        "delimiter": ",",
        "textqualifier": "'",
        "containsHeader": "true"
    }
}
```

## ４－２．データの収集

Dailyのコストと使用状況レポートを設定すると、月ごとのフォルダが作成され、その中に日々のレポートが追加されていきます。なので、過去のデータも含めた最新のデータを集めようと思うと、過去の、月ごとの、そのフォルダの中の最終データと、現在の月の最新データを収集する必要があります。
![dailybucket_detail](/images/dailybucket_detail.png)

## ４－３．データのマージとS3へのアップロード


Lambdaを使って過去の、月ごとの、そのフォルダの中の最終データと、現在の月の最新データを結合させます。

データの更新が自動で行われるように、S3へのPUTをトリガーにデータをマージし、S3のLatestフォルダにlatest.csvの名前でアップロードされるように設定、コーディングします。

サンプルコード全文は[Gist](https://gist.github.com/pistachiyoda/235ce579080e0a8b2e4b4f0dacc5e757)上にあげています。

**各月の最新データへのパスをkeys[]に格納する**

それぞれの月のどのファイルを最新ファイルとして読み込むのかを決定し、そのファイルへのパスを取得していきます。最新ファイルの選別方法として、ファイルがそれぞれの月のフォルダにアップロードされた日時でソートし、最新のものを取得するようにしています。

また、コストと使用状況レポートがS3にアップロードされる時には、レポートの本体と、レポートの項目などを情報を持ったファイル（manifest.json）が一緒にアップロードされるので、こちらが最新ファイルだと認識されないように、本体ファイルの拡張子（csv.gz）を持つファイルだけが最新ファイルとみなされるようにします。

```python:billingreport_merge.py
def lambda_handler(event, context):

    # 使用するバケット
    bucket = 'BUCKET_NAME'

    ########各月の最新データへのパスをKeys[]に格納する########

    def json_serial(obj):
        if isinstance(obj, datetime):
            return obj.isoformat()
        raise TypeError ("Type %s not serializable" % type(obj))

    # そのバケット内のフォルダごとの最新のオブジェクトを取得
    # Keysが格納されているPrefixを格納する配列
    Prefixes = []
    # 最新データへのパスを格納する配列
    Keys = []

    client = boto3.client('s3')

    result = client.list_objects_v2(Bucket=bucket, Prefix='/billingreport/', Delimiter='/')
    for o in result.get('CommonPrefixes'):
        prefix = o.get('Prefix')
        Prefixes.append(prefix)

    for prefix in Prefixes:
        response = client.list_objects(
            Bucket=bucket,
            Prefix=prefix
        )
        str_response = json.loads(json.dumps(response, default=json_serial))
        contents = str_response['Contents']
       
        #content部をLastModifiedで並び替え、最新のものを取得
        sorted_contents = sorted(contents,key=lambda x:x['LastModified'],reverse=True)
       
        #もしlatest_contentに格納されたのがcsv.gzだったら格納、ちがったら２周目
        for latest_content in sorted_contents:
            if 'csv.gz' in latest_content['Key']:
                Keys.append(latest_content['Key'])
                break
            else:
                continue
```

**Keys[]にパスが格納された各月のデータをトリムしてマージ**

前述したように、コストと使用状況レポートの項目は多岐にわたるので、必要な項目のみをfieldnamesに格納し、その項目だけがマージされるようにします。また、Lambda上では、/tmpの下に一時ファイルが作れるので、マージファイルはいったんここに作成し、最終的にS3にアップロードされるようにします。

```python:billingreport_merge.py
    client = boto3.client('s3')

    fieldnames = [
        'lineItem/UsageStartDate', 
        'lineItem/ProductCode', 
        'lineItem/BlendedCost', 
        'lineItem/UsageType',
        'lineItem/CurrencyCode',
        'lineItem/LineItemDescription',
        'product/productFamily',
        'product/region',
        'resourceTags/user:Cost'
    ]

    try:
        with open('/tmp/processed.csv', 'w') as f:
            writer = csv.DictWriter(f,fieldnames=fieldnames)
            writer.writeheader()
        for key in Keys:
            response = client.get_object(Bucket=bucket, Key=key)
            gz = gzip.GzipFile(fileobj=response['Body'])
            with open('/tmp/origin.csv', 'w') as f:
                f.write(gz.read().decode('utf-8'))

            with open('/tmp/origin.csv') as f:
                reader = csv.DictReader(f)
                with open('/tmp/processed.csv','a') as csvfile:
                    writer = csv.DictWriter(csvfile,fieldnames=fieldnames)

                    for row in reader:
                        writer.writerow(
                            {
                                'lineItem/UsageStartDate': row['lineItem/UsageStartDate'],
                                'lineItem/ProductCode': row['lineItem/ProductCode'],
                                'lineItem/BlendedCost': row['lineItem/BlendedCost'],
                                'lineItem/CurrencyCode': row['lineItem/CurrencyCode'],
                                'lineItem/UsageType': row['lineItem/UsageType'],
                                'lineItem/LineItemDescription': row['lineItem/LineItemDescription'],
                                'product/productFamily': row['product/productFamily'],
                                'product/region': row['product/region'],
                                'resourceTags/user:Cost': row['resourceTags/user:Cost'],
                            }
                        )

        with open('/tmp/processed.csv', 'r') as f:
            client.put_object(
                Bucket=bucket,
                Key='latest/marged_processed_latest.csv',
                Body=f.read()
            )

        return response['ContentType']

    except Exception as e:
        print(e)
        print('Error getting object {} from bucket {}. Make sure they exist and your bucket is in the same region as this function.'.format(key, bucket))
        raise e
```

## ４－４．QuickSightへのデータのインポート

QuickSightに、S3のデータソースを示すManifestとして、①で作成したManifestが保存されているS3上のURLを入力し、データをインポートします。

**Manifestのアップロード**

ここに4-1で作成したjsonのManifestファイルが保存されているS3のURLをアップロードします。

![manifest](/images/manifest.png)

データがインポートできたらData Setが作成されるので、視覚化する際に必要なデータの選別や、表示項目の編集を行い、Data Analsisを作成します。さらに必要に応じて、読み取り専用のDashboardを作成します。

最終的にこんな感じのダッシュボードができあがります。

![dashboard](/images/dashboard.png)

また、自動的にData Set内容が更新されるように、Schedule refresh(毎日〇時に参照データを更新する設定)をします。Data Setの内容が更新されれば、その内容はData Analsis、Dashboardへも反映されます。

![refresh](/images/refresh.png)

## ５．まとめ

データのインポートやLambdaでのデータ結合などで手間取りましたが、自動で、継続的にデータを視覚化できるシステムは、今回実践したようなAWSのサービス利用料の分析だけでなく、様々な分野での売り上げ分析などで活用することができると思います。

私は一社目がメーカーの営業職だったので、こういったツールがあれば、売り上げ報告会議などでかなり重宝したでしょう…。もし私がなにかしらの企業の情シスだったら、営業さんたちにこのツールの導入をお勧めたい…！とりあえず身近なところ（部内とか）で紹介させてもらって、誰かの役に立ったらいいなぁと思います。

