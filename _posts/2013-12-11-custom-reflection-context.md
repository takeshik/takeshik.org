---
layout: post
title: "CustomReflectionContext でリフレクションに介入"
date: 2013-12-11 18:57
categories:
- programming
tags:
- CSharp
- Reflection
- Advent Calendar
---

この記事は [C# Advent Calendar 2013](http://http://www.adventar.org/calendars/119) の 10 日目の記事です。相変わらず遅刻してしまったわけですが…

.NET 4.5 で追加されたクラスの中に、`CustomReflectionContext` というクラスがあります (アセンブリ: `System.Reflection.Context`)。MSDN ライブラリによれば、

> リフレクションのカスタマイズ可能なコンテキストを表します。

…とのこと。さっぱり意味不明なのですが、どうやら Reflection API の範囲内で、プロパティを増やしたり、付与されているカスタム属性を変更したりすることができるようです。これは試してみるほかない！

<!-- more -->

## コンテキストの実装

ひとまず、以下のような画竜点睛を欠く感に溢れたクラスを、リフレクション対象として使うことにします。

```csharp
[DataContract]
public class Person
{
    private int _id;

    [DataMember]
    public string Name { get; set; }

    [DataMember]
    public int Age { get; set; }

    public Person(int id, string name, int age)
    {
        this._id = id;
        this.Name = name;
        this.Age = age;
    }

    public override string ToString()
    {
        return string.Format("Person #{0}: {1} ({2})", this._id, this.Name, this.Age);
    }
}

var person = new Person(1, "John Doe", 20); // "Person #1: John Doe (20)"
```

そして、このクラスの少し足りてないところを無理やり補う感じで、よく分からないなりに IntelliSense 先生などにお伺いを立てつつ、クラスを実装してみます。

```csharp
class CustomContext : CustomReflectionContext
{
    private static readonly CustomContext _instance = new CustomContext();

    public static CustomContext Instance { get { return _instance; } }

    private CustomContext() {}

    protected override IEnumerable<PropertyInfo> AddProperties(Type type)
    {
        if (type != typeof(Person)) yield break;
        var id = type.GetField("_id", BindingFlags.NonPublic | BindingFlags.Instance);
        yield return this.CreateProperty(this.MapType(typeof(int).GetTypeInfo()), "Id",
            self => id.GetValue(self),
            (self, value) => id.SetValue(self, value),
            new [] { new DataMemberAttribute(), }, null, null
        );
    }
}
```

(上の例では Singleton にしていますが、別に普通にインスタンス化させても動作します。しかし、(後々) 複雑なことをやらせるに当たり事故の可能性を抑えるために適用しています)

オーバーライドした `AddProperties` メソッドの中で、問題のクラス `Person` の時のみ新しいプロパティを追加するようにしています。

`CreateProperty` のシグネチャは以下の通りです。

```csharp
protected PropertyInfo CreateProperty(
    Type propertyType,
    string name,
    Func<Object, Object> getter,
    Action<Object, Object> setter,
    IEnumerable<Attribute> propertyCustomAttributes,
    IEnumerable<Attribute> getterCustomAttributes,
    IEnumerable<Attribute> setterCustomAttributes
);
```

なるほど、いかにも「プロパティを実装して追加します！」と言わんばかりのシグネチャなので、割とわかりやすいですね。`getter` もしくは `setter` を `null` にすることで、read-only または write-only なプロパティも作れるようです。

## 実際に使ってみる

上で実装したコンテキストを使って、実際にプロパティが追加されたかを確認してみます。`CustomReflectionContext` の適用下でリフレクションを行うには、`Map` で始まるメソッドを呼べば達成できるようです。例えば、以下のように:

```csharp
var type = CustomContext.MapType(typeof(Person).GetTypeInfo());
```

上で得られた Type オブジェクトの型は `RuntimeType` ではなく `System.Reflection.Context.Custom.CustomType` となっているので、`CustomReflectionContext` の適用下でリフレクションが行われることが確認できます。

```csharp
var prop = type.GetProperty("Id");
```

プロパティも取得してみます。本来は定義されていないプロパティなので `null` となるところ、`System.Reflection.Context.Virtual.VirtualPropertyInfo` 型のオブジェクトが返ってきます。

読み書きももちろん動作します:

```csharp
prop.GetValue(person); // 1
prop.GetValue(person, 3);
Console.WriteLine(person); // "Person #3: John Doe (20)"
```

以上の通り、`CustomReflectionContext` を用いることで、なおかつ「CustomReflectionContext の適用下のリフレクション オブジェクトを渡せる場合に限り」、その型の内容を「CustomReflectionContext の機能の範囲内で」騙し変えることが可能です。

CustomReflectionContext の機能は、上で示した

* パブリック プロパティの追加

の他は、`GetCustomAttributes` メソッドを用いた、

* メンバに付与されたカスタム属性の追加・変更
* メソッドのパラメータに付与されたカスタム属性の追加・変更

くらいです。

この `CustomReflectionContext` は MEF (Managed Extensibility Framework) において、カスタム属性を事前に付与することなく、ある型を MEF コンポーネントとしてロードできるようにする機能のために用意されているらしく、その範囲内では機能的に不足はないのでしょうが、それ以外で色々考えてみると、意外ともう少し色々出来ても…と思わなくもない感じです。

## 様々な試行 (と称した無駄な足掻き)

ここで終わるのも勿体ない感じがするので、色々と足掻いてみます。その前に、色々と面倒なので以下のヘルパ メソッドを CustomContext に定義します。

```csharp
public TypeInfo Map(Type type)
{
    return this.MapType(type.GetTypeInfo());
}

public TypeInfo Map<T>()
{
    return this.Map(typeof(T));
}
```

いちいち `ctx.MapType(typeof(T).GetTypeInfo())` って書くの面倒ですし。

### with DataContractSerializer

せっかく例で `DataMemberAttribute` を付けているわけですし、実際にシリアライズしてみましょう。


```csharp
using (var ms = new MemoryStream())
{
    new DataContractSerializer(CustomContext.Instance.Map<Person>())
        .WriteObject(ms, person);
    Console.WriteLine(Encoding.UTF8.GetString(ms.ToArray()));
}
```

…の結果は:

```xml
<Person xmlns="http://schemas.datacontract.org/2004/07/" xmlns:i="http://www.w3.org/2001/XMLSchema-instance">
    <Age>20</Age>
    <Name>John Doe</Name>
</Person>
```

どうも不発だったようで…考えられる原因としては、

* DataContractSerializer がよっぽど複雑なことをしてすり抜けているか (RuntimeTypeHandle とか引き回してるので、十分あり得る)
* VirtualPropertyInfo の実装が DataContractSerializer にとって不十分 (GetCustomAttributesData メソッドで常に空の配列を返しているのが要因としてあるかも？)

などでしょうか。

### with Json.NET (JsonSerializer)

仕方がないので Json.NET (`Install-Package Newtonsoft.Json`) のシリアライザ… `JsonSerializer` も試してみましょう。

```csharp
using (var sw = new StringWriter())
{
    JsonSerializer.CreateDefault().Serialize(sw, person);
    Console.WriteLine(sw);
}
```

…の結果は:

```javascript
{
    "Name":"John Doe",
    "Age":20
}
```

と、これまた残念な結果に…しかしこれは `person.GetType()` の結果にプロパティ `Id` は当然存在しないわけで、必然でしょう。

`Serialize` メソッドには `Type` を渡すオーバーロードもあるみたいですが、どうも限定的にしか使われていないらしく、大勢に影響は無い模様。

…となると、`Person.GetType()` の結果が `CustomReflectionContext` の適用下の `Type` を返せば、上手く動くのではないかという考えに至ります。

`GetType` をオーバーライドすることなど出来ないわけで…となると、古の？黒魔術を用いて、無理やり結果をすげ替えてやるしかなさそうです。

まず、`Person` クラスの定義を以下のように変更して…

```csharp
[DataContract]
[ReflectionContextProxy]
public class Person : ContextBoundObject
```

…と、この時点で残念な空気が漂ってしまっている気がしますが、`RealProxy` と `ContextBoundObject` を用いてメソッドの呼び出しをフックしています。

詳細を説明しようとすると軽く本記事程度の長さになってしまいそうな気がするので詳しい内容については泣く泣く割愛させて頂きますので、詳細については各自ググって頂けますと幸いです。

```csharp
public class ReflectionContextProxy : RealProxy
{
    private MarshalByRefObject _obj;
    private CustomReflectionContext _context;

    public ReflectionContextProxy(Type type, MarshalByRefObject obj, CustomReflectionContext context)
        : base(type)
    {
        this._obj = obj;
        this._context = context;
    }

    public override IMessage Invoke(IMessage msg)
    {
        var ccm = msg as IConstructionCallMessage;
        if (ccm != null)
        {
            RemotingServices.GetRealProxy(this._obj).InitializeServerObject(ccm);
            return EnterpriseServicesHelper.CreateConstructionReturnMessage(
                ccm, (MarshalByRefObject) this.GetTransparentProxy()
            );
        }
        var mcm = msg as IMethodCallMessage;
        if (mcm != null)
        {
            if (mcm.MethodName == "GetType")
            {
                return new ReturnMessage(
                    this._context.MapType(this._obj.GetType().GetTypeInfo()),
                    null, 0, null, mcm
                );
            }
            return RemotingServices.ExecuteMessage(this._obj, mcm);
        }
        throw new NotSupportedException();
    }
}

[AttributeUsage(AttributeTargets.Class)]
public class ReflectionContextProxyAttribute : ProxyAttribute
{
    public override MarshalByRefObject CreateInstance(Type serverType)
    {
        var obj = base.CreateInstance(serverType);
        var proxy = new ReflectionContextProxy(serverType, obj, CustomContext.Instance);
        return (MarshalByRefObject) proxy.GetTransparentProxy();
    }
}
```

上の定義を済ませたところで動作確認します:

```csharp
var person = new Person(1, "John Doe", 20);

Console.WriteLine(person.GetType().GetType().FullName);
    // System.Reflection.Context.Custom.CustomType
Console.WriteLine(person.GetType().GetProperty("Id"));
    // System.Int32 Id
```

…と、無事 `GetType` メソッドの呼び出しのフックに成功しました。

ここでもう一度、先ほどのシリアライズ コードを実行してみると…

> Newtonsoft.Json.JsonSerializationException: Error getting value from 'Name' on 'Person'. ---> System.NotImplementedException: メソッドまたは操作は実装されていません。

…と、スタックトレースを見る限り、内部で動的コード生成してしまっている結果、残念なことになってしまっているみたいです。

### 結果

非常に残念な結果ではありますが、より単純にリフレクションを用いるコードでは普通に動いてくれる可能性もまだ期待できるでしょう。

また、この方向にしてもまだまだやりようがあるでしょうし、何より `RealProxy` の使い方を思い出すのに結構な時間を要したので捨てるに捨てられない…という残念な事情もあるので、全くオチてないのですが、今後の調査課題ということにしようと思います。

## まとめ

* `CustomReflectionContext` を用いることで、リフレクションの動作に (限定的に) 介入することができる。
* リフレクションの介入については `Type` や `Assembly` といった具体的なリフレクション オブジェクトを渡す段階で `CustomReflectionContext` を適用してやる必要がある。
* 動的に追加されるメンバについては、操作の制限を受ける。

所感としては、(今回冒頭にて例に挙げた MEF のように) Reflection API を用いたシステムにおいて、最初から `CustomReflectionContext` の存在および利用を念頭に置いてシステムを作りこめば、普通に利用のタイミングもあるのではないかなと考えられます。例えば、

* バージョンアップに際しての既存の外部ライブラリに対して無変更でパッチを当てて呼び出せるようにするとか、あるいは
* カスタム属性を用いてテキスト リソースを紐付けている場合、個々のアセンブリに対してではなく一括でローカライズを行う手段として

といった使途もあるのではないかなと (カスタム属性内の文字列ローカライズについては、割と真面目にイケるネタかもしれないと、個人的には考えています)。

あるいは、`CustomReflectionContext` の機能不足については、このクラスが行っているように、自分で `System.Reflection` 名前空間のクラスを全部継承して色々やれるようにすれば…とは思いますが、ちょっと苦行すぎる感がありますね。非現実的というほどでもないとは思いますが。

今回は、無茶な使い方をしても簡単に上手くいくはずがないよね、というのを改めて明らかにしたことに価値があると思いました (無理やり…)。
