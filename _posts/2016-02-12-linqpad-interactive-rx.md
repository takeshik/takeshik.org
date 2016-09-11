---
layout: post
title: "Interactive LINQPad: お手軽ポチポチ Rx"
date: 2016-02-12 07:35
comments: true
categories:
- programming
tags:
- CSharp
- LINQPad
- Rx
---

[LINQPad](http://www.linqpad.net/) は .NET コードのお手軽な実行環境、そして良質なデータ ビジュアライザとして全ての .NET ユーザの必携ツールといっても過言ではないといえるのではないでしょうか。入力補完などの一部の便利な機能は有料ではありますが、実際のところ、購入する価値は十分にあると思います。

それはさておき、LINQPad はどんなオブジェクトも `.Dump()` すれば見やすく表示してくれるというのが最高に魅力的です。これだけでも LINQPad は十分に便利で使う価値があります。ですが、LINQPad の便利さはそれに留まるものではありません。

…ということ等々を[以前書いたりもしました](http://takeshik.org/blog/2014/12/06/try-linqpad/)が、文章ばかりの記事でして面白みにも少し欠けていました。先の記事は割と総論的な内容だったので、趣向を変えて、面白く膨らませそうな LINQPad の便利な使い方について、折にふれて書いていこうと思います。

さて、LINQPad はその名の通り LINQ の学習に非常に有用ですが、[Reactive Extensions (Rx)](https://github.com/Reactive-Extensions/Rx.NET) の学習でも活躍します。本稿では、更にひと手間加える事で、LINQPad 上で Rx を interactive に利用する方法を紹介します。 


<!-- more -->

## Rx の学習・動作確認にこそ LINQPad

ここ最近では様々な環境で Rx 的なライブラリも普及してきて、.NET においても以前よりかはキワモノ感も薄れているのではないかと思います。[ReactiveX](http://reactivex.io/) などの良質な学習リソースが出てきているのも非常に心強いですが、とはいえ、それでもやはり Rx の学習は難しいと思います。

様々な要因が考えられますが、初見においては特に時間軸の存在―要素配信のタイミングが重要な意味合いを持つ―であったり、あるいは状態遷移が (`IEnumerable<T>` と比べて) 複雑なことも大きいのではないかと私は考えています。また、オペレータの数も非常に多く、その内容も多岐にわたるので、都度調査が必要な程度には学習量は大きいといえます。

そのような Rx の学習、あるいは日々の Rx ライフにおける動作の検証・確認に LINQPad はきっと役に立ちます。

とりあえず動かすだけなら何ということはありません。LINQPad で [F4] キーを押して [Query Properties] ダイアログを開き、そこから `Rx-Main` パッケージを参照し<sup>(※)</sup>、必要に応じて名前空間をインポートし、コードを書いて、いつものように `.Dump()` するだけです。

{% asset_img interval.gif IObservable`1 を Dump している様子 %}

このように、要素の配信がリアルタイムで確認できます。上の例では、実際に 0.5 秒ごとに `long` 値が配信されているのが確認できます。

もちろん、LINQPad の外と同じように `Subscribe` メソッドを用いて要素を購読することも可能ですが、その場合は上のような高度なビジュアライズは行われません。また、`Util.KeepRunning()` メソッドが返す `IDisposable` オブジェクトを保持して、クエリの実行終了を明示的に遅延させる必要があります。 

<small>※: LINQPad における NuGet の組み込みサポートは [Developer Edition 以上のみ利用可能](http://www.linqpad.net/Purchase.aspx)ですが、既に NuGet パッケージを参照しているクエリ ファイル (`*.linq`) を開くことで全ての環境で利用可能です。または、単にパッケージ内の DLL を参照することで代用可能です。</small>

### シーケンスの状態と Dump 表示

さて、上の例において、シーケンス (`IObservable<T>`) を `Dump` することで表示される表の枠線の色が (通常の) 青ではなく緑であることに気付かれたことと思います。

この緑色は、オブジェクトが `IObservable<T>` であることを表しているのですが、より正確には、そのシーケンスがまだ終了していない (`OnError` と `OnCompleted` のいずれもまだ配信されていない) ことを表します。

実際に終端を持つシーケンスを表示させて確認してみましょう。

{% asset_img interval-take.gif OnCompleted で枠が青くなる %}

最初の例に `.Take(3)` を足したので、要素が 3 つ配信された時点で `OnCompleted` が配信され、シーケンスが終了します。上の例でも明らかな通り、`OnCompleted` が配信されると枠の色が通常のオブジェクトと同様の青に変化します。

また、以下の例で示すように、`OnError` では同様に赤く変化します。

{% asset_img interval-take-throw.gif OnError で枠が赤くなる %}

このように、LINQPad を使って `IObservable<T>` を扱うことで、シーケンスの状態と、その変化を視覚的に、かつリアルタイムに確認することができます。

例として、`Observable.Empty` と対比される、特徴的なオペレータ `Observable.Never` の動作を確認してみます。

{% asset_img interval-take-never.gif Observable.Never で OnCompleted しないことがよくわかる %}

LINQPad が実行が終了させていないという点でも明らかですが、前掲の例と見比べることで `Observable.Never` の動作も視覚的に理解できます。[ReactiveX での図による解説](http://reactivex.io/documentation/operators/empty-never-throw.html)も十分に練られており、正確さは抜群だとは思いますが、実際に動作させて、状態がリアルタイムに表示されるこちらの方が、身に付けるものとしての感覚的な理解の手段としては優れているのではないかと考えます。

## ユーザ操作に応じた処理

ここまでの例は全て入力したコードを実行するだけでしたが、LINQPad ではユーザの操作を元にした、つまりインタラクティブな処理を行わせることもできます。この手の処理とよく合う Rx と組み合わせることで、その相乗効果をより高めることができます。

### Hyperlinq

ここで LINQPad の提供する機能のひとつ、`Hyperlinq` を紹介します。これは結果ペインにハイパーリンクを表示させ、これをクリックすることで指定した処理を行わせることができる、というものです。実際の利用例を見たほうが早いと思います:

{% asset_img hyperlinq-guid.gif Hyperlinq の用例: guidgen %}

上の例では、"Generate GUID" リンクをクリックしたタイミングで、引数に渡した `Action` の処理…つまり、`Guid.NewGuid().Dump()` が実行されます。これはクエリの実行が終了しているかどうかに関係なく、また、何度でも実行できます。

<small>これらの動作は、メニュー バー内 [Query] / [Cancel All Threads and Reset] が行われることにより、クエリを含んだアプリケーション ドメインが破棄されるまで有効です。</small>

### Subject + Hyperlinq = ポチポチ Rx

上で `Hyperlinq` を紹介したのは、`Subject<T>` を組み合わせることで「ポチポチ Rx」を実現させるための前フリです。

`Subject<T>` は `IObservable<T>` と `IObserver<T>` が一緒になった…つまり、自身で要素を配信できるようになった `IObservable<T>` シーケンスのようなものなわけですが、ならば、ユーザの入力に応じて実際に値を流せるようにして、それらを組み合わせたシーケンスと一緒に表示すれば、まさに最強の Rx テスト ベンチになるわけです。

善は急げ！早速、LINQPad のクエリ種別を [C# Program] に変更して、以下のコードを入力してください (記事の末尾でクエリ ファイルのダウンロードができます)。

```csharp
public static class ObserverExtensions
{
    public static object Controller<T>(
        this IObserver<T> observer,
        Func<T> nextValueGenerator = null,
        Func<Exception> errorGenerator = null)
    {
        return Util.HorizontalRun(true,
            new Hyperlinq(() => observer.OnNext((nextValueGenerator ?? (() =>
            {
                var t = typeof(T);
                if (t.Name == "Unit" && t.IsValueType)
                {
                    return default(T);
                }
                return Util.ReadLine<T>("Next value:", default(T));
            }))()), "Next"),
            new Hyperlinq(() => observer.OnError((errorGenerator ?? (() =>
                new Exception(Util.ReadLine("Error message:")))
            )()), "Error"),
            new Hyperlinq(() => observer.OnCompleted(), "Completed")
        );
    }

    public static object WithController<T>(
        this IObserver<T> observer,
        Func<T> nextValueGenerator = null,
        Func<Exception> errorGenerator = null)
    {
        return Util.VerticalRun(
            observer.Controller(),
            observer
        );
    }

    public static IObserver<T> DumpWithController<T>(
        this IObserver<T> observer,
        string header = null)
    {
        observer.WithController().Dump(header);
        return observer;
    }
}
```

少し長いですが、3 つあるメソッドのうち枢要なものは最初の `Controller` メソッドだけです。これらのメソッドは `ISubject<T>` をターゲットにしているとはいえ、機能的には `IObserver<T>` の範囲内に収まっているため、対象を広い方に取ってあることを付記しておきます。

`Util.HorizontalRun` メソッドは、その返り値を `Dump` することで、引数で指定したオブジェクトたちを横並びで一度に表示できるという、その名の通りなメソッドです <small>(第 1 引数 `withGaps` は要素間の隙間を開けるかを指定します)</small>。

で、それを使って複数の `Hyperlinq` を横並びに表示しているわけです…それぞれ `OnNext`、`OnError`、そして `OnCompleted` に対応しています。`OnNext` と `OnError` に関しては、メソッドの省略可能引数によって配信する値の生成処理を上書きできるようにしていますが、基本的には `null` を渡してデフォルトの処理のまま使うことを想定しています。

そのデフォルトの処理内で用いている `Util.ReadLine` メソッドは、LINQPad が表示するテキスト ボックスによってユーザ入力を受け付け、その内容を返すという便利な機能です。これを使って、配信する値の内容をユーザに入力してもらいます。

残り 2 つのメソッドは単なるヘルパ メソッドです。`WithController` は返り値を `Dump` することで、コントローラとシーケンスを一緒に表示します (`Util.VerticalRun` メソッドは引数に指定したオブジェクトを縦並びで一度に表示します)。`DumpWithController` は `.WithController().Dump()` 相当です。

以上が機能を実現するための準備となるライブラリ コードです。最後に、以下のメソッドを記述して実際に実行させてみましょう。

```csharp
void Main()
{
    var xs = new Subject<int>();
    var ys = new Subject<int>();

    Util.HorizontalRun("xs, ys", xs.WithController(), ys.WithController())
        .Dump();

    Util.HorizontalRun("xs.Zip(ys), ys.CombineLatest(ys)",
        xs.Zip(ys, (x, y) => new { x, y, }),
        xs.CombineLatest(ys, (x, y) => new { x, y, })
    ).Dump();
}
```

以下は実行例です。

{% asset_img subjects.gif Subject Controller の用例 (1) %}

このように、ユーザの操作を介在させることで `Zip` と `CombineLatest` の違いを分かりやすく図示できました。

もちろん、エラー時の挙動も簡単に確認できます。

{% asset_img subjects-error.gif Subject Controller の用例 (2) %}

## まとめ

LINQPad はその性質により、Rx の動作の学習や調査に非常に適しているといえます。また、LINQPad の更なる持ち味である、ユーザの操作に応じた処理の実現は Rx との相性も良く、コードを多少記述するだけでこれらを組み合わせ、より良い実験環境を構築できます。

Rx の疑問点が出てくることは多々あると思いますが…**困ったら、まずは LINQPad！**

是非、活用してみてください。

## ダウンロード

本稿の最後で紹介したコードを含んだ LINQPad クエリ ファイルは [GitHub で公開しています](https://github.com/takeshik/linqpad-queries)。

* [Rx-Subject-Controller.linq](https://github.com/takeshik/linqpad-queries/raw/master/rx/Rx-Subject-Controller.linq) <small>(右クリック メニュー等から保存してください)</small>
