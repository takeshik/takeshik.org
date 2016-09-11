---
layout: post
title: "Rx を可視化する ― LINQPad + Rx-Testing + α"
date: 2016-04-30 23:55
comments: true
categories:
- programming
tags:
- CSharp
- LINQPad
- Rx
---

Rx、というかリアクティブ プログラミングは時間に密接に関連した概念です。したがって、その挙動を再現可能な形で確認したり、あるいは可視化することは少なくとも簡単なことではないといえます。

Rx においては時間軸と実行処理の抽象化のために[スケジューラ](http://reactivex.io/documentation/scheduler.html)と呼ばれる概念が用意されており、これを選択したり、あるいは独自に実装することで Rx の処理に介入することができます。つまり、これを実装すればオペレータの挙動が綺麗に出力できるかも？…と思ったら、既にそういう用途に使えそうなものがありました。

<!-- more -->

## Materialize だけでは不足

ある `IObservable<T>` で発生する要素 (`OnNext`, `OnError`, `OnCompleted`) は、`Observable.Materialize` メソッドを用いることで解析に適した `Notification<T>` オブジェクトに変換でき、`OnNext` を通してこれを購読できます。実際に、以下の例を LINQPad で動作させてみます:

```csharp
var scheduler = Scheduler.Default;
var t = scheduler.Now;
Observable.Interval(TimeSpan.FromSeconds(0.5), scheduler)
    .Take(3)
    .Materialize()
    .Select(x => new { T = (scheduler.Now - t), x, })
    .Dump();
```

上のコードは、デフォルト スケジューラの返す時刻情報を基に、配信された要素と配信時の経過時間を出力します。結果は以下のようなものになります:

{% asset_img materialize.gif Materialize を用いた例の結果 %}

得られる情報は、内容としては悪くありませんが、現実の時刻情報を用いているため、配信タイミングには若干の誤差が出てしまいます。それで問題ない場合も多いでしょうが、再現可能な形での確認、あるいは可視化―即ち、ユニット テストやシーケンスの挙動の例示―においては、些か不適当といえるものとなっています。

## TestScheduler

ここで役に立つのが `TestScheduler` (`Microsoft.Reactive.Testing` 名前空間) です。これは [`Rx-Testing` パッケージ](https://www.nuget.org/packages/Rx-Testing/)に含まれるスケジューラで、その名の通り Rx のテストに役立つクラスが含まれています。

…というのは [@zoetro さん](https://twitter.com/zoetro)の [blog 記事](http://zoetrope.hatenablog.jp/entry/20111031/1320077799) で 4 年以上も前に題材に上がっておりまして…この先特に私が述べる必要もないくらいに詳細に紹介されていますので、是非ご一読ください。

…というのも残念なので、私の十八番であるところの LINQPad で、`TestScheduler` を便利に使ってみることにします。

## 便利パーツの定義

まずは、[以前の記事](http://takeshik.org/blog/2016/02/12/linqpad-interactive-rx/)と同じような感じで、`TestScheduler` を操作するためのコントロール パネル的なものを出せるようにします。

```csharp
public static object WithController(this TestScheduler sched)
{
    var dc = sched.ToDumpContainer();
    return Util.VerticalRun(
        Util.HorizontalRun(true,
            new Hyperlinq(() => dc.Refresh(x => x.AdvanceBy(TimeSpan.FromMilliseconds(100).Ticks)), "+0.1s"),
            new Hyperlinq(() => dc.Refresh(x => x.AdvanceBy(TimeSpan.FromMilliseconds(250).Ticks)), "+0.25s"),
            new Hyperlinq(() => dc.Refresh(x => x.AdvanceBy(TimeSpan.FromMilliseconds(500).Ticks)), "+0.5s"),
            new Hyperlinq(() => dc.Refresh(x => x.AdvanceBy(TimeSpan.FromMilliseconds(1000).Ticks)), "+1.0s"),
            new Hyperlinq(() => dc.Refresh(x => x.AdvanceBy(TimeSpan.FromMilliseconds(10000).Ticks)), "+10s")
        ),
        dc
    );
}

// Sample:
var scheduler = new TestScheduler();
scheduler.WithController().Dump();
```

これは以下のような形で表示されます:

{% asset_img scheduler-controller.gif WithController サンプルの出力 %}

`Clock` プロパティの値 (単位: tick = 100 ns) がそれぞれのリンクの値に応じて増加してゆくのが確認できます。

また、解説が前後しますが、上のコードでは `DumpContainer` を拡張したクラス `TypedDumpContainer<T>` を利用しています。

```csharp
public class TypedDumpContainer<T> : DumpContainer
{
    public new T Content
    {
        get { return (T) base.Content; }
        set { base.Content = value; }
    }

    public TypedDumpContainer() { }
    public TypedDumpContainer(T initialContent) : base(initialContent) { }

    public void Refresh(Action<T> action)
    {
        action(this.Content);
        this.Refresh();
    }
}

public static TypedDumpContainer<T> ToDumpContainer<T>(this T self)
    => new TypedDumpContainer<T>(self);
```

`DumpContainer` はこの blog でも何回か取り上げている LINQPad の便利クラスの一つで、これを `Dump` すると `Content` プロパティの値を変更を反映しつつ表示してくれる、一種のプレースホルダ的な機能です。なのですが、この `Content` プロパティは `object` 型なので、中の値を操作するにあたって不便です。上掲のクラスはそれを改善したものとなります。

最後に、リンクをクリックすることでオブジェクトの出力表示をトグルできる `Popup` 拡張メソッドを定義します。

```csharp
public static object Popup(this object obj, string text)
{
    var visible = false;
    var dc = new DumpContainer(obj);
    dc.Style = "display: none";
    return Util.VerticalRun(
        new Hyperlinq(() => dc.Style = (visible = !visible) ? "" : "display: none", text),
        dc);
}

// Sample:
new { Foo = "Hello,", Bar = typeof(Hyperlinq), Baz = "world!", }
    .Popup("Click Me!")
    .Dump();
```

これは以下のように表示されます:

{% asset_img popup.gif Popup() サンプルの出力 %}

長くなりましたが、便利パーツの用意は以上です。

## 出力の準備

次に、具体的な出力の型を用意します。`RecordEntry` 型は `Notification<T>` (およびそれを内包する `Recorded<T>` 型) が LINQPad 上では表示が冗長なため、リスト内で表示されると見辛い問題を解決し、さらにタグ付けにより、要素の出自を簡単に知れるようにしたクラスです。

```csharp
public class RecordEntry : IEquatable<RecordEntry>
{
    public long Time { get; private set; }
    public NotificationKind Kind { get; private set; }
    public object Value { get; private set; }
    public bool HasValue { get; private set; }
    public Exception Exception { get; private set; }
    public string Tag { get; private set; }

    public static RecordEntry Create<T>(long time, Notification<T> notification, string tag = null)
        => new RecordEntry()
        {
            Time = time,
            Kind = notification.Kind,
            Value = notification.HasValue ? (object) notification.Value : null,
            HasValue = notification.HasValue,
            Exception = notification.Exception,
            Tag = tag,
        };

    public bool Equals(RecordEntry other)
        => this.Time == other.Time
            && this.Kind == other.Kind
            && this.Value.Equals(other.Value)
            && this.Exception == other.Exception
            && this.Tag == other.Tag;
}
```

内容的には大したことない (雑な実装の) 代数的データ型です。前述した通り、

* `Notification<T>` の要素
    * `Kind`: 要素の種別 (`OnNext` / `OnError` / `OnCompleted`)
    * `Value`: 要素の内容
    * `HasValue`: `Value` の値が意味を持つか
    * `Exception`: `OnError` における例外オブジェクト
* Recorded<T> の要素
    * `Time`: 要素が配信された相対時刻
* 独自の要素:
    * `Tag`: タグ付けのためのユーザ定義文字列

によって構成されています。これを任意の `IObservable<T>` から構築するための拡張メソッドは、以下の通りです:

```
public static IObservable<RecordEntry> Record<TSource>(
    this IObservable<TSource> source,
    IScheduler scheduler,
    string tag = null)
    => source
        .Materialize()
        .Select(x => RecordEntry.Create(scheduler.Now.Ticks, x, tag));
```

最後に、この `RecordEntry` 型を最終的な表示に整形するための拡張メソッドを定義します:

```csharp
public static IEnumerable<object> Visualize(this IEnumerable<RecordEntry> records)
    => records.GroupBy(x => x.Time, (k, xs) => new
    {
        Time = k,
        Events = Util.HorizontalRun(true, xs
            .OrderBy(x => x.Kind)
            .ThenBy(x => x.Tag)
            .Select(x => new { x.Kind, x.Value, }
                .Popup(x.Kind == NotificationKind.OnNext ? x.Tag : $"{x.Kind}({x.Tag})"))
        ),
    });
```

## LINQPad + TestScheduler + α = Rx Visualization

では、実際に試してみましょう:

```csharp
// 他のシーケンス止める用
var stopper = new Subject<Unit>();
new Hyperlinq(() => stopper.OnNext(Unit.Default), "stop!").Dump();

// スケジューラ
var scheduler = new TestScheduler();
scheduler.WithController().Dump();

// シーケンスの用意
var xs = Observable.Interval(TimeSpan.FromSeconds(0.3), scheduler)
    .TakeUntil(stopper)
    .Record(scheduler, "X");
var ys = Observable.Interval(TimeSpan.FromSeconds(0.4), scheduler)
    .TakeUntil(stopper)
    .Record(scheduler, "Y");
var zs = Observable.Interval(TimeSpan.FromSeconds(0.5), scheduler)
    .TakeUntil(stopper)
    .Record(scheduler, "Z");
var xyzs = Observable.Merge(xs, ys, zs);

// 表示
var dc = new DumpContainer();
Util.HorizontalRun(false, xyzs, dc).Dump(); // 水平配置用
xyzs.ToArray().Subscribe(recs => dc.Content = recs.Visualize());
```

実行すると、以下のようになります:

{% asset_img visualize.gif Visualize サンプルの出力 %}


## まとめ

このように、再現可能な形で `IObservable<T>` シーケンスの動きを可視化することができました。可視化に際してはユーザのペースで時間を進めることができ、また、複数のシーケンスが介在していても簡単に区別を付けられるため、認識の容易さという観点でも優れているのではないでしょうか (とはいえ、改善の余地はまだまだ残されていますが)。

今回はコード量がやや多かったですが、LINQPad でもコードを蓄積させればここまでできる…という、単なるちょっとしたコードの実行環境に留まらない、一種の総合芸術性を感じて頂けたら幸いです。

## ダウンロード

本稿で紹介したコードを含んだ LINQPad クエリ ファイルは [GitHub で公開しています](https://github.com/takeshik/linqpad-queries)。

* [Rx-Visualization.linq](https://github.com/takeshik/linqpad-queries/raw/master/rx/Rx-Visualization.linq) <small>(右クリック メニュー等から保存してください)</small>
