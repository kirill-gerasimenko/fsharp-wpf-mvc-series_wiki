For a long time I've been obsessed with idea to use F# "Type providers" feature to generate  INotifyPropertyChanged implementation for model. Even before question/idea was brought up on [StackOverflow] (http://stackoverflow.com/questions/13786586/f-type-providers-and-inpc-metaprogramming). Ideally the usage INPC Type Provider would be like following:
```ocaml
type MyViewModel = NotifyPropertyChanged<SomeType>
```
Unfortunately, type providers in current implementation [do not accept type as parameters] (http://stackoverflow.com/questions/9547225/can-i-provide-a-type-as-an-input-to-a-type-provider-in-f).
As a reasonable compromise for the framework I came up with the following idea: 
  * There is separate assembly called "model prototypes assembly" where all model types are defined
  * To keep it declarative and simple model types should defined as F# records
  * This assembly must be referenced in project where views and controllers are defined
  * Assembly short name is supplied as input to NotifyPropertyChanged type provider

There are two kinds of F# type providers: "erased types" and "generated types". First is ubiquitously popular and has better development support. As for the second, to this point I don't know any serious "generated types" based implementation.
***
It's not that simple to make a design choice between "erased types" and "generated types". I found this  note from "Details about Erased Provided Types" section of "Creating a Type Provider" [MSDN tutorial] (http://msdn.microsoft.com/en-us/library/hh361034.aspx) particularly relevant to INPC Type Provider:

_...erased provided types ... are particularly useful in the following situations:
  ...
  When you are writing a provider where accurate runtime-type semantics aren't critical for practical use of the information space._
  ...

We'll see later in the chapter how this rule applies to design decisions.
***
###Round #1 - "erased types" + ExpandoObject
As a novice in type provider development, I decided to start from pilot version. I thought to try "erased types" first. Here is a test script that shows usage ([github] (https://github.com/dmitry-a-morozov/fsharp-wpf-mvc-series/blob/master/Chapter%2015%20-%20INPCTypeProvider/POC/TryExpando.fsx)):
```ocaml
#r @"SampleModelPrototypes\bin\Debug\SampleModelPrototypes.dll"
#r @"ExpandoObject\bin\Debug\ExpandoObject.dll"

open System
open System.ComponentModel
open ExpandoObject.INPCTypeProvider

type ViewModels = NotifyPropertyChanged<"SampleModelPrototypes">

let model = ViewModels.Person(FirstName = "F#", LastName = "Amazing", DateOfBirth = DateTime.Parse("2005-01-01"))

let inpc : INotifyPropertyChanged = upcast model
inpc.PropertyChanged.Add(fun args -> printfn "Property %s. Model: %A" args.PropertyName model)

model.FirstName <- "Dmitry"
model.LastName <- "Morozov"
model.DateOfBirth <- DateTime.Parse("1974-01-01")
```
The last three lines cause following being printed in FSI:
```
...
Property FirstName. Model: seq
  [[FirstName, Dmitry]; [LastName, Amazing]; [DateOfBirth, 1/1/2005 12:00:00 AM]]
Property LastName. Model: seq
  [[FirstName, Dmitry]; [LastName, Morozov]; [DateOfBirth, 1/1/2005 12:00:00 AM]]
Property DateOfBirth. Model: seq
  [[FirstName, Dmitry]; [LastName, Morozov]; [DateOfBirth, 1/1/1974 12:00:00 AM]]
...
```
`SampleModelPrototypes.dll` assembly contains [definition of Person model] (https://github.com/dmitry-a-morozov/fsharp-wpf-mvc-series/blob/master/Chapter%2015%20-%20INPCTypeProvider/POC/SampleModelPrototypes/Person.fs):
```ocaml
type Person = {
    mutable FirstName : string 
    mutable LastName : string
    mutable DateOfBirth : System.DateTime
}
```
Based on `Person`, type accessible via `ViewModel.Pereson` generated with support for INotifyPropertyChanged.
Second assembly "ExpandoObject.dll" contains type provider itself. As you probably guessed, "erased types" are replaced by [ExpandoObject] (http://msdn.microsoft.com/en-us/library/system.dynamic.expandoobject.aspx) class in the compiled  code. Property getters/setters are mapped to dictionary-style set/get key-based invocations. Setting key to value makes ExpandoObject raise INotidyProperyChanged.PropertyChanged event because it has built-in support for it. Not bad for the pilot but this version has numerous problems. I will mention only obvious ones: 
  * No support for INotifyDataErrorInfo ( IDataErrorInfo on .NET 4)
  * Data binding to dynamic objects works in WPF it's no-go for platform like WinRT 
(which is one of the primary reasons to replace dynamic proxy based approach with Type Provider). 
  * Data binding to dynamic objects is sub-optimal. Look [here] (http://blogs.msdn.com/b/silverlight_sdk/archive/2011/04/26/binding-to-dynamic-properties-with-icustomtypeprovider-silverlight-5-beta.aspx) for details ("What about WPF and DLR?" section).

###Round #2 - "erased types" + custom runtime base class
As the next step I introduced [custom run-time class] (https://github.com/dmitry-a-morozov/fsharp-wpf-mvc-series/blob/master/Chapter%2015%20-%20INPCTypeProvider/POC/CustomRuntimeClass/Model.fs) as base for models. Here is a brief description:
  * Supports INPC and INotifyDataErrorInfo  
  * Uses F# record prototypes as backing storage 
  * Provides dictionary-like key based set/get operations.
  * Usually data binding engine in WPF relies on reflection in run-time. But in our case real type/properties do not exist in compiled code. As a substitution model custom base class implements [ICustomTypeDescriptor] (http://msdn.microsoft.com/en-us/library/system.componentmodel.icustomtypedescriptor.aspx) interface. Alternatively, it could implement new on .NET 4.5 [ICustomTypeProvider] (http://msdn.microsoft.com/en-us/library/system.reflection.icustomtypeprovider.aspx), but it's more tedious despite of what some people [say] (http://blogs.msdn.com/b/silverlight_sdk/archive/2011/04/26/binding-to-dynamic-properties-with-icustomtypeprovider-silverlight-5-beta.aspx). 

Type provider [implementation] (https://github.com/dmitry-a-morozov/fsharp-wpf-mvc-series/blob/master/Chapter%2015%20-%20INPCTypeProvider/POC/CustomRuntimeClass/NotifyPropertyChangedTypeProvider.fs) didn't change much. It injects [Model] (https://github.com/dmitry-a-morozov/fsharp-wpf-mvc-series/blob/master/Chapter%2015%20-%20INPCTypeProvider/POC/CustomRuntimeClass/Model.fs) type instead of `ExpandoObject`. 

To test-drive this implementation I wanted to use something more elaborate than script. I tried a simplified version of [[Validation]] chapter application but kept model related functionality intact because that what's most affected by INPC Type Provider. I left out concepts of view and controller. They are very important but not relevant to our discussion at the moment. [[Data binding]] and [[Validation]] modules stay the same for now. 
Here is model prototype definition:
```ocaml
type Operations =
    | Add = 0
    | Subtract = 1
    | Multiply = 2
    | Divide = 3

type Calculator = {
    mutable AvailableOperations : Operations[] 
    mutable SelectedOperation : Operations
    mutable X : int
    mutable Y : int
    mutable Result : int
}
```
I post code for sample application because it's small:
```ocaml
module MainApp

open CustomRuntimeClass.INPCTypeProvider
open SampleModelPrototypes
...

let (?) (window : Window) name : 'T = name |> window.FindName|> unbox

type ViewModels = NotifyPropertyChanged<"SampleModelPrototypes">

[<STAThread>]
do 
    //Create Window 
    let window : Window = Application.LoadComponent(Uri("MainWindow.xaml", UriKind.Relative)) |> unbox 
    let x : TextBox = window?X 
    let y : TextBox = window?Y 
    let operations : ComboBox = window?Operation 
    let result : TextBlock = window?Result 
    let calculate : Button = window?Calculate 
    let clear : Button = window?Clear 
 
    //Create models 
    let model = ViewModels.Calculator() 
    model.AvailableOperations <- typeof<Operations> |> Enum.GetValues |> unbox 
 
    //Data bindings 
    Binding.FromExpression  
        <@ 
            x.Text <- string model.X 
            y.Text <- string model.Y 
            result.Text <- string model.Result 
            operations.ItemsSource <- model.AvailableOperations 
            operations.SelectedItem <- model.SelectedOperation 
        @> 
 
    window.DataContext <- model 
 
    //Event handlers 
    calculate.Click.Add  <| fun _ -> 
        match model.SelectedOperation with 
        | Operations.Add -> model.Result <- model.X + model.Y 
        | Operations.Subtract -> model.Result <- model.X - model.Y 
        | Operations.Multiply -> model.Result <- model.X * model.Y 
        | Operations.Divide -> 
            if model.Y = 0 
            then 
                model |> Validation.setError <@ fun m -> m.Y @> "Attempted to divide by zero." 
            else 
                model.Result <- model.X / model.Y 
        | _ -> () 

    clear.Click.Add <| fun _ -> 
        model.X <- 0 
        model.Y <- 0 
        model.Result <- 0 
 
   //Start
    window.ShowDialog() |> ignore 
```
Running application without any changes gives us:
[[Images/INPC.TypeProvider.Broken.Binding.png]]

Although we satisfied WPF data binding engine requirments by providing implementation of `ICustomTypeDescriptor` our data binding DSL still expects real typed properties (importance of run-time semantics). Small change in `Binding` module to fix the issue:
```ocaml
...
module FSharp.Windows.Binding
...
let rec (|PropertyPath|_|) = function 
    ...
    //Support for type provider erased types
    | Call((Some (Value (:? ICustomTypeDescriptor as model, _))), get_Item, [ Value(:? string as propertyName, _)]) 
        when get_Item.Name = "get_Item" && model.GetProperties().Find(propertyName, ignoreCase = false) <> null -> Some propertyName
    | _ -> None

```
Validation module has to be fixed for the very same reason:
```ocaml
...
module FSharp.Windows.Validation
...
let (|SingleStepPropertySelector|) (expr : PropertySelector<'T, 'a>) = 
    match expr with 
    ...
    | Lambda(arg, Coerce (Call (Some (Var selectOn), get_Item, [ Value(:? string as propertyName, _) ]), _)) when get_Item.Name = "get_Item" -> 
        assert(arg = selectOn)
        assert(typeof<ICustomTypeDescriptor>.IsAssignableFrom(selectOn.Type))
        propertyName, fun(this : 'T) -> get_Item.Invoke(this, [| propertyName |]) |> unbox<'a>
    ...
```
Now it works identically to application from [[Validation]] chapter. Is that all? Not really. This solution still has serious issues:
  1. The changes we made to binding and validation modules feel like hacks. They violate "Separation of concerns" principle. I was very keen to keep the framework design independent of model implementation (reminder: the only requirement for model is to implement INotifyPropertyChanged). Fact that we have different (type provider based) implementation should not force changes in binding or validation.
  2. It may be hard to port the solution to WinRT (which is again one of the drivers for my work on INPC type provider). Although it seems to be [possible] (http://jaylee.org/post/2012/03/07/Xaml-integration-with-WinRT-and-the-IXamlMetadataProvider-interface.aspx) but "generated types" [enable easier solution] (http://jaylee.org/post/2012/11/26/DataBinding-performance-in-WinRT-and-the-Bindable-attribute.aspx).
  3. Going forward I certainly plan to implement same [[Derived Properties]] feature as the one for dynamic proxy based models. I don't see how it can be done with "erased types".
  4. Once user refers to model prototypes assembly binary it's locked. Visual Studio has be re-opened in case prototypes assembly needs be recompiled which is certainly a sub-optimal experience.

\#4 is really nasty problem. I spent quite some time looking for solution (like custom remote domain + shadowing) but hit the wall. 

\#1, \#2 and \#3 can be solved if I'll switch to "generated types" type provider. I'm looking forward to this  challenge.

###Round #3 - "generated types" + custom runtime base class

I think "generated types" is the most useful version for practical application development. 
Here is the sample application used through the series updated to leverage "generated types" INPC type provider.

Model prototypes are defined in a separate assembly called "ModelPrototypes":
```ocaml
type Operations = 
    | Add 
    | Subtract 
    | Multiply 
    | Divide 

type CalculatorModel = { 
    mutable AvailableOperations : Operations[]  
    mutable SelectedOperation : Operations 
    mutable X : int 
    mutable Y : int 
    mutable Result : int
} 

type TempConveterModel = {
    mutable Celsius : float 
    mutable Fahrenheit : float
    mutable ResponseStatus : string
    mutable Delay : int 
}

type StockInfoModel = 
    {
        mutable Symbol : string
        mutable CompanyName : string
        mutable LastPrice : decimal
        mutable DaysLow : decimal
        mutable DaysHigh : decimal
        mutable Volume : decimal

        mutable AddToChartEnabled : bool
    }

    [<ReflectedDefinition>]
    member this.AccDist = 
        if this.DaysLow = 0M && this.DaysHigh = 0M then "Accumulation/Distribution: N/A"
        else
            let moneyFlowMultiplier = (this.LastPrice - this.DaysLow) - (this.DaysHigh - this.LastPrice) / (this.DaysHigh - this.DaysLow)
            let moneyFlowVolume  = moneyFlowMultiplier * this.Volume
            sprintf "Accumulation/Distribution: %M" <| Decimal.Round(moneyFlowVolume, 2)

type StockPricesChartModel = {
    mutable StocksInfo : StockInfoModel ObservableCollection
    mutable SelectedStock : StockInfoModel 
}

type MainModel = 
    { 
        mutable Calculator : CalculatorModel
        mutable TempConveter : TempConveterModel
        mutable StockPricesChart : StockPricesChartModel

        mutable ProcessName : string
        mutable ActiveTab : TabItem
        mutable RunningTime : TimeSpan
        mutable Paused : bool
        mutable Fail : bool
    }

    [<ReflectedDefinition>]
    member this.Title = sprintf "%s-%O" this.ProcessName this.ActiveTab.Header
```
In a project that needs to use view models (usually views/controllers project) it looks like the following
```ocaml
open FSharp.Windows.INPCTypeProvider

type ViewModels = NotifyPropertyChanged<"ModelPrototypes">

type CalculatorModel = ViewModels.CalculatorModel
type TempConveterModel = ViewModels.TempConveterModel
type StockInfoModel = ViewModels.StockInfoModel
type StockPricesChartModel = ViewModels.StockPricesChartModel
type HexConverterModel = ViewModels.HexConverterModel
type MainModel = ViewModels.MainModel
```
And that's it. Magically we have the same functionality as in dynamic proxy based model.
Key things to note:
  * Model protypes __HAVE__ to be defined in a separate assembly
  * Only fields defined as `mutable` participate in INPC   
  * Models can have dependencies on other models (for example `StockPricesChartModel` depends on `StockInfoModel` or MainModel on all others) 
  * F# record type signals intention to make it view model.
  * Dependencies on other types (like `Operations`) are copied as is
  * [[Derived Properties]] are supported. Property must be read-only and have `[<ReflectedDefinition>]`. You can use type synonym like `type NotifyDependencyChangedAttribute = ReflectedDefinitionAttribute` we had for dynamic proxy based model. I didn't want to drag another dependency into prototype assembly only for single type synonym.  

At first glance design seems very restrictive. But I believe it's advantageous because it takes functional style of data and code separation to the extreme. F# records as prototypes make a good choice - it's very concise, declarative and properly constrained (no augmentations allowed, for example).

There are shortcomings of this implementation which I plan to fix in the future versions:
  * Prototype assembly is not locked anymore but whenever it gets recompiled Visual Studio often shows "... FSC: error FS1135: Unexpected error creating debug information file...". It means you should reopen VS to recompile prototypes assembly.  
  * Some F# language constructs are not supported in derived property bodies. I will provided details on exact limitations in later updates.
  * To support derived properties custom model base class inherits from [DependencyObject] (http://msdn.microsoft.com/en-us/library/system.windows.dependencyobject.aspx). Therefore when user presses "." Intellisense shows derived from 'DependencyObject' members which is not a slick experience.
  * Units of measure are not yet supported
  * WinRT support is coming soon! 

It's was a bumpy road from erased to generated types. It was caused by the lack of other samples and documentation. I would like to thank personally to [@v2_matveev] (http://twitter.com/v2_matveev) from F# team . Without his assistance "generated types" version would not be possible. 

The implementation is a useful sample of "generated types" Type Provider for F# developers. 

### Conclusion

It is an important milestone for this framework/series. Chapters 3-15 each signify a useful feature that is based on some unique F# language capability (comparing to C#). [Chapter 2](Model), the only chapter that could have C# implementation, became obsolete.

INPC Type Provider brought minor performance improvement (build time vs run-time for dynamic proxy). Different model implementation caused no changes in the rest of the framework. Model independent design payed off.

***
At one point I seriously considered building support for C#-friendly view models. But there are no separate model definitions as declarative data structures. OOP with huge piles of imperative code makes generating  new assembly with amended types cost prohibitive. 

P.S. Dynamic proxy based model is still available but was moved to a separate assembly.