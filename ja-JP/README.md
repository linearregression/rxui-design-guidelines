<!--RxUI Design Guidelines-->
RxUI 設計ガイドライン
======================

<!--A set of practices to help maintain sanity when building an RxUI client 
application distilled from hard lessons learned.-->
厳しい教訓から蒸留された、RxUI のアプリケーションを構築する際に正気を保つためのプラクティス群

<!-- ## Best Practices-->
##ベストプラクティス

<!--The following recommendations are intended to help keep ReactiveUI code 
predictable, understandable, and fast.-->
以下は ReactiveUI のコードを、予測可能で理解しやすく効率がよくなるように手助けするための推奨事項だ。
<!--They are, however, only guidelines. Use best judgment when determining whether 
to apply the recommendations here to a given piece of code.-->
しかし、これらはガイドラインに過ぎない。コードに推奨事項を適用するかどうか、最良の判断をすること。

<!--- ### Commands-->
### コマンド

<!--Prefer binding user interactions to commands rather than methods.-->
メソッドよりもコマンドにユーザーインターフェースをバインドするほうが好ましい。

<!--__Do__-->
__よい__

```csharp
// XAML
<Button Command="{Binding Delete}" .../>

public class RepositoryViewModel : ReactiveObject
{
  public RepositoryViewModel() 
  {
    Delete = ReactiveCommand.CreateAsyncObservable(x => DeleteImpl());
    Delete.ThrownExceptions.Subscribe(ex => /*...*/);
  }

  public ReactiveAsyncCommand Delete { get; private set; }

  public IObservable<Unit> DeleteImpl() {...}
}
```

<!--__Don't__-->
__悪い__

<!--Use the Caliburn.Micro conventions for associating buttons and commands:-->
Caliburn.Micro の convention を使用してボタンとコマンドを関連付ける:

```csharp
// XAML
<Button x:Name="Delete" .../>

public class RepositoryViewModel : PropertyChangedBase
{
  public void Delete() {...}	
}
```

<!--Why? -->
#### なぜ？

<!-- 1. ReactiveCommand exposes the `CanExecute` property of the command to 
enable applications to introduce additional behaviour.-
 2. It handles marshaling the result back to the UI thread.
 3. It tracks in-flight items.-->
1. ReactiveCommand は追加の振る舞いに、コマンドを使用可能にするための `CanExecute` プロパティを公開する。
2. 結果を UI スレッドに戻すようマーシャリングする。
3. 実行中の項目を追跡する。



<!-- #### Command Names-->
#### コマンド名

<!--Don't suffix `ReactiveCommand` properties' names with `Command`; instead, name the property using a verb that describes the command's action. For example:-->
`ReactiveCommand` プロパティの名称の接尾辞に `Command` を使用しない; 代わりにコマンドの動作を説明する動詞を使う。例えば:

```csharp
	
public ReactiveCommand Synchronize { get; private set; }

// コンストラクタ内

Synchronize = ReactiveCommand.CreateAsyncObservable(
  _ => SynchronizeImpl(mergeInsteadOfRebase: !IsAhead));

```

<!-- When a `ReactiveCommand`'s implementation is too large or too complex for an anonymous delegate, name the implementation's method the same name as the command, but with `Impl` suffixed (for example, `SychronizeImpl` above).-->
匿名メソッドに対して `ReactiveCommand` の実装が大きすぎたり、複雑すぎるときは、実装名をコマンドの名称に接尾辞 Impl をつけたものにする (例えば、`SychronizeImpl`)。

<!-- ### UI Thread and Schedulers -->
### UI スレッドとスケジューラー

<!--Always make sure to update the UI on the `RxApp.MainThreadScheduler` to ensure UI  changes happen on the UI thread. In practice, this typically means making sure to update view models on the main thread scheduler.-->
UI スレッド上で UI の変更が発生するように、`RxApp.MainThreadScheduler` 上で更新されることを常に確認する。

<!--__Do__-->
__よい__

```csharp
FetchStuffAsync()
  .ObserveOn(RxApp.MainThreadScheduler)
  .Subscribe(x => this.SomeViewModelProperty = x);
```
<!--__Don't__-->
__悪い__

```csharp
FetchStuffAsync()
  .Subscribe(x => this.SomeViewModelProperty = x);
```

<!--Even better, pass the scheduler to the asynchronous operation - this is often
necessary for more complex tasks.-->
スケジューラーを非同期処理に渡すと更によい - これはしばしば、より複雑な処理で必要となる。

<!--__Better__-->
__よりよい__

```csharp
FetchStuffAsync(RxApp.MainThreadScheduler)
  .Subscribe(x => this.SomeViewModelProperty = x);
```

<!--　### Prefer Observable Property Helpers to setting properties explicitly-->
### Observable プロパティヘルパーを使用して明示的なプロパティ値を設定する

<!-- When a property's value depends on another property, a set of properties, or an 
observable stream, rather than set the value explicitly, use 
`ObservableAsPropertyHelper` with `WhenAny` wherever possible.-->
プロパティの値が他のプロパティやプロパティ群、observable ストリームに依存するとき、明示的に値を設定するよりも、
可能な限り `ObservableAsPropertyHelper` と `WhenAny` を合わせて使用する。

<!--__Do__-->
__よい__

```csharp
public class RepositoryViewModel : ReactiveObject
{
  public RepositoryViewModel()
  {
    someViewModelProperty = this.WhenAny(x => x.StuffFetched, y => y.OtherStuffNotBusy, 
         (x, y) => x && y)
      .ToProperty(this, x => x.CanDoIt, out canDoIt);
  }

  readonly ObservableAsPropertyHelper<bool> canDoIt;
  public bool CanDoIt
  {
    get { return canDoIt.Value; }  
  }	
}
```

<!--__Don't__-->
__悪い__

```csharp
FetchStuffAsync()
  .ObserveOn(RxApp.MainThreadScheduler)
  .Subscribe(x => this.SomeViewModelProperty = x);
```

<!-- #### Why?-->
#### なぜ？

<!-- - `ObservableAsPropertyHelper` will take care of raising `INotifyPropertyChanged`
   events - if you're creating read-only properties, this can save so much boilerplate
   code. 
 - `ObservableAsPropertyHelper` will use `MainThreadScheduler` to schedule subscribers,
  unless specified otherwise - no need to remember to do this yourself. 
 - `WhenAny` lets you combine multiple properties, treat their changes as observable
  streams, and craft ViewModel-specific outputs. 
 ### Almost always use `this` as the left hand side of a `WhenAny` call. -->
 - `ObservableAsPropertyHelper` は `INotifyPropertyChanged` イベントの発生を手掛ける。もし読み取り専用のプロパティを作成するのなら、
  多くのボイラープレートコードを省略できる。
 - `ObservableAsPropertyHelper` は他に指定が無い限り、スケジュールの購読者のために `MainThreadScheduler` を使用する。
 - `WhenAny` は複数のプロパティを結合し、それらの変更を observable ストリームとして扱い、ビューモデル特有の出力を作り出す。

### ほとんどの場合において、`this` を `WhenAny` 呼び出しの左側に使用する。

<!--__Do__-->
__よい__

```csharp
public class MyViewModel
{
  public MyViewModel(IDependency dependency)
  {
    Ensure.ArgumentNotNull(dependency, "dependency");

    this.Dependency = dependency;

    this.stuff = this.WhenAny(x => x.Dependency.Stuff, x => x.Value)
      .ToProperty(this, x => x.Stuff);
  }

  public IDependency Dependency { get; private set; }

  readonly ObservableAsPropertyHelper<IStuff> stuff;
  public IStuff Stuff
  {
    get { return this.stuff.Value; }
  }
}
```

<!--__Don't__-->
__悪い__

```csharp
public class MyViewModel(IDependency dependency)
{
  stuff = dependency.WhenAny(x => x.Stuff, x => x.Value)
    .ToProperty(this, x => x.Stuff);
}
```

<!-- #### Why? -->
#### なぜ？

<!-- - the lifetime of `dependency` is unknown - if it is a singleton it
 could introduce memory leaks into your application -->
 - `dependency` の生存時間が不明 - もしそれがシングルトンなら、メモリリークをあなたのアプリケーションに取り込んでしまう

<!-- ### Use descriptive variables in your `WhenAny`s-->
### `WhenAny` 内では説明的な変数を使用する

<!-- In situations where you are detecting changes in multiple expressions, ensure you name the variables in the `selector` -->
複数の式から変更を検出するようなときは、`selector` の変数名を確保する

```csharp
public class MyViewModel : ReactiveObject
{
  public MyViewModel()
  {
    this.WhenAny(
        x => x.SomeProperty.IsEnabled,
        x => x.SomeProperty.IsLoading,
        (isEnabled, isLoading) => isEnabled.Value && isLoading.Value)
        .Subscribe(_ => DoSomething());
  }
  
  // other code here
}
```

<!-- __Don't__-->
__悪い__

```csharp
public class MyViewModel : ReactiveObject
{
  public MyViewModel()
  {
    this.WhenAny(
        x => x.SomeProperty.IsEnabled,
        x => x.SomeProperty.IsLoading,
        (x, y) => x.Value && y.Value)
        .Subscribe(_ => DoSomething());
  }
  
  // その他のコード
}
```

<!-- #### Why? -->
#### なぜ？

<!-- This helps greatly with the readability of complex expressions, particularly when working with boolean values.-->
これは bool 値が動作するような複雑な式を読みやすくする。
