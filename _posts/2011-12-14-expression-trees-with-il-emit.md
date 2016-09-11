---
layout: post
title: "Expression Trees with IL Emit"
date: 2011-12-14 21:09:56
comments: true
categories:
- programming
tags:
- CSharp
- LINQ
- Expression Trees
- IL
- Advent Calendar
---

この記事は [C# Advent Calendar 2011](http://atnd.org/events/21988) 14 日目の記事として書かれたものです。

この記事では、Expression Trees API を Emit API と組み合わせて利用することで型を動的に構築するにあたっての問題点、及びそれを解決するための具体的な手段について解説します。

<!-- more -->

この記事に含まれるコードは目的通りに動作することを確認しましたが、その正しさ、特に生成されるアセンブリの内容における脆弱性・問題その他について一切の保証を負いかねます。

## 序

Expression Trees API は .NET Framework 4 になって機能が拡張されましたが、そのひとつとして、[LambdaExpression.CompileToMethod メソッド](http://msdn.microsoft.com/en-us/library/dd728258.aspx) の追加が挙げられるでしょう。

このメソッドは、通常ラムダ式木の内容をデリゲートとして取得するところ、引数として指定された [MethodBuilder](http://msdn.microsoft.com/ja-jp/library/system.reflection.emit.methodbuilder.aspx) オブジェクトに式木の内容を emit するものです。つまり、メソッドの定義に式木を使えるのです！

…と思いきや、このメソッドには、あるいは、Expression Trees には様々な制限があり、残念ながら、十全に型におけるメソッドの定義が行えるわけではありません。

### 準備

まずは、物事を始めるにあたって、コードの準備をしてしまいましょう。コードは [LINQPad](http://www.linqpad.net/) のような便利なツールでの実行を前提とした書き方ですが、もちろん適切な位置にコード片をコピペしても構いません。

```csharp
var asm = AppDomain.CurrentDomain.DefineDynamicAssembly(
    new AssemblyName("Test"),
    AssemblyBuilderAccess.RunAndSave
);
var mod = asm.DefineDynamicModule("Test.dll");
var type = mod.DefineType("TargetType", TypeAttributes.Public);
```

using ディレクティブの追加は IDE に教えてもらってください。

## Expression Trees の制限

以下のコードを続けて実行してみましょう。

```csharp
var field = type.DefineField("X", typeof(int), FieldAttributes.Public);
var method = type.DefineMethod("TestMethod", MethodAttributes.Public, typeof(int), new [] { typeof(int), typeof(int), });
var p_this = Expression.Parameter(typ);
var p_x = Expression.Parameter(typeof(int));
var p_y = Expression.Parameter(typeof(int));
var expr = Expression.Lambda(
    Expression.Assign(Expression.Field(p_this, field), Expression.Constant(100)),
    p_this, p_x, p_y
);
expr.CompileToMethod(method);
```

これは、以下のような要素を生成することを期待しています。なお、public なフィールドを利用しているのは単に簡単のためであって、許せないというのなら自動実装プロパティを emit するように読み替えて下さい。

```csharp
public int X;
public int TestMethod(int x, int y)
{
    return this.X = 100;
}
```

しかし、上のコードは例外

> NotSupportedException { Message = "Specified method is not supported.", TargetSite = TypeBuilderInstantiation.GetMethodImpl, }

が送出されてしまいます。

即ち、Expression Trees API は TypeBuilder などの System.Reflection.Emit 名前空間以下の Builder 系のオブジェクトを取り扱えていないことを表します。そしてそれは換言すれば、生成された型― [TypeBuilder.CreateType メソッド](http://msdn.microsoft.com/ja-jp/library/system.reflection.emit.typebuilder.createtype.aspx) によって得られた型であるならば、問題なく取り扱えるであろうことを示唆しているでしょう。

## CompileToMethod の制限

また、不正な IL が生成されることを承知で、以下のコードを試してみましょう。

```csharp
var method = type.DefineMethod("TestMethod", MethodAttributes.Public, typeof(int), Type.EmptyTypes);
var expr = Expression.Lambda(
    Expression.Constant(100)
);
expr.CompileToMethod(method);
```

上のコードでは、instance メソッドの暗黙の第 0 引数が宣言されていないため、不正となっています。しかしそこは一度棚に上げて、実行してみることにしましょう。

すると、今度は例外

> ArgumentException { Message = "Invalid argument value\nParameter name: method", TargetSite = LambdaExpression.CompileToMethod, }

が送出されてしまいます。

Mono 2.10.6 の System.Core.dll 内の `System.Linq.Expressions.LambdaExpression.CompileToMethodInternal` メソッドには、

```csharp
ContractUtils.Requires(method.IsStatic, "method");
```

というコードが含まれており、ここから `LambdaExpression.CompileToMethod` メソッドは instance メソッドをサポートしていない、と推定できます。

## 解決策の構築

以上のことより、Expression Trees API を用いて動的にメソッド、しいては型を構築するという企みは様々な制限により通常の手段では困難であり、これを回避するためには、何らかの策を講じなければならないことが明らかになりました。

検討の結果、以下のようにすることで制限を回避し、当初目的を達成することができます。

* LambdaExpression.CompileToMethod メソッドを実行する前に作成しようとする型 (以下''対象型''と呼称) の (外観的な) 定義を終了してしまう。
* 対象型 (のメソッド及びコンストラクタ) の実際の定義は、対象型のネストされた型に static メソッドとして定義し、対象型のメソッドは実際の定義を呼び出すスタブ コードを (`ILGenerator` を用いて) emit する。
    * フィールドの初期化も内部型に用意した static メソッドによって行う。

具体的には、以下のような定義の型:

```csharp
public class TargetType
{
    public int X = 123;
    
    public int Y;
    
    public TargetType(int y)
    {
        this.Y = y;
    }
    
    public int Func(int z)
    {
        return this.X + this.Y + z;
    }
}
```

を生成するために、以下のような型を実際には生成することとなります:

```csharp
public class TargetType
{
    class Impl
    {
        internal static Prologue(TargetType self)
        {
            self.X = X_Init(self);
        }
        
        static int X_Init(TargetType self)
        {
            return 123;
        }
        
        internal static void Ctor(TargetType self, int y)
        {
            self.Y = y;
        }
        
        internal Func(TargetType self, int z)
        {
            return self.X + self.Y + z;
        }
    }
    
    public int X;
    
    public int Y;
    
    public TargetType(int y)
    {
        Impl.Prologue(this);
        Impl.Ctor(this, y);
    }
    
    public int Func(int z)
    {
        return Impl.Func(this, z);
    }
}
```

上において、`LambdaExpression.CompileMethod` メソッドで生成する必要がある部分 (ユーザ定義部分) は、

* `TargetType.Impl.X_Init(TargetType)`
* `TargetType.Impl.Ctor(TargetType, int)`
* `TargetType.Impl.Func(TargetType, int)`

のみであり、残りは全て型の定義状況から導け、また、Emit API を用いて定義する必要があります。

このようにすることで、以下の効果が得られます:

* ユーザ定義部分を全て static メソッドにすることができる
    * `LambdaExpression.CompileToMethod` メソッドの、instance メソッドを生成できないという制限を回避できます。
* ユーザ定義部分が全て内部クラスに存在する形となる
* 内部クラスは外側のクラスが `TypeBuilder.CreateType` メソッドを呼び出した後に同じく `TypeBuilder.CreateType` メソッドを呼び出すことになる
    * ユーザ定義部分が全て内部クラス内にあることにより、先に外観部分を定義した外側のクラスを `CreateType` することができます。そうすることで、Expression Trees API が `...Builder` オブジェクトに触る可能性を回避することができます。
    * また、具体的なアクセシビリティ設定にも依りますが、内部クラスは一般に外側のクラスの private メンバにアクセスすることができます。

従って、先に示した問題点を全て解決することができます。

## 実施

それでは、実際に上に示したような型を作ることを目標として、コードを構築していきましょう。

```csharp
public class TypeGenerator
{
    // 対象型
    private readonly TypeBuilder _type;

    // 対象型のみを含んだ配列 (コーディング省力化)
    private readonly Type[] _typeArray;

    // 対象型の実装を定義する内部クラス
    private readonly TypeBuilder _implType;

    // 全てのコンストラクタで呼び出される共通初期化メソッド (Prologue)
    private readonly MethodBuilder _prologue;

    // 実際の実装を行うコードのキュー
    //   Item1: メソッドの実装を提供する関数オブジェクト Type 引数は CreateType された対象型が渡り、ここから型のメンバにアクセスできる
    //     返り値は LambdaExpression、あるいはその他の Expression: Expression.Lambda に包まれる (救済措置)
    //     (※) LambdaExpression.CompileToMethod は自身のシグネチャと対象の MethodBuilder のシグネチャが一致していることを確認していない模様
    //   Item2: 実装を注入する対象の MethodBuilder
    private readonly Queue<Tuple<Func<Type, Expression>, MethodBuilder>> _implementers;

    // 定義されたメンバのリスト (TypeBuilder.GetMembers メソッドは NotSupportedException で利用できないため)
    private readonly List<MemberInfo> _members;

    // CreateType されたかどうかのフラグ
    private bool _isCreated;

    public TypeGenerator(
        ModuleBuilder module,
        string name,
        IEnumerable<Type> baseTypes
    )
    {
        this._type = module.DefineType(
            name,
            TypeAttributes.Public,
            baseTypes.Any()
                ? baseTypes.First()
                : typeof(object),
            baseTypes.Skip(1).ToArray()
        );
        this._typeArray = new[] { this._type, };
        // 名前を <Impl> とすることで C# などの普通の言語上で認識されることを防ぐ
        this._implType = this._type.DefineNestedType("<Impl>", TypeAttributes.NestedPrivate);
        this._prologue = this._implType.DefineMethod(
            "Prologue",
            MethodAttributes.Assembly | MethodAttributes.Static,
            typeof(void),
            this._typeArray
        );
        this._implementers = new Queue<Tuple<Func<Type, Expression>, MethodBuilder>>();
        this._members = new List<MemberInfo>();
        this._isCreated = false;
    }

    public FieldBuilder DefineField(
        string name,
        Type type,
        Func<Type, Expression> initializer = null
    )
    {
        var field = this._type.DefineField(
            name,
            type,
            FieldAttributes.Public
        );
        if (initializer != null)
        {
            // 実装メソッドの定義
            var impl = this._implType.DefineMethod(
                field.Name + "_Init",
                MethodAttributes.Assembly | MethodAttributes.Static,
                type,
                this._typeArray
            );
            // Prologue メソッドに当該フィールドの初期化・代入コードを追加
            var prologueIl = this._prologue.GetILGenerator();
            LoadArgs(prologueIl, 0, 0);
            prologueIl.Emit(OpCodes.Call, impl);
            prologueIl.Emit(OpCodes.Stfld, field);
            this.AddImplementer(initializer, impl);
        }
        this._members.Add(field);
        return field;
    }

    public MethodBuilder DefineMethod(
        string name,
        Type returnType,
        IList<Type> parameterTypes,
        Func<Type, Expression> body = null
    )
    {
        var method = this._type.DefineMethod(
            name,
            MethodAttributes.Public | MethodAttributes.HideBySig,
            returnType,
            parameterTypes.ToArray()
        );
        var il = method.GetILGenerator();
        if (body != null)
        {
            // 実装メソッドの定義
            var impl = this._implType.DefineMethod(
                method.Name + "_Impl",
                MethodAttributes.Assembly | MethodAttributes.Static,
                returnType,
                _typeArray.Concat(parameterTypes).ToArray()
            );
            // 実装提供メソッドを呼び出す
            LoadArgs(il, Enumerable.Range(0, parameterTypes.Count + 1));
            il.Emit(OpCodes.Call, impl);
            il.Emit(OpCodes.Ret);
            this.AddImplementer(body, impl);
        }
        else
        {
            // return default(TReturn); もしくは単に return; を生成する
            if (returnType.IsValueType)
            {
                switch (Type.GetTypeCode(returnType))
                {
                    case TypeCode.Byte:
                    case TypeCode.SByte:
                    case TypeCode.Char:
                    case TypeCode.UInt16:
                    case TypeCode.Int16:
                    case TypeCode.UInt32:
                    case TypeCode.Int32:
                    case TypeCode.Boolean:
                        il.Emit(OpCodes.Ldc_I4_0);
                        break;
                    case TypeCode.UInt64:
                    case TypeCode.Int64:
                        il.Emit(OpCodes.Ldc_I4_0);
                        il.Emit(OpCodes.Conv_I8);
                        break;
                    case TypeCode.Single:
                        il.Emit(OpCodes.Ldc_R4, (float) 0);
                        break;
                    case TypeCode.Double:
                        il.Emit(OpCodes.Ldc_R8, (double) 0);
                        break;
                    default:
                        if (returnType != typeof(void))
                        {
                            il.Emit(OpCodes.Ldloca_S, (short) 1);
                            il.Emit(OpCodes.Initobj, returnType);
                        }
                        break;
                }
            }
            else
            {
                il.Emit(OpCodes.Ldnull);
            }
            il.Emit(OpCodes.Ret);
        }
        this._members.Add(method);
        return method;
    }

    public ConstructorBuilder DefineConstructor(
        IList<Type> parameterTypes,
        Func<Type, Expression> body = null
    )
    {
        var ctor = this._type.DefineConstructor(
            MethodAttributes.Public | MethodAttributes.HideBySig | MethodAttributes.SpecialName | MethodAttributes.RTSpecialName,
            CallingConventions.Standard,
            parameterTypes.ToArray()
        );
        var il = ctor.GetILGenerator();
        // 基底型のデフォルトコンストラクタを呼び出す
        LoadArgs(il, 0);
        il.Emit(OpCodes.Call, this._type.BaseType.GetConstructor(
            BindingFlags.Public | BindingFlags.NonPublic | BindingFlags.Instance,
            null,
            Type.EmptyTypes,
            null
        ));
        // Prologue メソッドを呼び出す
        LoadArgs(il, 0);
        il.Emit(OpCodes.Call, this._prologue);
        LoadArgs(il, Enumerable.Range(0, parameterTypes.Count + 1));
        if (body != null)
        {
            // 実装メソッドの定義
            var impl = this._implType.DefineMethod(
                "Ctor_Impl",
                MethodAttributes.Assembly | MethodAttributes.Static,
                typeof(void),
                this._typeArray.Concat(parameterTypes).ToArray()
            );
            // 実装提供メソッドを呼び出す
            il.Emit(OpCodes.Call, impl);
            this.AddImplementer(body, impl);
        }
        il.Emit(OpCodes.Ret);
        this._members.Add(ctor);
        return ctor;
    }

    public Type CreateType()
    {
        // 既に CreateType されている場合は処理を行わず単に結果を返す
        if (this._isCreated)
        {
            return this._type.CreateType();
        }
        // 対象型のデフォルトコンストラクタが定義されていない場合、定義する
        //   (Prologue が常に呼び出されるようにするため)
        if (!this._members.OfType<ConstructorBuilder>().Any())
        {
            this.DefineConstructor(Type.EmptyTypes);
        }
        // 対象型を CreateType する (※内部クラスは後で問題ない点に注目)
        var type = this._type.CreateType();
        // キューの内容を処分して実装を定義する
        foreach (var implementer in _implementers)
        {
            var expr = implementer.Item1(type);
            (expr is LambdaExpression
                ? (LambdaExpression) expr
                : Expression.Lambda(expr)
            ).CompileToMethod(implementer.Item2);
        }
        // Prologue メソッドをきちんと終わらせる
        this._prologue.GetILGenerator().Emit(OpCodes.Ret);
        // 内部クラスも CreateType する
        this._implType.CreateType();
        this._isCreated = true;
        return type;
    }

    private void AddImplementer(Func<Type, Expression> body, MethodBuilder impl)
    {
        this._implementers.Enqueue(Tuple.Create(body, impl));
    }

    // ldarg.N, ldarg.S, ldarg を一度に行うメソッド
    private static void LoadArgs(ILGenerator generator, IEnumerable<int> indexes)
    {
        foreach (var i in indexes)
        {
            switch (i)
            {
                case 0:
                    generator.Emit(OpCodes.Ldarg_0);
                    break;
                case 1:
                    generator.Emit(OpCodes.Ldarg_1);
                    break;
                case 2:
                    generator.Emit(OpCodes.Ldarg_2);
                    break;
                case 3:
                    generator.Emit(OpCodes.Ldarg_3);
                    break;
                default:
                    if (i <= short.MaxValue)
                    {
                        generator.Emit(OpCodes.Ldarg_S, (short) i);
                    }
                    else
                    {
                        generator.Emit(OpCodes.Ldarg, i);
                    }
                    break;
            }
        }
    }

    private static void LoadArgs(ILGenerator generator, params int[] indexes)
    {
        LoadArgs(generator, (IEnumerable<int>) indexes);
    }
}
```

## 動作確認

上のコードは、以下のように利用します:

```csharp
var asm = AppDomain.CurrentDomain.DefineDynamicAssembly(
    new AssemblyName("Test"),
    AssemblyBuilderAccess.RunAndSave
);
var mod = asm.DefineDynamicModule("Test.dll");
var gen = new TypeGenerator(mod, "TargetType");
gen.DefineField("X", typeof(int),
    // フィールドの初期化値
    t => Expression.Constant(123)
);
gen.DefineField("Y", typeof(int));
gen.DefineConstructor(new[] { typeof(int), }, t =>
{
    // コンストラクタの本文
    var p_this = Expression.Parameter(t);
    var p_y = Expression.Parameter(typeof(int));
    return Expression.Lambda(
        Expression.Block(
            Expression.Assign(Expression.Field(p_this, "Y"), p_y),
            Expression.Empty()
        ),
        p_this, p_y
    );
});
gen.DefineMethod("Func", typeof(int), new[] { typeof(int), }, t =>
{
    // メソッドの本文
    var p_this = Expression.Parameter(t);
    var p_z = Expression.Parameter(typeof(int));
    return Expression.Lambda(
        Expression.Add(
            Expression.Add(Expression.Field(p_this, "X"), Expression.Field(p_this, "Y")),
            p_z
        ),
        p_this, p_z
    );
});
gen.CreateType();
asm.Save("Test.dll");
```

上のコードを実行すると Test.dll が生成されます。以下は Test.dll を [ILSpy](http://wiki.sharpdevelop.net/ILSpy.ashx) で逆コンパイルした結果です:

```csharp
using System;
public class TargetType
{
  private class <Impl>
  {
    internal static void Prologue(TargetType targetType)
        {
            targetType.X = TargetType.<Impl>.X_Init(targetType);
        }
        internal static int X_Init(TargetType targetType)
        {
            return 123;
        }
        internal static void Ctor_Impl(TargetType targetType, int y)
        {
            targetType.Y = y;
        }
        internal static int Func_Impl(TargetType targetType, int num)
        {
            return targetType.X + targetType.Y + num;
        }
    }
    public int X;
    public int Y;
    public TargetType(int num)
    {
        TargetType.<Impl>.Prologue(this);
        TargetType.<Impl>.Ctor_Impl(this, num);
    }
    public int Func(int num)
    {
        return TargetType.<Impl>.Func_Impl(this, num);
    }
}
```

Test.dll を参照して、以下のコードを実行することで、アセンブリが正しく生成されていることが確認できます:

```csharp
TargetType x = new TargetType(30000);
Console.WriteLine(x.X);
Console.WriteLine(x.Y);
Console.WriteLine(x.Func(2000));
```

上のコードを実行すると、

```text
123
30000
32123
```

と実行され、期待した通りの結果となります。

## まとめ

以上によって、Expression Trees API と Emit API を組み合わせ、式木を用いて動的に型を定義することは一筋縄ではいかないが、可能ではあることが明らかになりました。

今回示したコードでは例えばプロパティの定義や static メンバの定義などが欠けていますが、以上に示した方針に依れば、これらも十分可能であると思われます。

## 最後に

C# Advent Calendar ということですが、なにやらかなり C# 分が少なくなってしまったような気がします…まあ、そこはご愛嬌ということで、お許し下さい！

一般に需要があるかどうかは激しく疑問ですが、Expression Trees や Emit API の使い方の一つとして、参考になれば幸いであります。

なお、この記事は私が制作している Expression Trees ベースのスクリプト言語 [Yacq](http://yacq.net) において[動的型生成機能](https://github.com/takeshik/yacq/blob/cd7159d34c09b35e72ceb9c6e35d15a2940392e7/Yacq/SystemObjects/YacqType.cs)を導入するにあたって得た知見を基に執筆されました。
