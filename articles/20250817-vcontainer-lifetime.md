---
title: "VContainer の Singleton/Scoped の挙動をみる"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["unity", "vcontainer"]
published: true
---

## TL;DR

- 私の理解が合っているかを確かめるために VContainer の挙動をみた
- Singleton:
  - `Register()` したスコープとその子孫のスコープで 1 つのインスタンスを使う
  - 子と親で同じ型を `Register()` できて、その時は自身か最も近い親のインスタンスを使う
- Scoped:
  - 各スコープで1つのインスタンスを使う
    - つまり、親と子では別々のインスタンスを使う
  - Singleton 同様に自身か最も近い親の `Register()` を見てインスタンス化する


## 準備

環境:
- Unity: 6000.0.25f1
- VContainer: v1.17.0

:::details コード全文
```cs
public class TestScript1
{
    readonly Guid instanceID;
    public string InstanceID => instanceID.ToString()[..7];

    public TestScript1()
    {
        instanceID = Guid.NewGuid();
    }
}

public class TestScript2 : IStartable
{
    readonly TestScript1 testScript1;
    readonly string lifetimeScopeName;

    public TestScript2(
        TestScript1 testScript1,
        string lifetimeScopeName)
    {
        this.testScript1 = testScript1;
        this.lifetimeScopeName = lifetimeScopeName;
    }

    void IStartable.Start()
    {
        Debug.Log($"{lifetimeScopeName} {testScript1.InstanceID}");
    }
}

public class ParentLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        builder.Register<TestScript1>(Lifetime.Singleton);
        //builder.Register<TestScript1>(Lifetime.Scoped);
        //builder.Register<TestScript1>(Lifetime.Transient);

        builder.Register<TestScript2>(Lifetime.Singleton)
            .AsImplementedInterfaces()
            .WithParameter("lifetimeScopeName", gameObject.name);
    }
}

public class Child1LifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        builder.Register<TestScript2>(Lifetime.Singleton)
            .AsImplementedInterfaces()
            .WithParameter("lifetimeScopeName", gameObject.name);
    }
}

public class Child2LifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        builder.Register<TestScript2>(Lifetime.Singleton)
            .AsImplementedInterfaces()
            .WithParameter("lifetimeScopeName", gameObject.name);
    }
}

public class Child3LifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        builder.Register<TestScript2>(Lifetime.Singleton)
            .AsImplementedInterfaces()
            .WithParameter("lifetimeScopeName", gameObject.name);
    }
}
```
:::

![](/images/20250817-vcontainer-lifetime/1.png)

 `TestScript1` がどのようにインスタンス化されるかをみていく。 `LifetimeScope` の親子関係は上の画像の GameObject の親子関係と同じにしている。



## Lifetime.Singleton の挙動

まずは `ParentLifetimeScope` で、 `TestScript1` を `Singleton` として `Register()` する。

```cs
public class ParentLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        builder.Register<TestScript1>(Lifetime.Singleton);

        builder.Register<TestScript2>(Lifetime.Singleton)
            .AsImplementedInterfaces()
            .WithParameter("lifetimeScopeName", gameObject.name);
    }
}
```

この場合は単純で、 [VContainer のドキュメント](https://vcontainer.hadashikick.jp/ja/scoping/lifetime-overview)にある通り `ParentLifetimeScope` 以下で同じインスタンスを使う。

![](/images/20250817-vcontainer-lifetime/2.png)

次は `ParentLifetimeScope` での `Register()` はそのままにして、 `Child1LifetimeScope` と `Child3LifetimeScope` でそれぞれ `Singleton` として `Register()` する。

```cs
public class Child1LifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        builder.Register<TestScript1>(Lifetime.Singleton);

        builder.Register<TestScript2>(Lifetime.Singleton)
            .AsImplementedInterfaces()
            .WithParameter("lifetimeScopeName", gameObject.name);
    }
}

public class Child3LifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        builder.Register<TestScript1>(Lifetime.Singleton);

        builder.Register<TestScript2>(Lifetime.Singleton)
            .AsImplementedInterfaces()
            .WithParameter("lifetimeScopeName", gameObject.name);
    }
}
```

これも[VContainer のドキュメント](https://vcontainer.hadashikick.jp/ja/scoping/lifetime-overview#lifetime%E3%81%A8%E3%82%B9%E3%82%B3%E3%83%BC%E3%83%97%E3%81%AE%E8%A6%AA%E5%AD%90%E9%96%A2%E4%BF%82)にある通り、子で `Register()` していれば親のインスタンスではなく子の `Register()` をみてインスタンスを生成して使う。

- `ParentLifetimeScope`
- `Child1LifetimeScope` 以下のすべての Scope
- `Child3LifetimeScope`

でそれぞれ 1 つずつ `TestScript1` のインスタンスが生成されているのが分かる。

![](/images/20250817-vcontainer-lifetime/3.png)

最後に、 `ParentLifetimeScope` を2つ複製して挙動を見てみる。
![](/images/20250817-vcontainer-lifetime/6.png)

最初の 3 行に複製した `ParentLifetimeScope` のログが並んでいて `instanceID` が異なるので、同じ型の `LifetimeScope` があるときは別々のインスタンスを使うことが分かる。
![](/images/20250817-vcontainer-lifetime/7.png)


## Lifetime.Scoped の挙動

複製した `ParentLifetimeScope` を消して、 `TestScript1` を `Scoped` として `Register()` して挙動をみる。

```cs
public class ParentLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        builder.Register<TestScript1>(Lifetime.Scoped);

        builder.Register<TestScript2>(Lifetime.Singleton)
            .AsImplementedInterfaces()
            .WithParameter("lifetimeScopeName", gameObject.name);
    }
}

public class Child1LifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        builder.Register<TestScript2>(Lifetime.Singleton)
            .AsImplementedInterfaces()
            .WithParameter("lifetimeScopeName", gameObject.name);
    }
}

public class Child3LifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        builder.Register<TestScript2>(Lifetime.Singleton)
            .AsImplementedInterfaces()
            .WithParameter("lifetimeScopeName", gameObject.name);
    }
}
```

`Lifetime.Scoped` は「自身かもっとも近い祖先の `Register()` を探し、コンテナはそれぞれがオブジェクトを生成して保持する」という挙動をする。ここでは `ParentLifetimeScope` のみで `Register()` しているので、 `LifetimeScope` ごとに別々のインスタンスを作っているのが分かる。

![](/images/20250817-vcontainer-lifetime/4.png)

ここで「自身かもっとも近い祖先の `Register()` を探し」の挙動を確認するために、 `TestScript1` に `IStartable` を実装させて、 `Child3LifetimeScope` でのみ実行されることをみる。 

```cs
public class TestScript1 : IStartable
{
    readonly Guid instanceID;
    public string InstanceID => instanceID.ToString()[..7];

    public TestScript1()
    {
        instanceID = Guid.NewGuid();
    }

    void IStartable.Start()
    {
        Debug.Log($"{instanceID} IStartable.Start()");
    }
}

public class Child3LifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        builder.Register<TestScript1>(Lifetime.Scoped)
            .AsImplementedInterfaces()
            .AsSelf();

        builder.Register<TestScript2>(Lifetime.Singleton)
            .AsImplementedInterfaces()
            .WithParameter("lifetimeScopeName", gameObject.name);
    }
}
```

`IStartable.Start()` が1度だけ呼ばれ `instanceID` も `Child3LifetimeScope` と一致しているので、 `Child3LifetimeScope` が自身の `Register()` に従って `TestScript1` をインスタンス化したことが分かる。

![](/images/20250817-vcontainer-lifetime/5.png)

## まとめ
[VContainer のドキュメント](https://vcontainer.hadashikick.jp/ja/scoping/lifetime-overview)の内容を VContainer を動かして確認した。

- Singleton:
  - `Register()` したスコープとその子孫のスコープで 1 つのインスタンスを使う
  - 子と親で同じ型を `Register()` できて、その時は自身か最も近い親のインスタンスを使う
- Scoped:
  - 各スコープで1つのインスタンスを使う
    - つまり、親と子では別々のインスタンスを使う
  - `Singleton` 同様に自身か最も近い親の `Register()` を見てインスタンス化する