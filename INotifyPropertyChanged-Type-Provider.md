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