With all the sophistication built into Binding DSL we still could not solve problem #1 from [previous chapter](Data-Binding. Growing Micro DSL). To remind, we'd like to show `ProcessName` and `ActiveTab` name on the title of main window. `MainModel` has two properties to hold proper state and there is `MultiBinding` with `StringFormat` property set usage in `MainView.SetBinding`: 

```ocaml
type MainModel() = 
    ...
    abstract ProcessName : string with get, set
    abstract ActiveTab : string with get, set
    
type MainEvents = 
    | ActiveTabChanged of string 
    
type MainView() as this = 
    inherit View<MainEvents, MainModel, MainWindow>()
    ...
    override this.EventStreams = 
        [   
            yield this.Control.Tabs.SelectionChanged |> Observable.map(fun _ -> 
                let activeTab : TabItem = unbox this.Control.Tabs.SelectedItem 
                let header = string activeTab.Header 
                ActiveTabChanged header) 
            ...
        ]
    
    override this.SetBindings model = 
        let titleBinding = MultiBinding(StringFormat = "{0} - {1}") 
        titleBinding.Bindings.Add <| Binding("ProcessName") 
        titleBinding.Bindings.Add <| Binding("ActiveTab") 
        this.Control.SetBinding(Window.TitleProperty, titleBinding) |> ignore 
        ...

type MainController...
    
    override this.InitModel model = 
        model.ProcessName <- Process.GetCurrentProcess().ProcessName 
        model.ActiveTab <- "Calculator" 
        ...
    
    override this.Dispatcher = Sync << function 
        | ActiveTabChanged header -> this.ActiveTabChanged header 
        ...
    
    member this.ActiveTabChanged header model = 
        model.ActiveTab <- header 
```
Note that `ProcessName` value is static while `ActiveTab` gets updated by event handler. 

Technically, this problem can be solved by extending Binding DSL to accept `String.Format` call with multiple parameters. This will result in following code: 
```ocaml
type MainView()...
    ...
    override this.SetBindings model = 
        Binding.FromExpression 
            <@ 
                this.Control.Title <- String.Format("{0} - {1}", model.ProcessName, model.ActiveTab)
             @>
```

In order for this to work [MultiBinding](http://blogs.mscommunity.net/blogs/borissevo/archive/2009/01/20/wpf-trick-3-multibinding-and-stringformat.aspx) support for  has to be introduced and this is not simple at all. We expressed everything in terms of [Binding](http://msdn.microsoft.com/en-us/library/system.windows.data.binding.aspx) class. This class along with [MultiBinding](http://msdn.microsoft.com/en-us/library/ms613634.aspx) are siblings sharing the common parent [BindingBase](http://msdn.microsoft.com/en-us/library/system.windows.data.bindingbase.aspx). Realistically, to make this work we should reformulate everything in terms of `BindingBase` and then use downcasts extensively. This is going to be ugly. To a certain extent it's a case of [Expression problem](http://en.wikipedia.org/wiki/Expression_problem) for functional languages - it's easy to add new operations, but really hard to add new types. Fortunately, there is a much better solution. 

Let's write down the ideal final solution we're looking for: 

```ocaml
type MainModel()... 
    ... 
    abstract ProcessName : string with get, set 
    abstract ActiveTab : TabItem with get, set 
    ... 
    member this.Title = sprintf "%s-%O" this.ProcessName this.ActiveTab.Header 
    ... 
type MainView()... 
    ... 
    override this.SetBindings model = 
        Binding.FromExpression 
            <@ 
                this.Control.Tabs.SelectedItem <- model.ActiveTab 
                this.Control.Title <- model.Title 
             @> 
```

This is impressive comparing to what we had: short, declarative, no imperative code, statically typed. `MainModel.Title` property is a key to the puzzle. This is what we call "derived" property - its value is a computation based on other properties `ProcessName` and `ActiveTab.Header`. As one might guess, `PropertyChanged("Title")` notification should be sent if either `ProcessName` or `ActiveTab.Header` have changed. Formally speaking this is a case of transitive dependency. Worth noting, that [ F# sprintf type-safe solution](http://diditwith.net/2008/01/16/WhyILoveFTypesafeFormatStrings.aspx) is better, than non-typed (obj-based) String.Format we would have, if taking the road of extending DSL with multi-binding. 

Others came up with different solutions to the same problem: 
  * IL [rewriting](http://www.sharpcrafters.com/solutions/notifypropertychanged) or [weaving](https://github.com/SimonCropp/NotifyPropertyWeaver). This seems to be to invasive until [Roslyn](http://msdn.microsoft.com/en-us/vstudio/roslyn.aspx) officially comes in. Also, this one doesn't handle nested dependency like model.property1.property2. 
  * C# [expression trees-based approach](http://knockoutcs.com/index.html). The major disadvantage is that "derived" properties are not looking anymore like normal ones and therefore are excluded from all language and tooling support. 

To provide implementation for the code above we need to dig really deep both into F# and WPF. 

### Step 1 - extract dependencies

First part of our plan is to parse "derived" property implementation and extract dependencies. As usual, [Quotations](http://msdn.microsoft.com/en-us/library/dd233212.aspx) are to rescue. To preserve original property we'll use powerful [ReflectedDefinitionAttribute](http://msdn.microsoft.com/en-us/library/ee353643.aspx), which, when applied to a method (or other language entity) instruct F# compiler to make the quotation expression that implements the method available for use at runtime. To have better contextual name we'll define a type synonym: 

```ocaml
type NotifyDependencyChangedAttribute = ReflectedDefinitionAttribute 
    
type MainModel() = 
    ... 
    [<NotifyDependencyChanged>] 
    member this.Title = sprintf "%s-%O" this.ProcessName this.ActiveTab.Header 
    ...
```

So, one should opt-in explicitly for "derived" property support. 

Once we have body quotation of "derived" property, the next thing is to parse it using [MethodWithReflectedDefinition](http://msdn.microsoft.com/en-us/library/ee353670.aspx) active pattern. Savvy F# developer might ask why not to use more specific [PropertyGetterWithReflectedDefinition](http://msdn.microsoft.com/en-us/library/ee370556.aspx) if we deal with read-only properties? As we'll see later this is due to constrains placed by [Castle Dynamic Proxy](http://www.castleproject.org/). 

```ocaml
...
module Mvc.Wpf.Binding
    ...
    let (|SourceAndPropertyPath|_|) expr = 
        ...
    let (|PropertyPath|_|) = function | SourceAndPropertyPath(path, _) -> Some path | _ -> None
    ...
module ModelExtensions = 
    type Expr with
        
        member this.ExpandLetBindings() = 
            match this with 
            | Let(binding, expandTo, tail) -> 
                tail.Substitute(fun var -> if var = binding then Some expandTo else None).ExpandLetBindings() 
            | ShapeVar var -> Expr.Var(var) 
            | ShapeLambda(var, body) -> Expr.Lambda(var, body.ExpandLetBindings())  
            | ShapeCombination(shape, exprs) -> ExprShape.RebuildShapeCombination(shape, exprs |> List.map(fun e -> e.ExpandLetBindings())) 
        
        member this.Dependencies = 
            seq { 
                match this with  
                | SourceAndPropertyPath x -> yield x 
                | ShapeVar _ -> () 
                | ShapeLambda(_, body) -> yield! body.Dependencies   
                | ShapeCombination(_, exprs) -> for subExpr in exprs do yield! subExpr.Dependencies 
            }
    
    let (|DependentProperty|_|) (memberInfo : MemberInfo) = 
        match memberInfo with 
        | :? MethodInfo as getter -> 
            match getter with 
            | PropertyGetter propertyName & MethodWithReflectedDefinition (Lambda (model, propertyBody)) -> 
                let binding = MultiBinding() 
                let self = Binding(path = "", RelativeSource = RelativeSource.Self) 
                binding.Bindings.Add self 
                
                propertyBody 
                    .ExpandLetBindings() 
                    .Dependencies 
                    |> Seq.choose(function 
                        | Some source, path when source = model -> Some(Binding(path, RelativeSource = RelativeSource.Self)) 
                        | _ -> None) 
                    |> Seq.distinct 
                    |> Seq.iter binding.Bindings.Add 
                
                binding.Converter <- { 
                    new IMultiValueConverter with 
                        
                        member this.Convert(values, _, _, _) = 
                            if values.Contains DependencyProperty.UnsetValue 
                            then 
                                DependencyProperty.UnsetValue 
                            else 
                                try 
                                    let model = values.[0] 
                                    getter.Invoke(model, [||]) 
                                with _ -> 
                                    DependencyProperty.UnsetValue 
                        
                        member this.ConvertBack(_, _, _, _) = undefined 
                } 
                Some(propertyName, getter.ReturnType, binding) 
            | _ -> None 
        | _ -> None 
```

It's a lot of code. Let's go step-by-step through it. 
  * `SourceAndPropertyPath` is active pattern similar to `PropertyPath` we used in [BindingMicroDSL previous chapter]. 

[[Images/DerivedProperties.SourceAndPropertyPath.ActivePattern.TypeSignature.png]]

The only difference is that it extracts binding source in addition to property path. For example, expression `this.ActiveTab.Header` in the body of `model.Title` will be parsed as `(model, "ActiveTab.Header")` tuple. As a matter of fact, `PropertyPath` now reuses `SourceAndPropertyPath` by discarding first item of resulting tuple. 
  * We'll skip `ExpandLetBindings` for now, because there is a more elaborate example down below. For now let's pretend it does nothing, like an identity function. 
  * Recursive property Dependencies extracts them all from a given quotation as a sequence of (model, property path) pairs. 

[[Images/DerivedProperties.Expr.Dependencies.Property.TypeSignature.png]]

See Tomas Petricek’s [example](http://fssnip.net/1i) on recursive quotation parsing.

  * To understand `DependentProperty` active pattern let's have a look at the type signature first: 

[[Images/DerivedProperties.DependentProperty.ActivePattern.TypeSignature.png]]

It takes `MemberInfo` (it would take [PropertyInfo](http://msdn.microsoft.com/en-us/library/system.reflection.propertyinfo.aspx), if we were not constrained by Castle Dynamic Proxy API) and returns a triple: property name, property return type and multi-binding instance. First two are obvious. Binding will have a collection of all dependencies converted to a regular single binding prepended with model self. There is also a converter attached. It invokes the saved reference to property complied implementation (getter) passing in the model instance as the first parameter. Converter invocation gets triggered when some dependencies change. Thus target of this multi-binding will get updated with property value whenever any of dependencies get changed. 

The best place to apply this logic is [intercept proxy creation](http://kozmic.pl/2009/01/17/castle-dynamic-proxy-tutorial-part-iii-selecting-which-methods-to) and specifically [handle non-virtual methods](http://kozmic.pl/2009/02/23/castle-dynamic-proxy-tutorial-part-vi-handling-non-virtual-methods). There is [whole tutorial](http://kozmic.pl/dynamic-proxy-tutorial/) on Castle Dynamic Proxy by Krzysztof Koźmic. Here is our implementation of `IProxyGenerationHook`: 

```ocaml
type Model() = 
    inherit DependencyObject()

    static let dependentProperties = Dictionary()
    ...
    static let options = 
        ProxyGenerationOptions( 
            Hook = { 
                new IProxyGenerationHook with 
                    member this.NonProxyableMemberNotification(_, member') = 
                        match member' with 
                        | DependentProperty(propertyName, propertyType, binding) -> 
                            let perTypeDependentProperties = 
                                match dependentProperties.TryGetValue member'.DeclaringType with 
                                | true, xs -> xs 
                                | false, _ -> 
                                    let xs = List() 
                                    dependentProperties.Add(member'.DeclaringType, xs) 
                                    xs 
                            let dp = DependencyProperty.Register(propertyName, propertyType, member'.DeclaringType) 
                            perTypeDependentProperties.Add(dp, binding) 
                        | _ -> () 
                    member this.ShouldInterceptMethod(_, method') = 
                        match method' with 
                        | PropertyGetter _ | PropertySetter _ -> method'.IsVirtual 
                        | _ -> false 
                    member this.MethodsInspected() = () 
            } 
        ) 
    ...
```

The hook invoked once per (model) type. "Derived" properties by definition are non-virtual therefore can be handled by `NonProxyableMemberNotification` method. Second parameter member' of type `MemberInfo` not `PropertyInfo` that is limitation mentioned above and the reason for `DependentProperty` active pattern to accept input of `MemberInfo` type. `NonProxyableMemberNotification` checks if input member is "derived" property. If so, it extracts property name, return type and multi-binding. Then, it adds to property the host type (which is model) WPF [custom dependency property](http://msdn.microsoft.com/en-us/library/ms753358.aspx) by calling `DependencyProperty.Register`. Dependency property has the same name and type. Both dependency property and multi-binding are added to static map `dependentProperties`. 

[[Images/DerivedProperties.dependentProperties.LocalBinding.png]]

By providing implementation of `ShouldInterceptMethod` we noted that interception should be applied to virtual property getters/setters. 

The final touch is modified `Model.Create` method:
```ocaml
...
type Model() = 
    ...
    static member Create<'T when 'T :> Model and 'T : not struct>()  : 'T = 
        let interceptors : IInterceptor[] = [| notifyPropertyChanged; AbstractProperties() |] 
        let model = proxyFactory.CreateClassProxy(options, interceptors) 
        match dependentProperties.TryGetValue typeof<'T> with 
        | true, xs -> 
            for dp, binding in xs do 
                let bindingExpression = BindingOperations.SetBinding(model, dp, binding) 
                assert not bindingExpression.HasError 
        | false, _ -> () 
        model 
    ...
```

After model has been created, the method looks up for the list of Dependency Properties (which are "derived" properties) and multi-bindings associated with them. Then it binds those pairs using model instance as a target. Now this dependency property gets updated with value of "derived" property whenever any of dependencies changes. To be WPF data binding target Model has to inherit from [DependencyObject](http://msdn.microsoft.com/en-us/library/system.windows.dependencyobject.aspx).

The last piece of puzzle is still missing. Those Custom Dependency Properties have same name as "derived" properties for a reason. Whenever WPF data binding engine sees call like : 
```ocaml
    control.SetBinding(DependencyProperty, "SomeProperty")
```


it always looks up for `DependencyProperty` with the same name first. Only if this was not successful it then tries regular .NET properties. 

Again, it's the case of transitive dependency. The big picture is following: 
  # Call `Binding.FromExpression <@ control.Text <- model.FullName @>` where `FullName` is "derived" property (non-virtual, read-only, annotated  with `[<NotifyDependencyChanged>]`) binds `control.Text` to `DependencyProperty` with name `FullName`.
  # Any dependency can change (let's by say setting property `model.FullName = ...`)
  # Through `MultiBinding` `DepedencyProperty` with name `FullName` gets updated
  # Through regular `Binding` control property `Text` gets updated too.

There is one more example of "derived" property. It's not much different from `MainModel.Title`, but hopefully will improve the understanding.  

For our "Stock Prices" control we'll change interface to have [Accumulation/Distribution](http://www.investopedia.com/terms/a/accumulationdistribution.asp) calculation and subset of editable properties (Last Price, High, Low and Volume) needed to perform it. 

[[Images/DerivedProperties.Accumulation-Distribution.png]]

Some minor changes to model are: 
```ocaml 
...
type StockInfoModel() = 
    inherit Model()
    
    abstract Symbol : string with get, set
    abstract CompanyName : string with get, set
    abstract LastPrice : decimal with get, set
    abstract DaysLow : decimal with get, set
    abstract DaysHigh : decimal with get, set
    abstract Volume : decimal with get, set
    ...
    [<NotifyDependencyChanged>]
    member this.AccDist = 
        let moneyFlowMultiplier = (this.LastPrice - this.DaysLow) - (this.DaysHigh - this.LastPrice) / (this.DaysHigh - this.DaysLow)
        let moneyFlowVolume  = moneyFlowMultiplier * this.Volume
        sprintf "Accumulation/Distribution: %M" <| Decimal.Round(moneyFlowVolume, 2)
```

So, `StockInfoModel.AccDist` is another example of "derived" property. It's easy to explain now what `Expr.ExpandLetBindings` does - it recursively expands all local name bindings/aliases like `moneyFlowVolume`. Because the same dependency can be used several times in computation (`this.DaysHigh` for example) there is `Seq.distinct` filter inside `(|DependentProperty|_|)` implementation. If any of the properties are updated through the user interface, `AccDist` get immediately recalculated (to make it really work `Binding.UpdateSourceOnChange` is used). 

There are minimal changes in `StockPricesChartView`:

```ocaml
...
type StockPricesChartView(control)...
    ...
    override this.SetBindings model = 
        ... 
        Binding.FromExpression 
            <@ 
                this.Control.CompanyName.Text <- model.SelectedStock.CompanyName 
                this.Control.AccDist.Text <- model.SelectedStock.AccDist 
            @> 
        
        Binding.UpdateSourceOnChange 
            <@ 
                this.Control.LastPrice.Text <- string model.SelectedStock.LastPrice 
                this.Control.DaysHigh.Text <- string model.SelectedStock.DaysHigh 
                this.Control.DaysLow.Text <- string model.SelectedStock.DaysLow 
                this.Control.Volume.Text <- string model.SelectedStock.Volume 
            @> 
```