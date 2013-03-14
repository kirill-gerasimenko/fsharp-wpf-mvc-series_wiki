Any real-world GUI application requires facilities to open child screens: both modal and non-modal. 

In order to add windows management functionality while preserving isolation from WPF we need to extend `IView` interface: 

```ocaml
type IView<'Events, 'Model> =
    inherit IObservable<'Events>

    abstract SetBindings : 'Model -> unit

    abstract ShowDialog : unit -> bool
    abstract Show : unit -> Async<bool>
    abstract Close : bool -> unit
```
  * `ShowDialog` method has the same meaning as standard [Window.ShowDialog](http://msdn.microsoft.com/en-us/library/system.windows.window.showdialog.aspx) but without pointless nullability. 
  * `Close` is a utility method. Single `boolean` parameter is propagated as a result of `Show*` methods. 
  * Return type of `Show` is very interesting. If a modal window can be viewed as synchronous computation, than consequently non-modal is an asynchronous one. This demonstrates the power of right abstraction - F# `Async<'T>` type can be used for expressing not only async I/O, but any kind of asynchronous computation. 

View implementation is straightforward:

```ocaml
[<AbstractClass>]
type View<'Events, 'Model, 'Window when 'Window :> Window and 'Window : (new : unit -> 'Window)>(?window) = 
    ...
    let mutable isOK = false
    ...
    interface IView<'Events, 'Model> with
        ...
        member this.ShowDialog() = 
            this.Window.ShowDialog() |> ignore
            isOK
        member this.Show() = 
            this.Window.Show()
            this.Window.Closed |> Event.map (fun _ -> isOK) |> Async.AwaitEvent 
        member this.Close isOK' = 
            isOK <- isOK'
            this.Window.Close()
```
Notice a nice usage of pipelining and `Event`/`Async` combinators inside `Show` method. Updated `Mvc` supports two versions of start: sync for modal windows and async for non-modal. Both versions return a `boolean` flag, which essentially indicates whether child `Mvc` confirms or discards `Model` state changes. 

```ocaml
[<AbstractClass>]
...
type Mvc... =
    ...
    member this.Activate() =
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

    member this.Start() =
        use subscription = this.Activate()
        view.ShowDialog()

    member this.AsyncStart() =
        async {
            use subscription = this.Activate()
            return! view.Show()
        }
```

It is worth noting that `Model` instance passed to both start variants is a primary communication vehicle between parent and child. If a parent Mvc wants to pass any state down to a child, the corresponding properties have to be set on the model. On the other hand, a child controller should be prepared to deal with either new or partially initialized model, so that passed down state won't be ignored or overridden. 

### Modal window example

Let's look first at the example of using modal window. Say, in our sample calculator application we want to be able to enter argument in hexadecimal format. We add a button next to each argument `TextBox` that opens up an auxiliary window to input value. Admittedly, this is not the best UI solution, but good enough for illustration purposes. 

[[Images/HexConverterWindowWithPassingState.png]]

Notice how state is communicated back and forth. This is achieved by the following logic inside `button.Click` event handler for the first argument: 

```ocaml
type SampleController() = 
    ... 
    member this.Hex1 model = 
        let view = HexConverter.view()
        let childModel = Model.Create() 
        let controller = HexConverter.controller() 
        let mvc = Mvc(childModel, view, controller)
        childModel.Value <- model.X

        if mvc.Start()
        then 
            model.X <- childModel.Value 
    ...
```

Depending on the result of `Mvc.Start` call the new state is either accepted or thrown away. 

The logic for second "H..." is intentionally a different. There is no state passed from parent. If user clicks "OK", then `TextBox` value gets overridden.

[[Images/HexConverterWindowWithoutPassingState.png]]

In such cases child should signal that model state is accepted by the user. The best way to express it in F# is `Option` type. Here is an event handler for the second argument "H..." button: 

```ocaml
type SampleController() = 
    ... 
    member this.Hex2 model = 
        (HexConverter.view(), HexConverter.controller())
        |> Mvc.start
        |> Option.iter(fun resultModel ->
            model.Y <- resultModel.Value 
        )
    ...
```

So, in case no state is passed, an instance of the model is created by the child Mvc. The extra module provides the required functionality: 

```ocaml
    ... 
[<RequireQualifiedAccess>]
module Mvc = 

    let inline start(view, controller) = 
        let model = (^Model : (static member Create : unit -> ^Model ) ())
        if Mvc<'Events, ^Model>(model, view, controller).Start() then Some model else None

    let inline asyncStart(view, controller) = 
        async {
            let model = (^Model : (static member Create : unit -> ^Model) ())
            let! isOk = Mvc<'Events, ^Model>(model, view, controller).AsyncStart()
            return if isOk then Some model else None
        }
```
Both functions in the module place [constraint](http://msdn.microsoft.com/en-us/library/dd548046.aspx) member  on the model to have static method `Create` with expected signature. The same technique was used in [Validation] module. These functions could be implemented as instance methods on `Mvc` class, but having them in a separate module gives an important advantage - flexibility in a model implementation. If we place them on the Mvc - member constraint becomes mvc-wide, thus forcing model to have `Create` method. In this design you'll pay the price only when you use the module. As an extra bonus, we are keeping public API surface of `Mvc` very small and hence easy to maintain. 

Alternative model implementation (like fully hand-written with parameterless constructor) can provide its own version of this. For example : 
```ocaml
[<RequireQualifiedAccess>]
module Mvc = 
     
    let start(view, controller) = 
        let model = new 'Model()
        if Mvc<'Events, 'Model>(model, view, controller).Start() then Some model else None

    let asyncStart(view, controller) = 
        async {
            let model = new 'Model()
            let! isOk = Mvc<'Events, 'Model>(model, view, controller).AsyncStart()
            return if isOk then Some model else None
        }
```

These Mvc extensions can be bundled together with `Model` definition and placed into a separate assembly, therefore allowing to include various model implementations. 

Before we get to the implementation details of `HexConverter` MVC-triple I would like to bring the reader's attention to `View` extensions module: 
```ocaml
[<AutoOpen>]
[<CompilationRepresentation(CompilationRepresentationFlags.ModuleSuffix)>]
module View = 
    
    type IView<'Events, 'Model> with

        member this.OK() = this.Close true
        member this.Cancel() = this.Close false

        member this.CancelButton with set(value : Button) = value.Click.Add(ignore >> this.Cancel)
        member this.DefaultOKButton 
            with set(value : Button) = 
                value.IsDefault <- true
                value.Click.Add(ignore >> this.OK)
```
Methods `OK` and `Cancel` are more readable shortcuts for calling `IView.Close`. Properties `OKButton` and `CancelButton` are helpful when it's only needed to attach a closing logic to a button without event handler in `Controller`. Defined in the extension module these helpers can be used with different `IView` implementations. 

While `HexConverter` MVC-triple implementation looks trivial (if not boring) I thought I would show a "cool" way to do this using module-scoped definitions, functions and objects expressions: 
```ocaml
...
module HexConverter =  

    type Events = ValueChanging of string * (unit -> unit)

    [<AbstractClass>]
    type Model() = 
        inherit FSharp.Windows.Model()

        abstract HexValue : string with get, set
        member this.Value 
            with get() = Int32.Parse(this.HexValue, NumberStyles.HexNumber)
            and set value = this.HexValue <- sprintf "%X" value

    let view() = 
        let result = {
            new View<Events, Model, HexConverterWindow>() with 
                member this.EventStreams = 
                    [
                        this.Window.Value.PreviewTextInput |> Observable.map(fun args -> ValueChanging(args.Text, fun() -> args.Handled <- true))
                    ]

                member this.SetBindings model = 
                    Binding.FromExpression 
                        <@ 
                            this.Window.Value.Text <- model.HexValue
                        @>
        }
        result.CancelButton <- result.Window.Cancel
        result.DefaultOKButton <- result.Window.OK
        result

    let controller() = 
        Controller.Create(
            fun(ValueChanging(text, cancel)) (model : Model) ->
                let isValid, _ = Int32.TryParse(text, NumberStyles.HexNumber, null)
                if not isValid then cancel()
            )
```
I'll let curious readers enjoy going through the code themselves. Hint: because only single event (`OK` button click) is needed, it's mapped to unit (a singleton value).  Also, because model initalizatio is not required, controller created from event handler using `Controller.Create` factory method.

I don't suggest using this technique in production code, but it's nice to have this option for short definitions. Pay attention to how event handlers can be inlined inside `Dispatch`. `HexConverter.view()` function uses `View` extensions module to define `Cancel` button. 

### Non-modal window example

To demonstrate non-modal windows we adopt Tomas Petricek [example](http://msdn.microsoft.com/en-us/library/hh297101.aspx) from MSDN. To the right side of our calculator we add a window chart control that shows stock prices using [MS Chart Control](http://msdn.microsoft.com/en-us/library/dd456632.aspx) for `WinForms`. In order to add another stock data to the chart user presses "Add Stock..." button. When "Stock Price" window is opened, user types in stock symbol and, once stock specific data is retrieved, "Add To Chart" button becomes enabled. Interestingly, many instances of "Stock Price" window can be opened simultaneously. 

[[Images/AsyncChildWindow-StockPricesChart.png]]

Below is "Add Stock..." related code: 
```ocaml
type SampleModel() = 
    ...
    abstract StockPrices : ObservableCollection<string * decimal> with get, set
...
type SampleEvents = 
    ...
    | AddStockToPriceChart 
... 
    
type SampleView() as this =
    inherit View<SampleEvents, SampleModel, SampleWindow>()
    ...        
    override this.EventStreams = 
        [
            ...
            this.Window.AddStock, AddStockToPriceChart
        ]
        |> List.map(fun(button, value) -> button.Click |> Observable.mapTo value)
...
type SimpleController() = 
...
    override this.Dispatcher = function
        ...
        | AddStockToPriceChart -> Async this.AddStockToPriceChart 
    ... 
    member this.AddStockToPriceChart model = 
        async {
            let! result = (StockPriceView(), StockPriceController()) |> Mvc.asyncStart  
            result |> Option.iter (fun stockInfo ->
                model.StockPrices.Add(stockInfo.Symbol, stockInfo.LastPrice)
            )
        }
```
What we've done is similar to the way we handled asynchronous I/O. No need to introduce new concept, but now it's not about I/O - it's about non-modal windows. It is a simple, composable and beautiful approach. And, to make things even more impressive - data retrieval from Yahoo finance web service inside `StockPriceController` is asynchronous too. 

One more thing I'd like to point out: imperative control initialization. For our example we need to manually setup some `MS Chart Control` properties: 
```ocaml
type SampleView() as this =
    inherit View<SampleEvents, SampleModel, SampleWindow>()
    
    do 
        let area = new ChartArea() 
        area.AxisX.MajorGrid.LineColor <- Color.LightGray 
        area.AxisY.MajorGrid.LineColor <- Color.LightGray        
        this.Window.StockPricesChart.ChartAreas.Add area 
        let series = 
            new Series( 
                ChartType = SeriesChartType.Column, 
                Palette = ChartColorPalette.EarthTones, 
                XValueMember = "Item1", 
                YValueMembers = "Item2") 
        this.Window.StockPricesChart.Series.Add series 
    ...
```
It is similar to [InitializeComponent](http://msdn.microsoft.com/en-us/library/system.windows.markup.icomponentconnector.initializecomponent.aspx) method. 