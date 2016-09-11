---
layout: post
title: "Unity batch build の罠"
date: 2014-09-01 01:30
comments: true
categories:
- programming
tags:
- CSharp
- Unity
- Automation
---

**転職のお知らせ**: 全く隠してないのでもはや筒抜け感がありますが、数週間前から[株式会社グラニ](http://grani.jp/)で働いてます。

で、最近は Unity の闇と日々格闘する毎日を送っているわけなのですが、Unity のビルド プロセス、あるいは batch mode については最低限の情報は纏まっており、とりあえず適用してみるのは簡単かなという一方、それ以上の情報というのは少ない印象があります。で、問題にぶち当たったときに情報が枯渇 (全くゼロというわけでもない) しており絶望、というビルドできないプロセスが完成している状況です。

ということで、自分が苦しんだ Unity (のビルド周り) の闇とその解決策について書いてみようと思い立った次第です。(しばらく同趣旨の内容が続きそうな予感…！)

<!-- more -->

## ビルド ターゲットが変更されない件

ビルドを自動化したい → batch mode でスクリプトを実行すればよい、ということはググれば割と簡単に出てくるので、以下のコードがサラサラと書けます:

```csharp
public class Batch : MonoBehaviour
{
    private static void BuildAndroid()
    {
        EditorUserBuildSettings.SwitchActiveBuildTarget(BuildTarget.Android);
        var msg = BuildPipeline.BuildPlayer(
            EditorBuildSettings.scenes
                .Where(x => x.enabled)
                .Select(x => x.path)
                .ToArray(),
            @"\path\to\bin\a.apk",
            BuildTarget.Android,
            BuildOptions.AllowDebugging | BuildOptions.ConnectWithProfiler | BuildOptions.Development
        );
        if (!string.IsNullOrEmpty(msg))
        {
            Debug.LogError(msg);
            Environment.Exit(123);
        }
    }
}
```

で、

```sh
/path/to/Unity -batchmode -quit -projectPath /path/to/project -executeMethod Batch.BuildAndroid
```

…と実行すれば大抵の場合は .apk が出力されるのです (ここまで前説) が、なぜか C# のコンパイル エラーが発生しビルドが通らない、もちろん editor からはビルドできるのに…という問題にぶち当たることが稀に (良く？) あります。

以下のコードをどこか適当な C# コードに貼り付ければ状況を故意に再現できます:

```csharp
#if UNITY_ANDROID
#warning *** Android! ***
#elif UNITY_IPHONE
#warning *** iOS! ***
#else
#error *** Other Target! ***
#endif
```

で、上のバッチ コードを走らせると、Android をターゲットに設定しているのに `*** Other Target! ***` とエラーが発生して死んでしまいます。

上述のエラー ログの少し上にあるコンパイラへの引数リストを精査してやると分かるのですが、バッチ ビルドではまず最初に選択中のターゲット、そのデフォルト値は PC and Mac Standalone でビルドが走ってしまってるらしく… (事実、上のコードの `#error` を `#warning` に置き換えてビルドしてやると `*** Other Target! ***` の警告の後に `*** Android! ***` の警告が出て正しくビルドされることが確認できます) 

なので、バッチ ビルドする前に editor から [Switch Platform] してやればちゃんとビルドできるのですが、どうも現在選択中のターゲット設定は `$(ProjectRoot)/Libraries` 以下に格納されてる模様で、一般的なプロジェクトではこのディレクトリはバージョン管理から弾いてると思われるので、「ターゲットを Android か iOS にしてプロジェクト保存しといてね！」とお願いしても全く意味がない、という残念な状況です。

なので、解決策としては、`#if` でターゲット プラットフォームごとにコードを分ける際に、どうせ Android や iPhone の実機でテストするからと **Standalone でビルドできないコードを書かない**、という、当然といえば当然なのかどうなのかイマイチぱっとしない答えになってしまいます。

そもそも、`EditorUserBuildSettings.SwitchActiveBuildTarget` メソッドは batch mode では機能しないと[マニュアルに書いてある](http://docs.unity3d.com/ScriptReference/EditorUserBuildSettings.SwitchActiveBuildTarget.html)のですが、そういう大切なことは IntelliSense にも書いておいてほしい…あるいは、動いてないのに正常実行しないでほしい… (つらい)

## ビルドでエラーが起きると Unity がフリーズする

上のコードで

```csharp
if (!string.IsNullOrEmpty(msg))
{
    Debug.LogError(msg);
    Environment.Exit(123);
}
```

というエラー ハンドリングのコードを書いてます。エラーの内容に応じて終了コードを変えて呼び出し側で色々対応したいという意図が感じられる素晴らしいコードなのですが、batch mode では `Environment.Exit` メソッドを呼ぶとフリーズします！危険！

終了コードなどという高度な手法を用いることを Unity は許してくれないので、適当な例外を `throw` して終了コード 1 を得るのが batch mode での最善手のようです。ひどい。

## 結論

Unity の batch mode は闇
