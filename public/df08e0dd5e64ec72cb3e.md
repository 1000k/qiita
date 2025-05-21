---
title: '抄訳: Constructor Injection vs. Setter Injection'
tags:
  - Java
  - テスト
private: false
updated_at: '2015-06-23T13:28:15+09:00'
id: df08e0dd5e64ec72cb3e
organization_url_name: null
slide: false
ignorePublish: false
---
[Constructor Injection vs. Setter Injection](http://misko.hevery.com/2009/02/19/constructor-injection-vs-setter-injection/)

2009年の記事ですが、依存性注入の基本を学ぶにあたり役立つ知識だったので、一部補足しながら訳しました。コードを単体テスト可能にする（＝「接合点を作る」）ために、ぜひ知っておくべき内容です。

クラスBに依存しているクラスAをテストするときに、依存性を排除する典型的な方法として **Constructor Injection** (コンストラクター・インジェクション) と **Setter Injection** (セッター・インジェクション) があります。この記事ではそれぞれの実現方法と、どちらがベターかを解説しています。

以下、抄訳です。


セッター・インジェクション とは？
-----------------------
- Springフレームワークで使用。
- コンストラクタは引数を取らない。
- インスタンス化後、そのインスタンスの setter メソッドを使ってオブジェクトを注入する。

```java
Database db = new Database();

OfflineQueue queue = new OfflineQueue();
queue.setDatabase(db);

CreditCardProcessor processor = new CreditCardProcessor();
processor.setOfflineQueue(queue);
processor.setDatabase(db);
```

コンストラクター・インジェクションとは？
----------------------------
- Pico-Container や GUICE で使用。
- コンストラクタの引数にインスタンスを渡し、メンバ変数に設定する方法。
- もし引数が渡されなかったらデフォルトのインスタンスを作成する。

```java
CreditCardProcessor processor = new CreditCardProcessor(?queue?, ?db?);
```

どちらが良いか？
----------------
一見すると、コンストラクタにごちゃごちゃ引数を渡さなくて済むぶん、セッター・インジェクション の方が楽に書けて良いように思えます。しかしコンストラクター・インジェクションの方が優れている部分があるため、私は後者を選びます。それは、コンストラクター・インジェクションはパラメータの順序を指定でき、[Circular Dependency（循環参照）](http://misko.hevery.com/2008/08/01/circular-dependency-in-constructors-and-dependency-injection/) に陥る危険が無いという点です。

一般的なアプリは多くのクラスと連携します。そこでセッター・インジェクションを採用してしまうと、何度も setter の呼び出しを行う事になってしまいます。これだと連携するクラスの数が増えるたびに setter をコールする行を追加しなくてはいけません。面倒ですし、コールし忘れの危険性も増えます。さらに `set` する順番が決まっている場合は最悪です。

一方で コンストラクター・インジェクションはコンストラクタさえ書いてしまえば、指定した順に、自動でクラスをインスタンス化してくれます。コンストラクトの時点ですべて設定が完了するので、後はそのオブジェクトを確実に使えます。

例
----
`CreaditCardProcessor` クラスをインスタンス化するケースを考えてみましょう。

```java
CreditCardProcessor processor = new CreditCardProcessor();
```

インスタンス化できてめでたしめでたし、とはなりません。実際にはこのクラスは `OfflineQueue` クラスと連携し、またDBとやりとりするために Database クラスとも連携します。これらのインスタンスを `set` してやらなければ使えません。

これらすべてをセッター・インジェクションで行うと、以下のようになります。

```java
// Database クラスのインスタンス化 
Database db = new Database();

// 設定
db.setUsername("username");
db.setPassword("password");
db.setUrl("jdbc:....");

// OfflineQueue クラスのインスタンス化
OfflineQueue queue = new OfflineQueue();

// 設定
queue.setDatabase(db);

// CreditCardProcessor クラスに、上で作ったインスタンスを setter でセットする
CreditCardProcessor processor = new CreditCardProcessor();
processor.setOfflineQueue(queue);
processor.setDatabase(db);
```

これで全部？いや、要件が増えればさらに setter でセットする内容は増えるかもしれません。何かフレームワークを使っていれば楽に書く方法があるかもしれませんが、使ってない場合はもうお手上げです。

では同じことをコンストラクター・インジェクションで実現してみましょう。`CreditCardProcessor` は以下のようにインスタンス化できます。

```java
CreditCardProcessor processor = new CreditCardProcessor(?queue?, ?db?);
```

コンストラクタは `OfflineQueue` と `Database` のインスタンスを必要とするので、両方ともインスタンス化しましょう。

```java
// 必要となるインスタンスを作成する
Database db = new Database("username", "password", "jdbc:....");
OfflineQueue queue = new OfflineQueue(db);

// コンストラクタに渡してやる
CreditCardProcessor processor = new CreditCardProcessor(queue, db);
```

コンストラクタにインスタンスを渡すことができました。これで完了です。

このコードの良いところは、コンストラクタに渡す引数が不足していたらコンパイルエラーがちゃんと出ることです。また、意図していない順番でインスタンス化される不具合も防げます。

というわけで、個人的にはコンストラクター・インジェクションがおすすめです。
