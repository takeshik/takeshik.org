layout: post
title: "エレメンタル プログラミングのすゝめ"
date: 2015-02-28 01:54:05
comments: true
categories:
- programming
tags:
- CSharp
- Joke
---

プログラマが日々コードを書くなかでも、そこには山があり、谷があり…そして、光があり、闇があります。プログラマは、いつも同じような無味乾燥なコードを書いているわけではないはずです。

周りの人に感動を与える芸術的な光のコードを書くときもあれば、目を背けざるを得ない、絶望を振りまく闇のコードを書かざるを得ないときもある。そして、そんな深い絶望を超絶技巧でねじ伏せる魔術的なコードは、両方の要素を具えているといえます。

プログラミングとは、そう、光と闇との大いなる戦い…今回提唱する**エレメンタル プログラミング**で、皆さんもこの大いなる戦いに身を投じましょう。

(注: 本稿は有用と思われる内容も含んでいますが、本質的にはネタ記事です)

<!-- more -->

## コードの光と闇を示す古典的手法

一般に、コードに直接現れない意図を示す手段としては、名前などでそれを示すことを別にすれば、コメントが挙げられるでしょう。

```csharp
// HACK: 無理を無理やり解決
public void SomeTrickyWorkaround()
{
   // ...
}
```

最近 (といっても結構前ですが) では、ドキュメント コメントといった仕組みで何らかの特別な意図を示すこともできます。

```csharp
/// <summary>
/// インフラストラクチャ。独自に作成したコードから直接使用するためのものではありません。
/// </summary>
public class SomeDangerousClass
{
   // なんかやばいクラス
}
```

しかし、これらコメントはコードにおいて何らの意味を提供することはありません。上述のドキュメント コメントのように、コンパイラやポストプロセッサによって特別な意味を見出されたとしても、それらはコードそれ自体の情報とはなれません。

## エレメンタル プログラミング: 属性で属性を与える

しかし、例えば C# (というより .NET) には[カスタム属性](https://msdn.microsoft.com/ja-jp/library/5x6cd29c.aspx)があります。カスタム属性はプログラム内の任意の要素に対して独自の意味を付け加えることができる言語要素です。この素晴らしい仕組みを使えば、自分、そして他人のコードに属性を与えることができます。

Java におけるアノテーションなど、同様の言語要素がある言語でもエレメンタル プログラミングが行えますが、属性という語のダブル ミーニングが楽しめるかは言語に依存します。

### カスタム属性 (attributes) で自分だけの属性 (elements) を作ろう！

早速、コードを属性で満たすための詠唱準備に取り掛かりましょう。以下のようにすればカスタム属性を定義できます。

```csharp
public enum CodeElement
{
    Light = 1,
    Dark = 2,
}

[AttributeUsage(System.AttributeTargets.All, AllowMultiple = true)]
[Conditional("DEBUG")]
public abstract class ElementalCodeAttribute : Attribute
{
    public abstract CodeElement Element { get; }
    public string Description { get; private set; }

    protected ElementalCodeAttribute(string description = null)
    {
        this.Description = description;
    }
}

[AttributeUsage(System.AttributeTargets.All, AllowMultiple = true)]
[Conditional("DEBUG")]
public sealed class LightAttribute : ElementalCodeAttribute
{
    public override CodeElement Element { get { return CodeElement.Light; } }

    public LightAttribute(string description = null) : base(description) { }
}

[AttributeUsage(System.AttributeTargets.All, AllowMultiple = true)]
[Conditional("DEBUG")]
public sealed class DarkAttribute : ElementalCodeAttribute
{
    public override CodeElement Element { get { return CodeElement.Dark; } }

    public DarkAttribute(string description = null) : base(description) { }
}
```

[`AttributeUsageAttribute`](https://msdn.microsoft.com/ja-jp/library/system.attributeusageattribute.aspx) でカスタム属性の詳細な挙動を定義するわけですが、`AttributeTargets.All` を指定することで全てのプログラム要素に対して属性を付与することができ、また、`AllowMultiple` を `true` にすることで属性を単に付与するだけでなく、複数回付与することでその威力を更に強大にすることを可能にしています。

また、カスタム属性の引数に文字列を与えることで、コードがその属性を纏うに至った経緯など (フレーバー テキスト) についても簡単に記すことができるようになっています。

そして、上で定義したカスタム属性には [`ConditionalAttribute`](https://msdn.microsoft.com/ja-jp/library/system.diagnostics.conditionalattribute.aspx) も付与しています。これが付与されると、引数で与えられたコンパイル時定数が定義されずにビルドされた場合、そのカスタム属性のプログラム要素への付与は全て取り除かれます。つまり、どのメンバが光なのか、あるいは闇なのかはリアル ワールドに Release ビルドの形で出荷された時点で分からなくなります (ただし、カスタム属性の定義自体は残ります)。

今回は光と闇の 2 属性のみ定義しましたが、もちろん一例にすぎません。コードの世界観に応じて四大でも五行でも、その他様々な属性に対応できます。

### 実際に属性を与えてみる

それでは、道具も揃ったところで、実際にコードを属性で満たしてみましょう。

```csharp
[Dark]
public void DirtyMethod()
{
    // ...
}
```

`DirtyMethod` が闇を纏ったメソッドであることが一目瞭然です。

```csharp
[Light("すごい")]
[Light("つよい")]
[Light("ありがたい")]
public class GreatClass()
{
    // ...
}
```

このクラスはあまりのすごさ、つよさ、ありがたさに平伏すしかないであろうことが容易に理解できます。複数記述は `[Light, Light, Light]` のようにも記述できるので、時と場合とコーディング規則に応じて使い分けましょう。

```csharp
public bool TryParse(string s, [Dark("ref args are evil")] out SomeValue value)
{
   // ...
}
```

出力 (`out`) 引数は利用者にある種の不便を強い、本質的に闇を背負っていることを主張しています。

```csharp
[return: Dark]
public object OperationWhichMayFail()
{
    // ...
    if (succeeded)
        return result;
    else
        return "エラーが発生しました";
}
```

返り値が闇に満ちているので、利用の際は細心の注意が必要なことが分かります。

```csharp
[module: Dark]
```

そもそもモジュール自体を相手にすることが一般的でないので、触れざるを得ない時点で闇があるのかもしれません。

```csharp
[assembly: Light]
[assembly: Dark]
```

このアセンブリは光と闇が両方そなわっており、最強に見えることが確定的に明らかです。

以上に示したように、細部から全体にわたり、コードの様々な部分に属性を与えることができます。

## 真面目な考察

エレメンタル プログラミングは単にコードに超自然的な力を与えるだけではなく、コメントを記述するより有用な面があるとも考えられます。

カスタム属性は上述のコードで示したとおり、コード中の様々なプログラム要素に対して付与できます。つまり、単なる行または範囲であるコメントと違い、その指し示す範囲が明確かつ追跡可能です。

追跡可能である、ということはつまり、IDE の支援によって属性を持つコードを検索し、また、プログラム中から属性の付与状況を取得することもできます (リフレクション)。これにより、例えば CI を用いてコード中の光と闇のバランスをモニタリングすることで終末的状況を事前に察知し、光の戦士たちを召喚するといったことが可能となります。これは、コードを単に文字列検索するよりも簡単かつ正確です。

コメント的な役割でカスタム属性を用いる例は例えば [`MonoTODOAttribute`](https://github.com/mono/mono/blob/master/mcs/build/common/MonoTODOAttribute.cs) などが挙げられるので、今回の手法は取り立てて新しいものではありませんが、プログラマを光と闇の戦いなどの状況下に置くことで、使命感を持ってプログラムの自己言及性を高めることができ、さらにはコミュニケーションを円滑化する効果も持っていると考えられます。

## まとめ

エレメンタル プログラミングを用いると、コードに超自然的な力を与えることができます。これは、目に見えない加護やステータス補正があるだけでなく、実際のプログラミングにおいても利点をもたらし得る可能性があります。

皆さんもエレメンタル プログラミングを試してみて、無属性のコードでは味わえないプログラミング体験を実感してみてはいかがでしょうか。

<blockquote class="twitter-tweet" lang="en"><p>エレメンタル プログラミングが提唱された。今後の展開に期待したい</p>&mdash; ぐらばく (@Grabacr07) <a href="https://twitter.com/Grabacr07/status/571306514507366400">February 27, 2015</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>
