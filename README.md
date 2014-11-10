RxUI Design Guidelines
======================

A set of practices to help maintain sanity when building an RxUI client 
application distilled from hard lessons learned.

## Best Practices

The following recommendations are intended to help keep ReactiveUI code 
predictable, understandable, and fast.

They are, however, only guidelines. Use best judgment when determining whether 
to apply the recommendations here to a given piece of code.

### Commands

Prefer binding user interactions to commands rather than methods.

__Do__

```csharp
// In XAML
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

__Don't__

Use the Caliburn.Micro conventions for associating buttons and commands:

```csharp
// In XAML
<Button x:Name="Delete" .../>

public class RepositoryViewModel : PropertyChangedBase
{
  public void Delete() {...}	
}
```

Why? 

1. ReactiveCommand exposes the `CanExecute` property of the command to 
enable applications to introduce additional behaviour.
2. It handles marshaling the result back to the UI thread.
3. It tracks in-flight items.


#### Command Names

Don't suffix `ReactiveCommand` properties' names with `Command`; instead, name the property using a verb that describes the command's action. For example:

```csharp
	
public ReactiveCommand Synchronize { get; private set; }

// and then in the ctor:

Synchronize = ReactiveCommand.CreateAsyncObservable(
  _ => SynchronizeImpl(mergeInsteadOfRebase: !IsAhead));

```

When a `ReactiveCommand`'s implementation is too large or too complex for an anonymous delegate, name the implementation's method the same name as the command, but with `Impl` suffixed (for example, `SychronizeImpl` above).

### UI Thread and Schedulers

Always make sure to update the UI on the `RxApp.MainThreadScheduler` to ensure UI  changes happen on the UI thread. In practice, this typically means making sure to update view models on the main thread scheduler.

__Do__

```csharp
FetchStuffAsync()
  .ObserveOn(RxApp.MainThreadScheduler)
  .Subscribe(x => this.SomeViewModelProperty = x);
```
__Don't__

```csharp
FetchStuffAsync()
  .Subscribe(x => this.SomeViewModelProperty = x);
```

Even better, pass the scheduler to the asynchronous operation - this is often
necessary for more complex tasks.

__Better__

```csharp
FetchStuffAsync(RxApp.MainThreadScheduler)
  .Subscribe(x => this.SomeViewModelProperty = x);
```

### Prefer Observable Property Helpers to setting properties explicitly

When a property's value depends on another property, a set of properties, or an 
observable stream, rather than set the value explicitly, use 
`ObservableAsPropertyHelper` with `WhenAny` wherever possible.

__Do__

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

__Don't__

```csharp
FetchStuffAsync()
  .ObserveOn(RxApp.MainThreadScheduler)
  .Subscribe(x => this.SomeViewModelProperty = x);
```

#### Why?

 - `ObservableAsPropertyHelper` will take care of raising `INotifyPropertyChanged`
   events - if you're creating read-only properties, this can save so much boilerplate
   code.
 - `ObservableAsPropertyHelper` will use `MainThreadScheduler` to schedule subscribers,
  unless specified otherwise - no need to remember to do this yourself.
 - `WhenAny` lets you combine multiple properties, treat their changes as observable
  streams, and craft ViewModel-specific outputs.

### Almost always use `this` as the left hand side of a `WhenAny` call.

__Do__

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

__Don't__

```csharp
public class MyViewModel(IDependency dependency)
{
  stuff = dependency.WhenAny(x => x.Stuff, x => x.Value)
    .ToProperty(this, x => x.Stuff);
}
```

#### Why?

 - the lifetime of `dependency` is unknown - if it is a singleton it
 could introduce memory leaks into your application

### Use descriptive variables in your `WhenAny`s

In situations where you are detecting changes in multiple expressions, ensure you name the variables in the `selector` 

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

__Don't__

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
  
  // other code here
}
```

#### Why?

This helps greatly with the readability of complex expressions, particularly when working with boolean values.
