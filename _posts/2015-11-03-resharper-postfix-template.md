layout: post
title: "ReSharper の Postfix Templates は神機能"
date: 2015-11-03 22:15:00
comments: true
categories:
- programming
tags:
- ReSharper
- Visual Studio
- CSharp
---

[ReSharper 10](https://www.jetbrains.com/resharper/) の RTM がリリースされました！色々と新機能や拡充された機能があるわけですが、今回紹介したいのはこれ！ **Postfix Templates** です。

この機能、ReSharper 9 までは拡張 (extension) として提供されていたのですが、10 になって本体に統合されました。これを機に是非ひとつ試してみては如何でしょうか？

<!-- more -->

## Postfix Templates とは

`.` を入力した直後の IntelliSense 表示に入力候補が追加され、それを選択することで様々な便利テンプレートが展開される…という、実に小気味良い機能です。

要は、こんな感じでコードが書けるようになります。

{% asset_img demo.gif Demo of Postfix Templates %}

これは…便利そうですよね！ <small>(※ デモ内の`cw` は標準のテンプレートです)</small>

個人的には、

* ファクトリ メソッドがあるかと思って型名の直後に `.` を打って不発だった時の絶望感から救ってくれる `.new`
* いちいち部分式の頭に戻って `await ` と入力する (そして必要に応じて括弧も足す必要がある！) 絶望感から救ってくれる `.await`
* 型変換するときに式を行ったり来たりして括弧の面倒をみてやらないといけない絶望感から救ってくれる `.cast`

は特におすすめしたいテンプレートです。

また、上のデモにも映っていますが、通常のテンプレート同様、識別子の部分などの入力が必要な箇所にジャンプできますし、括弧の追加などの際には、必要に応じて適用範囲を提案し、選択することができます。

利用可能なテンプレートは以下の 28 種類です。

{% asset_img list.png List of Postfix Templates %}

また、これとは別に、`Length` プロパティがあるところで `.Count` と入力したり、あるいはその逆の場合に、正しいプロパティ名に置き換えてくれる機能も付いています。下らない機能な感じもしますが、地味に便利です。

## ちょっと残念なところ

文脈にかかわらず、それなりの数の入力候補が出てきてしまうので、人によっては、また、テンプレートによっては誤選択が頻繁に発生することがあるかもしれません。その場合は、自分のスタイルに合わせて、不要なテンプレートを[設定](https://www.jetbrains.com/resharper/help/Reference__Options__Environment__Postfix_Templates.html)から無効にしてしまいましょう。

また、`.par` や `.sel`、`.cast` はどうも入力候補一覧には出てこないので、これは手で入力して `Tab` キーで展開してやる必要があるのですが、まあ、いちいち候補に出てきて鬱陶しくなるのを回避するためと好意的に解釈できなくもないかと…普通にバグなのかもしれませんが。

## まとめ

かねてから Postfix Templates は ReSharper で最も入れるべきプラグインだと主張してきたわけですが、その素晴らしさゆえに遂に標準搭載となったわけであります。

本当に便利な機能なので、ぜひ ReSharper 10 を導入して (あるいは ReSharper 8.x / 9.x の場合は[拡張をインストール](https://resharper-plugins.jetbrains.com/packages/ReSharper.Postfix.R90/)して) 素晴らしいコーディング体験をぜひ自分のものとしてみてください。
