---
title: 依存性注入(DI)の解説とやり方
tags:
  - PHP
  - PHPUnit
  - テスト
  - 単体テスト
private: false
updated_at: '2017-11-28T22:04:50+09:00'
id: aef6aed46b0fc34cc15e
organization_url_name: null
slide: false
ignorePublish: false
---
**依存性注入 (Dependency Injection)** は、クラスを単体テスト可能にするために使われるテクニックです。
これが意識されていないが故に単体テストが全くできないコードをよく見かけます。

単体テストの際には必ず必要になる知識なので、解説しておきます。
以下のサンプルでは [PHPUnit](http://www.phpunit.de/manual/3.8/ja/index.html) を利用しています。


悪い例: 単体テストができないケース
----------------------
以下のような Foo クラスの `play()` メソッドは、単体テストケースが書けません。

```php:foo.php
class Foo {
    public function play() {
        $bar = new Bar();

        if ($bar->getSomething() === 1) {
            return true;
        }

        return false;
    }
}
```

`play()` 内で外部クラス Bar をインスタンス化しています。つまり `Foo::play()` メソッドは Bar クラスに **依存** しており、このままでは単体テストができません。

無理矢理テストケースを書くと、以下のようになります。

```php:foo_test.php
class FooTest {
    public function testPlay() {
        $foo = new Foo;
        $this->assertTrue($this->foo->play());
    }
}
```

何が問題かお分かりでしょうか？

Foo クラスのテスト結果は、Bar クラスに依存してしまっています。つまり、Foo の実装を何も変更しなくても、Bar の実装を変更したタイミングで Foo の単体テストが壊れてしまう可能性があります。

上記の例で言えば、 `Bar::getSomething()` メソッドを編集して、戻り値が決して `1` にならないような実装に変えてしまうと、 `Foo::play()` メソッドの戻り値は常に `false` になってしまいます。つまり、 `testPlay()` テストケースは永遠に成功しません。

また、これは2つのクラスにまたがる **結合テスト** であり、単体テストになっていません。

これらの問題を解決するために、以下の2つの修正が必要です。

1. `Foo:play()` メソッド内で直接 Bar クラスを `new` するのではなく、外部から Bar インスタンスを受け取って利用するだけにする
2. テスト時は **モック化** した Bar クラスを使い、 `Bar::getSomething()` の実装を無視できるようにする


モック化とは？
------------
モック化とは、クラスの挙動を置き換えることです。例えば特定のメソッドの戻り値を好きな値に指定することができます。

今回のケースでは、`Bar::getSomething()` が常に `1` を返すようにすれば、「`return true`」の行が実行されることをテストできますし、`1` 以外を返すようにすれば「`return false`」の行が実行されることをテストできます。

しかし先述のコードのままでは、テスト対象のメソッド (`Foo::play()`) 内で外部クラスの Bar を直接インスタンス化しているため、モックを注入する手段がありません。

まだピンと来ない方も、この先の具体例を見ればイメージが掴めると思います。


メソッドを単体テスト可能にする
------------------------------
もしメソッドが下記のようになっていればどうでしょうか？

```php:foo.php
    public function play(Bar $bar) {  // Bar クラスのインスタンスを受け取るようにした。
        if ($bar->getSomething() === 1) {
            return true;
        }

        return false;
    }
```

メソッド内で Bar クラスを `new` するのでなく、Bar インスタンスを引数に受け取って利用するだけにしました。これならテストケースでモックを注入することができます。

```php:foo_test.php
    public function testPlay() {
        $foo = new Foo;

        // Bar クラスのモックを作成する
        $bar = $this->getMock('Bar');

        // Bar::getSomething() の戻り値が 1 になるよう設定する
        $bar->expects($this->any())
            ->method('getSomething')
            ->will($this->returnValue(1));

        // Foo::play() に注入 (インジェクション)
        $this->assertTrue($foo->play($bar));
    }
```

`Foo::play()` メソッドは Bar クラスとの依存が解消され、単体テストが可能になりました。これでもう Bar クラスの実装を変更しても Foo クラスのテストが壊れることはありません。これが**依存性注入**です。

これはメソッドの引数にオブジェクトを渡せるようにする手法で、**インターフェース・インジェクション** (Interface Injection)[^1] と呼ばれます。

[^1]: 厳密なインターフェース・インジェクションでは、まずクラスの interface を作成し、それを委譲したメソッドを実装する必要があります。上記の例では簡略化のため interface は作成していません。


コンストラクター・インジェクション
----------------------------------
依存性注入は以下のようなやり方でも実現が可能です。

```php:foo.php
class Foo {
    /** @var Bar */
    protected $bar;

    public function __constructor(Bar $bar = null) {
        // メンバ変数として Bar インスタンスを作成しておく
        $this->bar = $bar ? $bar : new Bar;
    }

    public function play() {
        // メンバ変数の Bar インスタンスからメソッドを呼び出す
        if ($this->bar->getSomething() === 1) {
            return true;
        }

        return false;
    }
}
```

このケースでは、Foo クラスのコンストラクタに Bar クラスのオブジェクトを渡せるようにしています。これは **コンストラクター・インジェクション** (Constructor Injection) と呼ばれるテクニックです。

この場合のテストケースは以下のようになります。

```php:foo_test.php
    public function testPlay() {
        // Bar クラスのモックを作成する
        $bar = $this->getMock('Bar');
        $bar->expects($this->any())
            ->method('getSomething')
            ->will($this->returnValue(1));

        // Constructor Injection でモックを注入する
        $foo = new Foo($bar);

        $this->assertTrue($foo->play($bar));
    }
```

※補足: コンストラクタ内の3項演算子について。

```php
    public function __constructor(Bar $bar = null) {
        $this->bar = $bar ? $bar : new Bar;
    }
```

Foo クラスのインスタンス作成時に引数に `$bar` が渡されなかった場合は、デフォルトの Bar クラスを `new` するようにしています。

このコードにより、テストの時はモック化した Bar を注入して使うことができ、本番コードではパラメータを何も指定しないことでデフォルトの Bar クラスを使えるようにしています。

なお、PHP7 以上なら、NULL合体演算子 `??` を使って、以下のようにエレガントに書けます。

```php
$this->bar = $bar ?? new Bar;
// 今回の例では ?: を使っても OK
```


セッター・インジェクション
--------------------------
別のアプローチとして、クラスにオブジェクトを注入するためセッターメソッドを用意する方法もあります。

```php:foo.php
class Foo {
    /** @var Bar */
    protected $bar;

    public function setBar(Bar $bar) {
        $this->bar = $bar;
    }

    public function play() {
        if ($this->bar->getSomething() === 1) {
            return true;
        }

        return false;
    }
}
```

コンストラクターで Bar インスタンスを注入するのではなく、`setBar()` メソッドで Bar オブジェクトを注入しています。

この場合のテストケースは以下のようになるでしょう。

```php:foo_test.php
    public function testPlay() {
        // Bar クラスのモックを作成する
        $bar = $this->getMock('Bar');
        $bar->expects($this->any())
            ->method('getSomething')
            ->will($this->returnValue(1));

        // セッター経由でモックを注入する
        $foo = new Foo;
        $foo->setBar($bar);

        $this->assertTrue($foo->play($bar));
    }
```

これが **セッター・インジェクション** (Setter Injection) です。


まとめ
------
* 単体テストしたければ、メソッド内で外部クラスを new してはいけない。
* モックを注入可能にするためには、以下のいずれか手法を採る。
  * メソッドの引数に使いたいオブジェクトを渡せるようにする。 (**Interface Injection**)
  * 使う予定のオブジェクトをコンストラクタに渡せるようにする。(**Constructor Injection**)
  * オブジェクトを setter メソッドで渡せるようにする。(**Setter Injection**)


補足1: runkitで強引にメソッドの挙動を変える方法
-----------------------------------------------
PHPにおいては [runkit](http://php.net/manual/ja/book.runkit.php) を使うことで、モック化をしなくてもメソッドの挙動を強引に変えることが可能です。ただし破壊的な手段であり、これが必要な時点でオブジェクト指向的な設計ができていない疑いが非常に強いです。また、テストケースのメンテナンスコストも大幅に上がってしまいます。

できるだけ Constructor Injection 等の方法を用い、テスト可能になるようリファクタリングをしましょう。リファクタリング中はコードが醜くなるかもしれませんが、最終的に設計が改善できるならば結果OKです。 **コードの美しさよりも安全性が重要** です。


補足2: Constructor Injection vs Setter Injection
------------------------------------------------
[抄訳: Constructor Injection vs. Setter Injection](http://qiita.com/1000k/items/df08e0dd5e64ec72cb3e) にて、コンストラクター・インジェクションとセッター・インジェクションのどちらが良いかを考察しています。結論としては **コンストラクター・インジェクションがベター** です。


参考
----
- [依存性の注入 - Wikipedia](http://ja.wikipedia.org/wiki/%E4%BE%9D%E5%AD%98%E6%80%A7%E3%81%AE%E6%B3%A8%E5%85%A5)
- [DI（依存性注入）を白紙から説明してみる - 檜山正幸のキマイラ飼育記](http://d.hatena.ne.jp/m-hiyama/20060926/1159253903)
- [Inversion of Control Containers and the Dependency Injection pattern](http://martinfowler.com/articles/injection.html)
  - リファクタリング界の大御所、Martin Fowler氏の考察。
- [Dependency Injection のタイプ | TECHSCORE(テックスコア)](http://www.techscore.com/tech/Java/Others/Spring/1-3/)
- [第10章 テストダブル](http://www.phpunit.de/manual/3.7/ja/test-doubles.html#test-doubles.mock-objects)
  - PHPUnit 公式マニュアルによるモックの使い方。
