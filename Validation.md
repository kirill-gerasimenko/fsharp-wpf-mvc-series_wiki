[Complex input validation](http://msdn.microsoft.com/en-us/magazine/ff714593.aspx) is usually done in WPF by implementing [IDataErrorInfo](http://msdn.microsoft.com/en-us/library/system.componentmodel.idataerrorinfo.aspx). There is also [INotifyDataErrorInfo](http://msdn.microsoft.com/en-us/library/system.componentmodel.inotifydataerrorinfo.aspx) interface available in .NET 4.5 and Silverlight, but I plan to have a separate post on taking advantage of .NET 4.5. For now let's consider we're on .NET 4.0. In addition, to have complete user experience proper [visual clues](http://msdn.microsoft.com/en-us/library/ms752347.aspx#invalidation_feedback) have to be set to display errors. 

`Model` has to implement `IDataErrorInfo` and provide some utility methods. Also, the framework has grown big enough to justify a separate module with utility functions and extensions. 
```ocaml
...
[<AutoOpen>]
module Extensions = 
    let inline undefined<'T> = raise<'T> <| NotImplementedException()
...
[<AbstractClass>]
type Model() = 
    ...
    static let notifyPropertyChanged = {
        new StandardInterceptor() with
            member this.PostProceed invocation = 
                match invocation.Method, invocation.InvocationTarget with 
                    | PropertySetter propertyName, (:? Model as model) -> model.ClearError propertyName 
                    | _ -> ()
    }
    
    let errors = Dictionary()
    ...
    interface IDataErrorInfo with
        member this.Error = undefined
        member this.Item 
            with get propertyName = 
                match errors.TryGetValue propertyName with
                | true, message -> message
                | false, _ -> null
    
    member this.SetError(propertyName, message) = 
        errors.[propertyName] <- message
        this.TriggerPropertyChanged propertyName 
    member this.ClearError propertyName = this.SetError(propertyName, null) 
    member this.ClearAllErrors() = errors.Keys |> Seq.toArray |> Array.iter this.ClearError 
    member this.HasErrors = errors.Values |> Seq.exists (not << String.IsNullOrEmpty) 
```
The following details may deserve your attention: 
 * `undefined` - seems to me cleaner than widely-accepted ... `raise(new NotImplementedException())` .... Inspired by Haskell function. 
 * `notifyPropertyChanged` implementation changed to clean up error info on property change. The way it's done is a bit awkward because of the strange nature of `IDataErrorInfo`: to set/remove error `PropertyChanged` of `INotifyPropertyChanged` should be raised first. Introduction of `INotifyDataErrorInfo` will make the code cleaner. 

Another small change we must perform is setting `ValidatesOnDataErrors` in data binding module: 
```ocaml
...
type Expr with
    member this.ToBindingExpr() = 
        ...
        let target : FrameworkElement = (view, [||]) |> window.GetValue |> control.GetValue |> unbox 
        let binding = Binding(path, ValidatesOnDataErrors = true) 
        target.SetBinding(targetProperty.DependencyProperty, binding) 
        ...
```
### Validation module

The deficiency of the current implementation is that `SetError` takes the property name as a string and therefore is not subject to compiler type checks and tooling support. The best way to address this is to use F# quotations. Introducing statically typed version of set error and some supportive types/functions below: 
```ocaml
[<AutoOpen>]
module FSharp.Windows.Validation
...
open Microsoft.FSharp.Quotations
open Microsoft.FSharp.Quotations.Patterns
    
type PropertySelector<'T, 'a> = Expr<('T -> 'a)>
    
let (|SingleStepPropertySelector|) (expr : PropertySelector<'T, 'a>) = 
    match expr with 
    | Lambda(arg, PropertyGet( Some (Var selectOn), property, [])) -> 
        assert(arg.Name = selectOn.Name)
        property.Name, fun(this : 'T) -> property.GetValue(this, [||]) |> unbox<'a>
    | _ -> invalidArg "Property selector quotation" (string expr)
    
let inline setError (SingleStepPropertySelector(propertyName, _) : PropertySelector< ^Model, _>) message model = 
    (^Model : (member SetError : string * string -> unit) (model, propertyName, message))
```
Let's go step by step through the snippet: 
 * `PropertySelector` is a type alias for quotations like `<@ fun model -> model.Property @>` 
 * `SingleStepPropertySelector` is an active pattern that ensures quotation conforms to the shape mentioned above and extracts two things: property name and getter function. 
 * breaking down `setError`: 
  * First parameter of quotation type is deconstructed in place extracting only property name. This technique is similar to [Pattern matching function](http://msdn.microsoft.com/en-us/library/dd233242.aspx).
  * [Member constraints](http://msdn.microsoft.com/en-us/library/dd548046.aspx) are placed on Model type. This is a reason for inlining and hat-notation on model type. The technique is a bit esoteric, you can find more examples [here](http://codebetter.com/matthewpodwysocki/2009/09/28/generically-constraining-f-part-ii/). 
  * This is done not for the sake of trying a cool feature. Considering variety of ways Model can be implemented (hand-written, Castle dynamic proxy, linfu dynamic proxy, IOC proxy, Notify Property Weaver, code-gen, etc.) it places a very lightweight constraint on the use of the `Validation` module: only to have method `SetError` of type: _`string * string -> unit`_.

Now it’s time for a usage example. If we want to display error whenever user tries to divide by zero in our sample calculator, the following code can appear in controller: 
```ocaml
...
    member this.Calculate model = 
        match model.SelectedOperation with
        ...
        | Divide -> 
            if model.Y = 0 
            then 
                model |> Validation.setError <@ fun m -> m.Y @> "Attempted to divide by zero." 
            else 
                model.Result <- model.X / model.Y 
```
Visually user will see this picture when trying division by zero Y 

[[Images/DivisionByZeroValidationError.png]]

Some other useful validation functions: 
```ocaml
...
let inline clearError expr = setError expr null
    
let inline invalidIf (SingleStepPropertySelector(propertyName, getValue : ^Model -> _)) predicate message model = 
    if model |> getValue |> predicate 
    then 
        (^Model : (member SetError : string * string -> unit) (model, propertyName, message)) 
    
let inline assertThat expr predicate = invalidIf expr (not << predicate)
    
let inline objectRequired expr = invalidIf expr ((=) null) "Required field."
let inline valueRequired expr = assertThat expr (fun(x : Nullable<_>) -> x.HasValue) "Required field."
let inline textRequired expr = invalidIf expr String.IsNullOrWhiteSpace "Required field."
...
```

Function like _`clearError`_ is built by means of [partial application](http://en.wikipedia.org/wiki/Partial_application) - one of the key composition tools of functional languages. 

I would like to show another cool trick. Let's say we want to enforce second argument for addition and subtraction to always be a positive number. 

Below is `SampleController`'s code snippet. 
```ocaml
...
    member this.Calculate model = 
        ...
        match model.SelectedOperation with
        | Add -> 
            model |> Validation.positive <@ fun m -> m.Y @>
            if not model.HasErrors
            then 
                model.Result <- model.X + model.Y
        | Subtract -> 
            model |> Validation.positive <@ fun m -> m.Y @>
            if not model.HasErrors
            then 
                model.Result <- model.X - model.Y
        ...
...
```
A sensible implementation can look like: 
```ocaml
let inline positive expr = assertThat expr (fun x -> x > 0) "Must be positive number."
```

Type signature inferred by compiler is: 

[[Images/PossibleImplValidationPositive.png]]

meaning this function is only suitable to work with int numbers. 

Wouldn't it be nice to have the same function to work with other kind of numbers: floats, decimals, etc. to avoid code duplication? [F# language primitives](http://msdn.microsoft.com/en-us/library/ee340276.aspx) which are based on "Member constraints" are to the rescue. To make it even more reusable we split it into two functions: one is in `Extensions` module, and the other is in `Validation`. 

```ocaml
open LanguagePrimitives
...
module Extensions = 
    ...
    let inline positive x = GenericGreaterThan x GenericZero 
... 
module FSharp.Windows.Validation
    ...
    let inline positive expr = assertThat expr positive "Must be positive number."
...
```

Let's look at the type signature again: 

[[Images/GenericImplValidationPositive.png]]

This function can work with any F# built-in numerics because it is based only on two constraints: `get_Zero` and `comparison`. Function `Extensions.positive` can be used in other places where generic verification for positive numbers is required: that's why we put it in a separate module. 

It's worth noting that the module is based on four language features: pattern matching, explicit member constraints, inlining and partial application. *None of them are available in C#.*

### P.S.
I also added convenient utility method: 

```ocaml
[<RequireQualifiedAccess>]
module Observable =
    let mapTo value = Observable.map(fun _ -> value)
```

It allows to write more palatable code to map View events:
```ocaml
type SampleView() =
    ...
    override this.EventStreams = 
        [
            this.Window.Calculate.Click |> Observable.mapTo Calculate
            this.Window.Clear.Click |> Observable.mapTo Clear
        ]
    ...
```
I was thinking to call method Observable.const following the established Haskell naming convention, but decided against this for two reasons: const is reserved keyword in F#, and mapTo seems readable. 