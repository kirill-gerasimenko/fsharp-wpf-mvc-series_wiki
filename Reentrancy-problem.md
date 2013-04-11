There is a well-known [problem](http://www.codinghorror.com/blog/2005/08/is-doevents-evil-revisited.html) of [reentrancy](http://blogs.msdn.com/b/cbrumme/archive/2004/02/02/66219.aspx) when dealing with WPF or Windows GUI applications in general. MSDN has a brief [memo](http://msdn.microsoft.com/en-us/library/ms741870.aspx) on the subject as well (see "reentrancy and locking" segment). 

To demonstrate, I put together a short example. It doesn't use the framework to keep it simple. 
```ocaml
open System
open System.Windows
open System.Windows.Controls
open System.ComponentModel 
open System.Linq
  
type Model() = 
    let mutable text = ""
    let propertyChangedEvent = Event<_,_>()
    
    member this.Text 
        with get() = text 
        and set value = 
            text <- value
            propertyChangedEvent.Trigger(this, PropertyChangedEventArgs "Text")
    
    interface INotifyPropertyChanged with
        [<CLIEvent>]
        member this.PropertyChanged = propertyChangedEvent.Publish
    
[<STAThread>] 
do
    let model = Model()
    let textBox = TextBox(DataContext = model)
    textBox.SetBinding(TextBox.TextProperty, "Text") |> ignore
    textBox.TextChanged |> Observable.add(fun _ ->
        printfn "Begin event handler. TextBox.Text value: %s. Reverting ..." model.Text
        model.Text <- String(model.Text.ToCharArray() |> Array.rev)
        printfn "End event handler."
    )
    model.Text <- "Hello"
    stdin.ReadLine() |> ignore
```
When you run you'll see is: 

[[Images/ReentranceDemoStackOverflowException.png]]

Notice, there are only "Begin ..." console outputs no "End ..." and it blows up with `StackOverflowException`. This is the essence of the problem step-by-step:  
  # Setting `model.Text` property to value "Hello" changes `Text`  
  # Property value set and `INotifyPropertyChanged.PropertyChanged` event is triggered 
  # Execution re-enters the callback via subscription before the one in flight had a chance to finish 
  # Recursive logic without stop-case causes eventual stack overflow.

We need to prevent #3 from happening. Some propose using `boolean` flag to discard incoming events while processing others. I don't think this is an acceptable solution. There is also [Dispatcher.DisableProcessing](http://msdn.microsoft.com/en-us/library/system.windows.threading.dispatcher.disableprocessing.aspx) approach, but it's very platform dependent (Yes, I do plan to make the framework multi-platform. Stick around to see this happening.) There are other reasons too. I will elaborate more on this in [[Composition]] chapter. 

Conceptually, it would be great if we could place incoming events in a queue (including ones triggered by current running event handler) and process them one by one in FIFO order. Rolling out this logic by hand is a non-trivial and error-prone exercise. Luckily, we have [Reactive Extensions](http://msdn.microsoft.com/en-us/data/gg577609.aspx) library. With recent (at the time of writing) [release v2.0](http://blogs.msdn.com/b/rxteam/archive/2012/08/15/reactive-extensions-v2-0-has-arrived.aspx) it has some helpful functionality. There are two methods of public static class System.Reactive.Observer : Synchronize and Checked. Unfortunately, documentation is not available yet on MSDN so I'll show screen shots from Visual Studio Object Browser: 

[[Images/Observer.Synchronize.Help.png]]
[[Images/Observer.Checked.Help.png]]

`Synchronize` with `preventReentrancy` being set does exactly what we're looking for. `Checked` is more like asserting that event source conforms `IObservable` semantics. All changes are localized in `Mvc.Activate` method: 
```ocaml
type Mvc... =
    
    member this.Activate() =
        controller.InitModel model
        view.SetBindings model

        let observer = Observer.Create(fun event -> 
            match controller.Dispatcher event with
            | Sync eventHandler ->
                try eventHandler model 
                with exn -> this.OnException(event, exn)
            | Async eventHandler -> 
                Async.StartWithContinuations(
                    computation = eventHandler model, 
                    continuation = ignore, 
                    exceptionContinuation = (fun exn -> this.OnException(event, exn)),
                    cancellationContinuation = ignore))
#if DEBUG
        let observer = observer.Checked()
#endif
        let observer = Observer.Synchronize(observer, preventReentrancy = true)
        view.Subscribe observer
```
That was easy. For the first time we have realized the advantage of using `IObservable` as opposed to standard F# events (see [View] chapter for more details). Rx.NET combinators library is a big help when it comes to concurrency and complex event processing. 

To demonstrate the problem in the context of our sample calculator we add a logic that filters out Divide operation, if second argument Y is zero. 

As you can imagine, event handler is hooked up to second argument `TextBox.TexhChanged` event. To see the effect without reentrancy prevention, comment out those lines in `Mvc`: 
```ocaml
type Mvc...
    ...
    member this.Activate model =
        ...
//#if DEBUG
//        let observer = observer.Checked()
//#endif
//        let observer = Observer.Synchronize(observer, preventReentrancy = true)
        view.Subscribe observer
```
Make sure there is a number other than 0 in the second argument text input then click "C" clear button. 

[[Images/YChangedWithoutReentrancyPrevention.png]]

Remember that `InitModel` serves as an event handler for `Clear`. We see that `YChanged` event handler was called from inside of `InitModel` - the case of the reentrance again. Remove comments in `Mvc.Activate` and run it again - now it should be alright. To make it even more interesting uncomment call to `Checked` and run - you'll see "Reentrancy has been detected." exception. 

### Small Embellishments

There are some other improvements I want to introduce.  

First, button click events are often mapped to `View.EventStreams` property. Let's have utility method that makes it terser: 
```ocaml
[<RequireQualifiedAccess>]
...
module List =
    open System.Windows.Controls

    let ofButtonClicks xs = xs |> List.map(fun(b : Button, value) -> b.Click |> Observable.mapTo value)
...
//Usage
...
type SampleView() as this =
    ...
    override this.EventStreams = 
        [ 
            yield! [
                this.Window.Calculate, Calculate 
                this.Window.Clear, Clear 
                this.Window.CelsiusToFahrenheit, CelsiusToFahrenheit 
                this.Window.FahrenheitToCelsius, FahrenheitToCelsius 
                this.Window.CancelAsync, CancelAsync 
                this.Window.Hex1, Hex1 
                this.Window.Hex2, Hex2 
                this.Window.AddStock, AddStockToPriceChart 
            ]
            |> List.ofButtonClicks 
                 
            yield this.Window.Y.TextChanged |> Observable.map(fun _ -> YChanged(this.Window.Y.Text))
        ] 
    ...
```
Second, more importantly, the framework grew big enough to package it into separate assembly. Done. 

### P.S
I would like to thank personally to [Bart de Smet](http://channel9.msdn.com/Tags/bart+de+smet) for [assistance](http://social.msdn.microsoft.com/Forums/en-US/rx/thread/2fc29cbc-145a-4aba-9820-84b7e5dbf908) in the problem solving and for moving forward great Rx.NET framework.