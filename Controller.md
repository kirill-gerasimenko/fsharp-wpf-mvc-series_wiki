Controller is the key component of MVC-triple because it contains most (if not all) presentation logic. It's essential responsibility is to execute specific computation as respond to event while reading/updating model state in process. A simplest variation of controller is event handling function of type `('Events -> 'Model -> unit)`. It can be either standalone function or type member.

Most of screens in GUI applications when first opened are not empty but show some information to start with. It means that view model needs to be initialized. I chose to go against the initial instinct of putting `Model` initialization logic into Model's constructor. If taken, it would force `Model` to have smelly dependencies on external resources like database connections, web services, etc. and eventually testability will suffer. Placing this code into Controller's `InitModel` and employing [Dependency inversion principle](http://en.wikipedia.org/wiki/Dependency_inversion_principle) really helps. 

With requirements above in mind controller interface can defined as following:
```ocaml
type IController<'Events, 'Model> =

    abstract InitModel : 'Model -> unit
    abstract EventHandler : ('Events -> 'Model -> unit)
```
Controller has dependency only on `Events`. It means loose coupling and as result testability. 

Note that `EventHandler` is not a method but a property of type “function”. It allows to use neat [Pattern matching function](http://msdn.microsoft.com/en-us/library/dd233242.aspx) syntax in derived type implementation (see `SampleController` below). Also, it signifies its role as event handler factory (See [[Intro]]). 

To support a rare case where there is no model initialization logic we define factory method:
```ocaml
module Controller = 

    let fromEventHandler callback = {
        new IController<'Events, 'Model> with
            member this.InitModel _ = ()
            member this.EventHandler = callback
    } 
```
### Sample Controller 
Let's have a look at a sample controller and small changes to events and view:
```ocaml
...
type SampleEvents = 
    | Add 
    | Subtract of int * int 
    | Clear 

type SampleView()... 
    ... 
    override this.EventStreams = 
        [
            ... 
            this.Window.Subtract.Click |> Observable.map(fun _ -> Subtract(int this.Window.X.Text, int this.Window.Y.Text)) 
        ] 
    
type SampleController() = 

    interface IController<SampleEvents, SampleModel> with 
        member this.InitModel model = 
            model.X <- 0
            model.Y <- 0
            model.Result <- 0

            model.Title <- 
                sprintf "Process name: %s" <| System.Diagnostics.Process.GetCurrentProcess().ProcessName

        member this.EventHandler = function
            | Add -> this.Add
            | Subtract(x, y) -> this.Subtract x y
            | Clear -> (this :> IController<_, _>).InitModel
    
    member this.Add(model : SampleModel) = 
        model.Result <- model.X + model.Y
        
    member this.Subtract x y (model : SampleModel) =  
        model.Result <- x - y
```
Key points:
* In the example below `SampleController` sets `Window` title to the running process name, but in reality application logic complexity can vary, so it would keep it localized in `InitModel`. 
* `EventHandler` 
    * Takes a form of classic high-order function: it performs pattern matching on the first argument and returns the corresponding event handler, which is another function. A careful look at the type signature `('Events -> 'Model -> unit)` reveals its role of event handler factory, indeed. 
    * Compiler will warn you if any of the pattern cases, that is, events are missing. Try commenting out any of the pattern matching cases to see the effect. 
    * It is beneficial to set model as the last parameter in an individual event handler to leverage function partial application and let `EventHandler` implementation be a compiler-checked map. 
* I made `Substract` event look different to show that extra state can be attached to Discriminated Union case which is an advantage over plain `Enums`. This is useful if some visual state cannot be bound to a model, though it's rare in WPF where most properties are dependency properties and thus represent eligible binding targets. 

### Mvc - gluing pieces together.
Now that all components defined we need to connect them, start event loop and coordinate event processing. Technically these responsibilities can be assigned to controller. We could create base class, put this logic there ans require all controller to inherit from this class. But it will lead to fragile subclass coupling. Better solution is to defined component that will play mediator/coordinator role. 
```ocaml
open System.ComponentModel

type Mvc<'Events, 'Model when 'Model :> INotifyPropertyChanged>(model : 'Model, view : IView<'Events>, controller : IController<'Events, 'Model>) =

    member this.Start() =
        controller.InitModel model
        view.SetBindings model
        view.Subscribe(callback = fun event -> controller.EventHandler event model)
```
A picture is worth a thousand words ...
[[Images/Mvc2.png]]

Look how loose coupled yet cohesive overall architecture:
* Model has no dependencies
* View depends on `'Events` type and implicitly (run-time through data binding) on `'Model`. We'll make second one explicit in [the next chapter](Data-Binding)
* Most important, Controller takes dependency only on `'Events` and `'Model`. It allows presentation logic it to be easy testable (see unit tests example below). On other side pattern matching ensure we process all event types coming from View (cohesion).
* As for Mvc type it depends on everything but it doesn't matter because it stays as is for most of applications. No need (almost) to extend or change it.  
