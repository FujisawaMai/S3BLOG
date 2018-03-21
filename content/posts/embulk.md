---
title: "EmbulkでS3のcsvデータをRDSにインポートする"
date: 2018-03-21T20:53:48+09:00
draft: false
tags: ["AWS","S3","RDS","EC2","Embulk"]
---

# 1.実施背景と記事概要

先日、["AWSサービスの使用料を、コストと使用状況レポートとQuicksightを使って継続的に自動で可視化する"](http://kotororo.net/posts/quicksight/)を投稿しました。この記事では、QuickSightでの可視化のためのデータをS3からインポートしています。しかし、もともとは、S3データをRDSにインポートし、それをさらにEmbulkにインポートするという方法を検討していました。

検討を進めていく中で、S3から直接、簡単にQuickSightへデータがインポートできるということが発覚したので、RDS～Embulkは使わないことになりましたが、Embulkは便利なツールで、覚えて置いたら今後なにかの役に立つかもしれないので、今回実施したテストの手順をひらたくまとめておこうと思います。

# 2.Embulkとは

Tresure Data社のOSSで、[公式レポジトリ](https://github.com/embulk/embulk)の説明をそのまま訳すと、

> Embulkは、さまざまなストレージ、データベース、NoSQL、クラウドサービス間のデータ転送を支援する並列バルクデータローダーです。

# 3.構成

構成図としては、こんな感じです。EmbulkはEC2にインストールして使います。

![embulk](/images/embulk.png)

# 4.EmbulkでS3に保存されているcsvをRDSにインポートするまでの手順

こんな感じのtest用のcsvファイルを事前に作成。

```csv:test.csv
fruits,color,price
apple,red,100
banana,yellow,200
melon,green,300
```

**4-1.aws cliをインストール**

```
$ sudo pip install awscli
```

**4-2.pipのインストール**

ubuntuだとデフォルトでpython3が入っている。

```
$ sudo python3 get-pip.py
$ sudo pip install awscli
```

pipが入っていない場合は、事前に以下を実行。

```
$ wget https://bootstrap.pypa.io/get-pip.py
```

**4-3.まずはS3に保存されているバケットのcsvをEC2上にコピー**

```
$ aws s3 cp s3://BUCKET_NAME/TEST.csv .
```

fatal error: Unable to locate credentials　となる場合は、aws configureをする。
[Amazon公式 AWS CLIの設定](https://docs.aws.amazon.com/ja_jp/cli/latest/userguide/cli-chap-getting-started.html)

**4-4.Embulkやそのほか必要なものインストール**

以下のEmbulk公式レポジトリのドキュメントを参照。
[Embulk公式レポジトリ](https://github.com/embulk/embulk)

Javaが必要なので、Javaもインストール。

```
$ sudo apt-get upate
$ apt search jre
$ sudo apt install default-jre
```

Embulkへインプット、アウトプットするためのプラグインをインストール。

```
$ embulk gem install embulk-input-s3
$ embulk gem install embulk-output-mysql
```

MySQLクライアントを入れておく。

```
$ sudo apt-get install mysql-client
```
データベースにログインし、フルーツテーブルを作っておく。

**4-5.Embulkの設定ファイル(test.yaml)を作る**

```
in:
  type: s3
  bucket: BUCKET_NAME
  path_prefix: test.csv
  endpoint: s3-ap-notrheast-1.amazonaws.com
  access_key_id: *****************
  secret_access_key: *****************
out:
  type: mysql
  host: ****.************.ap-northeast-1.rds.amazonaws.com
  user: ************
  password: ************
  database: fruits
  table: fruits
  mode: insert
```

**4-6.guessとrunの実行**

4-5で作成した設定ファイル（test.yaml）をもとにcsvをデータベースにインポートするためによしなに推測した、guessed.yamlを出力する。

```
$ embulk guess ./test.yaml -o guessed.yaml
```

以下によりS3からRDSへのインポートが実行される。

```
embulk run quessed.yaml
```

**4-7.データベースを確認しよう！**
```
mysql> show tables;
+------------------+
| Tables_in_fruits |
+------------------+
| fruits |
+------------------+
1 row in set (0.01 sec)

mysql> show columns from fruits;
+--------+------------+------+-----+---------+-------+
| Field | Type | Null | Key | Default | Extra |
+--------+------------+------+-----+---------+-------+
| fruits | text | YES | | NULL | |
| color | text | YES | | NULL | |
| price | bigint(20) | YES | | NULL | |
+--------+------------+------+-----+---------+-------+
3 rows in set (0.00 sec)

mysql> select * from fruits;
+--------+--------+-------+
| fruits | color | price |
+--------+--------+-------+
| apple | red | 100 |
| banna | yellow | 200 |
| melon | green | 300 |
+--------+--------+-------+
3 rows in set (0.00 sec)
```