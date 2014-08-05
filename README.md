RxUi Design Guidelines
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

public class RepositoryViewModel : ViewModelBase 
{
  public RepositoryViewModel() 
  {
    Delete = new ReactiveAsyncCommand();
    Delete.RegisterAsyncObservable(
      x => Delete(),  
      e => /* Do something with error */)
    .Subscribe();
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

1. ReactiveAsyncCommand exposes the `CanExecute` property of the command to 
enable applications to introduce additional behaviour.
2. It handles marshaling the result back to the UI thread.
3. It tracks in-flight items.


#### Command Names

Don't suffix `ReactiveCommand` properties' names with `Command`; instead, name the property using a verb that describes the command's action. For example:

```csharp
	
public IReactiveCommand Synchronize { get; private set; }

// and then in the ctor:

Synchronize = new ReactiveCommand();
Synchronize.RegisterAsync(_ => SynchronizeImpl(mergeInsteadOfRebase: !IsAhead))
    .Subscribe();

```

When a `ReactiveCommand`'s implementation is too large or too complex for an anonymous delegate, name the implementation's method the same name as the command, but with `Impl` suffixed (for example, `SychronizeImpl` above).

### UI Thread and Schedulers

Always make sure to update the UI on the `RxApp.DeferredScheduler` to ensure UI 
changes happen on the UI thread. In practice, this typically means making sure 
to update view models on the deferred scheduler.

__Do__

```csharp
.FetchStuffAsync()
.ObserveOn(RxApp.DeferredScheduler)
.Subscribe(x => this.SomeViewModelProperty = x);
```
__Don't__

```csharp
.FetchStuffAsync()
.Subscribe(x => this.SomeViewModelProperty = x);
```

Even better, pass in the scheduler to methods that take one in.

__Better__

```csharp
.FetchStuffAsync(RxApp.DeferredScheduler)
.Subscribe(x => this.SomeViewModelProperty = x);
```

### Prefer Observable Property Helpers to setting properties explicitly

When a property's value depends on another property, a set of properties, or an 
observable stream, rather than set the value explicitly, use 
`ObservableAsPropertyHelper` with `WhenAny` wherever possible.

__Do__

```csharp
public class RepositoryViewModel : ViewModelBase 
{
  ObservableAsPropertyHelper<bool> canDoIt;

  public RepositoryViewModel() 
  {
    someViewModelProperty = this.WhenAny(x => x.StuffFetched, 
									 y => y.OtherStuffNotBusy, 
									 (x, y) => x && y)
      .ToProperty(this, x => x.CanDoIt, setViaReflection: false);
  }

  public bool CanDoIt
  {
    get { return canDoIt.Value; }  
  }	
}
```

__Don't__

```csharp
.FetchStuffAsync()
.ObserveOn(RxApp.DeferredScheduler)
.Subscribe(x => this.SomeViewModelProperty = x);
```

### Prefer using `this` as the left hand side of a `WhenAny` call.

__Do__

```csharp
IDependency dependency;
public IDependency Dependency
{
  get { return this.dependency; }
  set { this.RaiseAndSetIfChanged(ref dependency, value); }
}

IObservableAsPropertyHelper<IStuff> stuff;

public IStuff Stuff
{
  get { return this.stuff.Value; }
}

public class MyViewModel(IDependency dependency)
{
  Ensure.ArgumentNotNull(dependency, "dependency");
  
  this.Dependency = dependency;
  
  this.stuff = this.WhenAny(x => x.Dependency.Stuff, x => x.Value)
    .ToProperty(this, x => x.Stuff);
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
