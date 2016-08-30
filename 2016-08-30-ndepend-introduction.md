---
layout: post
title: "NDepend: LINQ for お手軽コード解析"
date: 2016-08-29 15:00
comments: true
categories:
- programming
tags:
- NDepend
- Code Analysis
---

最初は小さいコードでも、日々進捗を重ねることで大きなものとなる。そしてある日、積み上がった成果の大きさに気付き、達成感に思わず笑顔になってしまうものですね。

成果が積み重なるのは喜ばしいことですが、同時に複雑さも折り重なり、問題も山積、問題に気付けないことにも同時に気付く、そういう未来も同じくらい存在します。長期的に育ってゆくコードベースの健康を維持したり、問題の芽を摘むための方策のひとつとして、**静的解析**が挙げられます。

.NET 環境向けにも様々なツールが存在するわけですが、今回、その中のひとつである [**NDepend**](http://www.ndepend.com/) のライセンスを開発元より頂戴しまして、実際に使ってみる機会を得ました。(ありがとうございます！)

日頃この手の解析ツールをしっかりと使ってきた方ではないのですが、せっかくの機会ですし、自分なりに遊んでみて、長短織り交ぜてレビューしてみようと思います。

<!-- more -->

## NDepend 事始め

この手の大規模ツールだとインストーラが付いてくるのを想定してしまったりもしますが、NDepend はダウンロードして任意の場所に展開するだけでインストール完了らしいです。`VisualNDepend.exe` を起動すると、Visual Studio っぽい感じの (というか名前も Visual NDepend で、実際似ているのですが) いい感じの GUI アプリケーションが起動します。

{% asset_img start-page.png Visual NDepend のスタート ページ %}

スタート ページのメニューにある通り、自分で解析対象のアセンブリを選択するか、あるいは Visual Studio のソリューション ファイルを用いて対象を選択することで、解析を始めることができます。今回は、[丁度良く熟成・放置された中規模のライブラリ](https://github.com/takeshik/yacq)があったので例として使ってみることにします。解析が完了すると、ダッシュボードが表示され、以下のような感じの HTML レポートも出力されます。

{% asset_img report.png レポート ファイル %}

始めたばかりの状態では当然過去の履歴も無いので、ダッシュボードのグラフにも線が引かれず、寂しい感じです。コードを編集・リビルドして再度解析を走らせればデータも蓄積されていくのですが、標準の設定だと 1 日 1 つまでのデータしか保存されず、それ以外は捨てられてしまいます。それで問題ない場合も多いのでしょうが、問題ある場合も NDepend プロジェクトの設定の変更 (メニュー バー内の [Project] - [Edit Project Properties] を選択) から挙動をカスタマイズすることができます。

プロジェクトの最初から NDepend の存在を前提としている場合は、ここですぐさまビルド プロセスなりに組み込んでしまえば何の問題もないでしょうが、実際のところ、そう綺麗に物事が始まるばかりでもないのではないかと思います。既存の、既に歴史が蓄積されているコードに対して解析を開始するには、先述した設定の変更によって全ての解析結果をサンプルとして残すようにした上で、データ採取対象となるバージョンを逐一チェックアウトして遡り、ビルド・解析することを繰り返せば、とりあえずは目的を達成できるようです。

解析対象にするバージョンごとにチェックアウト・ビルド・解析を都度繰り返すのはかなり面倒で、何とかして自動化したい感じです…あんまりにも面倒なので、この記事の後ろのほうで自動化に挑戦します。

それはさておき、過去のタグが打たれたバージョン及び最新のコードを解析にかけて、それなりに見栄え良くデータを蓄積した後のダッシュボードは、以下のような感じとなります (料理番組で調理済の鍋が横から出てくる感じで):

{% asset_img dashboard.png ダッシュボード (情報蓄積後) %}

ダッシュボードの表示も賑やかなものとなり、コードの行数やルール違反数の変遷が表示されたりと、なかなか興味深い感じです。ですが、個人的な感覚としては、標準の指標も十分に興味深くはある一方で、やはり個別の事情に応じた独自の指標のほうが、情報の有用性としては高いと思います。月並みな指標の類も有用かつ重要ですが、各々の課題に基づいた指標も色々表示したいと思いますし、それを得るためにコードを色々な切り口で素早く分析する必要性というのも感じるわけです。

## CQLinq: NDepend の力の源泉

…ということを感じるわけですが、NDepend はそういう思いを叶えてくれます。ダッシュボードの表示を構成している様々な指標やルールは、全て **CQLinq** という C# の LINQ を拡張したクエリ言語によって記述されています。また、ダッシュボード上の情報だけでなく、NDepend の様々な機能が CQLinq のクエリとして実現されています。

ハードコードではなく、クエリによって様々な機能が実現されている…ということは、つまり、それらの内容を精査し、自由に編集し、あるいは自分で新しく定義できる、ということです。そして NDepend には CQLinq を書いてその場で実行してくれる、便利なクエリ エディタも用意されています。

{% asset_img cqlinq.png CQLinq のクエリ エディタと実行例 %}

実際のところ、指標やルールの類は、個々コードベースにより特化したものこそ必要とされるであろうことは先述した通りですが、その点、NDepend は 300 を超えるクエリが標準で定義されており、自分でクエリを作る際の出発点も十分用意されているといえます。

例えば、`Expression` 型を継承したクラスは末尾に `Expression` が必ず付いていてほしい…という、標準では明らかに定義されていなさそうな一方で、ごく一部のライブラリにおいては重要な意味を持つであろうルールを定義することを考えます。NDepend では、標準で `Exception class name should be suffixed with 'Exception'` という、ごく一般的なルールが定義されています。

```csharp
// <Name>Exception class name should be suffixed with 'Exception'</Name>
warnif count > 0 from t in Application.Types where
  t.IsExceptionClass &&
  // We use SimpleName, because in case of generic Exception type
  // SimpleName suppresses the generic suffix (like <T>).
 !t.SimpleNameLike(@"Exception$") &&
 !t.SimpleNameLike(@"ExceptionBase$") // Allow the second suffix Base
                                      // for base exception classes.
select t
```

CQLinq の詳細は把握していなくても、これを書き換えれば恐らく簡単に目的が実現できることは想像がつきます。読んだだけで何となく意味は掴めてきますし、それに、入力の際は補完も効きますし、ドキュメントも表示されます (Visual Studio や LINQPad と比較するとやや癖がある動作ですが、書く分にはまあ困らない程度かなという印象です):

{% asset_img cqlinq-completion.gif クエリ エディタの入力補完 %}

というわけで、これを基に、末尾が `Expression` で終わらない `Expression` を継承した型が存在する場合、警告を発生させるルールを作ってみます:

```csharp
// <Name>Expression class name should be suffixed with 'Expression'</Name>
warnif count > 0 from t in Application.Types where
  t.DeriveFrom("System.Linq.Expressions.Expression") &&
  //            ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ ここと…
  !t.SimpleNameLike(@"Expression$")
  //                  ~~~~~~~~~~ ここを書き換えただけ！かんたん！
select t
```

CQLinq 独自の要素である `warnif` は、クエリ式とのフィット感のために、メソッド形式では使えないような印象もを感じますが、実際はそんなこともなく、以下のようにも書けるみたいです:

```csharp
// <Name>Expression class name should be suffixed with 'Expression'</Name>
warnif count > 0 Application.Types
    .Where(t => t.DeriveFrom("System.Linq.Expressions.Expression")
        && !t.SimpleNameLike(@"Expression$")
```

なるほど、簡単ですね！あまり個別の静的解析ツールに詳しくないので、他のツールでも手軽に規則を編集・追加できるのかについては分かりかねるところではありますが、この CQLinq のシステムは、恐らくお手軽さ、簡単さの観点で、かなり上の方に入るのではないかと思います。

また、前述のとおり、ルールだけでなく、ダッシュボード上のグラフにプロットする指標類も同様に CQLinq によって記述されており、ダッシュボード上のメニューから自由に編集・追加できます。ただ、グラフの指標の追加に際しては、値が収集されるのが次回解析以降からのようなので、途中から指標の不足に気付き、追加したくなったような際にはちょっと面倒なことになりそうな感じがあります。

## 差異検出も CQLinq でお手軽に

特にインターフェイスが API として公開されるライブラリの場合は特に顕著ですが、バージョン間のコードの差異は重大な関心事です。(というか、静的解析を走らせようとするコードなんて、大体がそんな感じなのかもしれません)

膨大なメタデータへのアクセスを提供する .NET ですから、もちろん、やろうと思えばリフレクるなり、セシるなり、いくらでもやりようはあります…が、考えただけでげんなりするくらい面倒な作業であろうことは、容易に想像がつきます。

さて、NDepend で解析を行うと、解析結果の情報が .ndar ファイルとして保存され、蓄積されます (アーカイブされる頻度は、前述の通りプロジェクトのプロパティから変更できます)。このファイルは、当該時点での解析結果が全て保存されている (具体的には、[`IAnalysisResult`](https://www.ndepend.com/api/NDepend.API~NDepend.Analysis.IAnalysisResult_members.html) オブジェクトのシリアライズによる) ので、その時点でのデータを基に、ルールの違反状況を取得したりといったクエリを実行したりすることができます。少し分かりにくい場所にありますが、メニュー バーの [Project] - [Load a Previous Analysis Result of the Current Project] から参照するデータを選択することができます。

それだけではなく、2 つの .ndar ファイルを用いて、両者間の解析結果の差異、つまり型やメンバの追加・削除や変更を取得したりすることもできます。これも、メニュー バー内の [Diff] - [Define the Two Snapshots to Diff] - [Define a Baseline for Comparison] を選択することで、比較に用いる対象を選択することができます。

{% asset_img compare-select.png 比較対象を選択するダイアログ %}

比較対象を選択したら、メニュー バーの [Diff] - [Code Diff Summary] から主要な処理が実行できるので、簡単に試してみることができます。

まあ、過去の解析情報を取得してきたり、あるいは二者間の差異を抽出したりという機能は (あまりこの類のツールに明るくないので、実際のところは分かりかねますが) あって当然な機能の範疇だろうとは思われます。ですが、ここでメニュー項目から処理の内容を選択すると、先ほどのルールの際と同様、処理の内容がクエリとして表示され、実行されるのが NDepend らしさ溢れるポイントです。例えば以下は、削除されたメソッドの一覧を取得するクエリです:

```csharp
from m in codeBase.OlderVersion().Application.Methods where
 !m.ParentType.WasRemoved() &&
  m.WasRemoved() &&
 !m.IsGeneratedByCompiler
select new { m, m.NbLinesOfCode }
```

例によって、これも (ほぼ) 普通の C# コードなので、入力補完に頼りながら、自分の欲しい内容に向けてクエリを改変していくことも容易に行えます。

例えば、上のクエリにおいて、`OlderVersion` および `WasRemoved` というメソッドが差異抽出において重要な位置を占めていることが想像できます。さらに、入力補完を頼りに探索することで、様々な型においても `OlderVersion` や `WasRemoved` が提供されており、さらに、それらと対になる `NewerVersion` や `WasAdded` 他、様々な便利メソッドが提供されていることがわかります。

{% asset_img cqlinq-members.png CQLinq のオブジェクト モデルと入力補完、ドキュメント %}

折角なので、私も差異抽出機能を活かしたオリジナルのクエリを書いてみました。

```csharp
codeBase.NewerVersion().Application.Methods
    .Where(x => x.VisibilityWasChanged())
    .Select(x => new
    {
        Method = x,
        Old = x.OlderVersion().Visibility,
        New = x.NewerVersion().Visibility,
    })
```

このクエリを実行すると、アクセシビリティの変更されたメソッドと、変更前・変更後のアクセシビリティが表示されます。

{% asset_img diff-visibility.png 実行結果 %}

先述した通り、このような作業は Reflection API や Mono.Cecil 等でもできなくはないでしょう。ですが、NDepend を使うことで、それよりも遥かに簡単で、読みやすいクエリの形で差異抽出が行えています。

NDepend の解析結果は、単なるメタデータの情報だけでなく、行数やコメント率、循環的複雑度といったコード上の指標、あるいはメンバ間の依存関係や適切なアクセシビリティといった、メタデータから導出できる様々な付加情報も含まれており、分析に適したものとなっています。システムによって適度に咀嚼され、静的解析に適した形に抽象化されたデータ ソースを介して解析を行うのは、メタデータ自体を相手に、気合いで解析するよりも間違いなく生産的といえます。

## 小括

私が上のクエリを仕上げるのに、特段ヘルプ ページを熟読して勉強したわけではありません。IDE 優勢な言語環境において、「入力補完の波に乗るだけでライブラリの使い方は一通り学習できる」というのは、所謂あるある話ではないかと思いますが、NDepend においても、それなりに充実した入力補完と、かなり充実したコード ドキュメントのおかげで、手探りで使い方を学習できてしまいます。

最初からクエリが豊富に用意されているので、平均的な処理はすぐにこなせる。それだけでなく、これら標準のクエリの内容も全て表示・編集できるので、自分でクエリを書いて独自の解析を行う際にもサンプル、あるいは出発点として活用できる。

その学習サイクルが、ドキュメントも表示される補完機能と、書いたその場で実行されるクエリ エディタによって自然に実現できるわけで、非常に良く出来ていると思います。

## その場限りのお手軽解析でも使える

ここまでの例では、継続的に保守されるコードベースに対し、開発プロセスの一環として継続的に解析を実行し、蓄積された情報を基に分析を行うことを暗黙に前提としていましたが、NDepend では、このような (NDepend の) プロジェクト ファイルを作成し、それに対して継続的に解析を実行するような使い方だけでなく、既存の DLL を参照して、それに対して解析・クエリ実行を行うような方法でも十分に活用できます。

一例として、[JSON.NET](http://www.newtonsoft.com/json) のアセンブリを取り上げます。JSON.NET は NuGet のパッケージの中身を見ればわかる通り、様々なターゲット向けのアセンブリが提供されており、興味深くもある一方で複雑さに頭が痛くなります。

NDepend のスタート ページから [Compare 2 versions of a code base] を選択し、アセンブリを選択します。今回は、NuGet のパッケージから持ってきた `net45` と `netstandard1.0` ターゲットのアセンブリを選択して、解析を開始します (ここではそれぞれに単一のアセンブリを指定していますが、複数個の指定も可能なので、かなり柔軟に使えそうです):

{% asset_img diff-select.png diff を取るアセンブリ (or NDepend の解析結果) を選択する %}

メニュー バーの [Diff] - [Code Diff Summary] から [New types] と [Types removed] からクエリを実行した結果は以下の通りです:

{% asset_img diff-instant.png net45 - netstandard1.0 で追加 / 削除された型の一覧 %}

一発で、かつ見やすい形式で変更点が抽出されました。(それにしても、`BindingFlags` を自前で定義していることがわかったり、色々と哀愁漂うものがありますね…)

このように、「プロジェクトを最初に作成して、一定期間ごとに解析を走らせて…」というような、継続的なフローの一部ではない、いわば「単発」の解析においても NDepend は十分に活用できます。「この単語はシステム ライブラリ中でも一般に使われているものなんだろうか…？」というような感じで、自分の命名や型のデザインが一般的なものかどうか悩むのはよく見られる光景なのではないかと思いますが、そのような用途にもうってつけではないかと思います。

## NDepend Power Tools と API

ところで、NDepend をダウンロードすると、Visual NDepend 以外にも様々なツールが同梱されていることに気付きます。

`NDepend.Console.exe` は解析をコマンドラインから行えるようにしてくれるツールだとすぐ理解できます。ビルド プロセスに組み込む際には、これを使えば捗りますね。

そして `NDepend.PowerTools.exe` は、キーを押下するとそれに応じた便利コマンドが実行される、昔懐かしい感じのコマンド ベースのコンソール アプリケーションです。例えば `p) .NET Framework v3.5 and v4.X installed, Core Public API Changes` は普通に便利な感じですね。いっそ Visual NDepend あたりに組み込んでくれればいいのに…

と思ったら、こんな作りになっているのは、この NDepend Power Tools は NDepend API、つまり NDepend をコードから利用するためのライブラリのサンプルとして提供されているからのようです。Power Tools と一緒に入っている `NDepend.PowerTools.SourceCode` がこのツールのソースコードで、中を覗いてみると各コマンドがそれぞれ別れて実装されています。だから探索しやすいようにコマンド ベースな感じになっているんですね。

ということで、NDepend API を使って、何か有用な処理をさせてみましょう。記事の最初のほうで、既存のコードベースに対して (途中から NDepend を導入する等の事由により) 過去のバージョンに対しても遡及的に解析を行う、という処理について、これを自動化することを考えます。

### アーカイブされたデータのタイムスタンプ変更

実現に取り掛かる前に、NDepend における解析結果とメトリクス データ (≒ ダッシュボードのグラフ情報) の保存先について概説します。

NDepend のアーカイブされた解析結果ファイルは、出力ディレクトリ (NDepend のプロジェクト設定で定義され、大抵の場合プロジェクト ファイルの位置にある `NDependOut` ディレクトリです) 内の `/{timestamp:yyyy_MM/dd_HH_mm}/` に保存されます。ここで、`timestamp` は単純に解析が行われた時刻が採用されます。ファイル名にも色々情報が入っていますが、少なくとも現状では無視され、NDepend はディレクトリ名のみによってタイムスタンプを認識するようです。従って、ディレクトリ名を適切に変更するだけで、解析結果のタイムスタンプを希望する日時に変更することができるみたいです。

メトリクス データについては、出力ディレクトリ内の `TrendMetrics/NDependTrendData{time:yyyy}.xml` に追記されてゆく形で保存されるため、アーカイブ データの場合に比べて面倒な感じです。とはいえ、中身は単なる XML ファイルのため、容易に編集が可能です。

```xml
<R D="24/1/2016 2:34:56 AM" L="v1.23" V="..." />
<R D="23/1/2016 1:23:45 AM" L="v1.22" V="..." />
```

のような感じでデータが蓄積されているので、属性 `D` の値を変更することでタイムスタンプを変更できます。

### 事例: Git と組み合わせて遡及的に解析

以上の前提を踏まえて、実際に NDepend API を用いて処理を自動化させてみます。処理にあたっては、Git リポジトリの操作 (タグの取得・チェックアウト) のために [libgit2sharp](https://github.com/libgit2/libgit2sharp/) を参照しています。

NDepend API を利用するには `NDepend.API.dll` を参照すればよいのですが、NDepend のアセンブリが特殊な配置になっている (`Lib` ディレクトリ以下に配置されている) 都合上、アセンブリの解決 (resolving) に手を加えてやる必要があります。

```csharp
AppDomain.CurrentDomain.AssemblyResolve += (sender, e) =>
{
    const string pathToLib = @"C:\path\to\NDepend\Lib";

    var assemblyName = new AssemblyName(e.Name);
    var asmFilePath = Path.Combine(pathToLib, assemblyName.Name + ".dll");
    return File.Exists(asmFilePath) ? Assembly.LoadFrom(asmFilePath) : null;
};
```

上のコードをアプリケーションのエントリポイントに追加することで、NDepend API の実行に必要なアセンブリが正しく解決されます。(NDepend Power Tools のソースコード内 `AssemblyResolver` クラスが参考になります)

準備が整ったら、さらに以下のコードを追加します:

```csharp
var settings = new
{
    ProjectName = "ConsoleApplication1",
    RepositoryRoot = @"C:\path\to\repo",
    BuildOutputDir = @"ConsoleApplication1\bin\Debug",
    NDependProjectPath = @"C:\NDependTest\ConsoleApplication1.ndproj",
    ApplicationAssemblyPattern = "My*.dll",

    MSBuildPath = @"C:\Program Files (x86)\MSBuild\14.0\Bin\amd64\MSBuild.exe",
    NuGetPath = Environment.ExpandEnvironmentVariables(@"%USERPROFILE%\.nuget\packages\NuGet.CommandLine\2.8.5\tools\NuGet.exe"),
};

// NDepend プロジェクトの作成
var projectManager = new NDependServicesProvider().ProjectManager;
var project = projectManager.CreateBlankProject(
    settings.NDependProjectPath.ToAbsoluteFilePath(),
    settings.ProjectName);

using (var repo = new Repository(settings.RepositoryRoot))
{
    foreach (var tag in repo.Tags
        .Cast<Tag>()
        .Select(x => new
        {
            Name = x.FriendlyName,
            CommitId = ((Commit)x.Target).Id.ToString(),
            Timestamp = ((Commit)x.Target).Author.When.LocalDateTime,
        })
        .OrderBy(x => x.Timestamp))
    {
        Console.WriteLine($"Analyzing: {tag.Name} ({tag.CommitId}) @ {tag.Timestamp}");

        Console.WriteLine("    Prepare"); // タグのチェックアウト & clean もどき
        var buildOutDir = Path.Combine(settings.RepositoryRoot, settings.BuildOutputDir);
        if (Directory.Exists(buildOutDir)) Directory.Delete(buildOutDir, true);
        repo.Checkout(tag.CommitId, new CheckoutOptions() { CheckoutModifiers = CheckoutModifiers.Force });

        Console.WriteLine("    Build assemblies"); // NuGet パッケージの復元 & ソリューションのビルド
        var slnPath = new DirectoryInfo(settings.RepositoryRoot).EnumerateFiles("*.sln").First().FullName;
        Process.Start(settings.NuGetPath, $"restore {slnPath}").WaitForExit();
        Process.Start(settings.MSBuildPath, slnPath).WaitForExit(); // 雑実装ではある

        Console.WriteLine("    Analyze"); // NDepend: アセンブリの解析
        // 解析対象のアセンブリを設定
        project.CodeToAnalyze.SetApplicationAssemblies(
            new DirectoryInfo(Path.Combine(settings.RepositoryRoot, settings.BuildOutputDir))
                .EnumerateFiles(settings.ApplicationAssemblyPattern)
                .Select(x => x.FullName.ToAbsoluteFilePath()));
        // 解析 & 結果の取得
        var analysisResult = project.RunAnalysis();
        // 解析結果のタイムスタンプを変更 (ファイルの移動)
        var outDir = project.GetOutputDirectoryAbsolutePath().DirectoryInfo
            .CreateSubdirectory(tag.Timestamp.ToString("yyyy_MM"))
            .CreateSubdirectory(tag.Timestamp.ToString("dd_HH_mm"));
        analysisResult.AnalysisResultRef.AnalysisResultFilePath.FileInfo
            .MoveTo(Path.Combine(outDir.FullName, $"{tag.Name}.ndar"));

        Console.WriteLine("    Log trend metrics"); // NDepend: メトリクス データの追記
        analysisResult.LogTrendMetrics(tag.Timestamp);

        Console.WriteLine("    Completed");
    }
}

projectManager.SaveProject(project);
```

上のコードの `settings` の値を適当に設定して実行すれば、指定したリポジトリ内の全てのタグについて、それが打たれた時点に遡ってビルドおよび解析を行い、正しいタイムスタンプを設定した上で結果を纏めてくれます。結果は保存された NDepend プロジェクトを開くことで閲覧可能です。

Git の処理やファイル操作も含まれているので、やや長めのコードとなってしまっていますが、感覚はそれなりに掴めるのではないかと思います。`IProjectManager` を取得し、そこからプロジェクトの操作を行い、解析およびメトリクス データの追記まで行っています。

結局、解析結果の移動は力業になってしまっていますが、一方で、メトリクス データの追記の際にはタイムスタンプの指定ができたりと、色々と Visual NDepend の画面からは見えてこない面白いポイントも見受けられます。プロジェクトの作成から設定の変更、そして保存まで行えてしまっているので、Visual NDepend というのは NDepend API のフロントエンドであると言えてしまう程度には、この API システムは単なるおまけではなく、きちんと作りこまれたものであるといえるでしょう。

<small>実際には、上のコードでも解析結果やメトリクスに現在時刻でのデータがゴミとして残ってしまうので、手作業で消す必要があったりと問題点は存在します。とはいえ、真面目に対応すると無駄にコードが長くなってしまうので、残念ですが上のコードでは割愛しています。</small>

### Visual Studio に溶け込む NDepend

フロントエンド…といえば、NDepend はスタート ページ内に案内があるように Visual Studio に組み込むことができ、拡張をインストールすることで Visual Studio からシームレスに NDepend の機能を利用することができます。

拡張をインストールすると、Visual NDepend で使っていたクエリ エディタやダッシュボードの表示が完全に Visual Studio に溶け込む形で追加されます。もはや Visual NDepend Studio と言ってしまえるようなものになっており、なかなか面白い感じです。

{% asset_img visual-ndepend-studio.png Visual NDepend を見た後だと謎な感じがすごいが、実際便利そう %}

## 気になる点

このように、NDepend はお手軽さと柔軟さを両立させた便利な解析ツールなわけですが、触っていると色々と気になる点もやはり出てきます。

* 過去のコードに対する遡及的な解析、即ちプロジェクトの中途からの NDepend の導入や、指標を変更しての再解析といった作業がフローとしてあまり想定されていないのが個人的に気になったのは先述の通りです。
    * また、解析結果にアセンブリ バージョンは記録されるとはいえ、複数結果を識別する情報としてはタイムスタンプしか用意されていないので、デイリー レポートを生成するような場合では問題にはならないのでしょうが、それ以外の場合はある種の不便が発生する可能性を感じてしまいます。
    * とはいえ、今後の機能追加に Git のサポートなども予定されているらしいので、これらについては将来に期待できそうです。
* 入力補完は完璧とはいかずとも十分なものではありますが、Visual Studio の IntelliSense 等と比べると若干の癖があり、慣れが必要です。
    * 一例としては、Visual Studio のように `.` を入力したタイミングで入力候補が確定する動作などが再現されるだけで、使い心地はかなり向上するのではないかと感じました。
* CQLinq は C# とほぼ同じように書けるので、習得が容易、かつ柔軟な記述が可能ではあります…が、裏で色々と面倒なことをやっているようで、複雑なクエリを書くと、稀に解釈に失敗してしまう事例が見受けられました。
    * こればかりは仕方ないので、地道にバグ レポートを上げていくしかないでしょう。
    * 将来のバージョンでは CQLinq の表現力がさらに向上するとのことで、これについても今後に期待が持てます。

とはいえ、これらの問題も十分折り合いを付けて便利に使っていける程度という印象ですし、例によって要望を上げたりバグ レポートを書いたりして、より良い未来を楽しみにしてゆけば良いのではないかと思います。

## 総評

一通り NDepend の機能を見てきました…が、まだまだ触れてない機能が多々あります (今後折に触れて紹介していきたいところです) し、上手く乗り回せている自信もあんまりないわけですが、個人的には非常に面白いツールだと思います。

NDepend は CQLinq というクエリ システムの存在が明らかに大きなポイントなわけですが、使って覚えれば覚えるほど便利になっていく、という成果のフィードバックの楽しさ、という LINQ を使い始めた頃のワクワク感というのが非常に感じられると思いますし、豊富なサンプルとしての標準のクエリ セットの存在というのも、学習障壁を非常に緩やかなものにしてくれており、取っ付きやすいものになっていると思います (ちょっとポエムっぽい)。

決して安い製品ではないので、誰にでもオススメ！と言い切るのは少々難しくはありますが、まずは試用してみて、NDepend と CQLinq の力、面白さを体感するだけの価値は間違いなくあると思いますし、今後の更なる機能追加にも期待したいところです。