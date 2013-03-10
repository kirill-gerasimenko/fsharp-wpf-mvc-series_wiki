Controller is the key component of MVC-triple because it contains most (if not all) presentation logic. It's essential responsibility is to execute specific computation as respond to event while reading/updating model state in process. A simplest variation of controller is event handling function of type `('Events -> 'Model -> unit)`. It can be either standalone function or type member.

Most of screens in GUI applications when first opened are not empty but show some information to start with. It means that view model needs to be initialized. I chose to go against the initial instinct of putting `Model` initialization logic into Model's constructor. If taken, it would force `Model` to have smelly dependencies on external resources like database connections, web services, etc. and eventually testability will suffer. Placing this code into Controller's `InitModel` and employing [Dependency inversion principle](http://en.wikipedia.org/wiki/Dependency_inversion_principle) really helps. 

With requirements above in mind controller interface can defined as following:
```ocaml
type IController<'Events, 'Model> =

    abstract InitModel : 'Model -> unit
    abstract EventHandler : ('Events -> 'Model -> unit)
```
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
* With explicit IContoller<_, _> interface implementation type inference breaks down a bit therefore `model` parameter in `Add` and `Subtract` event handlers has to be type annotated. A reason is unknown to me because it seems like compiler has enough information to figure it out on its own. We'll look how the problem can be alleviate in the next chapter.

### Mvc - gluing pieces together.
Now that all components defined we need to connect them, start event loop and coordinate event processing. Technically these responsibilities can be assigned to controller. We could create base class, put this logic there ans require all controller to inherit from this class. But it will lead to fragile subclass coupling. Better solution is to defined separate component that will play mediator/coordinator role. 
```ocaml
open System.ComponentModel

type Mvc<'Events, 'Model>(model : 'Model, view : IView<'Events>, controller : IController<'Events, 'Model>) =

    member this.Start() =
        controller.InitModel model
        view.SetBindings model
        view.Subscribe(callback = fun event -> controller.EventHandler event model)
```
A picture is worth a thousand words ...
[[Images/Mvc2.png]]

Look how loose coupled yet cohesive overall architecture:
* Model has no dependencies
* View depends on `'Events` type and implicitly (run-time, through data binding) on `'Model`. We'll make second one explicit in [the next chapter](Data-Binding)
* Most important, Controller takes dependency only on `'Events` and `'Model`. It allows presentation logic it to be easy testable (see unit tests example below). On other side pattern matching ensures we process all event types coming from View (cohesion).
* As for Mvc type it depends on everything but it doesn't matter because it stays as is for most of applications. No need (almost) to extend or change it.  

### INotifyPropertyChanged model constraint
In order for data binding to function properly `Model` should implement `INotifyPropertyChanged` (there are exceptions to this assumption where suggested architecture may not be applicable). Ideally, this constraint should be reflected in the system. `Mvc<_, _>` seems to be the right place because the only other alternative - `IView` - can technically function in static fashion being bound to `Model` without `INotifyPropertyChanged` support (think of [OneTime](http://msdn.microsoft.com/en-us/library/system.windows.data.bindingmode.aspx) binding mode). 

There are three ways to enforce this constraint: 
 1. Placing the following code in constructor or at the beginning of `Mvc.Start`:
```ocaml
open System.Diagnostics
...
    let t = model.GetType() 
    Debug.WriteLineIf(t.GetInterface(typeof<INotifyPropertyChanged>.Name) = null, sprintf "Model of type %s doesn't implement INotifyPropertyChanged" t.Name)
...
```
 This is similar to the way WPF warns about data binding errors. Still, I feel it does not sufficiently draw developer's attention. 
 
 2. Instead of writing a warning to debug output a more strong alternative - assertion - can be used:
```ocaml
...
    assert(match box model with | :? INotifyPropertyChanged -> true | _ -> false)
...
```
 3. Define a type constraint on generic `'Model` type:
```ocaml
type Mvc<'Events, 'Model when 'Model :> INotifyPropertyChanged>(model : 'Model, view : IView<'Events>, controller : IController<'Events, 'Model>) =
    ...
```

After contemplating on this issue for quite a long time I decided to go with generic constraint as being the most compiler/developer friendly. It comes with two minor drawbacks:
 * There is no code that relies on declared generic constraint, which is probably not a good sign.
 * Some dynamic proxy-based approaches allow to weave `INotifyPropertyChanged` [without declaring it on original type](http://www.codewrecks.com/blog/index.php/2008/08/04/implement-inotifypropertychanged-with-castledynamicproxy/).

### Unit-testing
As a container of most if not all presentation logic controller is the primary target for unit testing. Let's take it for a test drive. We'll use this opportunity to show F#-specific unit-testing extension framework - [Unquote](http://code.google.com/p/unquote/). It has a nice integration with well-known frameworks XUnit.NET and NUnit. The major contribution of Unquote to the commonly used libraries is the ability to show descriptive information for failed assertions. See this framework ["Getting Started"](http://code.google.com/p/unquote/wiki/GettingStarted) topic for further details. The example below is [ XUnit.NET](http://xunit.codeplex.com/)-based. 
```ocaml
module FSharp.Windows.Sample.Test

open System 
open System.Diagnostics
open Xunit
open Swensen.Unquote.Assertions
open FSharp.Windows
open FSharp.Windows.Sample

let controller = SampleController()
let icontroller : IController<_, _> = upcast controller

[<Fact>]
let InitModel() = 
    let model = SampleModel.Create()
    icontroller.EventHandler Clear model
    model.X =? 0
    model.Y =? 0
    model.Result =? 0 
    model.Title =? ("Process name: " + Process.GetCurrentProcess().ProcessName)

[<Fact>]
let Add() = 
    let model : SampleModel = Model.Create()
    model.X <- 3
    model.Y <- 5
    icontroller.EventHandler Add model
    test <@ model.Result = 8 @>

[<Fact>]
let Subtract() = 
    let model = SampleModel.Create()
    controller.Subtract 11 9 model
    test <@ model.Result = 2 @>
```
Despite being written for a toy application, this code looks refreshingly simple for UI-based testing. This is due to the framework design and, partially, due to the language used. Event handlers can be tested either directly like `Subtract` or through `controller.EventHandler` call like `Add` which effectively tests `EventHandler` implementation too. This seems excessive, because `EventHandler` is already a compiler-checked map. 