So far we discussed composition of visual components as event sources. But what about non-UI sources? Let’s imagine a trading application. Besides historical statistics, its screen may also show a real-time information, like a last stock trade price. This stream of incoming price ticks is a good example of non-UI event source. There is a good chance of having `ComboBox`/`Dropdown` on that screen for switching between stocks. We'll address this case later; this is like a knob to tune this non-UI event source. 

To simulate the experience we add a stopwatch-like component to the status line. It also has pause and restart functionality. We'll discuss strange "Fail" `CheckBox` down below.

[[Images/CompositionWithNonUISources.png]]

It's worth nothing that both UI and non-UI event sources composition is immutable by its nature - once it is done there is no way to disassemble it later, replace one event source by another, and assemble back. This means that in order to have a capability of amending/tuning event source some API has to be provided and a reference to event source should be persisted. 

With this assumption in mind here is the code for the stopwatch event source: 
```ocaml
...
type StopWatchObservable(frequency, failureFrequencyInSeconds) = 
    let watch = Stopwatch.StartNew() 
    let paused = ref false 
    let generateFailures = ref false 
    
    member this.Pause() = 
        watch.Stop() 
        paused := true 
    member this.Start() = 
        watch.Start() 
        paused := false 
    member this.Restart() = 
        watch.Restart() 
        paused := false 
    
    member this.GenerateFailures with set value = generateFailures := value 
    
    interface IObservable<TimeSpan> with 
        member this.Subscribe observer = 
            Observable.Interval(period = frequency) 
                .Where(fun _ -> not !paused) 
                .Select(fun _ -> 
                    if !generateFailures && watch.Elapsed.TotalSeconds % failureFrequencyInSeconds < 1.0 
                    then failwithf "failing every %.1f secs" failureFrequencyInSeconds 
                    else watch.Elapsed) 
                .Subscribe(observer) 
```
Notice that for the reason mentioned above it has two flavors: immutable - `IObservable<TimeSpan>` and mutable - `Pause`, `Start`, `Restart` and `GenerateFailures` flag that controls periodic failure generation.

The main MVC-triple went through some changes: 
```ocaml
type MainModel() = 
    ...
    abstract RunningTime : TimeSpan with get, set
    abstract Paused : Nullable<bool> with get, set
...
type MainEvents = 
    ...
    | StopWatch 
    | StartWatch 
    | RestartWatch 
    | StartFailingWatch 
    | StopFailingWatch 
    
type MainView() as this = 
    inherit View<MainEvents, MainModel, MainWindow>()
    
    let pause = this.Control.PauseWatch 
    let fail = this.Control.Fail 

    override this.EventStreams = 
        [   
            ...
            yield this.Control.RestartWatch.Click |> Observable.mapTo RestartWatch 
            yield pause.Checked |> Observable.mapTo StopWatch 
            yield pause.Unchecked |> Observable.mapTo StartWatch 
            yield fail.Checked |> Observable.mapTo StartFailingWatch 
            yield fail.Unchecked |> Observable.mapTo StopFailingWatch 
        ]
    
    override this.SetBindings model = 
        ...
        this.Control.SetBinding(Window.TitleProperty, titleBinding) |> ignore
        Binding.FromExpression 
            <@ 
                this.Control.PauseWatch.IsChecked <- model.Paused 
                this.Control.Fail.IsChecked <- model.Fail 
            @>
        this.Control.RunningTime.SetBinding(TextBlock.TextProperty, Binding(path = "RunningTime", StringFormat = "Running time: {0:hh\:mm\:ss}")) |> ignore
        
type MainController(stopWatch : StopWatchObservable) = 
    ...
    override this.InitModel model = 
        ...
        model.RunningTime <- TimeSpan.Zero
        model.Paused <- Nullable false
        model.Fail <- Nullable false 

    override this.Dispatcher = Sync << function
        ...
        | StopWatch -> ignore >> stopWatch.Pause
        | StartWatch -> ignore >> stopWatch.Start
        | RestartWatch -> this.RestartWatch 
        | StartFailingWatch -> fun _ -> stopWatch.GenerateFailures <- true 
        | StopFailingWatch -> fun _ -> stopWatch.GenerateFailures <- false 
    
    member this.RestartWatch model =
        stopWatch.Restart()
        model.Paused <- Nullable false 
```

  * `MainModel` obtained the properties to support new visual state/functionality. As we'll see later, there is no separate model for non-visual event sources. 
  * Nothing interesting took place in `MainView`, except that there are more questions about data binding, which will be all addressed in the [upcoming chapter](Data-Binding. Growing Micro DSL). 
  * `MainController` only deals with mutable part of `StopWatchObservable`. As a matter of fact, due to always explicit interface implementation (right design choice Don, despite of what [some people say](http://visualstudio.uservoice.com/forums/121579-visual-studio/suggestions/2313586-don-t-implement-interfaces-explicitly-in-f-) `MainController` does not see stopwatch as `IObservable<TimeSpan>` at all.

`Mvc` got the new overload of `Compose` member in order to support non-visual event sources: 
```ocaml
...
type Mvc... =
    ...
    member this.Compose(childController : IController<_, _>, events : IObservable<_>) = 
        let childView = {
            new IPartialView<_, _> with
                member __.Subscribe observer = events.Subscribe observer
                member __.SetBindings _ = () 
        }
        this.Compose(childController, childView, id)
```
This is simplification of visual source variation: instead of usual MVC-triple one needs to provide only controller and event source itself. Again, there is no separate model. Controller is supposed to work on parent's model. 

Here is the code that composes the final application controller: 
```ocaml
...
    let stopWatch = StopWatchObservable(frequency = TimeSpan.FromSeconds(1.), failureFrequencyInSeconds = 5.)
    let stopWatchController = Controller.Create(fun (runningTime : TimeSpan) (model : MainModel) -> 
        model.RunningTime <- runningTime)

    let view = MainView()
    let mvc = 
        Mvc(MainModel.Create(), view, MainController(stopWatch))
            .Compose(stopWatchController, stopWatchController)
            <+> (CalculatorController(), CalculatorView(view.Control.Calculator), fun m -> m.Calculator)
            <+> (TempConveterController(), TempConveterView(view.Control.TempConveterControl), fun m -> m.TempConveter)
            <+> (StockPricesChartController(), StockPricesChartView(view.Control.StockPricesChart), fun m -> m.StockPricesChart)
...
```

  * `stopWatchController` created from sync event handler that works on *`MainModel`*. It can easily be `Async`, or full-blown controller – it doesn't matter. There is a lot of flexibility. 
  * Again, I want to emphasize the dual role (event source/knob or immutable/mutable) of `stopWatchController`. Going back to the example of trading application, if in order to change stock symbol (read event source) it's acceptable to close and reopen screen, than the whole complexity of mutability/tuning is needless. But it's useful to have these options. 

## Processing events on UI thread

Non-visual event sources impose new challenge by introducing extra concurrency. Remember, model updates flow down to the GUI through data binding. By default, event handlers get invoked on the same thread that raises a notification. It was not much of a problem for GUI-initiated events because they were originated on UI thread anyway. Events sources such as stock price ticks or timers are different because they usually push events on some sort of thread pool. It makes GUI controls subject to a concurrent updates which is a really bad idea. In practice WPF shields a developer from this kind of mistakes and one ends up with some sort of not-supported/cross-threading exception.

Let's try find best solution by looking at different options.

### Solution 1 

The first idea that came to my mind is the following: right before event streams get composed, we modify external ones to raise notifications on correct thread: 
```ocaml
type Mvc...
    ...
    member this.Compose(childController : IController<_, _>, events : IObservable<_>) = 
        let compositeView = view.Compose(events.ObserveOnDispatcher())
        ...
```

This solution has several drawbacks - some are minor, and some are significant: 
  # In case of several external event sources (which is pretty rare, I must admit, but still...) an overhead of the context switch will be even bigger. An ideal solution would be doing this only once.  
  # `ObserveOnDispatcher` will unconditionally post/queue callback even though sometimes the event source might be already notifying on correct thread 
  # One day I still plan to make the core library to be multi-platform. `ObserveOnDispatcher` is very WPF/Silverlight specific. 

### Solution 2

Other approach is to put the same call to `ObserveOnDispatcher` into `Mvc.Activate`: 
```ocaml
    member this.Activate model =
        ...
        let observer = Observer.Synchronize(observer, preventReentrancy = true)
        ...
        view.ObserveOnDispatcher()
            .Subscribe(observer)
```
This would solve drawback#1 from the previous solution, but introduces another completely unexpected problem, which is another manifestation of drawback#2. To demonstrate it we have added some logic to `Calculator` component that prevents entering non-number characters into `X` and `Y` fields. The easiest way to implement it is to hook up `PreviewTextInput` events for both `TextBoxes` and cancel input if it's invalid. 
```ocaml
type CalculatorEvents = 
    ... 
    | XotYChanging of string * (unit -> unit) 
    
type CalculatorView... 
    
    override this.EventStreams = 
        [ 
            ... 
            yield this.Control.X.PreviewTextInput |> Observable.map(fun x -> XotYChanging(x.Text, fun() -> x.Handled <- true)) 
            yield this.Control.Y.PreviewTextInput |> Observable.map(fun y -> XotYChanging(y.Text, fun() -> y.Handled <- true)) 
        ] 
```
Because of [tunneling](http://msdn.microsoft.com/en-us/library/ms742806.aspx#routing_strategies) nature of `Preview*` events it falls through when combined with `ObserveOnDispatcher`. To rephrase, those events have to be processed synchronously, but `ObserveOnDispatcher` always makes asynchronous post into WPF Dispatcher queue. 

### Solution 3 - correct one

The best cross-platform abstraction to ensure that code runs on proper thread is [SynchronizationContext](http://msdn.microsoft.com/en-us/magazine/gg598924.aspx). Rx.NET library had [ObserveOn](http://msdn.microsoft.com/en-us/library/hh229634.aspx) overload for a long time:
```c#
public static class Observable
{
    ...
    public static IObservable<TSource> ObserveOn<TSource>(this IObservable<TSource> source, SynchronizationContext context) ...
    ...
}
```

Recently released [Reactive Extensions for .NET 2.0](http://blogs.msdn.com/b/rxteam/archive/2012/06/20/reactive-extensions-v2-0-release-candidate-available-now.aspx) (thanks [Bart](http://channel9.msdn.com/Tags/bart+de+smet) has added support for inlining calls that already are in the proper context. Read "A smarter `SynchronizationContextScheduler`" segment for details. Resulting code follows: 
```ocaml
type Mvc... 
    ...
    member this.Activate model =
        ...
        view
            .ObserveOn(
                scheduler = SynchronizationContextScheduler(SynchronizationContext.Current, alwaysPost = false))
            .Subscribe(
                observer = Observer.Synchronize(observer, preventReentrancy = true))
```
"alwaysPost" is the magic switch that tells `SynchronizationContextScheduler` to inline calls whenever possible and this will make those tunneling event handlers working as expected. 

### Handling event source errors 
When were dealing with GUI-based event source it was safe to assume that there were no event-generation errors. But adding non-visual sources to the mix changes the picture. No events can follow [OnError] (http://msdn.microsoft.com/en-us/library/dd781657.aspx), therefore event stream dies and exception gets rethrown. Exception handling should be provided on stream-by-stream basis. Look at amended composition logic. 

```ocaml
let stopWatch = StopWatchObservable(frequency = TimeSpan.FromSeconds(1.), failureFrequencyInSeconds = 5.)
    let stopWatchController = Controller.Create(fun (runningTime : TimeSpan) (model : MainModel) -> 
        model.RunningTime <- runningTime)

    let rec safeStopWatchEventSource() = Observable.Catch(stopWatch, fun(exn : exn)-> Debug.WriteLine exn.Message; safeStopWatchEventSource()) 

    let view = MainView()
    let mvc = 
        Mvc(MainModel.Create(), view, MainController(stopWatch))
            .Compose(stopWatchController, safeStopWatchEventSource())
            <+> ...
```

Now safeStopWatchEventSource() returns infinite stream. Exception gets logged and stream continues. For details on Rx error handling look [here](http://www.introtorx.com/Content/v1.0.10621.0/11_AdvancedErrorHandling.html#AdvancedErrorHandling).

If application is executed inside Visual Studio and "Fail" flag is on, we'll see error message pumping into "Output" window. 

[[Images/CompositionWithNonUISources.ObservableCatch.png]]

### Application module

The final solution assumes that appropriate `SynchronizationContext` has been established before `Mvc.*Start` call. Using WPF [Application](http://msdn.microsoft.com/en-us/library/system.windows.application.aspx) class is a good way to ensure this. Application module contains helpers that make it easy to bootstrap root `Mvc`: 
```ocaml
open System.Runtime.CompilerServices
open System.Windows
open System.Threading

[<AutoOpen>]
[<CompilationRepresentation(CompilationRepresentationFlags.ModuleSuffix)>]
[<Extension>]
module Application = 

    [<Extension>] //for C#
    let AttachMvc(mvc : Mvc<_, _>) =
        let cts = new CancellationTokenSource()
        Async.StartImmediate(mvc.AsyncStart() |> Async.Ignore, cts.Token)
    
    type Application with 
        member this.Run(mvc, mainWindow) =
            this.Startup.Add <| fun _ -> AttachMvc mvc
            this.Run mainWindow
```
The first visible extension method `AttachMvc` is meant for application hosts written in C#. It is needed, for example, if you want to leverage [ClickOnce](http://msdn.microsoft.com/en-us/library/t71a733d.aspx) deployment. F# projects do not support this. Usage will be similar to something like below: 
```C#
namespace ...
{
    public partial class App : Application
    {
        public App ()
        {
            this.Startup += delegate
            {
                var model = Model.Create<MainModel>();
                var view = new MainView(new MainPage());
                var controller = new MainController();
                this.RootVisual = view.Control;
                var mvc = new Mvc<MainPageEvents, MainModel>(model, view, controller);
                this.AttachMvc(mvc);
            };
            ...
        }
    }
}
```

Application extension method Run is for F# hosted application main: 
```ocaml
[<STAThread>] 
[<EntryPoint>]
let main _ = 
    ...
    let mvc = 
        ...

    app.Run(mvc, mainWindow = view.Control)
```