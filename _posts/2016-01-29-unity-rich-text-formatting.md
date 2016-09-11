---
layout: post
title: "Unity の Debug.Log 着色かんたん化への道"
date: 2016-01-29 04:53
comments: true
categories:
- programming
tags:
- Unity
- CSharp
---

**イェーイ！printf デバッグ最高！！！**

実際、お手軽なのは素晴らしいことですし、安定してる <small>(最近はかなり改善されたとはいえ、Unity で VS のデバッガにアタッチしたりすると稀によく死にますし…)</small> というのも重要なわけで、皆の頼れる味方、[`UnityEngine.Debug`](http://docs.unity3d.com/ja/current/ScriptReference/Debug.html) クラスのロギング系メソッドは最高に便利ですね。しかも、[Rich Text](http://docs.unity3d.com/ja/current/Manual/StyledText.html) 機能を用いることで、

```html
<color=red>こんな</color>感じで<color=blue>色</color>とか<color=green>指定</color>できたり<b>超便利</b>
```

<span style='color:red'>こんな</span>感じで<span style='color:blue'>色</span>とか<span style='color:green'>指定</span>できたり**超便利** です。最高ですね。ただ、ちょっとこれは書きづらい…なんとかしたい…

<!-- more -->

## Rich Text の構築は実際つらい

単にベタなテキストなら `<color=red>こんな感じ</color>` に書くだけでおしまいなので、まあ何の問題もないわけですが、ちょっと複雑になると…

```csharp
Debug.Log("x = <color=yellow>" + x + "</color>, y = <color=yellow>" + y + "</color>");
```

うわっ、悲惨…あるいは、ここは順当にフォーマット文字列使うにしても:

```csharp
Debug.LogFormat("x = <color=yellow>{0}</color>, y = <color=yellow>{1}</color>", x, y);
```

ちょっとこれも割と最悪な感じですね…ここはひとつ、簡単に書ける手段を考えねば…

## 拡張メソッドは便利、最高！

真っ先に思いついた手法は <small>(私の十八番らしい)</small> 拡張メソッドです。

```csharp
namespace Somewhere.Isolated
{
    public static class ColorfulExtensions
    {
        public static string Red(this object self)
        {
            return "<color=red>" + self + "</color>";
        }
    
        public static string Green(this object self)
        {
            return "<color=green>" + self + "</color>";
        }
    
        public static string Blue(this object self)
        {
            return "<color=blue>" + self + "</color>";
        }
    }
}
```

上のコードでは端折ってますが、色々なメソッドを更に定義することで、他の色や修飾要素もどんどん追加してしまえます。使い方は以下の通りです:

```csharp
// using Somewhere.Isolated;
Debug.LogFormat("How many {0}({1}-{2}){3}", "files".Red(), 0.Green(), 15.Blue(), '?');
// "How many <color=red>files</color>(<color=green>0</color>-<color=blue>15</color>)?"
```

これで無事 How many <span style='color: red'>files</span>(<span style='color: green'>0</span>-<span style='color: blue'>15</span>)? と表示されます。

上に示した例でも明らかですが、拡張メソッドの `this` 引数が `string` ではなく `object` なのは、文字列以外の、例えば `int`、`float`、`Vector2` などの多様な型からも直接利用させたいからですね (`vec.ToString().Red()` とか面倒だし！)。

…という意図があるにしても、あらゆる型を対象に取れる、範囲が広すぎて邪悪な拡張メソッドなわけで、なので名前空間を隔離して必要なコード以外からは見えないようにする、というわけです。

### …が、ReSharper が！

{% asset_img resharper-autousing.png ReSharper の IntelliSense が気を利かせすぎてつらい %}

ReSharper を導入していると、この IntelliSense 拡張が気を利かせすぎて…なんということでしょう、`using` していない名前空間の拡張メソッドまで全て検索して、候補にちゃっかり追加してしまうのです。もちろん、これを選択すれば一緒に `using` ディレクティブも追加してくれる優れものです…せっかく奥深くに隔離したのに！

他の人に発見されて、「なんて邪悪な拡張メソッドなんだ！許すまじ！！」と糾弾されること間違いなしです。

**[というか](https://twitter.com/Grabacr07/status/586467636948508673)、[されました](https://twitter.com/Grabacr07/status/586469215097331712)**。

## 安全な手法を求めて: IFormattable

このような苦難を経て、邪悪な拡張メソッドに頼らずとも簡単にログに着色する手法を求め、今回思いついたのが `IFormattable` を用いた手法です。

[`IFormattable`](https://msdn.microsoft.com/ja-jp/library/system.iformattable.aspx) インターフェイスは `ToString()` メソッドのオーバーロードや、あるいはフォーマット文字列内のプレースホルダにおいて書式指定文字列 (と[カルチャ情報](https://msdn.microsoft.com/ja-jp/library/system.globalization.cultureinfo.aspx)) を与えることで、出力である文字列の書式を制御できるようになります。以下のような例で、割とお馴染みなのではないでしょうか:

```csharp
var s1 = DateTime.MaxValue.ToString("s"); // "9999-12-31T23:59:59"
var s2 = string.Format("{0:#,###} = 0x{0:x}", 65535); // "65,535 = 0xffff"
var s3 = $"New GUID: {Guid.NewGuid():d}"; // "New GUID: 01716235-8bcb-47d3-b719-d64ff00b5deb"
```

これをログ着色に応用したのが、以下に示す実装です:

```csharp
public partial class Colorful
    : IFormattable
{
    private readonly object _obj;

    public Colorful(object obj)
    {
        this._obj = obj;
    }

    public string ToString(string format, IFormatProvider formatProvider)
    {
        if (string.IsNullOrEmpty(format)) return this._obj.ToString();
        return "<color=" + format + ">" + this._obj + "</color>";
    }
}
```

そして、この `IFormattable` 実装を利用するためのヘルパ メソッドも定義します:

```csharp
partial class Colorful
{
    public static string Format(string format, params object[] args)
    {
        return string.Format(format, Array.ConvertAll(args, x => new Colorful(x)));
    }
}
```

これにより、`string.Format` メソッドと同等のシグネチャを保ちつつ、引数 `args` の要素を全て `Colorful` オブジェクトにラップさせることで、今回実装した書式処理を適用させるようにしています。

<small>割と細かいネタ: [`Array.ConvertAll`](https://msdn.microsoft.com/ja-jp/library/exc45z53.aspx) メソッドは `array.Select(selector).ToArray()` と同等の効果を持ちますが、途中で `IEnumerable<T>` を経由しないため、より高速に処理されます。PCL 環境では利用できないのが残念ではありますが…</small>

このヘルパ メソッドを用いることで、以下のように今回の書式処理を利用できます:

```csharp
Debug.Log(Colorful.Format("How many {0:red}({1:green}-{2:blue}){3}", "files", 0, 15, '?'));
// "How many <color=red>files</color>(<color=green>0</color>-<color=blue>15</color>)?"
```

あるいは、`Colorful.Format` ヘルパ メソッドを用いず、`string.Format` メソッド上で直接利用する場合は以下のようになります:

```csharp
Debug.LogFormat("Hello, {0:blue}.", new Colorful("world"));
// "Hello, <color=blue>world</color>."
```

これで、無駄に適用範囲の広い邪悪な拡張メソッドに頼らずとも、柔軟かつ簡単に Rich Text を組み立てることができました。

### 今回の実装の制限

以上に示した実装は、少なくとも以下の機能が欠けているので、そのままでは実用性に多少の影響があると考えています:

* 既に `IFormattable` を実装したオブジェクトをラップした際に、元の書式指定文字列も指定できるようにする
* 太字 `<b>` やイタリック `<i>` など、その他の修飾要素のサポート

気が向いたら、これらの課題を解決した実装を GitHub か何処かに公開しようと思います。

## 結び

Unity のコンソール自体を見やすくするエディタ拡張もいくつかあるようですが、それにしても大量に流れるログの中から必要な情報を素早く見つけるためには、文字の着色など様々な強調が必要なのは間違いないでしょう。素早いデバッグには、情報を素早く、かつ柔軟に強調する仕組みが求められるといえます。

拡張メソッドによる手法も、入力補完の正確さや、`Debug.LogFormat` のような `string.Format` をラップしたメソッドとの親和性の高さが魅力ではありますが、書式指定文字列による手法は `this` 引数を `object` 型にするといった邪悪な手法に頼らずとも、自然に任意の型を対象とすることができますし、名前空間を汚染することもありません。結構アリな策なのではないでしょうか。

### ライセンス

本稿中のライブラリ コードについては、[zlib License](https://opensource.org/licenses/Zlib) を適用するものとします。
