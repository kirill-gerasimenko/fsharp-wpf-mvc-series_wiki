### Async event handlers 
There is no need to explain the importance of asynchronous programming. These days it's everywhere. F# was one of the first mainstream languages to provide [first-class](http://msdn.microsoft.com/en-us/library/dd233250.aspx) support for it. Plenty of [examples](http://msdn.microsoft.com/en-us/library/hh273070.aspx) using [ F# for async client-side](http://tomasp.net/blog/async-non-blocking-gui.aspx) development are available on the Web. Support for asynchronous programming has to be built-in into the framework. In the context of the framework it means first and foremost support for asynchronous event handlers. In typical GUI application most events are processed synchronously - therefore we need to explicitly support both options. This requires changing controller type definition: 

```ocaml
type EventHandler<'Model> = 
    | Sync of ('Model -> unit)
    | Async of ('Model -> Async<unit>)

type IController<'Events, 'Model> =

    abstract InitModel : 'Model -> unit
    abstract Dispatcher : ('Events -> EventHandler<'Model>)
```

`EventHandler` is extracted into the own type. It comes now in two flavors: synchronous and asynchronous. Asynchronous one uses F# `Async<'T>` type to express asynchronous computation. Both of them return unit to emphasize their side-effectful nature. Controller function that returns event handler for specific event was renamed `Dispatcher`. This is what it does: map(dispatch) event to handler. Type signature of `Dispatcher` now is very crisp too: read-only property of `('Event -> EventHandler<'Model>)`. 

`Mvc` type `Start` method implementation adopts changes introduced above: 
```ocaml
type Mvc<'Events, 'Model when 'Model :> INotifyPropertyChanged>(model : 'Model, view : IView<'Events, 'Model>, controller : IController<'Events, 'Model>) =
    
    member this.Start() =
        controller.InitModel model
        view.SetBindings model
        view.Subscribe (fun event -> 
            match controller.Dispatcher event with
            | Sync eventHandler -> eventHandler model
            | Async eventHandler -> Async.StartImmediate(eventHandler model)
        )
```

### Sample

We are going to use a call to [w3schools temperature converter service](http://www.w3schools.com/webservices/ws_example.asp) as an example of asynchronous I/O. 

Bottom section of the Calculator window is dedicated to issuing of async calls to the service. 

[[Images/AsyncController.png]]

It converts temperature from Celsius to Fahrenheit and back, showing request status either "Waiting for response...", or "Response received.". The artificial delay is added for demonstration purposes, because the response is still too quick to be easily noticeable. 

Service client was created using Visual Studio standard "Add Service Reference ..." dialog: 

[[Images/w3schooltempconverter.png]]

Because in Visual Studio 2010 async operations are generated (don't forget to click check-box) using old-fashion Begin/End style we'll use F# extensions methods and [Async.FromBeginEnd](http://msdn.microsoft.com/en-us/library/ee340508.aspx) to make integration with F# async smoother: 

```ocaml
...
type TempConvertSoapClient with
        
    member this.AsyncCelsiusToFahrenheit(celsius : float) = 
        async { 
            let! result = Async.FromBeginEnd(string celsius, this.BeginCelsiusToFahrenheit, this.EndCelsiusToFahrenheit) 
            return float result 
        } 
    
    member this.AsyncFahrenheitToCelsius(fahrenheit : float) = 
        async { 
            let! result = Async.FromBeginEnd(string fahrenheit, this.BeginFahrenheitToCelsius, this.EndFahrenheitToCelsius) 
            return float result 
        } 
```

Visual Studio 2012 generates more modern Task-based operations. 

Sample Events, Model, and View are trivial. Most interesting changes happen in `SampleController`: 
```ocaml
type SimpleController() = 
    inherit Controller<SampleEvents, SampleModel>()
    
    let service = new TempConvertSoapClient(endpointConfigurationName = "TempConvertSoap")
    
    override this.InitModel model = 
        ...
        model.TempConverterHeader <- "Async TempConveter"
        model.Delay <- 3
        ...

    override this.Dispatcher = function
        | Calculate -> Sync this.Calculate 
        | Clear -> Sync this.InitModel 
        | CelsiusToFahrenheit -> Async this.CelsiusToFahrenheit 
        | FahrenheitToCelsius -> Async this.FahrenheitToCelsius 
        ...
        
    member this.CelsiusToFahrenheit model = 
        async {
            let context = SynchronizationContext.Current
            ...
            model.TempConverterHeader <- "Async TempConverter. Waiting for response ..."            
            do! Async.Sleep(model.Delay * 1000)
            let! fahrenheit = service.AsyncCelsiusToFahrenheit model.Celsius
            do! Async.SwitchToContext context
            model.TempConverterHeader <- "Async TempConverter. Response received."            
            model.Fahrenheit <- fahrenheit
        }
        ...
```

The following details deserve our attention: 
  * Notice local "service" binding. It's what I call "operational" state. It should be in Controller as opposed to the visual state in `Model`. Another example would be a database connection, etc. Also, in real-world application [Dependency inversion principle](http://en.wikipedia.org/wiki/Dependency_inversion_principle) can be used to enable loose coupling and therefore ease of unit testing. 
  * Event handlers are prefixed with either `Sync`, or `Async` union cases.
  * Do not forget about `Async.SwitchToContext`. Tomas Petricek has an excellent [post](http://tomasp.net/blog/async-non-blocking-gui.aspx) on this subject. 
  * Notice that contrary to F#, C# 5.0 new async feature by default runs async continuations in [SynchronizationContext](http://msdn.microsoft.com/en-us/magazine/gg598924.aspx) where the computation was initially started. F# certainly allows having more control over context switches and, thus, caters to a more advanced user. I wonder if F# can provide a method like `Async.RunInContext` that performs the same kind of thing. [Don?](https://twitter.com/dsyme) Tomas has a [post](http://tomasp.net/blog/safe-gui-async.aspx) where he shows a different approach to ensure that continuation runs in the right context. 

Specialized `SyncController` is provided to support a very common case of sync-only event handlers: 
```ocaml
[<AbstractClass>]
type SyncController<'Events, 'Model>(view) =
    inherit Controller<'Events, 'Model>()

    abstract Dispatcher : ('Events -> 'Model -> unit)
    override this.Dispatcher = fun e -> Sync(this.Dispatcher e)
```

### Exception handling

So far we ignored important aspect - exception handling. In presence of asynchronous computation it becomes even more relevant. 

`Mvc<_, _>` type defines abstract `OnException` hook and channels both sync and async computation exception through it.

```ocaml
type Mvc... =

    member this.Start() =
        controller.InitModel model
        view.SetBindings model
        view.Subscribe (fun event -> 
            match controller.Dispatcher event with
            | Sync eventHandler ->
                try eventHandler model 
                with exn -> this.OnException(event, exn)
            | Async eventHandler -> 
                Async.StartWithContinuations(
                    computation = eventHandler model, 
                    continuation = ignore, 
                    exceptionContinuation = (fun exn -> this.OnException(event, exn)),
                    cancellationContinuation = ignore
                )
        )

    abstract OnException : 'Events * exn -> unit
    default this.OnException(_, exn) = ... 
```
If you're happy with default implementation (which re-throws exception __preserving stack trace__) then most sensible way to handle exceptions is to provide callback for global application-wide [Application.DispatcherUnhandledException](http://msdn.microsoft.com/en-us/library/system.windows.application.dispatcherunhandledexception.aspx). 
```ocaml
    ...
    let app = Application()
    app.DispatcherUnhandledException.Add <| fun args ->
        let why = args.Exception
        Debug.Fail("DispatcherUnhandledException handler", string why.Message)
        args.Handled <- true
    ...
```
The alternative to global handler is to override `OnException` by providing Mvc subtype or creating local definition using [object expressions](http://msdn.microsoft.com/en-us/library/dd233237.aspx) (see example below). This is certainly more powerful because override will have access not only to exception instance but also event, model, view and controller. For example, by logging event and model state (maybe also some controller state), exception can be easily reproduced.  

Some noted about default implementation of `OnException`. There are [two ways](http://stackoverflow.com/questions/57383/in-c-how-can-i-rethrow-innerexception-without-losing-stack-trace) to do it on .NET 4.0. Here we use call to undocumented "InternalPreserveStackTrace". It's done in hope that most users will be able switch to .NET 4.5 and use fully supported [ExceptionDispatchInfo](http://msdn.microsoft.com/en-us/library/system.runtime.exceptionservices.exceptiondispatchinfo.aspx). As hypothetical example of custom exception-handling strategy, let's say you're stuck on .NET 4.0 and you don't like using undocumented methods. Let's use exception-wrapper approach (F# has a bit nicer way than C#  to define and unwrap those).
```ocaml
...
exception PreserveStackTraceWrapper of exn

type System.Exception with
    member this.Unwrap() = 
        match this with
        | PreserveStackTraceWrapper inner -> inner.Unwrap()
        | exn -> exn
...
let mvc = {
    new Mvc<_, _>(model, view, controller) with
        member this.OnException(_, exn) = 
            let wrapperExn = match exn with | PreserveStackTraceWrapper _  -> exn | inner -> PreserveStackTraceWrapper inner
            raise wrapperExn
}
let app = Application()
app.DispatcherUnhandledException.Add <| fun args ->
    let why = args.Exception.Unwrap()
    Debug.Fail("DispatcherUnhandledException handler", string why.Message)
    args.Handled <- true
...
```
In this example override is local object expression, but `Mvc<_, _>` subtype can be defined as well. Using this subtype through application will make it effectively global strategy. 

To test exception handling use "Kaboom !" button or disconnect from network before call to temperature converter service.

### Cancellation
I assume the reader is familiar with a way cancellation is handled in F# async workflows. For more details look at [ Chris's book](http://www.amazon.com/Programming-comprehensive-writing-complex-problems/dp/0596153643/) p.271 and PFX team blog [ post](http://blogs.msdn.com/b/pfxteam/archive/2009/05/22/9635790.aspx). In the current design we start all async workflows without explicitly specifying `CancellationToken` in `Async.StartWithContinuations` call. This means that shared global `CancellationToken` will be used. The only way to cancel it is to call `Async.CancelDefaultToken` method which cancels all running asynchronous workflows (ones that share the same token). "Cancel Async" button initiates the cancellation. 
```ocaml
    override this.Dispatcher = function
        ...
        | CelsiusToFahrenheit -> Async this.CelsiusToFahrenheit
        | FahrenheitToCelsius -> Async(fun model -> 
            let context = SynchronizationContext.Current
            Async.TryCancelled(
                computation = this.FahrenheitToCelsius model,
                compensation = fun error -> 
                    context.Post((fun _ -> model.TempConverterHeader <- "Async TempConverter. Request cancelled."), null) 
            ))
        | CancelAsync -> Sync(ignore >> Async.CancelDefaultToken)
        ...
```
Often an application needs some compensation function to run as a reaction to async computation cancellation. It can be done either through [Async.TryCancelled](http://msdn.microsoft.com/en-us/library/ee370399.aspx) (see FahrenheitToCelsius case above) combinator or inside workflow by calling [ Async.OnCancel method](http://msdn.microsoft.com/en-us/library/ee340460.aspx) (see CelsiusToFahrenheit handler below). In our specific example we set status message to "... Request cancelled." 
```ocaml
...
    member this.CelsiusToFahrenheit model = 
        async {
            let context = SynchronizationContext.Current
            use! cancelHandler = Async.OnCancel(fun() -> 
                context.Post((fun _ -> model.TempConverterHeader <- "Async TempConverter. Request cancelled."), null)) 
            model.TempConverterHeader <- "Async TempConverter. Waiting for response ..."            
            do! Async.Sleep(model.Delay * 1000)
            let! fahrenheit = service.AsyncCelsiusToFahrenheit model.Celsius
            do! Async.SwitchToContext context
            model.TempConverterHeader <- "Async TempConverter. Response received."            
            model.Fahrenheit <- fahrenheit
        }
...
```
Because compensation function makes changes to Model we need to ensure execution in the proper context. Simulated delay comes before web service call in the workflow so that user is  able to see the clear effect of cancellation - network call simply does not take place. 

_Ideas presented below are not fully-developed mostly because I didn't have a chance using them in a real-world application. For the same reason they did not make it into the framework. Still, they may have value for a curious reader. Feel free to try them out._

Cancelling the most recent set of asynchronous computations is ideal most of the times, but can cause issues if your application wants to cancel async workflows on individual basis. To be able to cancel an arbitrary asynchronous workflow you’ll need to create and keep track of a `CancellationTokenSource` object. Along with async computation event handler returns `CancellationToken` instance of the computation it's running with. 
```ocaml
open System.Threading
    
type EventHandler<'Model> = 
    ...
    | Async of ('Model -> Async<unit> * CancellationToken)
...
type Mvc ...
    ...
    member this.Start model =
        ...
            | Async eventHandler -> 
                let computation, ct = eventHandler model
                Async.StartWithContinuations(
                    computation, 
                    continuation = ignore, 
                    exceptionContinuation = (fun exn -> this.OnException(event, exn)),  
                    cancellationContinuation = ignore
                    cancellationToken = ct
                )
...

type SimpleController ...
    ...
    let mutable cancellationSource : CancellationTokenSource = null
    
    override this.Dispatcher = function
        ...
        | CancelAsync -> Sync(fun _ -> if cancellationSource <> null then cancellationSource.Cancel())
    ...
    member this.CelsiusToFahrenheit model = 
        cancellationSource <- new CancellationTokenSource()
        async {
            ...
        },
        cancellationSource.Token
    ...
```
Explicit control over `CancellationTokenSource` makes all kinds of scenarios possible: selective, grouped, linked cancellations. 

### Async model initalization

Async model initialization is another speculative feature. It doesn't mean it's useless but I didn't put it through a real-life test. 

A scenario where loading all data required for initialization takes significant time sounds like a real one. Necessity to reduce start-up time in exchange to gradually enabling functionality as data gets available can be quite important. Be prepared to write a complex logic dealing with a partial initialization. 

A possible solution is splitting up `InitModel` into 2 pieces: the first, synchronous, loads data that are absolutely required before `View` shows up, and the second initializes data that can be loaded asynchronously. Async part is executed by `Async.StartImmediate` after the bindings are set. Speaking of bindings, sync part should set all Model properties that are loaded asynchronously to some reasonable default values. 

To implement custom controller with async `InitModel` inherit from new base class `AsyncInitController<_, _>` and override `Dispatcher` and `InitModel`. If you don't like to be constraint by subtyping just create type that has two members with appropriate names (`Dispatcher` and `InitModel`) and types then use `AsyncInitController.Create` factory method.
```ocaml
[<AbstractClass>]
type AsyncInitController<'Events, 'Model>() =
    inherit Controller<'Events, 'Model>()

    abstract InitModel : 'Model -> Async<unit>
    override this.InitModel model = model |> this.InitModel |> Async.StartImmediate

    static member inline Create(controller : ^Controller) = {
        new IController<'Events, 'Model> with
            member this.InitModel model = (^Controller : (member InitModel : 'Model -> Async<unit>) (controller, model)) |> Async.StartImmediate
            member this.Dispatcher = (^Controller : (member Dispatcher : ('Events -> EventHandler<'Model>)) controller)
    } 
```
As example of async model initialization, `SampleController` counts files in "...\ProgramFiles" folder (which is usually a lot). 
```ocaml
type SampleController() = 
    inherit AsyncInitController<SampleEvents, SampleModel>()
    ...
    override this.InitModel(model : SampleModel) = 
        ...
        let folderToSearch = Environment.GetFolderPath Environment.SpecialFolder.ProgramFiles
        model.Title <- sprintf "Files in %s: ..." folderToSearch

        async {
            try 
                let context = SynchronizationContext.Current
                do! Async.SwitchToThreadPool()
                let totalFiles = Directory.GetFiles(folderToSearch, "*.*", SearchOption.AllDirectories).Length
                do! Async.SwitchToContext context
                model.Title <- sprintf "Files in %s: - %i" folderToSearch totalFiles
            with e ->
                System.Diagnostics.Debug.WriteLine e.Message
                model.Title <- sprintf "Failed to count files in in %s." folderToSearch 
        }
    
``` 
Exceptions are still possible. It's reasonable to have `try ... with` clause inside `async { ... }` block and to handle it there. For example, unless you run this chapter application with administrative credential it fails to access "ProgramFiles" folder and report it to window title. 

Cancellation doesn't makes sense in context of async `InitModel`.