For long time I've been obsessed with idea to use F# "Type providers" feature to generate  INotifyPropertyChanged implementation for model. Even before question/idea was brought up on [StackOverflow] (http://stackoverflow.com/questions/13786586/f-type-providers-and-inpc-metaprogramming). Ideally INPC Type Provider usage would like following:
```ocaml
type MyViewModel = NotifyPropertyChanged<SomeType>
```
Unfortunately, type providers in current implementation [do not accept type as parameters] (http://stackoverflow.com/questions/9547225/can-i-provide-a-type-as-an-input-to-a-type-provider-in-f).
As reasonable compromise for the framework I come up with following idea: 
  * There is separate assembly called "model prototypes assembly" where all model types defined
  * To keep it declarative and simple model types should defined as F# records
  * This assembly must be referenced in project where views and controllers defined
  * Assembly short name supplied as input to NotifyPropertyChanged type provider

There are two kinds of F# type providers: erased types and generated types. First is ubiquitously popular and has better development support. As matter fact, to this pint I don't know any serious "generated types" based implementation.
***
It's not that simple to make a design choice between "erased types" and "generated types". I found particularly relevant to INPC Type Provider this note from "Details about Erased Provided Types" section of "Creating a Type Provider" [MSDN tutorial] (http://msdn.microsoft.com/en-us/library/hh361034.aspx):

... When you are writing a provider where accurate runtime-type semantics aren't critical for practical use of the information space.

We'll see later in the chapter how this rule applies to design decisions.
***
###Attemp #1 - "erased types" + ExpandoObject
Being novice in type provider development, I decided to start from pilot version. Naturally, I though to try "erased types" first. Here is test script that shows usage ([github] (https://github.com/dmitry-a-morozov/fsharp-wpf-mvc-series/blob/master/Chapter%2015%20-%20INPCTypeProvider/ErasedTypesPilot/TryExpando.fsx)):
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
`SampleModelPrototypes.dll` assembly contains [definition of Person model] (https://github.com/dmitry-a-morozov/fsharp-wpf-mvc-series/blob/master/Chapter%2015%20-%20INPCTypeProvider/ErasedTypesPilot/SampleModelPrototypes/Person.fs):
```ocaml
type Person = {
    mutable FirstName : string 
    mutable LastName : string
    mutable DateOfBirth : System.DateTime
}
```
Based on `Person`, type accessible via `ViewModel.Pereson` generated with support for INotifyPropertyChanged.
Second assembly "ExpandoObject.dll" contains type provider itself. As you probably guessed, it's "erased types" replaced by [ExpandoObject] (http://msdn.microsoft.com/en-us/library/system.dynamic.expandoobject.aspx) class in final code. Properties getters/setters mapped to dictionary-style set/get key-based invocations. Setting key to value causes ExpandoObject raise INotidyProperyChanged.PropertyChanged event to be risen because ExpandoObject has built-in support for it. Not bad for pilot but this version has numerous problem. I will mention only obvious ones: 
  * No support for INotifyDataErrorInfo ( IDataErrorInfo on .NET 4)
  * Even though data binding to dynamic objects works in WPF it's no-go for platform like WinRT 
(which is one of the primary reasons to replace dynamic proxy based approach with Type Provider). 
  * Data binding to dynamic objects is sub-optimal. Look [here] (http://blogs.msdn.com/b/silverlight_sdk/archive/2011/04/26/binding-to-dynamic-properties-with-icustomtypeprovider-silverlight-5-beta.aspx) for details ("What about WPF and DLR?" section).

###Attemp #2 - "erased types" + custom runtime base class
As next step I introduced [custom run-time class] (https://github.com/dmitry-a-morozov/fsharp-wpf-mvc-series/blob/master/Chapter%2015%20-%20INPCTypeProvider/ErasedTypesPilot/CustomRuntimeClass/Model.fs) as base for models. Here is brief description:
  * Supports INPC and INotifyDataErrorInfo  
  * Uses F# record prototypes as backing storage 
  * Provides dictionary-like key based set/get operations.
  * In absence of real properties to support data binding it implements old-fashion [ICustomTypeDescriptor] (http://msdn.microsoft.com/en-us/library/system.componentmodel.icustomtypedescriptor.aspx) interface which was way easier than new on .NET 4.5 [ICustomTypeProvider] (http://msdn.microsoft.com/en-us/library/system.reflection.icustomtypeprovider.aspx) (which is only choice on Silverlight). For details see [this] (http://blogs.msdn.com/b/silverlight_sdk/archive/2011/04/26/binding-to-dynamic-properties-with-icustomtypeprovider-silverlight-5-beta.aspx).

Type provider [implementation] (https://github.com/dmitry-a-morozov/fsharp-wpf-mvc-series/blob/master/Chapter%2015%20-%20INPCTypeProvider/ErasedTypesPilot/CustomRuntimeClass/NotifyPropertyChangedTypeProvider.fs) didn't change much. Instead of `ExpandoObject` it injects [Model] ((https://github.com/dmitry-a-morozov/fsharp-wpf-mvc-series/blob/master/Chapter%2015%20-%20INPCTypeProvider/ErasedTypesPilot/CustomRuntimeClass/Model.fs) type. 

To test-drive this implementation I wanted to use something more elaborate than script. I tried to simplified version of [[Validation]] chapter application  and kept model related functionality because that what's most affected by INPC Type Provider. I left out concept of view and controller. They are very important but irrelevant for our discussion now. [Data binding] and [Validation] modules were taken as is for now. 
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
Here code that replaced view:
```ocaml
let (?) (window : Window) name : 'T = name |> window.FindName|> unbox

type ViewModels = NotifyPropertyChanged<"SampleModelPrototypes">

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
model.SelectedOperation <- model.AvailableOperations.[0] 

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
```