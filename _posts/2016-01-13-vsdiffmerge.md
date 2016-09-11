---
layout: post
title: "Visual Studio を diff / merge ツールとして使う"
date: 2016-01-13 03:20
comments: true
categories:
- programming
tags:
- Visual Studio
- Git
---

Git などのバージョン管理システムを用いるにあたり重要な機能はマージであり、避けられないのはコンフリクトです。私は rebase 教徒なのでマージではないかもしれませんが…いずれにせよ、コンフリクトからは逃れられない。こればかりは仕方ありません。

Windows においてグラフィカルにコンフリクトを解決するためのツールは数多く…と言いたいところですが、決して多くありません。どれが使いやすいかというと、うーん…一長一短ということにしておきたい感じです (私見)。そんな中、Visual Studio をマージ ツールとして使うことができ、触り心地も悪くない感じです。Windows 環境のマージ ツールでお悩みの方は、一度試してみては如何でしょうか。

<!-- more -->

## Visual Studio as a (Diff / Merge) Tool

とりあえず、百聞は一見に如かずということで、どんな具合に表示されるかをスクリーン ショットで紹介してしまいます。

* diff (2 ペイン表示): {% asset_img vs-difftool-split.png Visual Studio as a difftool (split) %}
* diff (インライン表示): {% asset_img vs-difftool-inline.png Visual Studio as a difftool (inline) %}
* merge: {% asset_img vs-mergetool.png Visual Studio as a mergetool %}

こんな感じでグラフィカルに <small>(当然だ)</small> 差異を表示してくれます。コンフリクトの解決も、hunk 部分の左側にあるチェック ボックスをトグルすることで簡単に採用させることができます。

まあ、普通のマージ ツールなんですが、ちゃんと一通りのことを直観的に操作できる感じでして、結構いい感じではないかなと個人的には思います。あくまで主観的な話なので、誰にでもマッチするかというと、まあ…そういう感じでもないとは思いますが、冒頭でも述べた通り、Windows のグラフィカル環境で一通りちゃんと処理できるマージ ツールの絶対数がそこまで多くはないという感じなので (片手でぎりぎり収まるくらいか？)、折角なので一度試してみてほしいかな、とは思います。

## Git で Visual Studio をマージ ツールに登録する

Git で Visual Studio を diff / merge ツールとして登録するには、以下の設定を `.gitconfig` なりに記述してやります <small>(注: 水平方向に長いです)</small>:

```ini
[diff]
    tool = vsdiffmerge

[merge]
    tool = vsdiffmerge

[difftool "vsdiffmerge"]
    cmd = \"C:\\Program Files (x86)\\Microsoft Visual Studio 14.0\\Common7\\IDE\\vsdiffmerge.exe\" \"$LOCAL\" \"$REMOTE\" //t
    keepbackup = false
    trustexitcode = true

[mergetool "vsdiffmerge"]
    cmd = \"C:\\Program Files (x86)\\Microsoft Visual Studio 14.0\\Common7\\IDE\\vsdiffmerge.exe\" \"$REMOTE\" \"$LOCAL\" \"$BASE\" \"$MERGED\" //m
    keepbackup = false
    trustexitcode = true
```

上の内容もググれば出てくるといえばその通りなのですが、あんまり日本語の情報で残ってないのと、会社内で何回か聞かれたりもしたので、まあ、ぎりぎり有用情報の範囲内かな…という感じです。

あと、当然ですが、Visual Studio のパスは適宜修正してください。

以下は `.gitconfig` とかじゃなくて、(SourceTree みたいなのの) 設定ダイアログ内のテキスト ボックスとかでコピペする用です。

```sh
# difftool
"C:\Program Files (x86)\Microsoft Visual Studio 14.0\Common7\IDE\vsdiffmerge.exe" "$LOCAL" "$REMOTE" /t
# mergetool
"C:\Program Files (x86)\Microsoft Visual Studio 14.0\Common7\IDE\vsdiffmerge.exe" "$REMOTE" "$LOCAL" "$BASE" "$MERGED" /m
```

少し脇道に逸れまして、`devenv.exe` じゃなくて `vsdiffmerge.exe` なんですね…っていう話なんですが、このヘルパ プログラムを介してやることで、既存の Visual Studio インスタンス上で差異を表示してやるためのものみたいです (当然、既存のインスタンスが存在しない場合は新しく `devenv.exe` が起動する)。

## 残念なところ

あんまり褒めちぎっても仕方ないので、少し残念なところについても言及しておきます。

* 上掲のスクリーン ショットでもよく見れば判るのですが、diff と merge で微妙に機能の有効化状況に差が…換言すれば、**両機能間での統一性には若干の難がある**みたいです。例ではまあ、例えば merge の方では [Web Essentials](http://vswebessentials.com/) による Markdown のシンタックス ハイライトが動いてますが、diff の方はそうではありません(これが例えば C# だとちゃんと表示されるんですけどね)。<br /><small>拡張が色々動いてたり、配色が色々カスタマイズされたままですみません…まあ、素の状態でも大筋には差はない…はずということで、平にご容赦ください。</small>
* 相対的に**重い**です。まあ、これも主観的な観点ですし、どこまで許容できるか、という話に尽きるのですが…少なくとも [WinMerge](http://winmerge.org/) よりは確実に立ち上がりまでに時間が掛かります。

以上を踏まえても、使いやすさと表示の洗練具合については他のツールより優れており、十分選択に値すると私は考えています。

## まとめ

私はいつまで経ってもコンフリクトの解決が不得手、というか苦手意識が抜けなかったのですが、`vsdiffmerge` を知ってからは随分と心が楽になりました。

割と便利で使いやすく、綺麗に出来ていると思うので、使ったことない方は是非一度でも試してみてほしい機能です。
