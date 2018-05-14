---
title: "ローカルにNode.jsでサーバーをたててAjaxを体験したい"
date: 2018-05-7T22:36:51+09:00
draft: false
tags: ["Node.js"]
---
# 1.記事概要

最近、あるアレクサアプリをExpressやNode.jsやDyanmoDB、などなどで作ろうとするも、それぞれの技術理解が中途半端で全体的によくわからなくなってきてしまったため、いきなりアプリを作るのではなく、アプリ制作に必要な要素、技術をひとつずつ理解していこう、という結論に至りました。

近頃はまず、JavaScriptの基礎を勉強しています。

この記事ではローカルにNode.jsのコマンドラインでサーバーをたててAjaxを体験する方法をメモしています。

# 2.Ajaxって？

Asynchronous JavaScript + XMLの略で、JavaScriptがサーバーと通信した結果を使ってHTMLを操作するのがAjaxです。非常によく使われるのがGoogle Mapの例で、Ajaxにより以下のプロセスでインタラクティブなレスポンスが実現できます。

1. Google Mapでユーザーが地図を動かす
2. 動かした先の地図の情報をサーバーに取得しにいく
3. サーバーから地図データが返ってくる
4. その地図データをHTMLに埋め込まれる

私はとりあえず、JavaScriptによって実行されるHTTPリクエストはAjax、と理解しています。

# 3.サーバーを作るのには"http-server"を使う

http-serverというコマンドラインを使います。

以下のコマンドでインストールして、

```
> npm install http-server -g
```

実行したいアプリファイルがあるパスを指定して、アプリを実行します。

```
> http-server [path] [options]
```

これにより、http://localhost:8080でサーバーにアクセスできます。

詳細は[公式](https://www.npmjs.com/package/http-server)を参照ください。

# 4.AjaxによるHTTPリクエストでJSONデータを取得する

練習で、”IDを入力したら、そのIDを持つ人物の名前をAjaxで取得するプログラム”を作ってみました。

メンバーはこんな感じで。

```member_info.json
[
    {"id":"1","name":"山田"},
    {"id":"2","name":"佐藤"},
    {"id":"3","name":"松本"},
    {"id":"4","name":"高田"}
]
```
   
```index.html
<html>
    <head>
        <title>example</title>
        <script type="text/javascript" src="app.js"></script>
    </head>
    <body>
        <h1>ID番号を入力したら、そのIDを持つ人物の名前をAjaxで取得する</h1>
        ID番号を入力してください:<input type = "text" id = "id">
        <button onclick="btnclick();">送信</button>
    </body>
</html>
```

```app.js
function btnclick(){
    var id = document.getElementById("id").value;
    //XMLHttpRequestオブジェクトを作る
    var xhr = new XMLHttpRequest();
    //リクエストを送る
    xhr.open("GET", "member_info.json", true);
    //読み込みが完了したときに実行される関数を定義
    xhr.onload = function(e){
        var json_data = xhr.responseText;
        var parsed = JSON.parse(json_data);
        for(i = 0; i < parsed.length; i++){
            if (id == parsed[i].id){
                var name = parsed[i].name;
            }
        }
        alert(name);
    }
    //送信する
    xhr.send();
}
```

こんな感じになります。
![ajax_experience_1.PNG](/images/ajax_experience_1.PNG)

↓

![ajax_experience_2.PNG](/images/ajax_experience_2.PNG)