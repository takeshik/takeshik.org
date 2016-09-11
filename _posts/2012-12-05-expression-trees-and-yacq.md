---
layout: post
title: "言語基盤としての Expression Trees、そして Yacq"
date: 2012-12-05 05:40:51
comments: true
categories:
- programming
tags:
- CSharp
- Yacq
- LINQ
- Expression Trees
- Advent Calendar
---

これは [Esolang Advent Calendar 2012](http://atnd.org/events/33784) の 4 日目の記事 <small>(えっ)</small> らしいです。
.NET における Expression Trees の簡単な紹介と、言語処理系のインフラストラクチャとしての利用についての簡単な紹介です。

<!-- more -->

## はじめに

言語処理系を構築するにあたっては AST (抽象構文木) を定義し、構築し、解析し最終的な出力を得る、というのがひとつの定石であるのは、たぶん疑うところではないと思われます。

また、近頃はリッチな言語の上で言語を構築することにより、基盤言語のライブラリ資産をそのまま適用できるようにしてしまおう、という発想が割と幅を利かせているように思われます。JVM とか、CLI とか、あと最近の流行は JavaScript ですね。

今回話題にするのは CLI、即ち .NET を基盤にした言語における話となります。

## Expression Trees in .NET

.NET Framework という一般的なインフラストラクチャにおいて、ひときわ逸般的な標準機能を挙げるなら、 **[Expression Trees](http://msdn.microsoft.com/ja-jp/library/bb397951.aspx)** ではないかと私は考えています。

.NET における Expression Trees は、現行のものは [DLR (Dynamic Language Runtime)](http://dlr.codeplex.com/) 由来とも言われますが、要するに **.NET Framework 標準の AST** です。

### Expression Trees の特徴

Expression Trees は以下の特徴を持っています。

#### コンパイル可能

Expression Trees は AST としてこれを独自に解析し、独自の処理を適用することもできます (LINQ to SQL や to Entities はこれによって式を SQL に変換したりしている) が、これを実行時にコンパイルし、単なる関数オブジェクトとしてを出力として受け、そのまま呼び出すことができます。

例えば、整数値を自乗する関数は:

```csharp
var param = Expression.Parameter(typeof(int));
var expr = Expression.Lambda<Func<int, int>>(
    Expression.Multiply(param, param),
    param
); // expr is Expression<Func<int, int>>
Console.WriteLine(expr.Compile()(16)); // 256
               // ~~~~~~~~~~~~~~ returns Func<int, int>
```

のようにして構築できます。

#### 言語中立

Expression Trees は C# などの .NET 言語を表現するための AST ではなく、.NET の、換言すれば [CIL](http://en.wikipedia.org/wiki/Common_Intermediate_Language) を表現するための AST となっています。

機能が洗練され、他の言語に引っ張られることがないとも考えられますし、逆に機能不足とも考えられるでしょう。

Expression Trees に標準で用意されている式ノードの種類は [こちら](http://msdn.microsoft.com/ja-jp/library/system.linq.expressions.expression.aspx) からだいたい確認できます。

#### 幅広い適用範囲

Expression Trees は .NET の標準機能であり、システム全体で幅広く使われています。

Expression Trees は LINQ のインフラストラクチャを構成しており、また、C# をはじめとしたいくつかの .NET 言語はコンパイル時に式から Expression Trees を構築することができます。

先ほどの自乗関数も、

```csharp
Expression<Func<int, int>> expr = n => n * n;
// expr.Compile(); ...
```

のようにして構築できます。(先の例より遥かに簡単ですが、これはこれで実行時の構築ができなかったり、一部の表現が利用できない等の問題があります)

#### 拡張可能

.NET 4 で Expression Trees の機能が大拡充されたのですが、その一環として、Expression Trees 自体を拡張できます。

即ち、自分で式ノード型を定義し、標準のノードへの変換を実装することで、最終的に標準の定義範囲内に収まることを保証してやることで、自前の AST を拡張の形で定義することができたりします。

例えば、以下のようにして新しい式ノード型 `DumpExpression` を定義します。この式ノード型はオペランドとして式を 1 つ受け取り、それを `Console.WriteLine` し、オペランドをそのまま返すものです。

```csharp
public class DumpExpression : Expression
{
    public Expression Expression
    {
        get; private set;
    }

    public override bool CanReduce
    {
        get { return true; }
    }

    public override ExpressionType NodeType
    {
        get { return ExpressionType.Extension; }
    }

    public override System.Type Type
    {
        get { return this._expr.Type; }
    }

    public DumpExpression(Expression expr)
    {
        this._expr = expr;
    }

    public override Expression Reduce()
    {
        // n => { Console.WriteLine(n); return n; }
        return Expression.Block(
            Expression.Call(
                typeof(Console).GetMethod("WriteLine", new [] { typeof(object), }),
                Expression.Convert(this._expr, typeof(object))
            ),
            this._expr
        );
    }
}
```

これは以下のように利用します:

```csharp
var param = Expression.Parameter(typeof(int));
// n => DUMP(n + 123)
var expr = Expression.Lambda<Func<int, int>>(
    new DumpExpression(Expression.Add(param, Expression.Constant(123))),
    param
);
var ret = expr.Compile()(1000);
```

上のコードで 1123 が出力され、ret にも 1123 が代入されます。

### Expression Trees の esolang 的価値

つまり、 **Expression Trees を使うと、.NET 上で動作する言語が簡単に作れてしまうのです！**

<small>(もちろん DLR を使う方が簡単なのかもしれませんが、DLR はむしろ型システムの構築まで面倒を見る感があります。外部ライブラリをたくさん参照するのも何かアレですし。あと、dynamic 割と前提だったり、相互運用性の面でちょっと面白みに欠ける気がします (単に触ったことがないだけ…)</small>

そして、これは単にコードを実行できるだけに留まりません。

単に実行できるだけではなく、コンパイル、即ち実行可能ファイルの作成も難しいことを考えるまでもなく可能ですし、さらには構築した Expression Trees を標準、あるいはサードパーティの LINQ のクエリプロバイダに引渡し、様々な変換や解析、特別な実行処理を受けることも可能になるのです。

完全に透過的に言語システムに溶け込んでいるわけではありませんが (それでも結構なレベルだと思いますが)、Code as Data の思想を .NET システムは標準で取り込んでいるということは、特筆に値するのではないでしょうか。

## .NET 言語 Yacq

さて、上で Expression Trees を使うと .NET 言語を簡単に作れる、ということに言及しましたが、私の言語 **[Yacq](http://yacq.net/)** <small>(GitHub: [takeshik/yacq](http://github.com/takeshik/yacq/))</small> は、これを実証するものです。(尤も、Yacq は現在かなりの機能を網羅しているため、簡単に作れる、という範囲のコード量ではないかもしれませんが…)

Yacq は、Expression Trees の特徴、即ちコンパイル可能性、言語中立性、可搬性、および拡張性を忠実に活かしつつ、その上で使いやすいものへと仕上げています。

例えば、int 型の引数を引数に取り、それを画面に表示し、引数値をそのまま返す関数を Yacq で記述し、その式構造を得るには:

```csharp
var expr = YacqService.Parse("(\ [n:Int32] n.(print) n)");
// to compile and invoke:
((Expression<Func<int, int>>) expr).Compile()(123);
```

と記述します。Parse メソッドは `Expression<Func<int, int>>` 型を返すので、(キャストして) 普通に `Compile()` して関数オブジェクトを生成することができます。

そして、Yacq が Expression Trees の拡張性を十分に活かした構造となっているために、システムの利用可能な範囲を狭めることなく、より使いやすいものに仕上がっています。

先ほどの Yacq のコード文字列 `(\ [n:Int32] n.(print) n)` は、パーザによって (上における DumpExpression のように) Expression Trees を独自に拡張したツリー構造へと変換されます。

<small>(因みに、Yacq のパーザには linerlock さんの [Parseq](https://github.com/linerlock/parseq) を利用しています)</small>

ここで特筆すべきは、Yacq はパーザによるコードの構文解析を通した利用だけでなく、Expression Trees を拡張した AST を直接利用することもできるという点です。

Yacq コード `(\ [n:Int32] n.(print) n)` を表す式ツリーは、次のように構築できます:

```csharp
var expr = YacqExpression.List(
    YacqExpression.Identifier("\\"),
    YacqExpression.Vector(
        YacqExpression.List(
            YacqExpression.Identifier(":"),
            YacqExpression.Identifier("n"),
            YacqExpression.Identifier("Int32")
        )
    ),
    YacqExpression.List(
        YacqExpression.Identifier("."),
        YacqExpression.Identifier("n"),
        YacqExpression.List(
            YacqExpression.Identifier("print")
        )
    ),
    YacqExpression.Identifier("n")
);
```

このように、`foo.bar` を `(. foo bar)`、`foo:bar` を `(: foo bar)` とすることさえ把握すれば、そのままの構造です。

<small>(ちょっとコード量長いですけど、この内容だったらパーザ叩けばよいので、割とどうでもよいかと…)</small>

上の式ツリーは、システム標準の Reduce() メソッドによる変換機構により、システム標準の式ノードのみの構造となるまで変換されます。

だいたいこのあたりが最終段階の一歩手前くらいです:

```csharp
var expr = YacqExpression.AmbiguousLambda(
    new Expression[] {
        YacqExpression.Identifier("n")
            .Method("print"),
        YacqExpression.Identifier("n"),
    },
    YacqExpression.AmbiguousParameter(typeof(int), "n")
);
```

割と最終的な形が見えてきたのではないかと思われます。Identifier からの Method メソッドのチェインが目を引きます。

そして、最後はこうなります:

```csharp
var param = Expression.Parameter(typeof(int), "n");
var expr = Expression.Lambda<Func<int, int>>(
    Expression.Block(
        Expression.Call(
            typeof(Console).GetMethod("WriteLine", new [] { typeof(int), }),
            param
        ),
        param
    ),
    param
);
```

`Identifier` は名前ベースの解決が行われ `Parameter` の参照となり、また `.Method("print")` は `Console.WriteLine` メソッドの呼出へと変換されます。

やや掻い摘んだ説明となりましたが、以上のように、Yacq の AST は Expression Trees をベースとし、そこにライブラリ的要素を導入しつつも、あくまで Expression Trees の拡張として振る舞えるものとなっていることを説明しました。

また、Yacq を、Yacq とは全く関係ない部分で、単純な Expression Trees のヘルパとして利用することもできます。

```csharp
var expr = YacqExpression.Candidate(typeof(Enumerable))
    .Method("Range",
        Expression.Constant(1),
        Expression.Constant(100)
    )
    .Select("Select",
        YacqExpression.AmbiguousLambda(
            YacqExpression.Function("+",
                YacqExpression.Identifier("e"),
                Expression.Constant(100)
            ),
            YacqExpression.AmbiguousLambda("e")
        )
    )
    .Method("Sum");
```

普通の Expression Trees のみで書くと:

```csharp
var e = Expression.Parameter(typeof(int));
var expr = Expression.Call(typeof(Enumerable), "Sum", null,
    Expression.Call(typeof(Enumerable), "Select", new [] { typeof(int), typeof(int), },
        Expression.Call(typeof(Enumerable), "Range", null,
            Expression.Constant(1),
            Expression.Constant(100)
        ),
        Expression.Lambda<Func<int, int>>(
            Expression.Add(
                e,
                Expression.Constant(100)
            ),
            e
        )
    )
);
```

型推論もしてくれないし、拡張メソッドの扱いもしてくれない (params 引数の扱いも…) し、とても手で組めたもんじゃありません。

ということで、こういう所でも Yacq の「Expression Trees 拡張」としての旨味があると考えています。

# まとめ

* .NET における Expression Trees は Esolang 的にも興味深いシステムである。
* Expression Trees を用いれば、有用な処理系を簡単に作ることができそうである。
* Yacq は言語としても、Expression Trees 拡張としても色々便利である (宣伝)。

遅くなってしまってすみませんでした＞＜

# 関連記事

* [Expression Trees with IL Emit](/blog/2011/12/14/expression-trees-with-il-emit/)
    * [C# Advent Calendar 2011](http://atnd.org/events/21988) で寄稿した記事。<br />Expression Trees 単体では型・アセンブリが作成できないので、IL Emit との禁断の合わせ技について解説
