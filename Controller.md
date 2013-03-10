Controller is the key component of MVC-triple because it contains most (if not all) presentation logic. It's essential responsibility is to execute specific computation as respond to event reading/updating model state in process.
A simplest variation of controller is event handling function of type `('Events -> 'Model -> unit)`. It can be either standalone function or type member property.

### Sample Controller 
Let's have a look at a sample controller and changes to events and view:
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

    member this.EventHandler = function
        | Add -> this.Add
        | Subtract(x, y) -> this.Subtract x y
        | Clear -> fun(model : SampleModel) ->
            model.X <- 0
            model.Y <- 0
            model.Result <- 0

    member this.Add model = 
        model.Result <- model.X + model.Y
        
    member this.Subtract x y model = 
        model.Result <- x - y
```
Key points:
  * Controller has dependency only on `SampleEvents`. It means loose coupling and testability. 
  * Takes a form of classic high-order function: it performs pattern matching on the first argument and returns the corresponding event handler, which is another function. A careful look at the type signature `('Events -> 'Model -> unit)` reveals its role of event handler factory, indeed. 
  * Compiler will warn you if any of the pattern cases, that is, events are missing. Try commenting out any of the pattern matching cases to see the effect. 
  * It is beneficial to set model as the last parameter in an individual event handler to leverage function partial application and let `EventHandler` implementation be a compiler-checked map. 
  * I made `Substract` event look different to show that extra state can be attached to Discriminated Union case which is an advantage over plain `Enums`. This is useful if some visual state cannot be bound to a model, though it's rare in WPF where most properties are dependency properties and thus represent eligible binding targets. 

### Mvc