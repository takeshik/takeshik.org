---
layout: post
title: "Expression Trees をシリアライズする"
date: 2012-12-14 22:58:07
categories:
- programming
tags:
- CSharp
- LINQ
- Expression Trees
- Serialization
- Advent Calendar
---

この記事は [C# Advent Calendar 2012](http://atnd.org/events/33905) の 14 日目の記事です。
なぜかシリアライズできない Expression Trees をなんとかしてシリアライズします。

<!-- more -->

## はじめに

こんにちは！LINQ してますか！？

当然皆さん LINQ してると思いますが、Expression Trees は割と触ってる方が少ない印象で、ちょっと寂しい感があります。

とはいえ LINQ to SQL や to Entities、あるいは [先日の t-takano さんの記事](http://cerberus1974.hateblo.jp/entry/2012/12/13/000245) で取り上げられた MongoDB の LINQ プロバイダなどを使えば、知らぬ間に使っているものです。

今回は、そんな Expression Trees とシリアライズのお話です。

なお、今回の記事は拙作 [Yacq](https://github.com/takeshik/yacq) の開発において得られた知見を基にしています。



## 衝撃の事実

こんな記事を書くくらいなので、もう既に明らかになっているようなことですが…

式木を構成する式ノードの既定型 `Expression` クラスの[定義](http://msdn.microsoft.com/ja-jp/library/system.linq.expressions.expression.aspx)を見てみると…

```csharp
public abstract class Expression
```

…と、`Serializable` 擬似カスタム属性が付与されていないことが分かります。

実際のところ、式木はシリアライズできないのです。残念！

…となってしまうと面白くないので、何とかシリアライズする方法を探しましょう。

CodePlex を見てみると、[Expression Tree Serializer](http://expressiontree.codeplex.com/) なんてライブラリがあったりするのですが、コードを見てみると異様に複雑だったり、NuGet で公開されてなかったり (終わコン！) と散々なので、ここはひとつ自作するしかなさそうです。

## 式木を無理やりシリアライズ

ということで、式木をシリアライズしましょう。方法は一つしかないでしょう。一種のプロキシ <small>([`RealProxy`](http://msdn.microsoft.com/ja-jp/library/system.runtime.remoting.proxies.realproxy.aspx) じゃないです)</small> 的な型を作って、それを経由してシリアライズ・デシリアライズするのです！

具体的な式ノードの種別は、 [`ExpressionType` 列挙体](http://msdn.microsoft.com/ja-jp/library/bb361179.aspx) を参照すれば一望できます…これを全部網羅するのが目標となります…

とはいえ、たとえば Add や Subtract 等は [`BinaryExpression`](http://msdn.microsoft.com/ja-jp/library/system.linq.expressions.binaryexpression.aspx) という具合に、いくつかの種別に関しては一つの型として纏められていますので、`ExpressionType` のメンバ数だけプロキシ型を作る必要性は必ずしもありません。とはいえ、その型の単位でプロキシ クラスを作るといえども、演算種別を保持するために先掲の `ExpressionType` 値を保持する必要があるため、結果的には `ExpressionType` のメンバと一対一対応で型を作るのと、意味論的には大差は無いでしょう。ちなみに、Yacq では `ExpressionType` のメンバと、付随するその他のクラスに対応するプロキシ クラスを全て作成しました。

さて、ここでは `BinaryExpression` として示される式ノードについてシリアライズすることを検討しましょう。`BinaryExpression` でシリアライズする必要があるプロパティは以下の通りです。

* [`NodeType` : `ExpressionType`](http://msdn.microsoft.com/ja-jp/library/system.linq.expressions.expression.nodetype.aspx)
* [`Left` : `Expression`](http://msdn.microsoft.com/ja-jp/library/system.linq.expressions.binaryexpression.left.aspx)
* [`Right` : `Expression`](http://msdn.microsoft.com/ja-jp/library/system.linq.expressions.binaryexpression.right.aspx)
* [`IsLifted` : `bool`](http://msdn.microsoft.com/ja-jp/library/system.linq.expressions.binaryexpression.islifted.aspx)
* [`Method` : `MethodInfo`](http://msdn.microsoft.com/ja-jp/library/system.linq.expressions.binaryexpression.method.aspx)
* [`Conversion` : `LambdaExpression`](http://msdn.microsoft.com/ja-jp/library/system.linq.expressions.binaryexpression.conversion.aspx)

ということで、これを単に対応付けた、シリアライズ可能なクラスを定義するまでです。

```csharp
[DataContract]
[Serializable]
public class BinaryNode
    : Node
{
    [DataMember]
    public ExpressionType NodeType { get; set; }

    [DataMember]
    public Node Left { get; set; }

    [DataMember]
    public Node Right { get; set; }
    
    [DataMember]
    public bool IsLifted { get; set; }

    [DataMember]
    public MethodRef Method { get; set; }

    [DataMember]
    public Lambda Conversion { get; set; }
    
    public static BinaryNode Serialize(BinaryExpression expr)
    {
        return new BinaryNode()
        {
            NodeType = expr.NodeType,
            Left = Node.Serialize(expr.Left),
            Right = Node.Serialize(expr.Right),
            IsLifted = expr.IsLifted,
            Conversion = Lambda.Serialize(expr.Conversion),
        };
    }
    
    public BinaryExpression Deserialize()
    {
        return Expression.MakeBinary(
            this.NodeType,
            this.Left.Deserialize(),
            this.Right.Deserialize(),
            this.IsLifted,
            this.Method,
            this.Conversion.Deserialize()
        );
    }
}
```

こんな感じです。上のコードで `Node` は `Expression`、`MethodRef` は `MethodInfo`、`Lambda` は `LambdaExpression` に対応する型を想定しています。

こんな感じで全ての式ノード種別を網羅していくわけで、相当に酷い作業なわけですが、これで式木のシリアライズが可能となります。


## ParameterExpression のシリアライズに関し注意すべき点

`ParameterExpression` は、今回のようなシリアライズに限らず、Expression Trees の処理において注意を要する型です。というのも、`ParameterExpression` はオブジェクト参照を厳密に区別するため、換言すれば、パラメータの型や名前が同一でも参照が別であれば別のパラメータとして認識され、よって例外の射出原因となるためです ( [参考](http://qiita.com/items/438b845154c6c21fcba5) )。

ですから、ParameterExpression のシリアライズに際しても、パフォーマンスのためではなく、正しい式木の再構築のために、積極的なキャッシュ機構の実装を要します。

```csharp
[DataContract]
[Serializable]
public class Parameter : Node
{
    private static readonly Dictionary<ParameterExpression, Parameter> _parameterReverseCache
        = new Dictionary<ParameterExpression, Parameter>();

    private static readonly Dictionary<Parameter, ParameterExpression> _cache
        = new Dictionary<Parameter, ParameterExpression>();

    [DataMember]
    public String Name { get; set; }
    
    public static Parameter Serialize(ParameterExpression expression)
    {
        return _parameterReverseCache.TryGetValue(expression)
            ?? new Parameter()
               {
                   Type = TypeRef.Serialize(expression.Type),
                   Name = expression.Name,
               }.Apply(p => _parameterReverseCache.Add(expression, p));
    }

    public override Expression Deserialize()
    {
        return _cache.TryGetValue(this)
            ?? Expression.Parameter(
                   this.Type.Deserialize(),
                   this.Name
               ).Apply(p => _cache.Add(this, p));
    }
}

// ※ 上のコード中で使われている拡張メソッドの定義は以下の通りです:

internal static TReceiver Apply<TReceiver>(this TReceiver self, params Action<TReceiver>[] actions)
{
    Array.ForEach(actions, a => a(self));
    return self;
}

internal static TValue TryGetValue<TKey, TValue>(this IDictionary<TKey, TValue> dictionary, TKey key, TValue defaultValue = default(TValue))
{
    TValue value;
    return dictionary.TryGetValue(key, out value)
        ? value
        : defaultValue;
}
```

さらに、`DataContractSerializer` でシリアライズする際に、オブジェクト グラフの参照関係を維持するために、

```csharp
[DataContract(IsReference = true)]
```

とカスタム属性を付与する必要があります。

以上のような施策により、`LambdaExpression` もきちんとシリアライズできる、かなり強力なシステムとなるのです。

## 式木をシリアライズできることによる利点

式木をシリアライズできるということは、シリアライズにより発生する利点を全て享受できるということです。当然ですが。つまり:

* 式木の内容をファイルに保存し、読み込むことができる。
* 式木を `AppDomain` 境界を超えて引き渡すことができる。
    * リモート ホストや [sandbox ドメイン上で実行させたりする](http://msdn.microsoft.com/ja-jp/library/bb763046.aspx) ことができるようになる

といった内容です。べんり！

ちなみに、デリゲートをシリアライズしても `MethodInfo` を得るための間接的な識別情報と `this` 参照をシリアライズしたものが得られるだけで、[`DynamicMethod`](http://msdn.microsoft.com/ja-jp/library/system.reflection.emit.dynamicmethod.aspx) 等にそのまま投げ込める IL 列が得られたりするわけでもなく、実行に際してはデリゲートが指すアセンブリのロードが前提となっています (そういう用途で間に合うことが大半というのは、まあそうなのですが)。そういう観点からも、Expression Trees のシリアライズによるコードの保存には一定の意味があると考えられます。

以上が本題で、後は雑多なオマケな気がします。

## MethodInfo 等のシリアライズにまつわる話

`MemberInfo` 派生型、例えば `MethodInfo` は serializable なのですが、`DataContractSerializer` でシリアライズしようとすると例外が射出されてしまいます。

これに関しては単純に抽象型の `MethodInfo` でシリアライズしようとしているからであり ([`KnownTypes` の問題](http://msdn.microsoft.com/ja-jp/library/ms730167.aspx))、具象型の `RuntimeMethodInfo` 型 (※ほとんどの場合) でシリアライズすればうまくいきます:

```csharp
new DataContractSerializer(Type.GetType("System.Reflection.RuntimeMethodInfo"))
    .WriteObject(s, typeof(Enumerable).GetMethod("Range"));
```

ですが、これで得られた `RuntimeMethodInfo` のシリアライズされたデータは、だいたいこんな感じです:

```xml
<RuntimeMethodInfo z:FactoryType="MemberInfoSerializationHolder" xmlns="http://schemas.datacontract.org/2004/07/System.Reflection" xmlns:i="http://www.w3.org/2001/XMLSchema-instance" xmlns:x="http://www.w3.org/2001/XMLSchema" xmlns:z="http://schemas.microsoft.com/2003/10/Serialization/">
  <Name i:type="x:string" xmlns="">Range</Name>
  <AssemblyName i:type="x:string" xmlns="">System.Core, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089</AssemblyName>
  <ClassName i:type="x:string" xmlns="">System.Linq.Enumerable</ClassName>
  <Signature i:type="x:string" xmlns="">System.Collections.Generic.IEnumerable`1[System.Int32] Range(Int32, Int32)</Signature>
  <Signature2 i:type="x:string" xmlns="">System.Collections.Generic.IEnumerable`1[[System.Int32, mscorlib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089]] Range(System.Int32, System.Int32)</Signature2>
  <MemberType i:type="x:int" xmlns="">8</MemberType>
  <GenericArguments i:nil="true" xmlns="" />
</RuntimeMethodInfo>
```

ちょっと汚すぎます。個人的には、[`RuntimeMethodHandle`](http://msdn.microsoft.com/ja-jp/library/system.runtimemethodhandle.aspx) は別環境ででシリアライズできなくなりそうな懸念を考えれば仕方ないものの、もう少し密度の高い情報を期待していたのですが。

…ということで、よりスマートな形式で `MethodInfo` をシリアライズできないものか検討していたのですが、どうも上の XML の `Signature2` 要素くらい冗長なデータで記録しているのには、それなりの理由があるみたいでした。

上の XML において記述されている `MemberInfoSerializationHolder` というのが、恐らくこのシリアライズ処理を担っているクラスで間違いないでしょう。

ということで、これを [見てみる](https://github.com/mono/mono/blob/master/mcs/class/corlib/System.Reflection/MemberInfoSerializationHolder.cs) と、やはり `ISerializable` を実装していました。

`GetRealObject()` メソッドで `MethodInfo` を取得していますが、型に含まれるメソッド一覧に対し、片っ端から `MethodInfo.ToString()` メソッドを呼び出し、それが一致するかどうかで検索を行なっています。ちょっとあんまりな実装です。

ですが、CLR をちょっと覗き見してみると、以下のような呼び出しが見受けられ、`RuntimeType.ToString()` メソッドの返り値をキャッシュしているようです。

```csharp
RuntimeType.GetCachedName(TypeNameKind.ToString)
```

やや内部に立ち入った話となってしまいましたが、以上の結果から、`MethodInfo.ToString()` を用いて `MethodInfo` をデシリアライズすることは、シリアライズにあたっての安全性を考慮すると、そこまで残念な実装ではなさそうです。

(ですが、現在の Mono の実装では特にリフレクション データに関するキャッシュは適用している様子は見受けられず、パフォーマンス面で劣ることが予測されます)。

## 結論

**なぜ Expression をシリアライズできないようにした！！ (絶望)**

## 参考

* [Yacq における式木シリアライズのためのクラス群](https://github.com/takeshik/yacq/tree/b2325f8cdb590d601621cff28d58b816ec36dfdd/Yacq/Serialization)
* 他の式木に絡んだ Advent Calendar 記事
    * Esolang Advent Calendar 2012 - [言語基盤としての Expression Trees、そして Yacq](/blog/2012/12/05/expression-trees-and-yacq/)
    * C# Advent Calendar 2011 - [Expression Trees with IL Emit](/blog/2011/12/14/expression-trees-with-il-emit/)
