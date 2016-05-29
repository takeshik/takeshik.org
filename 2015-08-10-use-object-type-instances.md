layout: post
title: "Object 型のインスタンスを活用"
date: 2015-08-10 4:45:00
comments: true
categories:
- programming
tags:
- CSharp
---

例えば C# で `lock` 構文を用いるとき、

```csharp
private static readonly _gate = new object();
// ...
lock (_gate) { /* do something... */ }
```

と、特別に用意した `static readonly` な `Object` 型のインスタンスを同期オブジェクトに用いるのがお約束となっています。何気なくスルーしてしまうこのイディオムですが、きちんと向き合ってみると意外と便利に活用できるみたいです。

本稿では、いわゆるシンボル型の実装を題材にして、`Object` 型のインスタンスの活用について記していきます。記事は C# コードを取り上げていますが、比較的言語中立な話なはず。

<!-- more -->

## `Object` は `Equals` と `GetHashCode` <small>(& その他)</small> できる (そのまんま…)

`Object` 型で公開されているメソッドを見ると、デバッグ用途の側面が強い `ToString` とか、型の自己記述性の保障である `GetType` をひとまず脇に避けると、結局のところ `Equals` と `GetHashCode` の 2 つの大きな機能によって特徴付けられているといえるのではないでしょうか。

結局のところ、`Object` 型は

```csharp
var x = new object();
var y = new object();
// x == y って書きゃいいんだけど、Equals で言及した手前、話を複雑にしたくないので…
Console.WriteLine(x.Equals(y)); // False
```

という操作が成立する―つまり、一意性を持ち、自身と他者を区別することができる、という機能が `Object` 型、およびこれを継承する全ての型の本質的特徴なわけですね。(※値型などにおける等値性の話とかは話が複雑になるのでスルー)

冒頭の `lock` ステートメントの例に戻って総括すると、要は特別な機能のない素のオブジェクトを 1 つ static に作るだけでロック管理に必要な一意性が保証できるので、それで同期オブジェクトとしては十分、ということになります。

## JavaScript にシンボル型が！ (唐突)

ところで、今更な話題なのかもとはいえ最近知ったのですが、最近の JavaScript には[シンボル型](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/Symbol)が導入されつつあるみたいです。

色々な言語にシンボルという概念がありつつ、皆それぞれ微妙に意味付けや挙動が違ったりするのがアレなところですが、JavaScript においては、文字列によるものとは独立してオブジェクトのプロパティ識別子になれるオブジェクトらしいです。

```javascript
var obj = {};
var symA = Symbol();
var symB = Symbol();
obj[symA] = 'foo';
obj[symB] = 'bar';
console.log(obj); // Object {Symbol(): "foo", Symbol(): "bar"}
```

例えば Scheme における `gensym` みたいなものと理解したのですが多分間違いないはず。プロパティをプライベートにできるし、絶対に衝突しない名前を安全に生成できるので、非常に便利そうな感じですね。

## C# でもこういうのを実装したい…！

で、[未だ引きずっている特殊な事情](http://yacq.net/)により、自分でも C# で実装してみたくなってきたのですが、ここで先述の `Object` 型の件を持ち出せば、何も難しいことなく実装できると気付きました。

```csharp
// license: MIT
public struct Symbol : IEquatable<Symbol>
{
    private readonly object _value;

    public static implicit operator string(Symbol obj)
    {
        if (obj.Name == null) throw new InvalidOperationException();
        return obj.Name;
    }

    public static bool operator ==(Symbol left, Symbol right)
        => left.Equals(right);

    public static bool operator !=(Symbol left, Symbol right)
        => !left.Equals(right);

    public static implicit operator Symbol(string name)
        => Create(name);

    private Symbol(object value)
    {
        this._value = value;
    }

    public static Symbol Create(string name)
    {
        if (name == null) throw new ArgumentNullException(nameof(name));
        return new Symbol(string.Intern(name));
    }

    public static Symbol Generate()
        => new Symbol(new object());

    public string Name
        => this._value as string;

    public override int GetHashCode()
        => this._value.GetHashCode();

    public override bool Equals(object obj)
        => obj is Symbol && this.Equals((Symbol)obj);

    public bool Equals(Symbol other)
        => this._value == other._value;
}
```

今回の例では、匿名の識別子だけでなく、一般の文字列による識別子についてもシンボル型の枠内で取り扱いたいので、作りもそれに応じたものとなっています。

この型では、`String` を受け取って (インターンした上で) それを元に `Symbol` を生成するものと、何も受け取らず、内部で `Object` インスタンスを新たに生成してそれを元に生成するものと、2 つの生成ルートを用意しています。

その一方で、シンボルの同一性については `Object` 型の機能による、最も単純なもので両者統一されている…というのがポイントです。

残りは、文字列とシンボルを透過的に両立できるようにするための暗黙のユーザ定義型変換類と、等値性に関する演算子のオーバーロードによって `Symbol` 型が構成されています。

このクラスを用いることで:

```csharp
var symA = Symbol.Generate();
var symB = Symbol.Generate();

var dic = new Dictionary<Symbol, string>()
{
    { "hoge", "foo" },
    { symA, "bar" },
    { symB, "baz" },
};

Console.WriteLine(symA == symA); // False
Console.WriteLine(Symbol.Create("hoge") == dic.First(x => x.Value == "foo").Key); // True
Console.WriteLine(symA == symB); // False
Console.WriteLine(dic["hoge"]); // foo
Console.WriteLine(dic[symA]); // bar
Console.WriteLine(dic[symB]); // baz
```

JavaScript における `Symbol` 型を用いたときのように、文字列によるキー アクセスと、生成したシンボル オブジェクトを経由しないと到達できないキー アクセスを両立させ、かつ、シンボルの一意性を保証することができました。

## まとめ

以上のように、意識して Object 型それ自体を扱うことで、有用な処理を簡潔な構造で実装できることを示しました。

かつて Java だか .NET だかで、共通基底型であるこの `Object` 型を抽象型とするかどうかの議論もあったそうです<sup>[要出典]</sup> が、結果的にはインスタンス化できるようなものとされたわけで、恐らく、冒頭の `lock` ステートメントや、今回挙げたような例での利用方法が念頭に置かれていたのではないかと思われます。

改めて見返してみると、なんということはない、非常に当たり前な話のようにも見えるのですが、一方で、ついつい忘れがちな前提でもあるような気がします。

一度手法として理解してしまえば、読む方でも書く方でもすぐに気付けるものの、この手のコードに初めて出会うと、意味を理解するまでにどうしても無駄に長い時間がかかってしまうのが、現実の悲しいところです。
