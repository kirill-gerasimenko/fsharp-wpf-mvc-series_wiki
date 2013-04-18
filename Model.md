Let's begin with the first constituent of MVC - Model.

Model has two primary responsibilities:

1. It serves as a state container.
2. It implements [INotifyPropertyChanged](http://msdn.microsoft.com/en-us/library/ms229614.aspx) which is required for [data binding](http://martinfowler.com/eaaDev/DataBinding.html)-based solution.

### INotifyPropertyChanged implementation
There are [many ways](http://10rem.net/blog/2010/12/16/strategies-for-improving-inotifypropertychanged-in-wpf-and-silverlight) to implement `INotifyPropertyChanged`, including some  [minor enhancements] (http://jesseliberty.com/2012/06/28/c-5making-inotifypropertychanged-easier/) introduced in C# 5.0. Hopefully, [Roslyn] (http://msdn.microsoft.com/en-us/vstudio/roslyn.aspx) is going to be a technology to further improve the matter. 

For this framework I chose an approach that doesn't require writing implementation code (just following the convention is sufficient) and provides acceptable performance: dynamic proxy interception (specifically, one from [Castle.Core](http://www.castleproject.org/)). This approach is not novel at all - [people](http://lostechies.com/rayhouston/2009/06/03/fluent-silverlight-auto-wiring-inotifypropertychanged/) have been using it for quite some time. [Different](https://gist.github.com/166110) techniques exist, but the idea is pretty much the same - make notification from inside of property set to call interceptor. Important limitation to remember is: a property eligible for interception must be defined as virtual (or abstract). As long as we don't offer any new approach, the best we can do is implementing the existing one in more concise and readable way using F#. Let me post the  whole implementation here because it's relatively short: 
```ocaml
[<AbstractClass>]
type Model() = 

    static let proxyFactory = ProxyGenerator()

    static let notifyPropertyChanged = {
        new StandardInterceptor() with
            member this.PostProceed invocation = 
                match invocation.Method.Name.Split('_'), invocation.InvocationTarget with 
                    | [| "set"; propertyName |], (:? Model as model) -> model.TriggerPropertyChanged propertyName
                    | _ -> ()
    }

    let propertyChangedEvent = Event<_, _>()

    interface INotifyPropertyChanged with
        [<CLIEvent>]
        member this.PropertyChanged = propertyChangedEvent.Publish

    member internal this.TriggerPropertyChanged propertyName = 
        propertyChangedEvent.Trigger(this, PropertyChangedEventArgs propertyName)

    static member Create<'T when 'T :> Model and 'T : not struct>() : 'T = 
        proxyFactory.CreateClassProxy notifyPropertyChanged
```

To be fair, the equivalent C# implementation would not be much longer. I would like to comment on some code-level stylistic differences between F# and C#:
 * F# opts-in for immutability. There is no need to declare `static readonly ProxyGenerator proxyFactory = new ProxyGenerator()`. It is such by default. 
 * C# solution would have an extra class `NotifyPropertyChanged` derived from `StandardInterceptor`. In F# we use an object expression. 
 * Notice how many things the powerful pattern matching does inside `PostProceed` method :
  1. It checks if method starts from `set_` (this is a rather naive check for property setter but it just works) 
  2. It extracts the property name 
  3. It checks if the model is of type `Model`  
  4. It downcasts model and assigns it to the named value 
  This one F# expression is equivalent to 3-4 C# statements. 
 * F# event implementation takes care of ridiculous null-check in typical C# code:
```c#
...
    if (this.PropertyChanged != null)
        this.PropertyChanged(this, new PropertyChangedEventArgs(propertyName));
...
```
  Just call `propertyChangedEvent.Trigger`.
 * Last but not the least: there is only a single type annotation in the code – at the definition of `Create` method. The powerful F# type inference figures out all other types. 

### Sample model 
As a sample project I chose implementing a calculator with some additional utility features. Though some of these won't make sense in real-life projects, they are just complex enough to demonstrate the solution technology. 
Here is the partial sample model. 
```ocaml
type SampleModel() = 
    inherit Model()
    
    let mutable x = 0
    let mutable y = 0
    ...
    abstract X : int with get, set
    default this.X with get() = x and set value = x <- value
    
    abstract Y : int with get, set
    default this.Y with get() = y and set value = y <- value
    ...
```
It is somewhat awkward compared with more succinct C# version: 
```ocaml
    public class SampleModel: Model
    {
        public virtual int X { get; set; }
        public virtual int Y { get; set; }
        ...
    }
```
To define a virtual property and a backing storage F# syntax requires 2 lines of code (abstract + default). F# 3.0 [auto-implemented] (http://msdn.microsoft.com/en-us/library/dd483467.aspx) properties won't be of much help because they are not supported for virtual properties. It would be nice if in F# we could just declare properties as in C#. Let me introduce ...

### Abstract properties support
Here is result model:
```ocaml
[<AbstractClass>]
type SampleModel() = 
    inherit Model()
    
    abstract X : int with get, set
    abstract Y : int with get, set
...
```
Sweet ! As long as compiler doesn't provide any backing storage we need to do it ourselves. The simplest way to do this is to have another set/get property interceptor backed by a dictionary. 
```ocaml
[<AbstractClass>]
type Model() = 
...
    static member Create<'T when 'T :> Model and 'T : not struct>() : 'T = 
        let interceptors : IInterceptor[] = [| notifyPropertyChanged; AbstractProperties() |]
        proxyFactory.CreateClassProxy interceptors    
            
and AbstractProperties() =
    let data = Dictionary()
    
    interface IInterceptor with
        member this.Intercept invocation = 
            match invocation.Method with
 
                | Abstract & PropertySetter propertyName -> 
                    data.[propertyName] <- invocation.Arguments.[0]
                
                
                | Abstract & PropertyGetter propertyName ->
                    match data.TryGetValue propertyName with 
                    | true, value -> invocation.ReturnValue <- value 
                    | false, _ -> 
                        let returnType = invocation.Method.ReturnType
                 
                         if returnType.IsValueType then 
                            invocation.ReturnValue <- Activator.CreateInstance returnType
                
                | _ -> invocation.Proceed()
```
It makes sense to define `AbstractProperties` as a separate type because it's not as small as `notifyPropertyChanged`. `Model.Create` attaches a new instance of the interceptor because it holds model instance-specific state. This interceptor carries a piece of logic similar to `notifyPropertyChanged`: it checks if the method is a property setter and extracts property name. The best way to achieve reuse here is to apply [active patterns](http://msdn.microsoft.com/en-us/library/dd233248.aspx).  
```ocaml
module MethodInfo = 
    let (|PropertySetter|_|) (m : MethodInfo) =
        match m.Name.Split('_') with
        | [| "set"; propertyName |] -> assert m.IsSpecialName; Some propertyName
        | _ -> None
    
    let (|PropertyGetter|_|) (m : MethodInfo) =
        match m.Name.Split('_') with
        | [| "get"; propertyName |] -> assert m.IsSpecialName; Some propertyName
        | _ -> None
    
    let (|Abstract|_|) (m : MethodInfo) = if m.IsAbstract then Some() else None
```
[Active](http://blogs.msdn.com/b/chrsmith/archive/2008/02/21/introduction-to-f_2300_-active-patterns.aspx) patterns represent THE WAY to extend [pattern matching] (http://fsharpnews.blogspot.com/2009/08/advantages-of-pattern-matching.html) in F#. Together they enable to write readable, composable (thus reusable) and highly compiler-optimized code. Just try to rewrite the code for `AbstractProperties` in C# - and you'll see the difference. 