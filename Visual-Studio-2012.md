It's important to distinguish two kind of changes that come with Visual Studio 2012: new 3.0 version of F# language (which still can be compiled down to .NET 4.0) and new 4.5 version of .NET runtime.

# F# 3.0

## Queries

The first of [two new majors features](http://msdn.microsoft.com/en-us/library/hh370982.aspx) is [Query Expressions](http://msdn.microsoft.com/en-us/library/hh225374.aspx). This enables F# to be on a par with Haskell - now it has both imperative-style [Computation Expressions](http://msdn.microsoft.com/en-us/library/dd233182.aspx) (do notation in Haskell) and Query Expressions (List comprehension in Haskell). 

A query builder must be available for a data source to be used inside Query Expression. Out of the box F# 3.0 comes with [Linq.QueryBuilder](http://msdn.microsoft.com/en-us/library/hh323943.aspx) which provides support for all `IEnumerable` and `IQueryable` based sources.

There is one place where we can use query expression: 
```ocaml
type StopWatchObservable(frequency, failureFrequencyInSeconds) = 
    ...
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

It would be nice to re-write this piece as query expression (as we would use LINQ query in C#). We need to roll out query builder implementation for `IObservable<'T>`. A simplified version of it below:
```ocaml
...
[<RequireQualifiedAccess>]
module Observable =
    ...
    open System.Reactive.Linq
    ...
    type QueryBuilder internal() = 
        member this.For(source : IObservable<_>, selector : _ -> IObservable<_>) = source.SelectMany(selector) 
        member this.Zero() = Observable.Empty() 
        [<CustomOperation("where", MaintainsVariableSpace = true, AllowIntoPattern = true)>] 
        member this.Where(source : IObservable<_>, [<ProjectionParameter>] predicate : _ -> bool ) = source.Where(predicate) 
        member this.Yield value = Observable.Return value 
        [<CustomOperation("select", AllowIntoPattern = true)>] 
        member this.Select(source : IObservable<_>, [<ProjectionParameter>] selector : _ -> _) = source.Select(selector) 
    
    let query = QueryBuilder() 
```

Usage:
```ocaml
type StopWatchObservable(frequency, failureFrequencyInSeconds) =
    ...
    interface IObservable<TimeSpan> with 
        member this.Subscribe observer = 
            let xs = Observable.query { 
                for _ in Observable.Interval(period = frequency) do 
                where (not !paused) 
                select (if !generateFailures && watch.Elapsed.TotalSeconds % failureFrequencyInSeconds < 1.0 
                    then failwithf "failing every %.1f secs" failureFrequencyInSeconds 
                    else watch.Elapsed) 
            } 
            xs.Subscribe(observer) 
```

Note that F# allows adding custom operators to query expressions and you get syntax highlighting for them! Awesome. See [CustomOperationAttribute](http://msdn.microsoft.com/en-us/library/hh289709.aspx) MSDN documentation for details.

## Type Providers

F# 3.0 introduced a ground-shaking feature: type providers. For details watch Don's introductionary Build 2011  [presentation](http://channel9.msdn.com/Events/Build/BUILD2011/SAC-904T) or [the latest one](http://skillsmatter.com/podcast/scala/practical-functional-first-programming-with-f) from Progressive F# Tutorials 2012 conference.

### [WSDL Type Provider](http://msdn.microsoft.com/en-us/library/hh156503.aspx)

One place in sample application that begs for WSDL Type Provider usage is access to [w3schools temperature converter service](http://www.w3schools.com/webservices/ws_example.asp). 
```ocaml
[<AutoOpen>]
[<CompilationRepresentation(CompilationRepresentationFlags.ModuleSuffix)>]
module FSharp.Windows.Sample

open Microsoft.FSharp.Data.TypeProviders

type TempConvert = WsdlService<"http://www.w3schools.com/webservices/tempconvert.asmx?WSDL">

[<Measure>] type celsius
[<Measure>] type fahrenheit

type TempConvert.ServiceTypes.SimpleDataContextTypes.TempConvertSoapClient with

    member this.AsyncCelsiusToFahrenheit(value : float<celsius>) = 
        async {
            let! response = value |> string |> this.CelsiusToFahrenheitAsync |> Async.AwaitTask 
            return response.Body.CelsiusToFahrenheitResult |> float |> LanguagePrimitives.FloatWithMeasure<fahrenheit>
        }

    member this.AsyncFahrenheitToCelsius(value : float<fahrenheit>) = 
        async {
            let! response = value |> string |> this.FahrenheitToCelsiusAsync |> Async.AwaitTask 
            return response.Body.FahrenheitToCelsiusResult |> float |> LanguagePrimitives.FloatWithMeasure<celsius>
        }
```

We made two additions to types generated by WSDL type provider:
  * Two extension methods that return `Async` type instead of generated by default `Task` to smooth usage from F#.
  * [Units of Measure](http://msdn.microsoft.com/en-us/library/dd233243.aspx) were added to input/output values. Even though we don't have any operations on those values that can benefit from it, the immediate gain is a better documented code. Also, units of measure were moved from F# !PowerPack to standard F# 3.0 language distribution. It makes them particularly attractive to use.

### [XAML Type Provider](http://www.navision-blog.de/2012/03/22/wpf-designer-for-f/)

XAML type provider was intended to be a key to pure F# solution.

To start, we convert `HexConverter` MVC-triple to use XAML type provider. Below is a relevant code excerpt:
```ocaml
...
open FSharpx

module HexConverter =  

    type HexConverterWindow = XAML<"View\HexConverterWindow.xaml">

    let view() = 
        let window = HexConverterWindow()
        let value = window.Value
        value.ShowErrorInTooltip()
        let result = {
            new View<Events, Model, Window>(window.Root) with 
                member this.EventStreams = 
                    [
                        window.Value.PreviewTextInput |> Observable.map(fun args -> ValueChanging(args.Text, fun() -> args.Handled <- true))
                    ]

                member this.SetBindings model = 
                    Binding.FromExpression 
                        <@ 
                            window.Value.Text <- model.HexValue
                        @>
        }
        result.CancelButton <- window.Cancel
        result.DefaultOKButton <- window.OK
        result
...
module FSharp.Windows.Sample.Extensions 
    ...
open System.Windows.Data
open System.Windows.Controls

type TextBox with
    member this.ShowErrorInTooltip() = 
        let binding = Binding("(Validation.Errors).CurrentItem.ErrorContent", RelativeSource = RelativeSource.Self)
        this.SetBinding(TextBox.ToolTipProperty, binding) |> ignore
```

Let's compare it to the mixed F#/C# approach:
  * For data binding DSL to work all accessed controls need to be saved as local values (`let value = window.Value`). This is because XAML Type Provider emits [erased types](http://msdn.microsoft.com/en-us/library/hh361034.aspx#BK_Erased). It means that what you see is not what you get in compiled code. It's like a static micro expansion system. For example, `window.Value.Text <- model.HexValue` will be translated to `(window.GetChild "Value").Text <- model.HexValue` and binding cannot operate on this.
  * `value.ShowErrorInTooltip()` binds tooltip content to associated error message. Previously, we had it defined in shared resource style but XAML Type Provider doesn't support resource dictionaries.
  * Generated window type has `Root` property of type `Window`. Child controls defined as siblings to it. It means we cannot pass windows as a root and fetch children off it. Very unfortunate design decision.

It gets even worse with MVC-triples that define view type explicitly (which is typical case). Let's convert `StockPicker` triple as example.

First we build helper base class View:
```ocaml
...
open FSharpx.TypeProviders.XamlProvider

[<AbstractClass>]
type XamlProviderView<'Events, 'Model>(window : Window) = 

    let mutable isOK = false

    interface IView<'Events, 'Model> with
        member this.Subscribe observer = 
            let xs = this.EventStreams |> List.reduce Observable.merge 
            xs.Subscribe observer
        member this.SetBindings model = 
            window.DataContext <- model 
            this.SetBindings model
        member this.ShowDialog() = 
            window.ShowDialog() |> ignore
            isOK
        member this.Show() = 
            window.Show()
            window.Closed |> Event.map (fun _ -> isOK) |> Async.AwaitEvent 

    member this.Close isOK' = 
        isOK <- isOK'
        window.Close()

    abstract EventStreams : IObservable<'Events> list
    abstract SetBindings : 'Model -> unit
```

It is generalized only by event and model types because `Root` property of generated type is always `Window`. Here is derived `StockPickerView`:
```ocaml
open FSharpx 
type StockPickerWindow = XAML<"View\StockPickerWindow.xaml"> 
    
type StockPickerView(xaml : StockPickerWindow) as this = 
    inherit XamlProviderView<unit, StockInfoModel>(xaml.Root) 
    
    let companyName = xaml.CompanyName 
    let addToChart = xaml.AddToChart 
    let retrieve = xaml.Retrieve 
    let symbol = xaml.Symbol 
     
    do 
        symbol.CharacterCasing <- CharacterCasing.Upper 
        this.CancelButton <- xaml.CloseButton 
        this.DefaultOKButton <- xaml.AddToChart 
        symbol.ShowErrorInTooltip() 
    
    override this.EventStreams = 
        [ 
            xaml.Retrieve.Click |> Observable.mapTo() 
        ] 
    
    override this.SetBindings model = 
        Binding.FromExpression 
            <@ 
                companyName.Text <- model.CompanyName 
                addToChart.IsEnabled <- model.AddToChartEnabled 
                retrieve.IsEnabled <- isNotNull model.Symbol 
            @> 
        
        Binding.UpdateSourceOnChange <@ symbol.Text <- model.Symbol @> 
        
        let converter = BooleanToVisibilityConverter() 
        Binding.FromExpression  
            <@ 
                addToChart.Visibility <- converter.Apply model.AddToChartEnabled 
            @> 
```

It has all same problems as view for `HexConverter`. Additionally, instance of window has to be passed, therefore it needs to be created explicitly at calling site: 
```ocaml
type StockPricesChartController() =  
    ...
    override this.Dispatcher = fun() -> 
        Async <| fun model ->
            async {
                let window = StockPickerWindow()
                let view = StockPickerView(window)
                ... 
            } 
```

An attempt to convert `StockPricesChart` failed because I could not get XAML type provider to work with reference to MSChart in xaml:
```ocaml
<UserControl 
    ...
    xmlns:WinFormsChart = "clr-namespace:System.Windows.Forms.DataVisualization.Charting;assembly=System.Windows.Forms.DataVisualization" 
    ... />
```

[This post](http://www.mindscapehq.com/blog/index.php/2012/04/) claims it should work. May be I'm missing something.

Another issue is that the standard way to embed user controls doesn't work either:
```ocaml
<Window x:Class="FSharp.Windows.UIElements.MainWindow"
        ...
        xmlns:local="clr-namespace:FSharp.Windows.UIElements"> 
    ....
        <TabControl x:Name="Tabs" x:FieldModifier="public"> 
            <TabItem Header="Calculator"> 
                <local:CalculatorControl x:Name="Calculator" x:FieldModifier="public"/> 
            </TabItem> 
            <TabItem Header="Async Temperature Converter"> 
                <local:TempConveterControl x:Name="TempConveterControl" x:FieldModifier="public"/> 
            </TabItem> 
            <TabItem Header="Stock Prices"> 
                <local:StockPricesChartControl x:Name="StockPricesChart" x:FieldModifier="public"/> 
            </TabItem> 
        </TabControl> 
    ...
```

It means that our `MainView` has to be composed manually in code and full designer visualization is lost.

After all we have two views `HexConverter` and `StockPicker` that use XAML type provider. The rest stays C# generation-based.

As much as I appreciate the effort of the guys working on [FSharpx](https://github.com/fsharp/fsharpx) XAML Type Provider it is not yet ready for production use. But make your own judgment, I might be wrong.

# .NET 4.5

## INotifyDataErrorInfo

Starting 4.5 WPF supports [INotifyDataErrorInfo](http://msdn.microsoft.com/en-us/library/system.componentmodel.inotifydataerrorinfo.aspx) interface. It has been available for a while in Silverlight. Basically it makes error notification more natural. I'm not going to delve into boring implementation details. Follow the link  above to see MSDN implementation guidelines or look at source code for the chapter. It is worth nothing that semantic has changed a little: zero or many errors can be associated with single property. It is a deviation from `IDataErrorInfo` where one to one relationship exists. 

`Model.SetError` method was renamed to `AddError` to better reflect semantic, as a result member constraint for Validation module changed too.

## ExceptionDispatchInfo

Support for re-throwing exception with original stack trace was a long awaited feature. Finally [ExceptionDispatchInfo](http://msdn.microsoft.com/en-us/library/system.runtime.exceptionservices.exceptiondispatchinfo.aspx) has arrived. Changes to `Mvc.OnError` are minimal:
```ocaml
...
open System.Runtime.ExceptionServices
...
type Mvc ... =

    let mutable onError = fun _ exn -> ExceptionDispatchInfo.Capture(exn).Throw()
    ...
    abstract OnError : ('Events -> exn -> unit) with get, set
    default this.OnError with get() = onError and set value = onError <- value
    ...
```