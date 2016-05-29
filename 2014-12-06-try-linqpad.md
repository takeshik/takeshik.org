---
layout: post
title: "今日からはじめる LINQPad"
date: 2014-12-06 22:00
comments: true
categories:
- programming
tags:
- CSharp
- LINQPad
- Advent Calendar
---

この記事は [C# Advent Calendar 2014](http://qiita.com/advent-calendar/2014/csharp) の 6 日目の記事です。LINQPad の基礎的な機能について解説します。

<!-- more -->

**[LINQPad](http://www.linqpad.net/)** をご存知の方はもしかしたら多いのかもしれず、あるいは少ないのかもしれませんが、このツール、LINQPad という名前ではありますが **LINQ とはあまり関係ありません。**名前で結構損をしてるのではないかと常々思ってしまうので、ちょっと残念な感じではあります。

もちろん LINQ のお勉強にも便利なのは間違いないのですが、お手軽な C# / F# / VB コードの実行環境として、また、.NET オブジェクトを見やすく出力することのできるツールとして、十分に便利なツールです。Visual Studio の C# Interactive もしばらく出てこなさそうですし、ちょっとしたコード パッドとしてだけでも十分有用ではないかと思います。

ですが、LINQPad には単にコードを実行し、それなりの出力を得る以外にも、様々な便利機能が多く存在します。使いこなさずにいるには勿体ない。一方で、LINQPad のドキュメントはあまり纏まっておらず (書いてあっても分散してたり)、それらの便利な機能になかなか気付きにくいというのもまた事実。

そこで、この記事では LINQPad の導入、簡単な使い方から、特に便利な機能について掻い摘んで紹介したいと思います。割と幅広く取り上げるので、ご存知の内容も多いかもしれませんがご容赦ください。

なお、本稿では一部 [Pro または Premium Edition の機能](http://www.linqpad.net/Premium.aspx)についても言及していますが、本稿中のコードについてはエディションにかかわらず全て実行可能なはずです。

## ダウンロード

まずは LINQPad をダウンロードしましょう…と [公式サイト](http://www.linqpad.net/) にアクセスして、すぐに目につく右上の [Download now] をクリックしてしまいそうになるわけですが…

だがちょっと待ってほしい。LINQpad には [β 版の提供](http://www.linqpad.net/Beta.aspx)があり、新しい機能をすぐに試すことができます。安定性は実際のところどちらも変わらない感じなので、積極的に使っていきたいところです。

また、標準では x86 ビルドが提供されているところ、AnyCPU 版のバイナリも用意されているので、サーバ コードのテストをする等の際にはこちらの利用を選択するのも手かと思います。

## コードの実行

テキスト エディタ部分にコードを貼り付けて F5 で実行、出力したい内容に `.Dump()` 拡張メソッドを呼ぶ。それだけ。大変わかりやすいですが、注意すべき点として…

### Language の指定

* Expression にすると式のみ (`( ... )`) 記述できる (式以外のものは書けない。文末の `;` は含めない)
* Statement(s) にすると複文のみ (`void Main() { ... }`) 記述できる (メソッドの定義は不可)
* Program にするとクラス内部の定義が記述できる (`namespace` や `using` ディレクティブは記述不可、型を定義すると内部型 (nested type) となる)

ただし、Program モードの際に、拡張メソッドを含んだ static クラスを定義すると、実行の都合上、そのクラスは内部クラスではなく通常のクラスとして定義されます。この挙動は以下のコードで確認可能です (C# Program モードで実行してください):

```csharp
void Main()
{
    typeof(Klass).FullName.Dump(); // UserQuery+Klass
    typeof(Klass2).FullName.Dump(); // Klass2
}

public class Klass
{
    public static void Method(object self) { } 
}

public static class Klass2
{
    public static void Method(this object self) { } 
}
```

また、(左下のペインに表示されている) My Extensions クエリに拡張メソッドやその他のメンバ、クラスを定義することで、全てのクエリからその定義を参照することができます。

### (app|web).config を LINQPad でも使いたい

`app.config` や `web.config` などの .config ファイルでの設定を前提としたコードを実行したり、あるいはそのようなライブラリを参照する場合も多々あるかと思います。そのような場合に LINQPad.exe.config に設定を追加してもうまくいきません。

LINQPad.exe と同じパスに `LINQPad.config` ファイルを作成し、ここに設定を記述することで、実行されるコードに設定を反映させることができます。

### コードの部分実行

コードの一部を範囲選択して F5 キーを押すことで、選択した範囲のコードのみが実行されます (SQL Server Management Studio っぽい挙動)。分かってて使う分には便利な機能なのですが、稀によく誤爆してしまうことがあり…焦らずにすむように、頭の片隅にこの機能の存在を留めておきたいところです。

### 実行停止

Shift+F5 キーでコードの実行は停止されますが、static メンバの内容は依然として維持され、非同期実行中の処理も自発的に終了するまで続行されます。

メニューの [Query] / [Cancel All Threads and Reset] (Ctrl-Shift+F5) を選択することでアプリケーション ドメインをアンロードし、全ての非同期処理スレッドを停止させられます。

上述の `LINQPad.config` の内容も当然再読み込みされますし、一種のおまじないとしても有用です。

### ライブラリの参照

F4 キーを押すとクエリのプロパティ ― アセンブリの参照と名前空間のインポートが行えます。基本的な機能は見れば分かる感があるのですが、いくつかポイントだけ:

* Premium Edition では NuGet パッケージを参照できます (いまどきこの手のツールで NuGet 使えないのは痛いですよね！)
* 同じく Premium Edition では、[Save as snippet...] でスニペット ファイルを保存することで、アセンブリの参照と名前空間のインポートを簡単に行えます。
    * 例えば `sample` という名前でスニペットを保存した場合は、テキストエディタで `sample` と入力 (補完一覧に表示されます) した後に Tab キーを押すだけでこれらの設定を一度に適用できます。
* 名前空間のインポート設定では、[Pick from assemblies] からアセンブリごとに名前空間を一覧表示し、ここから選択して追加できます。また、[Cleanup...] で (参照を外して) もはや存在しない名前空間を一度に取り除くこともできます。

### 出力ペインの機能

コードを実行すると下のほうに出てくる出力ペイン。左側の [▼] をクリックすると様々な設定が可能です。特に便利なのは:

* [Full Screen Results] (F8) で出力ペインを切り離せます。複数ディスプレイでの作業向け。
* [Arrange Panel Vertically] (Ctrl+F8) で出力ペインの表示を垂直分割状態に変更できます。横長のワイド ディスプレイでの作業に最適。

また、[Export] から出力の内容を Word、Excel、または HTML として出力できます。Word での出力にはあまり有用性が感じられませんが、Excel 出力は CSV を作るよりもお手軽な実行結果の受け渡し手法です。

さらに見落としがちな機能として、ウィンドウの右下にある [/o-] / [/o+] ボタンをクリックしてトグルさせることで、コードの最適化フラグを変更させることができます。ちょっとした機能ではありますが、地味に便利です。

### internal メンバへのアクセス

(端的に言ってしまえば) LINQPad は `InternalsVisibleToAttribute` をサポートします。

1. [Edit] / [Preferences] 内 [Advanced] の [Allow LINQPad to access internal types…] を True にする
2. 参照するアセンブリに以下の定義を追加する: `[assembly: InternalsVisibleTo("LINQPadQuery")]`
3. LINQPad から当該アセンブリを参照する
4. public メンバに加えて internal メンバにアクセスできるようになる！(補完一覧にも出てくる！)

LINQPad で実行するコード以外には変化がないので、LINQPad を用いたテストの際にはとても有用な機能かと思います。

## コードの編集

もはや言うまでもない重大な問題…入力補完は LINQPad Pro または Premium 限定の機能です。つまり有料なのですが…こればかりはぜひ買って試してみて！としか言いようがありません。

あくまで IntelliSense "もどき" なので、ちょっと複雑なコードになると混乱されてしまうこともありますが、かといってそこまで普段使いに困るほど馬鹿というわけでもないです。

感覚としては、Visual Studio では IntelliSense で出てこない・怒られる == 自分が間違っている が必ず成立する感じですが、LINQPad の場合は数回悩んでみてダメならとりあえず F5 で実行を試してみたら？という程度です。その場合でも、メソッド チェインを短くしてみたり、少し型の注釈を足してやったりするだけでなんとかなってしまうくらいです。

少し入力補完の賢さを dis ってみましたが、LINQPad には Visual Studio にも ReSharper にも無い便利な機能があります。

補完リストが表示されているときに Ctrl+H を押すと、なんと一覧から拡張メソッドを消してくれます！

たとえば `IEnumerable<T>` の補完で拡張メソッドが大量に出てきて、型本来のメンバ一覧を見たいときとか、あるいは `Enumerable.` と入力したはいいけど拡張メソッドだらけで、本来の (ただの static メソッドであるところの) 生成系のメソッドだけ一覧したい、というときに大いに活躍します。

## .Dump() メソッド

LINQPad といえば `Dump` メソッド。これは以下のような定義です:

```csharp
public static T Dump<T>(this T obj /* , ... */)
{
    // ...
    return obj;
}
```

したがって、式中の任意の位置に `.Dump()` を挿入して、出力を得ることができます。

複雑なオブジェクトは表として出力されますが、階層が深い場合は一部が折り畳まれたり、リンクとして表示されることで隠されるので、クリックすることで表示可能です。また、表のヘッダの右側の ▶ をクリックすることで、票をグリッド ビューとして表示することもできます。

`IEnumerable` や `IEnumerable<T>` を `.Dump()` するとリストが出てくるのは当然な感がありますが、以下の型のオブジェクトについても `.Dump()` すると特別な挙動を示します:

* `Task` / `Task<T>`
* Dump されると `awaiting...` を表示し、実行が完了した段階で結果の値に置き換わります
* `Lazy<T>` - 表示されるリンクをクリックするまで実行が遅延されます
* `IObservable<T>` - 結果を受け取るたび、リストが伸びます。リストの枠の色で完了したかどうか判断できます
* `XDocument` / `XElement` - XML ビューで表示されます
* `Control` (Windows Forms), `UIElement` (WPF) - UI 要素の内容が出力に表示されます
* 等

特に `IEnumerable<Task<T>>` や `IObservable<T>` の出力は非常に面白いものなので、皆様もぜひ一度お試しいただきたいところです。

また、`Dump()` メソッドには様々なオーバーロードがあり、出力を細かく制御できるので、こちらも色々試してみては如何でしょうか。

## Util クラス

LINQPad 上のコードでは `.Dump` メソッドと同じように `Util` クラスが参照されており、コード内で自由に呼び出すことができるのですが、このクラス、あまり丁寧に解説が書かれてません。…ですが、**このクラスのヘルパ メソッドはどれも大変便利です。これを使わないわけにはいかない！**

LINQPad の入力補完から、あるいは ILSpy みたいなツールでメソッド一覧を眺めるなりしてしまえばおしまいなのですが、特に便利なメソッドについていくつか紹介したいと思います。

### Util.Cache

魔法のようなメソッドです。なんと、コードの実行を跨いで値を保持しておけるのです！以下のように使います:

```csharp
var dic = Util.Cache(() => new Dictionary<string, string>());
// dic に対していろいろ操作
dic["foo"] = "hoge";
// …
// で、表示
dic.Dump();
```

これを実行するとディクショナリの内容が表示されるわけですが…ここで `var dic = ...` の下の方を書きなおして…

```csharp
var dic = Util.Cache(() => new Dictionary<string, string>());
// dic に対していろいろ操作
dic["bar"] = "fuga";
// …
// で、表示
dic.Dump();
```

と書き直し、再度実行してみると、なんとディクショナリの中身は `"foo"` と `"bar"` 2 つのキーが存在しているのです！

C# Program モードにして static フィールドなりに値を保存しておけば同じことができるじゃないかと思いますが、こちらの場合はコードを変更して再実行すると以前の環境が破棄されてしまうため、こうもうまくはいきません。

これを使えば、LINQPad を REPL 的に使うことも簡単にできます。

キャッシュしたデータを飛ばして最初からやり直したいときは、前述した [Query] / [Cancel All Threads and Reset] を選択すれば OK です。

### Util.GetPassword / SetPassword

このメソッドを使うと、パスワード (等の固有情報) をコード内にベタ書きすることなく指定することができます。

まず、パスワードの設定です。

```csharp
var key = "user";
var password = "pass";
Util.SetPassword(key, password);
```

のように `Util.SetPassword` を呼ぶか、メニューの [File] / [Password Manager] からパスワードを管理できるので、そこから設定します。設定したパスワードは `%LOCALAPPDATA%\LINQPad\Passwords\` に暗号化されて保存されます。

このようにして設定されたパスワードは `Util.GetPassword(key)` で取り出すことができます。あるいは、`Util.SetPassword` で存在しないキーを指定するとパスワードを尋ねるダイアログが表示され、そこでパスワードの保存を行うこともできるので、あえてパスワードを事前設定しないという手もあります。

### Util.Cmd

```csharp
string[] output = Util.Cmd("dir C:");
```

などと書くと、`dir` コマンドの出力が行の配列として受け取ることができます。真面目にやろうとすると `ProcessStartInfo` を組み立てたりと微妙に面倒だったりするので、このお手軽さは良いのではないでしょうか。

既定では (返り値とは別に) 出力の内容が吐き出されます。第 2 引数 `quiet` (オーバーロード) に `true` を渡してやることで抑止できます。また、実行したプロセスが終了コードとして 0 以外を返した場合は、必ず `LINQPad.CommandExecutionException` が送出されるので注意が必要です。

### DumpContainer

`Util` クラスではありませんが、DumpContainer というクラスも同じように定義されており、こちらも組み合わせると便利にデータを出力できます。

```csharp
var dc = new DumpContainer().Dump();
while (true)
{
    var now = DateTime.Now;
    dc.Content = new
    {
        Y = now.Year,
        M = now.Month,
        D = now.Day,
        h = now.Hour,
        m = now.Minute,
        s = now.Second,
        ms = now.Millisecond,
    };
    DateTime.Now.ToString("yyyy/MM/dd hh:mm:ss").Dump();
    await Task.Delay(250);
}
```

このコードを実行すると、現在時刻の文字列が 1/4 秒ごとに流れるのとは別に、現在時刻を格納した匿名型の出力である表の内容も (追加して出力されるのではなく) 更新され続けます。

このように `DumpContainer` を使うと動的に更新される内容を出力することができます。使いどころも例のような単純なものだけでなく、多々あるのではないかと思います。

### Hyperlinq

またしても `Util` 以外のクラスですが、`Hyperlinq` (これでハイパーリンクと読ませたいんでしょうね…) は、"クリックすると処理を実行するリンク" を出力できます。

```csharp
new Hyperlinq(() => DateTime.Now.Dump(), "Click me!").Dump();
```

このコードを実行すると、画面に "Click me!" というリンクが表示され、これをクリックするたびに現在時刻が出力されます。

前述の `DumpContainer` と組み合わせた、やや複雑なサンプルです (サンプルなのでコードは汚い):

```csharp
var src = new DirectoryInfo(".").GetFiles();
var idx = 0;
var dcIndex = new DumpContainer();
var dcData = new DumpContainer();

new Hyperlinq(() =>
{
    if (idx == 0) return;
    --idx;
    dcIndex.Content = "index = " + idx;
    dcData.Content = src[idx];
}, "↑ prev").Dump();

dcIndex.Dump().Content = "index = " + idx;

new Hyperlinq(() =>
{
    if (idx >= src.Length - 1) return;
    ++idx;
    dcIndex.Content = "index = " + idx;
    dcData.Content = src[idx];
}, "↓ next").Dump();

dcData.Dump().Content = src[idx];
```

これを実行すると、変数 `src` で指定した配列の内容を 1 つずつページをめくる感覚で表示します。

### その他のメソッド

他にも、以下のような出力を細かく制御するメソッドが定義されています:

* `Util.Highlight` / `Util.HighlightIf` - (条件を満たした場合のみ) 文字に着色する
* `Util.HorizontalRun` - 複数のオブジェクトを横並びで Dump できるようにする
* `Util.VerticalRun` - 複数のオブジェクトを縦並びで Dump できるようにする
* `Util.Image` - 画像を表示する
* `Util.RawHtml` - HTML をそのまま出力する
* `Util.WithStyle` - 出力に CSS を適用する
* `Util.ReadLine` - ユーザから入力を受け付け、その入力を返す
    * LINQPad 中では、`Console.WriteLine` は `.Dump()` での出力に、`Console.ReadLine` はこの `Util.ReadLine` と同じ処理になります
 
これらのメソッドは呼んだだけでは画面に表示されず、返り値を `.Dump()` しないと実際に出力されません。(忘れがちなので注意！)

## lprun.exe

最後に、LINQPad に付属するツール `lprun.exe` をご紹介します。

lprun を使うと、保存した LINQPad のコード (`.linq` ファイル) をコマンドラインで実行できます。以下のように実行します:

```bat
lprun -format=FORMAT path\to\query.linq
```

とすると、`path\to\query.linq` のコードを実行します。

`FORMAT` は以下のうちのどれか 1 つです:

* text – テキスト + JSON (デフォルト)
* html – 完全な HTML (CSS 等を含む)
* htmlfrag – HTML (`<body>` 内部のみ)
* csv – CSV
* csvi – CSV (Invariant culture で出力)

結果は標準出力に書き込まれ、また、`Console.In` や `Util.ReadLine()` を使えば標準入力を読み込めるので、`lprun` でパイプ処理を構築することも可能です。

テキストが出力されるだけなので、さすがに全ての出力がインタラクティブに扱えるわけではありませんが、例えば `Util.GetPassword` のような表示に関係のないようなものについては同じように利用できますし、表示に大きく関わるようなものでも本来の機能をなるべく残した出力がされます。

`lprun` を使えば、面倒なアセンブリ参照やビルド処理を行うことなく、書いた C# コードをそのまま実行でき、様々な形式でデータの出力も可能なので、ちょっとしたバッチ処理にも十分使えるのではないかと思います。

## まとめ

LINQPad の基礎的な機能について、長くなってしまいましたがざっとまとめてみました。長くなってしまったにもかかわらず、まだまだ紹介していない機能があったりするわけで…

巨大なデータを一発でビジュアライズしたり、あるいはそれに対して何かしらの操作を加えて結果を見てみる、という用途に LINQPad は非常にマッチすると思います。そういう点では、公式にプッシュされている LINQ のお勉強、という用途よりは一種のテスト ベンチ、あるいはバッチ処理システムとしての機能について、より注目できるのではないかなと思います。そして、LINQPad にはそのような使い方をサポートする機能が、あまり表には出てきていませんが、多々用意されています。

また、in-memory な処理にとどまらず、本稿では解説していませんが LINQPad 自身が提供するデータ ソースへの接続機能、また、

* [LINQ to BigQuery - C#による型付きDSLとLINQPadによるDumpと可視化](http://neue.cc/2014/09/24_479.html) - [neue.cc](http://neue.cc)

のように、大きめのライブラリを組み立てたりすることで、より本格的に LINQPad を使い倒す余地も十分にあります。

C# を書いてて LINQPad をまだ使ったことがない方は、ぜひこれを機会に試してみてはいかがでしょうか。
