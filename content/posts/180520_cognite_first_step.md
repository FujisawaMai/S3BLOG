---
title: "はじめてのCognito!CognitoのログインUIを使って認証システムを実装したまとめ"
date: 2018-05-20T23:53:39+09:00
draft: false
tags: ["AWS","S3","Cognito","CloudFront"]
---

# 1.記事概要

業務でCognitoを軽く触ることになったので、その練習用にサンプルアプリを作りました。AWSのチュートリアルをベースに、サンプルアプリの作り方と、引っ掛かりそうなところをまとメモしておきます。

# 2.Cognitoって？

公式様からそのまま引用させていただきます…

>Amazon Cognito は、ウェブアプリケーションやモバイルアプリケーションの認証、許可、ユーザー管理をサポートしています。ユーザーは、ユーザー名とパスワード、または Facebook などのサードパーティーを通じて直接サインインできます。

[AWS公式 Amazon Cognito とは](https://docs.aws.amazon.com/ja_jp/cognito/latest/developerguide/what-is-amazon-cognito.html#pricing-for-amazon-cognito)

# 3.OAuth2.0って？

OAuth2.0とは、あるクライアントアプリが、あるIDプロバイダーのIDを使って権限を得るときに、クライアントアプリがアクセストークンを認可サーバーへ要求～認可サーバーがクライアントアプリにアクセストークンを発行するフローを標準化たもの。

詳しくは、私が今回OAuth2.0の理解の際、とてもとてもお世話になったこちらのサイトをご参照ください。

[一番分かりやすい OAuth の説明](https://qiita.com/TakahikoKawasaki/items/e37caf50776e00e733be)

[OAuth 2.0 全フローの図解と動画](https://qiita.com/TakahikoKawasaki/items/200951e5b5929f840a1f)

# 4.サンプルアプリ

## 4-1.サンプルアプリ概要 ##

このアプリは、以下のように動作します。

1. ユーザがCognitoにより認可されたら、ログイン後ページへ変移する
2. アーティスト名を入力すると、DynamoDBに格納されている、私が一番好きなそのアーティストの曲が表示される

![180520_cognite_first_step_7](/images/180520_cognite_first_step_7.png)

アプリの構成、各サービスの役割こんな感じです。

![180520_cognite_first_step_1](/images/180520_cognite_first_step_1.png)


- **Cognito User Pool**

  CognitoのUser Poolでは、その名の通り、ユーザを管理し、ユーザのサインイン、サインアップの設定ができます。また、Cognito独自のログインUIの設定、アプリクライアント（この場合はS3の静的コンテンツ）の設定等ができる。

- **Cognito Identity Pool**

  Identity Poolにより、サインインしたユーザに割り当てるロールの設定ができます。このサンプルアプリの場合、サインインしたユーザにDynamoDBへのアクセス権を割り当てるようにする。

- **DynamoDB**
  
  こんな感じで、私が好きなアーティスト、私的そのアーティストのベストソングを格納。選曲に意外と時間がかかってしまいまして…

  もし音楽の趣味が合う人がいてもいなくても、ドラムンとかハウス流しながらもくもく会したいな…

  もはや別にエレクトロじゃなくても、おのおののおすすめ音楽を聴きながらもくもくする…みたいなもくもく会いいなぁ…

```
{
    "Count": 7,
    "Items": [
        {
            "SongTitle": {
                "S": "Take Her Place"
            },
            "Artist": {
                "S": "Don Diablo"
            }
        },
        {
            "SongTitle": {
                "S": "Perfect Stranger"
            },
            "Artist": {
                "S": "Jonas Blue"
            }
        },
        {
            "SongTitle": {
                "S": "Alone"
            },
            "Artist": {
                "S": "Mashmello"
            }
        },
        {
            "SongTitle": {
                "S": "Beastmode"
            },
            "Artist": {
                "S": "Yellow Claw"
            }
        },
        {
            "SongTitle": {
                "S": "Gecko"
            },
            "Artist": {
                "S": "Oliver Heldens"
            }
        },
        {
            "SongTitle": {
                "S": "Dreamer"
            },
            "Artist": {
                "S": "Axwell λ Ingrosso "
            }
        },
        {
            "SongTitle": {
                "S": "Blood Sugar"
            },
            "Artist": {
                "S": "Pendulum"
            }
        }
    ],
    "ScannedCount": 7,
    "ConsumedCapacity": null
}
```

- **S3**

  今回のアプリはS3で作っています。ログインUIはCognitoのものを使いますが、それ以外のログイン前、ログイン後のページのhtmlやDynamDBからデータを取ってくる部分やUser PoolとID Poolの紐づけの部分のJavaScriptをS3に置く。

## 4-2.User Poolの設定 ##

まずはUser Poolの設定。

公式チュートリアル、[ユーザープールの開始方法](https://docs.aws.amazon.com/ja_jp/cognito/latest/developerguide/cognito-user-pool-as-user-directory.html)の[ステップ1.ユーザープールを使用して、ユーザーディレクトリを作成する](https://docs.aws.amazon.com/ja_jp/cognito/latest/developerguide/cognito-user-pool-as-user-directory.html)、[ステップ 2. アプリケーションを追加して、ホストされたウェブの UI を有効にする (オプション)](https://docs.aws.amazon.com/ja_jp/cognito/latest/developerguide/cognito-user-pools-configuring-app-integration.html)を実施。

このアプリではID プロバイダーにCognito User Poolを使用しているので、ソーシャルサインイン（FacebookなどのIDを使用したサインイン）を実装するステップ３などはいったん次の機会に置いておきます…。

基本的にチュートリアル通りなのですが、注意点や私的ポイントを以下にちょっとメモ。

### 4-2-1.ユーザープールを作成するときの注意点 ###

アプリクライアント設定時、デフォルトではクライアントシークレットの生成にチェックがついているが、これをはずす。これは、Amazon Cognito JavaScriptSDKは、クライアントシークレットを使用しないので、例外処理が行われてしまうため。
[チュートリアル: JavaScript アプリのユーザープールを統合する](https://docs.aws.amazon.com/ja_jp/cognito/latest/developerguide/tutorial-integrating-user-pools-javascript.html)のステップ9の注記を参照ください。

![180520_cognite_first_step_2](/images/180520_cognite_first_step_2.png)

### 4-2-2.アプリクライアントのコールバックURL ###

CognitoのコールバックURLはhttps通信のものしか設定できず（Oauth2の仕様としてhttpsでの通信しか許容されていない）、S3の静的ウェブサイトだとhttpsの設定ができないが、ローカルホストは例外。後々CloudFrontを使ってhttpsの設定するが、キャッシュとかいちいち消すのが大変なので、まずはローカルで開発するためコールバックは(http://localhost:8080)にしておく。

ローカルサーバーをさくっと作るには[http-serverコマンド](https://www.npmjs.com/package/http-server)を使いました。[前回の記事のステップ3](http://kotororo.net/posts/ajax_experience/)でも使ってるのでよければ参照してください。


### 4-2-3.アプリクライアントで設定するOAuthのフロー、スコープ ###

- **許可されているOAuthフロー**

  Cognitoでの認可にはOAuth2.0が使われている。OAuth2.0には認可するための方法（フロー）が何種類かあるが、Cognitoはその中のAuthorization code grant, Implicit grant, Client credentialsを採用できる。

- **許可されているOAuthスコープ**

  スコープとは、そのユーザのどのリソースを扱いたいか。たとえばfacebookなら、profileのスコープを要求したアクセストークンを使ったら、プロフィール情報の閲覧はできるけど、そのほかの情報は取得できない。postのスコープを要求した取得したアクセストークンなら、投稿はできるけどプロフィール情報の閲覧はできない。スコープは複数要求することもできる。また、openidのスコープなら、そのidのidpでのユーザIDが取得できる。

Authorization code grantは認可サーバーが発行した認可コードによりアクセストークンを得ますが、この処理を実行するためにはサーバーが必要となる。また、Client credentialsはユーザー認証は行わず、クライアントアプリケーションの認証のみがおこなわれるフローなので、ユーザー認証したい今回のアプリには適さない。なので、今回のアプリでは、サーバー不要で、認可サーバから直接アクセストークンを受け取るImplicit grantを採用。また、AWSの認証情報を渡すユーザーを識別するためのユーザーIDを取得するため、OAuthスコープはopenidを選択する。

![180520_cognite_first_step_3](/images/180520_cognite_first_step_3.png)

## 3-3.Identity Poolの設定 ##

### 3-3-1.認証プロバイダーの設定 ###

今回のアプリではCognitoを認証プロバイダーにするので、以下のように設定する。

### 3-3-2.ロールの設定 ###

【認証されたロール】はユーザに与えられるアクセス権限を定義するので、このロールがDynamoDBにアクセスできるようIAMロールの設定をする。

## 4.htmlとapp ##

まずhtmlはこういう感じです。

```index.html
<html>
    <head>
        <title>cognito test page</title>
    </head>
    <body>
        <h1>Cognito test app</h1>
        <h2 id='appTitle'>アーティスト名を入力したら私が一番好きなそのアーティストの曲を表示する</h2>
        <h3 id='before'>ログイン前に表示する画面</h3>  
        <button id='loginBtn'>login</button>
        <h3 id='loggedIn'></h3>
        
        <div id='artistInput'></div>
        <div id='showDbItemBtn'></div>
        <div id='dbItem'></div>
        <script src="https://sdk.amazonaws.com/js/aws-sdk-2.243.1.min.js"></script>
        <script src='app.js'></script>
    </body>
</html>
```

ログインボタンクリック～ログインURL生成～id poolとuser poolを紐づけて、IDプールのクレデンシャルを取得～DynamoDBから情報を取得して表示、までの処理。

```app.js
//cognitoのログインURLを生成して、そこへ移動する（OAuth2）
document.getElementById('loginBtn').addEventListener('click', () => {
    //Cognitoがホストするログインページのドメイン。ユーザープールで設定したやつ。最後に\loginをつける。
    const loginEndPoint = '---domain---/login';
    const params = new URLSearchParams({
        //implicitなのでresponse_typeはtoken
        response_type: 'token',
        client_id: '---app_client_ID---',
        redirect_uri: 'http://localhost:8080',
        scope: 'openid',
    });
    //location.hrefはそのページのURLを指すので、そこにログイン画面のURLを代入したなら、ログイン画面に移動することになる。
    //URLSearchParams.toString()で、URLで使用するのに適したクエリー文字列を生成する。
    location.href = `${loginEndPoint}?${params.toString()}`;;
});

//AWSのAPIを呼び出すためのキーを初期化
let accessKeyId = '';
let secretAccessKey = '';
let sessionToken = '';

(() => {
    //Location.hash ・・・URLのうち、'#'とそれに続くフラグメント識別子を収めたDOMString
    const params = new URLSearchParams(location.hash.slice(1));
    //loginしてなかったら、ここでreturnされる
    //URLSearchParams.has()で、指定された検索パラメーターが存在するかを表すBoolean値を返す。
    if (!params.has('id_token')) return;
    //loginしてたらを非表示にするもの
    document.getElementById('loginBtn').hidden = true;
    document.getElementById('before').hidden = true;
    //ログイン後に表示するもの 
    document.getElementById('loggedIn').innerHTML = "ログイン後に表示する画面";
    document.getElementById('showDbItemBtn').innerHTML = "<button id='showDbItem'>DynamoDBのデータを取得する</button>";
    document.getElementById('artistInput').innerHTML = "<input type='text' id='artistName' type='text'>";
    //id poolとuser poolを紐づけて、IDプールのクレデンシャルを取得する
    AWS.config.region = 'ap-northeast-1'; // リージョン
    AWS.config.credentials = new AWS.CognitoIdentityCredentials({
        IdentityPoolId: '---Identity_Pool_Id---',
        Logins: {
            'cognito-idp.ap-northeast-1.amazonaws.com/---Pool_ID---': params.get('id_token'),
        },
    });
    //AWSのAPIを呼び出すためのキーが取得できる
    AWS.config.credentials.get(function(err){
        if (err) {
            alert(err);
        }
    });
})()

document.getElementById('showDbItemBtn').addEventListener('click', () => {
    showDbItem(function(err, data){
        document.getElementById('dbItem').innerHTML = "好きな曲:"+data.Item.SongTitle;
    });
});

function showDbItem(callback){
    var Artist = document.getElementById('artistName').value;
    var dynamodb = new AWS.DynamoDB();
    var params = {
        TableName : 'test_MusicTable',
        Key: {
          "Artist": Artist,
        }
      };
      var documentClient = new AWS.DynamoDB.DocumentClient();
      documentClient.get(params, function(err, data) {
          callback(err, data);
          console.log(data);
      });

}
```

これで、ローカルホストをコールバックとしたアプリができました。

あとはこのコールバックURLをhttpsで通信できるように、S3で静的ホストしてるURLをCloudFrontでssl化して、リダイレクト先をlocalhostから設定しなおせば完成です。

## 5.CloudFrontでhttps通信にする ##

CloudFrontを使ってhttps通信をするには、BehaviorのViewer Protocol Policyを”Redirect HTTP to HTTPS”に設定します。

![180520_cognite_first_step_8](/images/180520_cognite_first_step_8.png)

## 6.まとめ ##

完成イメージはこんな感じです。

![180520_cognite_first_step_9](/images/180520_cognite_first_step_9.png)

わからないことが多すぎてお世話になっているメンターさんにたくさん助けていただきました…認証や認可、OAuth2のフローなど、学ぶことが多いサンプルアプリでした。