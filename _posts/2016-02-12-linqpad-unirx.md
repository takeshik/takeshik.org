---
layout: post
title: "UniRx でも LINQPad"
date: 2016-02-12 09:00
comments: true
categories:
- programming
tags:
- CSharp
- LINQPad
- Rx
- Unity
---

**Rx で困ったら、まずは [LINQPad](http://www.linqpad.net/)！**

…とはいっても、世の中には非標準な Rx もあったりします。[UniRx](https://github.com/neuecc/UniRx) は Unity 環境における Rx 移植ですが、そのままでは LINQPad で便利に使えません。そこで、UniRx でも本家と同じように便利に `Dump` したりするためのひと工夫を紹介します。

<!-- more -->

## LINQPad の Rx 向け表示は Rx 不要

LINQPad で `IObservable<T>` を `Dump` すると非常に分かりやすく表示してくれることは[前の記事](http://takeshik.org/blog/2016/02/12/linqpad-interactive-rx/)で紹介しましたが、この表示には Rx のライブラリを参照する必要はありません。

というのも、`IObservable<T>` や `IObserver<T>` は `mscorlib` アセンブリで定義されており、LINQPad 側で表示する機能は Rx のコードに頼らず自前で実装しているためです。 <small>(…と思われるけど、恐らく間違いない)</small>

## UniRx を LINQPad で動かせはするが…

UniRx の標準ライブラリ自体は `UnityEngine.dll` を参照しているため LINQPad 上では満足に動作させることができませんが、[NuGet で公開されているパッケージ](https://www.nuget.org/packages/UniRx/)は Unity に依存しているコードが取り除かれているため、LINQPad からも安全に参照できます。

UniRx には本家 Rx には無いオペレータが (Unity に依存するものを除いたとしても) 多く存在していたり、あるいは内部構造や細部の挙動に差異もあるため、UniRx の動作を LINQPad 上で検証するために本家 Rx を用いるよりは、このサブセットを用いるほうが適切でしょう。

しかし、Unity の参照するフレームワークには `System.IObservable<T>` や `System.IObserver<T>` インターフェイスが存在しないため、UniRx ではこれらを全て自前で定義しています…具体的には、`UniRx.IObservable<T>` であったり、あるいは `UniRx.IObserver<T>` といった具合に。つまり、本家 Rx とはインターフェイス上の互換性が一切ありません。

そのため、UniRx の `IObservable<T>` を `Dump` しても、要素の配信をリアルタイムに表示してくれたりはせず、味気ない表示しかしてくれません。もちろん、`Subscribe` するなどの通常の利用は可能ですが、要素配信のリアルタイムな表示などは一切ありません。

{% asset_img unirx-dump.png 通常のオブジェクト表示なので使いものにならない %}

そのため、本家 Rx のように使うためにはひと工夫が必要となります。

## ラップするだけでよし

なんということはない、`UniRx.IObservable<T>` を `System.IObservable<T>` でラップしてしまえばいい。ただそれだけです。


<!-- C# シンタックス ハイライトがネスト型を解釈できなくて真っ白テキストになってしまうので、仕方なく C++ に… -->

```c++
public static class UniRxBridgeExtensions
{
    private class SystemObservable<T> : System.IObservable<T>
    {
        private readonly UniRx.IObservable<T> _inner;

        public SystemObservable(UniRx.IObservable<T> inner)
        {
            this._inner = inner;
        }

        public IDisposable Subscribe(System.IObserver<T> observer)
            => this._inner.Subscribe(new UniRxObserver<T>(observer));
    }

    private class UniRxObservable<T> : UniRx.IObservable<T>
    {
        private readonly System.IObservable<T> _inner;

        public UniRxObservable(System.IObservable<T> inner)
        {
            this._inner = inner;
        }

        public IDisposable Subscribe(UniRx.IObserver<T> observer)
            => this._inner.Subscribe(new SystemObserver<T>(observer));
    }

    private class SystemObserver<T> : System.IObserver<T>
    {
        private readonly UniRx.IObserver<T> _inner;

        public SystemObserver(UniRx.IObserver<T> inner)
        {
            this._inner = inner;
        }

        public void OnNext(T value)
            => this._inner.OnNext(value);

        public void OnError(Exception error)
            => this._inner.OnError(error);

        public void OnCompleted()
            => this._inner.OnCompleted();
    }

    private class UniRxObserver<T> : UniRx.IObserver<T>
    {
        private readonly System.IObserver<T> _inner;

        public UniRxObserver(System.IObserver<T> inner)
        {
            this._inner = inner;
        }

        public void OnNext(T value)
            => this._inner.OnNext(value);

        public void OnError(Exception error)
            => this._inner.OnError(error);

        public void OnCompleted()
            => this._inner.OnCompleted();
    }

    public static System.IObserver<TSource> AsSystemObserver<TSource>
        (this UniRx.IObserver<TSource> observer)
        => new SystemObserver<TSource>(observer);

    public static UniRx.IObserver<TSource> AsUniRxObserver<TSource>
        (this System.IObserver<TSource> observer)
        => new UniRxObserver<TSource>(observer);

    public static System.IObservable<TSource> AsSystemObservable<TSource>
        (this UniRx.IObservable<TSource> source)
        => new SystemObservable<TSource>(source);

    public static UniRx.IObservable<TSource> AsUniRxObservable<TSource>
        (this System.IObservable<TSource> source)
        => new UniRxObservable<TSource>(source);
}
```

本当にラップしているだけなので、解説するのもアホらしいくらい単純なコードです。

とはいえ、実際にこれを用いれば、本家 Rx のような感覚で UniRx で `Dump` できます。例えば、以下のような感じで:

```csharp
Observable.Interval(TimeSpan.FromSeconds(1))
    .Take(3)
    .DoOnCompleted(() => "completed!".Dump())
    .AsSystemObservable()
    .Dump();
```

`IObservable<T>` と `IObserver<T>` の識別子が衝突してしまうので、名前空間名で修飾してやる必要があるのが面倒ではありますが、とはいえ、これらインターフェイスの名前をコード内で直接記述する機会は限られていますし、日頃ちょっとしたコードを動かす中で問題になることはないのではないかと思います。

## まとめ

**Unity ユーザの方にも知ってほしい…LINQPad は便利！！**

これを言いたいがためにこの記事を書きました。

流石に、普通の .NET 環境と同じくらい便利…と上手くはいきませんが、それでも C# は C#、十分便利だと思います。是非試してみてほしいです。

## ダウンロード

本稿で紹介したコードを含んだ LINQPad クエリ ファイルは [GitHub で公開しています](https://github.com/takeshik/linqpad-queries)。

* [Rx-UniRx-Bridge.linq](https://github.com/takeshik/linqpad-queries/raw/master/rx/Rx-UniRx-Bridge.linq) <small>(右クリック メニュー等から保存してください)</small>
