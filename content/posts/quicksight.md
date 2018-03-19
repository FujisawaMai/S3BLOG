---
title: "AWSサービスの使用料を、コストと使用状況レポートとQuicksightを使って継続的に自動で可視化する"
date: 2018-03-19T21:16:58+09:00
draft: false
tags: ["AWS","QuickSight","S3","Lambda"]
---

# １．目的

AWSにはサービス使用料を確認する方法として、請求ダッシュボードから、概要や料金明細を確認でき、またコストエクスプローラーからはサービス別、AZ別などのフィルターをかけることもできます。その他に請求明細レポートなど各種レポートから使用状況を確認することもできます。

しかし、これらツールによる料金確認だと、ダッシュボードからだと直近1か月、コストエクスプローラーも過去１２か月分しか確認ができないということから長期的な料金確認はできず、分析のためのグラフのカスタマイズはできません。また、各種レポートの中には今後利用できなくなるものがあり、AWSからはコストと使用状況レポートを使用することが推奨されています。

なので、AWSサービスの使用料を、長期的に、カスタマイズされた形で確認するために、コストと使用状況レポートとQuicksightを使って継続的に自動的に可視化してみようと思います。これは、S3に継続的に吐き出されるデータを自動的に可視化するので、コストと使用状況レポートに限らず、様々なものに転用できます。

# ２．QuickSightとは

QuickSightはデータを視覚化するAWSのサービスです。QuickSightがサポートするデータソース（S3やAthena、RDSなど）からData setを作成し、これをもとに視覚化を行います。詳しくは公式ドキュメントを参照ください。Amazon公式YoutubeのQuickSight Overviewがさっくり概要をつかみやすくておすすめです。

[**Amazon公式 QuickSight Overview（Youtube）**](https://www.youtube.com/watch?v=C_eT0xRNjCs "Amazon公式 QuickSight Overview（Youtube）")
[![IMAGE ALT TEXT HERE](http://img.youtube.com/vi/C_eT0xRNjCs/0.jpg)](http://www.youtube.com/watch?v=C_eT0xRNjCs)


[**Amazon公式Document　AWS document Amazon QuickSight**](https://docs.aws.amazon.com/ja_jp/quicksight/latest/user/welcome.html "Amazon公式Document　AWS document Amazon QuickSight")

# ３．コストと使用状況レポート

AWSサービスの使用状況についてかなり細かいレポートを出してくれるサービスで、請求ダッシュボード＞レポートから、S3にレポートを吐き出す設定ができます。

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

Lambdaを使って過去の、月ごとの、そのフォルダの中の最終データと、現在の月の最新データを結合させます。Lambdaには一時ファイルを作れるtmp/フォルダがあるので、それを使えばサーバレスにシステムを動かすことができます。

データの更新が自動で行われるように、S3へのPUTをトリガーにデータをマージし、S3のLatestフォルダにlatest.csvの名前でアップロードされるように設定、コーディングします。

サンプルコードは[Gist](https://gist.github.com/pistachiyoda/235ce579080e0a8b2e4b4f0dacc5e757)上にあげています。

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
データのインポートやLambdaでのデータ結合などで手間取りましたが、自動で、継続的にデータを視覚化できるシステムは、今回実践したようなAWSのサービス利用料の分析だけでなく、様々な分野での売り上げ分析などで活用することができると思います。私は一社目がメーカーの営業職だったので、こういったツールがあれば、売り上げ報告会議などでかなり重宝したでしょう…。もし私がなにかしらの企業の情シスだったら、営業さんたちにこのツールの導入をお勧めたい…！とりあえず身近なところ（部内とか）で紹介させてもらって、誰かの役に立ったらいいなぁと思います。

