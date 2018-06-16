---
title: "Expressでアプリを作る① ExpressとNode.jsを使ってTodolistアプリを作る"
date: 2018-06-17T00:59:15+09:00
draft: true
tags: ["AWS","Node.js","Express","MongoDB"]
thumbnailImage: /images/180616_first_express_app_1/Sample_Express_App_Architecture.jpg
---

アレクサスキルをExpressを使って開発しようとするも、基礎的なことがわかってなさすぎてつまりまくってしまいました…
<!--more-->

あとデプロイとかもよくわかっていなくて…まずはExpressでTodoListアプリを作ってデプロイする、というとこまでを実践してみたので、役立ちそうなポイントをメモしておきます。

実践内容としては、AWSを使用し、こんな構成で、アプリを作りました。

![Sample_Express_App_Architecture.jpg](/images/180616_first_express_app_1/Sample_Express_App_Architecture.jpg)

記事は以下の3部作になります。

1. ExpressとNode.jsとMongoDBを使ってTodolistアプリを作る
2. デプロイする 方法①EC2とELBとR53の設定
3. デプロイする 方法②Elastic Beanstalkを使ってみる

この記事では【1. ExpressとNode.jsとMongoDBを使ってTodolistアプリを作る】をまとめています。

# 1.ExpressとNode.jsを使ってTodolistアプリを作る

## 1-1.Expressとは

Node.jsのWebアプリケーションフレームワークで、Webアプリを作るためのひな形や骨組みのこと。

今回サンプルアプリを作るにあたって、mozillaのtutorialを参考にしました。英語ですが細かく書いてあってわかりやすかったので、Express x Node.js入門の際、ぜひ参考にしてみてください！

[Express Tutorial: The Local Library website](https://developer.mozilla.org/en-US/docs/Learn/Server-side/Express_Nodejs/Introduction)

Expressフレームワークの構成、各要素の役割は以下の通り。

![express_framwork.png](/images/180616_first_express_app_1/express_framwork.png)

新規のTodolistを作成する場合の流れを以下にまとめます。

まず、RoutesはHTTPメソッドとパスをControllerに紐づけます。
例えば、以下の例だと、'/create_todo'に対するPOSTでのHTTP Requestと、controllers/create_todo_controller.jsのcreate_todo_postメソッドを紐付けている。

```javascript:list.js
var create_todo_controller = require('../controllers/createtodoController');
router.post('/create_todo', create_todo_controller.create_todo_post);
```

Modelsはアプリケーションに出てくる概念の持つデータと振るまいを定義します。
以下の例だとtodolistSchemaにtodolistの項目（宿題をする,etc）を定義しており、ふるまいに関してはmongooseの中でいろいろ定義されている。

```javascript:todolist.js
//Require Mongoose
var mongoose = require('mongoose');

//Define a schema
var Schema = mongoose.Schema;

var todolistSchema = new Schema({
    item: { type: String, required: true, max: 100 },
    add_date: { type: Date },
    limit_date: { type: Date }
});

//Export model
module.exports = mongoose.model('ToDoList', todolistSchema );

```

createtodoController.jsのcreate_todo_postメソッドでは、新規のTodolistを作成する処理を記述。結果をテンプレートエンジンにレンダリングし、結果をHTMLで表示する。

```javascript:createtodoController.js
const { body,validationResult } = require('express-validator/check');
const { sanitizeBody } = require('express-validator/filter');
var ToDoList = require('../models/todolist');
var async = require('async');

exports.create_todo_post = [
   
    // Validate that the item field is not empty.
    body('item', 'item name required').isLength({ min: 1 }).trim(),

    // Sanitize (trim and escape) the name field.
    sanitizeBody('item').trim().escape(),

    // Process request after validation and sanitization.
    (req, res, next) => {

        // Extract the validation errors from a request.
        const errors = validationResult(req);

        // Create a item object with escaped and trimmed data.
        var new_item = new ToDoList(
          { item: req.body.item }
        );

        if (!errors.isEmpty()) {  //= If there is error
            // There are errors. Render the form again with sanitized values/error messages.
            res.render('createtodo', { title: 'Create Todo',errors: errors.array()});
            return;
        }
    
        // Data from form is valid.
        // Check if item with same name already exists.
        ToDoList.findOne({ 'item': req.body.item })
            .exec( function(err, found_item) {
                    if (err) { return next(err);
                    if (found_item) {
                        // Item exists, render message.
                        res.render('error', {message: 'the item is already exist'});
                        return;
                    }
                    else {
                    new_item.save(function (err) {
                        if (err) { return next(err); }
                        // Item saved. Redirect to genre detail page.
                        res.render('create_result', {message: "'" +  new_item.item + "'" + " is added to ToDo List"});
                        });
                    }
                });
    }
];
```

## 1-2.MongoDBとは

MongoDBはNoSQLのデータベースです。NoSQLデータベースとしては、世界で一番使われているようです。
https://db-engines.com/en/ranking

[Express Tutorial Part 3: Using a Database (with Mongoose)](https://developer.mozilla.org/en-US/docs/Learn/Server-side/Express_Nodejs/mongoose)の中でも紹介されていますが、このアプリでは無料で使用できる[mLab's](https://mlab.com/welcome/)から利用しています。

このアプリでは、MongooseというMongoDBのモジュールを使用し、これを利用してデータベースにアクセスします。Mongooseは、ドキュメント指向のデータモデルを使用するオープンソースのNoSQLデータベースであるMongoDBのフロントエンドとして機能します。
http://mongoosejs.com/

## 1-3.できたもの

MozillaにのっていたExpressのチュートリアルを参考にしながら、一旦ローカルでこんな感じでアプリが完成しました。

![todolist_TODO_LIST.PNG](/images/180616_first_express_app_1/todolist_TODO_LIST.PNG)

アプリ全体のプログラムはこちら。
https://github.com/pistachiyoda/express-todolist/tree/default


### [メモ] ユーザ名とかパスワードとがをコードに含めるとき

コード中にパスワードなどをそのまま含めるのはとても危険。そこで環境変数を利用して、プログラム中にはパスワードに対して環境変数を設定し、その環境変数の中身はサーバーに置いておくことで、セキュリティが保てます。
node.jsの場合だと、[dotenv](https://www.npmjs.com/package/dotenv)を使って、これを実現できます。

詳しくはこちら！
https://www.npmjs.com/package/dotenv



次は、一旦ローカルで開発したこのアプリをデプロイしていきます!

