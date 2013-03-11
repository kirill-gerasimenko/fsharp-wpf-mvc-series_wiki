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
type SyncController<'Event, 'Model when 'Model :> INotifyPropertyChanged>(view) = 
    inherit Controller<'Event, 'Model>(view) 
    
    abstract Dispatcher : ('Event -> 'Model -> unit) 
    override this.Dispatcher = fun e -> Sync(this.Dispatcher e) 
``` 