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
###Round #1 - "erased types" + ExpandoObject
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
  * Data binding to dynamic objects works in WPF it's no-go for platform like WinRT 
(which is one of the primary reasons to replace dynamic proxy based approach with Type Provider). 
  * Data binding to dynamic objects is sub-optimal. Look [here] (http://blogs.msdn.com/b/silverlight_sdk/archive/2011/04/26/binding-to-dynamic-properties-with-icustomtypeprovider-silverlight-5-beta.aspx) for details ("What about WPF and DLR?" section).

###Round #2 - "erased types" + custom runtime base class
As next step I introduced [custom run-time class] (https://github.com/dmitry-a-morozov/fsharp-wpf-mvc-series/blob/master/Chapter%2015%20-%20INPCTypeProvider/ErasedTypesPilot/CustomRuntimeClass/Model.fs) as base for models. Here is brief description:
  * Supports INPC and INotifyDataErrorInfo  
  * Uses F# record prototypes as backing storage 
  * Provides dictionary-like key based set/get operations.
  * Usually data binding engine in WPF relies on reflection in run-time. But in our case real type/properties do not exits in compiled code. As substitution, model custom base class implements [ICustomTypeDescriptor] (http://msdn.microsoft.com/en-us/library/system.componentmodel.icustomtypedescriptor.aspx) interface. Alternatively, it could implement new on .NET 4.5 [ICustomTypeProvider] (http://msdn.microsoft.com/en-us/library/system.reflection.icustomtypeprovider.aspx), which is also the only choice on Silverlight. But it's more tedious despite of what some people [say] (http://blogs.msdn.com/b/silverlight_sdk/archive/2011/04/26/binding-to-dynamic-properties-with-icustomtypeprovider-silverlight-5-beta.aspx). 

Type provider [implementation] (https://github.com/dmitry-a-morozov/fsharp-wpf-mvc-series/blob/master/Chapter%2015%20-%20INPCTypeProvider/ErasedTypesPilot/CustomRuntimeClass/NotifyPropertyChangedTypeProvider.fs) didn't change much. Instead of `ExpandoObject` it injects [Model] ((https://github.com/dmitry-a-morozov/fsharp-wpf-mvc-series/blob/master/Chapter%2015%20-%20INPCTypeProvider/ErasedTypesPilot/CustomRuntimeClass/Model.fs) type. 

To test-drive this implementation I wanted to use something more elaborate than script. I tried to simplified version of [[Validation]] chapter application  and kept model related functionality because that what's most affected by INPC Type Provider. I left out concept of view and controller. They are very important but irrelevant for our discussion now. [[Data binding]] and [[Validation]] modules were taken as is for now. 
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
Running application without more changes gives us:
[[Images/INPC.TypeProvider.Broken.Binding.png]]

This is because even though we satisfied WPF data binding engine requirment by providing implementation of `ICustomTypeDescriptor` our data binding DSL still expects real typed properties (ahh! those run-time semantics). Small change in `Binding` module to fix the issue:
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
Validation module has to be fixed for very same reason:
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
Now it works identically to application from [[Validation]] chapter. Is that all? Not really. This solution still has numerous problems:
  1. Those changes we made to binding and validation modules feels like hacks. They violate "Separation of concerns" principle. I was very keen to keep the framework design independent to model implementation (to remind: the only requirement for model is to implement INotifyPropertyChanged). Fact that we have different (type provider based) implementation should not force changes in binding or validation.
  2. The solution can either hard to port to WinRT (which is again one of the drivers for my work on INPC type provider). Thought it still seems to be [possible] (http://jaylee.org/post/2012/03/07/Xaml-integration-with-WinRT-and-the-IXamlMetadataProvider-interface.aspx) but seems like "generated types" [enable easier solution] (http://jaylee.org/post/2012/11/26/DataBinding-performance-in-WinRT-and-the-Bindable-attribute.aspx).
  3. Going forward I certainly plan to implement same [[Derived Properties]] feature as one for dynamic proxy based models. I don't see how it can be implemented for "erased types".
  4. Once user refers to model prototypes assembly binary it's locked. Visual Studio has be re-opened in case prototypes assembly needs be recompiled which is certainly sub-optimal experience.

#4 is really nasty problem. I spent quite some looking for solution (like custom remote domain) but hit the wall. 

#1, #2 and #3 can be solved if I'll switch to "generated types" type provider. I'm lookinf forward to this  challenge.