Let's have another look at MVC components [diagram](Intro). There is an implicit dependency between View and Model, even if it exists only at run time. Hiding it behind a generic obj type took away all the benefits of the statically typed language: compiler-checked program correctness and better tooling support (!IntelliSense, refactoring, etc.). Unfortunately, this is the usual way data binding is implemented for being generic enough. 

Here is a more simple and specific example to illuminate the problem. Let us assume during refactoring we renamed `X` and `Y` `SampleModel` properties to `Arg1` and `Arg2` respectively. Compile. What do we see? Hah, obviously we need to change code in `SampleController` too. Done. So far so good. Compile again. Looks like no errors. Run the application. Strange: there is no exception and still it doesn't work. Let's look at debug output window. Aha!  
```
 System.Windows.Data Error: 40 : BindingExpression path error: '...' property not found on 'object' ''...' (HashCode=21520579)'. BindingExpression:Path=...; DataItem='...' (HashCode=21520579); target element is 'TextBox' (Name='...'); target property is 'Text' (type 'String')
```
I'm sure this is a painful experience any WPF/Silverlight/Winforms developer is going through quite often. Is there a better way? Of course. 

First, let's update IView definition in order to reflect dependency on Model: 
```ocaml
type IView<'Events, 'Model> =
    inherit IObservable<'Event>
    
    abstract SetBindings : 'Model -> unit
```
'Mvc' changes too because of the transitive dependency: : 
```ocaml
...
type Mvc<'Events, 'Model when 'Model :> INotifyPropertyChanged>(model : 'Model, view : IView<'Events, 'Model>, controller : IController<'Events, 'Model>) =
...
```

It's good to have explicitly declared dependency, but we don't take advantage of it anywhere in the code i.e. the renaming problem is still not addressed. Introducing: 

### Statically typed data binding
The idea is simple - mapping F# quotation, like 
```ocaml
<@ control.Property <- model.Property @>
```
to the data binding expression roughly making it equivalent to the imperative call 
```ocaml
control.SetBinding(Control.DependencyProperty, "ModelProperty")
```
This approach is a simple variation of [Language](http://tomasp.net/blog/fsharp-iv-lang.aspx) Oriented [Programming](http://blogs.msdn.com/b/chrsmith/archive/2008/05/30/language-oriented-programming-in-f.aspx): in the specific context of data binding we give a different meaning to the standard F# assignment statement.

Let's look at another practical example. For this one we'll extend the sample calculator window in order to support multiple operations selected using a drop-down list. 

[[Images/BasicBindingWindow.png]]

Code in `SetBindings` method will look like the following: 
```ocaml
    override this.SetBindings model = 
        //Binding to property of type string
        <@ this.Window.X.Text <- model.X @>.ToBindingExpr()
        <@ this.Window.Y.Text <- model.Y @>.ToBindingExpr()
        <@ this.Window.Result.Text <- model.Result @>.ToBindingExpr()
        //Binding to property of type IEnumerable
        <@ this.Window.Operation.ItemsSource <- model.AvailableOperations @>.ToBindingExpr()
        //Binding to property of type obj
        <@ this.Window.Operation.SelectedItem <- model.SelectedOperation @>.ToBindingExpr()
```
Sweet! This code can be verified by compiler. It means that for our property rename we'll get compiler error instead of just text warning in debug output window. 

The implementation of `ToBindingExpr` extension method is given below: 

```ocaml
[<AutoOpen>]
[<CompilationRepresentation(CompilationRepresentationFlags.ModuleSuffix)>]
module FSharp.Windows.Data
    
open System.Reflection
open System.Windows
open System.Windows.Data 
open Microsoft.FSharp.Quotations
open Microsoft.FSharp.Quotations.Patterns
    
type PropertyInfo with 
    member this.DependencyProperty : DependencyProperty = 
        this.DeclaringType 
            .GetField(this.Name + "Property", BindingFlags.Static ||| BindingFlags.Public ||| BindingFlags.FlattenHierarchy) 
            .GetValue(null, [||]) 
            |> unbox    
    
type Expr with 
    member this.ToBindingExpr() = 
        match this with
        | PropertySet  
            (
                Some( FieldGet( Some( PropertyGet( Some (Value( view, _)), window, [])), control)),
                targetProperty, 
                [], 
                PropertyGet( Some( Value _), sourceProperty, [])
            ) ->
                let target : FrameworkElement = (view, [||]) |> window.GetValue |> control.GetValue |> unbox
                let bindingExpr = target.SetBinding(targetProperty.DependencyProperty, path = sourceProperty.Name)
                assert not bindingExpr.HasError
   
        | _ -> invalidArg "expr" (string this) 
```
The code is relatively straightforward by F# standards. I'd like to contrast it with hypothetical C# implemenation:

  | F# | C# 
---|----|---------
Representation | F# [Code Quotations](http://msdn.microsoft.com/en-us/library/dd233212.aspx) | Unfortunately, assignment statements are not allowed inside [Expression Trees](http://msdn.microsoft.com/en-us/library/bb397951.aspx) created from Lambda Expressions. Building them manually defeats the purpose. In comparision, F# quotations are full-blown i.e. support complete language.
Parsing/processing | [Predefined](http://msdn.microsoft.com/en-us/library/ee370259.aspx) active patterns. | Hypothetically, if assignment statement limitation doesn't exist, to parse expression tree one should use [Visitor](http://msdn.microsoft.com/en-us/library/system.linq.expressions.expressionvisitor.aspx) - a lot of [tedious](http://msdn.microsoft.com/en-us/library/bb546136.aspx) coding.
Extension property - readability | `PropertyInfo.DependencyProperty` | NA 
Module names same as classes - cohesion | module FSharp.Windows.Data. Using `[<CompilationRepresentation`<br/>`<(CompilationRepresentationFlags.ModuleSuffix)>]` attribute | There are no modules in C# but names of static classes (often used as containers for extensions) cannot clash with other class names either.

Here is `SampleModel` we expected to see: 
```ocaml
type Operations =
    | Add = 0
    | Subtract = 1 

[<AbstractClass>]
type SampleModel() = 
    inherit Model()
    
    abstract AvailableOperations : Operations[] with get, set
    abstract SelectedOperation : Operations with get, set
    abstract X : int with get, set
    abstract Y : int with get, set
    abstract Result : int with get, set
```
Compilation will fail with: "Error 3 This expression was expected to have type string but here has type int ..." pointing to expression 
```ocaml
<@ this.Window.X.Text <- model.X @> ...
```
Ouch! We are used to data binding magically knowing how to convert almost any type to string or object and back. But F# compiler won't accept such inaccuracy. For now, let's make compiler happy and convert `X`, `Y`, and `Result` model properties to strings. Some `SampleController` code requires fixing as well. Compile again. All is fine. Run... 
```
System.ArgumentException Message=... Parameter name: expr
```
Another Ouch! Let us explain what's happening here. F# assignment expression 
```ocaml
<@ this.Window.Operation.SelectedItem <- model.SelectedOperation @> 
```
gets converted in compile time to 
```ocaml
<@ this.Window.Operation.SelectedItem <- model.SelectedOperation :> obj @> 
```
F# compiler does these tricks to have smoother integration with imperative nature of .NET/BCL. Let's make a quick-and-dirty fix and convert `SelectedOperation` to obj. Now it compiles and runs just fine. 

Let's pause here for a moment to see where we are. View dependency on Model is explicit now. A small "rename" problem is solved too, but `Model` is less typed in some sense, having `string` and `obj` instead of real types. I intentionally made us jumping through these hoops, so it will be clear what the problem is we're trying to solve. 

Obj coercion needs to be fixed inside pattern matching of `ToBindingExpr` method. In order to resolve binding for handling a `Text` property of type string we need to be more explicit and modify quotation: 
```ocaml
<@ this.Window.X.Text <- string model.X @> ...
```
Keeping in mind the idea of Language Oriented Programming we'll give the following interpretation of this approach: let’s make compiler ignore type discrepancy, but use data binding magic for converting to and from string; "string" function call is a fake shim. At one point I thought of making a generic version of such shim similar to [Seq.cast](http://msdn.microsoft.com/en-us/library/ee370344.aspx), but then backed up. Surely, binding knows how to deal with obj and string conversions, but for everything else (boolean, for example) it is better to have the explicit converter. We 'll see examples of such converters later in the series. 

Another stylistic improvement is to support batching for binding quotations as opposed to doing it on individual basis. Resulting `SampleModel` (all strong typed: no string or obj) and `SetBindings` method of `SampleView` are given below: 
```ocaml
type Operations =
    | Add
    | Subtract

    override this.ToString() = sprintf "%A" this

[<AbstractClass>]
type SampleModel() = 
    inherit Model()

    abstract AvailableOperations : Operations[] with get, set
    abstract SelectedOperation : Operations with get, set
    abstract X : int with get, set
    abstract Y : int with get, set
    abstract Result : int with get, set
...
type SampleView() =
...    
override this.SetBindings model = 
    Binding.FromExpression 
        <@ 
            this.Window.Operation.ItemsSource <- model.AvailableOperations 
            this.Window.Operation.SelectedItem <- model.SelectedOperation 
            this.Window.X.Text <- string model.X 
            this.Window.Y.Text <- string model.Y 
            this.Window.Result.Text <- string model.Result 
        @>
```
And implementation:

```ocaml
...
module FSharp.Windows.Binding
...
let rec (|PropertyPath|_|) = function 
    | PropertyGet( Some( Value _), sourceProperty, []) -> Some sourceProperty.Name
    | Coerce( PropertyPath path, _) 
    | SpecificCall <@ string @> (None, _, [ PropertyPath path ]) -> Some path
    | _ -> None
    
type Expr with
    member this.ToBindingExpr() = 
        match this with
        | PropertySet
            (
                Some( FieldGet( Some( PropertyGet( Some (Value( view, _)), window, [])), control)),
                targetProperty, 
                [], 
                PropertyPath path  
            ) ->
                let target : FrameworkElement = (view, [||]) |> window.GetValue |> control.GetValue |> unbox
                target.SetBinding(targetProperty.DependencyProperty, path)
        | _ -> invalidArg "expr" (string this) 
     
type Binding with
    static member FromExpression expr = 
        let rec split = function 
            | Sequential(head, tail) -> head :: split tail
            | tail -> [ tail ] 
    
        for e in split expr do
            let be = e.ToBindingExpr()
            assert not be.HasError
```
Again, behold the power and beauty of F# language :
 * No explicit type declarations are given 
 * *Recursive* `PropertyPath` active pattern is factored out to improve readability, reuse, and testing 
 * Powerful predefined active patterns like [SpecificCall](http://msdn.microsoft.com/en-us/library/ee370245.aspx) and [Sequential](http://msdn.microsoft.com/en-us/library/ee353710.aspx) used in implementation
 * `Binding.FromExpression` - *static* extension member (hello C#!)

This approach of setting up data binding from quotations prevents another kind of silly mistakes or typos. Without it nothing prevents you from writing a code like one below: 
```ocaml
this.Window.X.SetBinding(TextBlock.TextProperty, "X") ...
```
or even this one: 
```ocaml
this.Window.X.SetBinding(CheckBox.IsCheckedProperty, "X") ...
```
Both use wrong dependency properties: `TextBlock.TextProperty` instead of `TextBox.TextProperty` in the first case and completely bogus `CheckBox.IsCheckedProperty` in the second. It's really the worst case scenario - no compile error, no run-time exception, no warning in Debug Output, nothing. Except that your application doesn't work as expected. 

Using `DependencyProperty` extension property as part of the implementation guarantees the right dependency property is used. Validity is always preserved. 

I realize that the suggested version of Binding support is somewhat limited and can be difficult to use in real applications. But please be patient, [later](Data-Binding.-Growing-Micro-DSL) in the series we'll have another post dedicated to Binding where we'll roll out a production-quality library. 

### Controller base class

Explicit implementation of `IContoller<_, _>` interface introduced in previous chapter breaks type inference  down a bit. `model` parameter in `Add` and `Subtract` event handlers had to be type annotated. A reason is unknown to me because it seems like compiler has enough information to figure it out on its own. To alleviate the problem this chapter defines `Controller<_, _>` base class. Subclass it and type annotations are not needed. 
```ocaml
[<AbstractClass>]
type Controller<'Events, 'Model when 'Model :> INotifyPropertyChanged>() =

    interface IController<'Events, 'Model> with
        member this.InitModel model = this.InitModel model
        member this.EventHandler = this.EventHandler

    abstract InitModel : 'Model -> unit
    abstract EventHandler : ('Events -> 'Model -> unit)

    static member FromEventHandler callback = {
        new IController<'Events, 'Model> with
            member this.InitModel _ = ()
            member this.EventHandler = callback
    } 
...
type SimpleController() = 
    inherit Controller<SampleEvents, SampleModel>()

    override this.InitModel model = 
        ...

    override this.EventHandler = function
        | Calculate -> this.Calculate
        | Clear -> this.InitModel

    member this.Calculate model = 
        ...
```
There is downside - it makes user-defined more tightly couple to base class. 
To sum up, it gives a choice to define controller in three different ways:

1. Define event handler function. Create controller instance by calling static `Controller.FromEventHandler` factory method. 
2. Just implement `IController<'Events, 'Model>` (be prepared to add some type annotations)
3. Inherit from `Controller<_, _>` and accept some coupling.

### Mvc.start shortcut

There is new function that allows to start Mvc when `'Model` has static member `Create` factory method:
```ocaml
[<RequireQualifiedAccess>]
module Mvc = 

    let inline start(view, controller) = 
        let model = (^Model : (static member Create : unit -> ^Model ) ())
        Mvc<'Events, ^Model>(model, view, controller).Start()
```
Usage:
```ocaml
Mvc.start(view, controller) ...
```