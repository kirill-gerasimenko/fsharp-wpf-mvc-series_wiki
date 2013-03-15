Creating data bindings out of F# assignment statement quotation as introduced in [Binding] chapter still stands. It just needs to be extended to handle various real-world scenarios. 

  `Binding.FromExpression <@ ... @>` is a nice facade method to our data binding micro DSL. It has support for batch statements. Mapping from single assignment  statement to a single binding happens in `Expr.ToBindingExpr` extension method. In this post we'll mostly focus on internals of this method. 

### Binding Target

Let's have a look at `MainView` from previous [External Event Sources] chapter.
```ocaml
type MainView() as this = 
    inherit View<MainEvents, MainModel, MainWindow>()
    
    let pause = this.Control.PauseWatch 
    let fail = this.Control.Fail 
    
    override this.EventStreams = 
        [   
            ... 
            yield pause.Checked |> Observable.mapTo StopWatch 
            yield pause.Unchecked |> Observable.mapTo StartWatch 
            yield fail.Checked |> Observable.mapTo StartFailingWatch 
            yield fail.Unchecked |> Observable.mapTo StopFailingWatch 
        ] 
    
    override this.SetBindings model = 
        let titleBinding = MultiBinding(StringFormat = "{0} - {1}") 
        titleBinding.Bindings.Add <| Binding("ProcessName") 
        titleBinding.Bindings.Add <| Binding("ActiveTab") 
        this.Control.SetBinding(Window.TitleProperty, titleBinding) |> ignore 
        
        Binding.FromExpression 
            <@ 
                this.Control.PauseWatch.IsChecked <- model.Paused 
                this.Control.Fail.IsChecked <- model.Fail 
            @> 
        this.Control.RunningTime.SetBinding(TextBlock.TextProperty, Binding(path = "RunningTime", StringFormat = "Running time: {0:hh\:mm\:ss}")) |> ignore 
```

It has places where we could not use our Binding DSL:
  # Window Title property binding requires [MultiBinding](http://msdn.microsoft.com/en-us/library/ms613634.aspx) and [StringFormat](http://msdn.microsoft.com/en-us/library/system.windows.data.bindingbase.stringformat.aspx) support.
  # When some control is referred many times inside a view (like `this.Control.PauseWatch` in our example), it is useful to define a shortcut like `let pause = this.Control.PauseWatch`. Unfortunately, if we try to use it in `Binding.FromExpression` we’ll get an exception. 
  # `TextBlock` `RunTime.TextProperty` binding requires `StringFormat` support. 

To find the right approach for solving the issues above let's look at correspondence between key components of binding, parts of single assignment statement, and items available after decomposition using [PropertySet](http://msdn.microsoft.com/en-us/library/ee353701.aspx) active pattern:

[[Images/BindingTargetDiagram.png]]

We'll skip problem #1 for now. To solve #2 we clearly need to handle more scenarios for parsing target. `(|Target|_|)` active pattern would do just that: 
```ocaml
...
module BindingPatterns = 
    
    let (|Target|_|) expr = 
        let rec loop = function 
            | Some( Value(obj, viewType) ) -> obj 
            | Some( FieldGet(tail, field) ) ->  field.GetValue(loop tail) 
            | Some( PropertyGet(tail, prop, []) ) -> prop.GetValue(loop tail, [||]) 
            | _ -> null 
        match loop expr with 
        | :? FrameworkElement as target -> Some target 
        | _ -> None 
...
```
Type signature:

[[Images/BindingDSL.TargetActivePattern.TypeSignature.png]]

As a result `ToBindingExpr` is much simpler and more readable: 
```ocaml
type Expr with 
    member this.ToBindingExpr() = 
        match this with 
        | PropertySet(Target target, targetProperty, [], PropertyPath path) -> 
            let binding = Binding(path, ValidatesOnDataErrors = true, ValidatesOnExceptions = true) 
            target.SetBinding(targetProperty.DependencyProperty, binding) 
        | _ -> invalidArg "expr" (string this) 
```

Moving on to problem #3. Contrary to the binding target object, the source object is never set explicitly by Binding DSL. It was a conscious architectural choice - binding property path is always relative to root (Window) data context which in our case is a model instance. Hence, right-hand side id mostly mapped to [Path property](http://msdn.microsoft.com/en-us/library/system.windows.data.binding.path.aspx) of `Binding` object. There are different language contracts that can be a part of the assignment statement and at the same time can be nicely mapped to Binding class properties. Let's look at the variations that make sense to support: 

 # | Original assignment <br/>statement quotation | Resulting Binding | Notes 
---|---|----|---------
 1| `<@ textBox.Text <- `<br/>`model.StringProperty @>` | `Binding(path = "StringProperty")` | straightforward mapping 
 2| `<@ textBox.Text <- `<br/>`string model.NonStringProperty @>` | `Binding(path = "NonStringProperty")` | straightforward mapping, except that string "shim" happily discards type system. See chapter [[Data Binding]] for details.
 3 | `<@ checkBox.IsChecked <- `<br/>`Nullable model.BoolProperty @>` | `Binding(path = "BoolProperty")` | same as #2 but for Nullable.
 4 | `<@ comboBox.SelectedItem <- `<br/>`model.StringProperty @>` | `Binding(path = "StringProperty")` | straightforward mapping, except that the compiler <br> generated coercion is discarded. See chapter [[Data Binding]] for details.
 5 | `<@ textBlock.Text <- `<br/>`String.Format(format, model.Property) @>` | `Binding(path = "Property", `<br>`StringFormat = format)` | #1 + `StringFormat`
 6 | `<@ control.Property <- `<br/>`func model.Property @>` | `Binding(path = "Property",`<br>`Converter = IValueConverter.OneWay func,`<br>`Mode = BindingMode.OneWay)` | #1 + Converter 
 6a | `<@ button.IsEnable <- `<br/>`not model.BoolProperty @>` | same as #6 with func = not | | 
 7 | `<@ this.Control.AddToChart.Visibility <- `<br/>`converter.Apply model.AddToChartEnabled @>` | `Binding(path = "Property",`<br/>`Converter = converter)` | Using `IValueConverter` implemenations
 8 | [Selector](http://msdn.microsoft.com/en-us/library/ms595227.aspx)-based controls binding |  |
 9 | [DataGrid](http://msdn.microsoft.com/en-us/library/system.windows.controls.datagrid.aspx) binding| | 
 10 | `<@ control.Property <- `<br/>`model.Collection.CurrentItem @>` | `Binding(path = "Collection/")` | See [MSDN](http://msdn.microsoft.com/en-us/library/ms742451.aspx#sourcetraversal) for details.
 11 | `<@ textBox.Text <- `<br/>`model.Property1.StringProperty2 @>` | `Binding(path = "Property1.StringProperty2")` | See [MSDN](http://msdn.microsoft.com/en-us/library/ms742451.aspx#multipleindirect)
 11a | `<@ control.Property <- `<br/>`model.Collection.CurrentItem.Property1 @>` | `Binding(path = "Collection/Property1")` | Same as 11, just includes collection in path 

To handle all combinations we factor-out parsing right-hand side of assignment statement into a separate `BindingExpression` active pattern: 
 
```ocaml
...
    member this.ToBindingExpr() = 
        match this with 
        | PropertySet(Target target, targetProperty, [], BindingExpression binding) -> 
            ... 
            target.SetBinding(targetProperty.DependencyProperty, binding) 
        ...
```

We've seen *#1*, *#2* and *#4* in chapter [[Data Binding]] and through the series. 

*#3*. In [previous chapter](External Event Sources) it was a bit inconvenient to work with `Paused` property of type `Nullable<bool>`, because it had to match the type of `ToggleButton.IsChecked`. Data binding magic knows how to map `Nullable<'T>` to `'T` on model. To take advantage of #3 we change type of `Paused` to bool and insert call to `Nullable` constructor in binding expression to keep compiler happy. Similar to string "shim" and coercion to obj it's discarded during binding setup. 

```ocaml
module BindingPatterns = 
    ...
    let (|Nullable|_|) = function 
        | NewObject( ctorInfo, [ propertyPath ] ) when ctorInfo.DeclaringType.GetGenericTypeDefinition() = typedefof<Nullable<_>> -> 
            Some propertyPath 
        | _ -> None    
    ...   
    let rec (|BindingExpression|) = function 
        | PropertyPath path -> 
            Binding path 
        | Coerce( BindingExpression binding, _) 
        | SpecificCall <@ string @> (None, _, [ BindingExpression binding ]) 
        | Nullable( BindingExpression binding) -> 
            binding 
        ... 
        | expr -> invalidArg "binding property path quotation" (string expr)
...
type MainModel() = 
    inherit Model()
    ...
    abstract Paused : bool with get, set 
    abstract Fail : bool with get, set 
    
type MainView() as this = 
    ...
    override this.SetBindings model = 
        Binding.FromExpression 
            <@ 
                pause.IsChecked <- Nullable model.Paused 
                fail.IsChecked <- Nullable model.Fail 
                ... 
            @> 
```

*#5*. All we need is another active pattern and to handle the case: 
```ocaml
module BindingPatterns = 
    ...
    let (|StringFormat|_|) = function 
        | SpecificCall <@ String.Format : string * obj -> string @> (None, [], [ Value(:? string as format, _); Coerce( propertyPath, _) ]) -> 
            Some(format, propertyPath) 
        | _ -> None    
    ...   
    let rec (|BindingExpression|) = function 
        ... 
        | StringFormat(format, BindingExpression binding) -> 
            binding.StringFormat <- format 
            binding 
        ... 
type MainView() as this = 
    ...
    override this.SetBindings model = 
        Binding.FromExpression 
            <@ 
                ... 
                this.Control.RunningTime.Text <- String.Format("Running time: {0:hh\:mm\:ss}", model.RunningTime)
                ...
            @> 
```

To demostrate *#6* let's look at our `StockPicker` MVC-triple. If user clicks "Retrieve" button before entering anything into "Symbol" `TextBox`, the error pops up. 

[[Images/BindingDSL.BeforeIsNotNullConverter.png]]

It is handled by validation logic in the controller: 

```ocaml
...
model |> Validation.textRequired <@ fun m -> m.Symbol @>
...
```

Let's make user interface more strict by keeping the "Retrieve" button disabled unless a text is entered into the "Symbol" field. This can be done in declarative fashion through the data binding: 

```ocaml
[<AutoOpen>]
module Mvc.Wpf.Sample.Extensions 
    
let isNotNull x = x <> null 
...
type StockPickerView() as this = 
    ...
    override this.SetBindings model = 
        Binding.FromExpression 
            <@ 
                ...
                this.Control.Retrieve.IsEnabled <- isNotNull model.Symbol 
            @>
    ...
```

The corresponding pattern matching case and helper methods on `IValueConverter` are: 

```ocaml
...
type IValueConverter with 
    static member Create(convert : 'a -> 'b, convertBack : 'b -> 'a) =  
        { 
            new IValueConverter with 
                member this.Convert(value, _, _, _) = try value |> unbox |> convert |> box with _ -> DependencyProperty.UnsetValue 
                member this.ConvertBack(value, _, _, _) = try value |> unbox |> convertBack |> box with _ -> DependencyProperty.UnsetValue 
        } 
    static member OneWay convert = IValueConverter.Create(convert, fun _ -> DependencyProperty.UnsetValue) 
...
module BindingPatterns = 
    ...
    let (|Converter|_|) = function 
        | Call(instance, method', [ propertyPath ]) -> 
            let instance = match instance with | Some( Value( value, _)) -> value | _ -> null 
            Some((fun(value : obj) -> method'.Invoke(instance, [| value |])), propertyPath ) 
        | _ -> None    
    ...
    let rec (|BindingExpression|) = function 
        ...
        | Converter(convert, BindingExpression binding) -> 
            binding.Mode <- BindingMode.OneWay 
            binding.Converter <- IValueConverter.OneWay convert 
            binding 
        ...
```

It gets mapped to one-way binding/converter, which is reasonable considering the semantics. To have it two-way, two functions are required: convert and convert back, and there is no way the second can be inferred from the first. 

Unfortunately, it won't exactly work because [TextBox.Text](http://msdn.microsoft.com/en-us/library/system.windows.controls.textbox.text.aspx) has a default `UpdateSourceTrigger` value of `LostFocus`. It means that if an application has a `TextBox` with a data-bound `TextBox.Text` property, the text you type into the `TextBox` does not update the source until the `TextBox` loses focus (for instance, when you click away from the `TextBox`). If you want the source to get updated as you are typing, set the `UpdateSourceTrigger` of the binding to `PropertyChanged`. For our Binding micro DSL it means accepting override as a parameter to `Binding.FromExpression` methods and piping it down to `Expr.ToBindingExpr`. We'll make it optional for convenience: 

```ocaml
type Binding with 
    static member FromExpression(expr, ?updateSourceTrigger) = 
    ...
type Expr with
    member this.ToBindingExpr(?updateSourceTrigger) = 
        match this with
        | PropertySet(Target target, targetProperty, [], BindingExpression binding) ->
            ...
            if updateSourceTrigger.IsSome then binding.UpdateSourceTrigger <- updateSourceTrigger.Value
            BindingOperations.SetBinding(target, targetProperty.DependencyProperty, binding)
```

Notice, that the call with `updateSourceTrigger` parameter set to `UpdateSourceTrigger.PropertyChanged` is used often enough to make a shortcut: 
```ocaml
type Binding with 
    ...
    static member UpdateSourceOnChange expr = Binding.FromExpression(expr, updateSourceTrigger = UpdateSourceTrigger.PropertyChanged) 
```

Going back to `StockPickerView`:

```ocaml
type StockPickerView() ...
    override this.SetBindings model =  
        Binding.FromExpression 
            <@ 
                ...
                this.Control.Retrieve.IsEnabled <- isNotNull model.Symbol
            @>
        
        Binding.UpdateSourceOnChange <@ this.Control.Symbol.Text <- model.Symbol @>
```

In practice, allowing to fine tune any binding property, not only `updateSourceTrigger`, would be nice. Following is the final version of API: 
```ocaml
type Binding with 
    static member FromExpression(expr, ?mode, ?updateSourceTrigger, ?fallbackValue, ?targetNullValue, ?validatesOnDataErrors, ?validatesOnExceptions) = 
        ...
        for e in split expr do 
            let be = e.ToBindingExpr(?mode = mode, ?updateSourceTrigger = updateSourceTrigger, ?fallbackValue = fallbackValue, 
                                     ?targetNullValue = targetNullValue, ?validatesOnDataErrors = validatesOnDataErrors, ?validatesOnExceptions = validatesOnExceptions) 
            ...
    
    static member UpdateSourceOnChange expr = Binding.FromExpression(expr, updateSourceTrigger = UpdateSourceTrigger.PropertyChanged) 
    static member TwoWay expr = Binding.FromExpression(expr, BindingMode.TwoWay) 
    static member OneWay expr = Binding.FromExpression(expr, BindingMode.OneWay) 
...
type Expr with 
    member this.ToBindingExpr(?mode, ?updateSourceTrigger, ?fallbackValue, ?targetNullValue, ?validatesOnDataErrors, ?validatesOnExceptions) = 
        match this with 
        | PropertySet(Target target, targetProperty, [], BindingExpression binding) -> 
            
            if mode.IsSome then binding.Mode <- mode.Value 
            if updateSourceTrigger.IsSome then binding.UpdateSourceTrigger <- updateSourceTrigger.Value 
            if fallbackValue.IsSome then binding.FallbackValue <- fallbackValue.Value 
            if targetNullValue.IsSome then binding.TargetNullValue <- targetNullValue.Value 
            binding.ValidatesOnDataErrors <- defaultArg validatesOnDataErrors true 
            binding.ValidatesOnExceptions <- defaultArg validatesOnExceptions true 
            
            target.SetBinding(targetProperty.DependencyProperty, binding) 
            
        | _ -> invalidArg "expr" (string this) 
```

It's worth noting that contrary to WPF `ValidatesOnDataErrors` and `ValidatesOnExceptions` are set on by default, because the framework relies on validation. 

To demonstrate application of #6a let's make Restart button enabled only when "Fail" `CheckBox` is unchecked: 

[[Images/BindingDSL.RestartDisableWhenFailOn.png]]

Implementation in `MainView`:
```ocaml
type MainView() as this = 
   ...
    override this.SetBindings model = 
        ...
        Binding.FromExpression 
            <@ 
                ...
                this.Control.RestartWatch.IsEnabled <- not model.Fail
            @>
```

This declarative and simple approach saves a lot of manual coding. A reasonable question to be asked is why not to make it two-way data binding? After all, not is a function easy to reverse. Answer to this simple question reveals key design decisions. Remember, the model is an in-memory image of the view and has to be its close reflection. All complex and especially imperative logic goes into the controller. Therefore, converters are less important than in [other](http://martinfowler.com/eaaDev/AutonomousView.html) architectures. In the snippet above we see that `fail.IsChecked` binds directly to the model.Fail. Whatever required conversion logic (including negation) can be applied inside event handler, i.e if not model.Fail then .... These kind of solution is easy to understand and test. On the other hand `Binding.FromExpression <@ this.Control.RestartWatch.IsEnabled <- not model.Fail @>` lets button enable state to be calculated off the `model.Fail` property (which should participate in `INotifyPropertyChanged` for this to work) and two-way binding doesn't make sense either. To emphasize the point let's look at #7. 

`StockPickerController` controller sets `model.AddToChartEnabled` to true once data for requested symbol are retrieved successfully, which in turn enables "Add To Chart" button. Imagine that we want to take a more radical approach and have "Add To Chart" button being hidden unless `model.AddToChartEnabled` is true. 

[[Images/BindingDSL.StockPicker.AddToChartVisibility.png]]

The sensible approach would be to bind `Button.Visibility` of type [Visibility] (http://msdn.microsoft.com/en-us/library/system.windows.visibility.aspx) property to `model.AddToChartEnabled`. Unfortunately, it cannot be done directly because Visibility type is 3-value enum, thus it cannot be mapped to a bool value. WPF has built-in [BooleanToVisibilityConverter](http://msdn.microsoft.com/en-us/library/system.windows.controls.booleantovisibilityconverter.aspx) type to mitigate the issue. An assignment statement, the cornerstone of our binding DSL, gives a natural one-way mapping (source -> target), but not the other way around. To have it two-way and keep it sound we need some creative idea. We'll use "pseudo" methods. Intriguing, right? Let's add the following extensions method: 

```ocaml
type IValueConverter with 
    ...
    member this.Apply _ = undefined
</code>  

of type signature: 

[[Images/BindingDSL.IValueConverter.Apply.TypeSignature.png]]

Usage:
```ocaml
type StockPickerView() as this = 
    ...
    override this.SetBindings model = 
        ...
        let converter = BooleanToVisibilityConverter()
        Binding.FromExpression 
            <@ 
                this.Control.AddToChart.Visibility <- converter.Apply model.AddToChartEnabled 
            @> 
```

"Pseudo" methods are never meant to be called (undefined ensures that it will throw an exception if an attempt is made), but serve as hints to Binding DSL interpreter. The funny thing is that this actually works. But how? Just another pattern matching case: 
```ocaml
    let rec (|BindingExpression|) = function 
        ...
        | Call(None, method', [ Value(:? IValueConverter as converter, _); BindingExpression binding ] ) when method'.Name = "IValueConverter.Apply" -> 
            binding.Converter <- converter 
            binding 
        ...
```

To be accurate, this won't guarantee two-way binding/conversion - it goes with default. To override the default, provide parameters to `Binding.FromExpression` method. Make sure to use converter instance created outside of quotation to avoid evaluation inside active patterns. Equipped with `IValueConverter.Create`, one can easily create local instances out of local lambdas. 

Let's go back to the discussion of the role of converters in the framework architecture. It has all the drawbacks mentioned for 6a. Unless there is a handy built-in converter, this is not a good approach. Even for built-in ones, for example, there is no harm to declare property of type `Visibility` on the model and let the controller to work off it. Converters are less testable and reusable. Error handling gets tricky for two way converters too. To sum it up, use this feature judiciously. 

We'll look at 8-11a in one step, because this way it's easier to demonstrate the approach. In the sample application we'll change stock prices chart MVC-triple to display additional stock properties. 

[[Images/BindingDSL.ComboBox.DataGrid.png]]

The drop-down list just below the chart contains all stocks we've added so far. Detailed information grid on the right side shows properties for the selected stock. Note that this is a classic example of master-detail view. 

`StockInfoModel` returned by auxiliary `StockPickerController` controller now has detailed properties as a dictionary in addition to some major ones: 
```ocaml
...
type StockInfoModel() =  
    inherit Model() 
    ... 
    abstract Details : IDictionary<string, string> with get, set 
```

`StockPricesChartModel` has been extended to have full information for all stocks and separate property to denote selected item: 
```ocaml
type StockPricesChartModel() = 
    inherit Model() 
    
    abstract StocksInfo : ObservableCollection<StockInfoModel> with get, set 
    abstract SelectedStock : StockInfoModel with get, set 
```
 
Here is how we do `Symbol ComboBox` binding: 

```ocaml
type StockPricesChartView(control) as this =
    ... 
    override this.SetBindings model = 
        ... 
        this.Control.Symbol.SetBindings( 
            itemsSource = <@ model.StocksInfo @>, 
            selectedItem = <@ model.SelectedStock @>, 
            displayMember = <@ fun m -> m.Symbol @> 
        ) 
        ...
```
It is a nice alternative to less cohesive and more error-prone (especially for string-based `DisplayMemberPath` ) 
```ocaml
        ... 
        Binding.FromExpression 
            <@ 
                this.Control.Symbol.ItemsSource <- model.StocksInfo 
                this.Control.Symbol.SelectedItem <- model.SelectedStock 
            @> 
        this.Control.Symbol.DisplayMemberPath <- "Symbol" 
        ... 
```

Implementation and type signature are: 
```ocaml
type Selector with
    member this.SetBindings(itemsSource : Expr<#seq<'Item>>, ?selectedItem : Expr<'Item>, ?displayMember : PropertySelector<'Item, _>) = 
        
        let e = this.SetBinding(ItemsControl.ItemsSourceProperty, match itemsSource with BindingExpression binding -> binding) 
        assert not e.HasError 
        
        selectedItem |> Option.iter(fun(BindingExpression binding) -> 
            let e = this.SetBinding(DataGrid.SelectedItemProperty, binding) 
            assert not e.HasError 
            this.IsSynchronizedWithCurrentItem <- Nullable true 
        ) 
        
        displayMember |> Option.iter(fun(SingleStepPropertySelector(propertyName, _)) -> 
            this.DisplayMemberPath <- propertyName 
        ) 
```

[[Images/BindingDSL.ComboBox.SetBindings.png]]

Compared to independent bindings for `Selector`, this one forces right types. Type of `selectedItem` is the same as the element type of `itemsSource` collection and `displayMember` is picked off that type too. Note how implementation sets `IsSynchronizedWithCurrentItem` when `selectedItem` is provided. 

Shown below is a `DataGird` bind to stock details dictionary as well as implementation details (in additon to #9, `itemsSource` binding is an example of #11) 
```ocaml
type StockPricesChartView(control) as this =
    ...
    override this.SetBindings model = 
        ...
        this.Control.Details.SetBindings( 
            itemsSource = <@ model.SelectedStock.Details @>, 
            columnBindings = fun stockProperty -> 
                [ 
                    this.Control.DetailsName, <@@ stockProperty.Key @@> 
                    this.Control.DetailsValue, <@@ stockProperty.Value @@> 
                ] 
        ) 
...
type DataGrid with 
    member this.SetBindings(itemsSource : Expr<#seq<'Item>>, columnBindings : 'Item -> (#DataGridBoundColumn * Expr) list, ?selectedItem) = 
        
        this.SetBindings(itemsSource, ?selectedItem = selectedItem) 
        
        for column, BindingExpression binding in columnBindings Unchecked.defaultof<'Item> do 
            column.Binding <- binding 
```

[[Images/BindingDSL.DataGrid.SetBindings.png]]

Again, this produces better cohesion, than unrelated bindings. `SetBindings` for `DataGrid` reuses `Selector.SetBindings` to set `ItemsSource` and `SelectedItem` (`displayMember` doesn't make much sense for a grid). `columnBindings` function that returns list of pairs: column + expression that is translated to data binding. 

Speaking of master-detail, in our example, this is <br>`StockPricesChartModel.SelectedStock` -> `StockPricesChartModel.SelectedStock.Details`<br> relation. It was possible in part, because we explicitly mapped selected item to `SelectedStock`. It is often the case, because usually there is logic in controller based on the value of the currently selected item. But sometimes it's unnecessary; especially when details are read-only,  or controller has no dependency on it. To make declarative binding to details possible without `SelectedItem` we'll introduce yet another pseudo method: 
```ocaml
type IEnumerable<'T> with
    member this.CurrentItem : 'T = undefined
```

Semantics of this are the same as [`IEnumerator<T>.Current`](http://msdn.microsoft.com/en-us/library/58e146b7.aspx).

We can rewrite details grid binding like this: 

```ocaml
type StockPricesChartView(control) as this = 
    override this.SetBindings model = 
        this.Control.Details.SetBindings( 
            itemsSource = <@ model.StocksInfo.CurrentItem.Details @>, 
            ...
        ) 
```

This is a demo for #10 and #11a. In order for this to work, `PropertyPath` active pattern has support for this pseudo method. 
```ocaml
    let (|PropertyPath|_|) expr = 
        let rec loop e acc = 
            match e with 
            | PropertyGet( Some tail, property, []) -> 
                loop tail (property.Name :: acc) 
            | SpecificCall <@ Seq.empty.CurrentItem @> (None, _, [ tail ]) -> 
                loop tail ("/" :: acc) 
            | Value _ | Var _ -> acc 
            | _ -> [] 
        
        match loop expr [] with 
        | [] -> None 
        | x::_ as xs -> 
            xs 
            |> Seq.pairwise 
            |> Seq.map (function 
                | "/", x -> x 
                | _, "/" -> "/" 
                | x, y -> "." + y) 
            |> String.concat "" 
            |> ((+) x) 
            |> fun propetyPath -> Some propetyPath 
```

This `CurrentItem` functionality is nicely mapped to WPF [PropertyPath XAML Syntax](http://msdn.microsoft.com/en-us/library/ms742451.aspx). It's not critical to have, but the technique is nice. Other mappings, like numeric and symbolic indexers or even multi-indexers, can be done too, but are not particularly useful. 

Binding DSL tried to cover a lot, but not everything. The backdoor to use standard API calls like [FrameworkElement.SetBinding](http://msdn.microsoft.com/en-us/library/ms598270.aspx) or [BindingOperations.SetBinding](http://msdn.microsoft.com/en-us/library/system.windows.data.bindingoperations.setbinding.aspx) is always available. Some additional extensions are definitely possible. For example:
  * [DataGridComboBoxColumn](http://msdn.microsoft.com/en-us/library/system.windows.controls.datagridcomboboxcolumn.aspx) doesn't inherit from `DataGridBoundColumn` and therefore requires special support. Ironically, `DataGridComboBoxColumn` has the same binding API as `Selector/ComboBox`, but because of "brilliant" single inheritance idea (thanks again, Java), or lack of mixins they do not share same base class or interface. 
  * Quotation like <@ model.Property <- control.Property @> can be mapped to a binding with the mode set to [OneWayToSource](http://msdn.microsoft.com/en-us/library/system.windows.data.bindingmode.aspx).

Whatever is added has to make sense in current framework and not introduce needless complexity. 

Let's look again at most important active patterns of our Binding DSL: `(|Target|_|)` (at beginning of article), `(|PropertyPath|_|)` (full code just above) and here is full code for `(|BindingExpression|)`: 
```ocaml
    let rec (|BindingExpression|) = function 
        | PropertyPath path -> 
            Binding path 
        | Coerce( BindingExpression binding, _) 
        | SpecificCall <@ string @> (None, _, [ BindingExpression binding ]) 
        | Nullable( BindingExpression binding) -> 
            binding 
        | StringFormat(format, BindingExpression binding) -> 
            binding.StringFormat <- format 
            binding 
        | Converter(convert, BindingExpression binding) -> 
            binding.Mode <- BindingMode.OneWay 
            binding.Converter <- IValueConverter.OneWay convert 
            binding 
        | Call(None, method', [ Value(:? IValueConverter as converter, _); BindingExpression binding ] ) when method'.Name = "IValueConverter.Apply" -> 
            binding.Converter <- converter 
            binding 
        | expr -> invalidArg "binding property path quotation" (string expr) 
```

All of them use one very basic but key tool of functional programming. That’s right, recursion. Without it (in absence of fold for Quotations), it would be impossible to avoid code duplication. Recursion is praised in [classic](http://www.amazon.com/Structure-Interpretation-Computer-Programs-Second/dp/0070004846) books, but seems to be rarely used in imperative languages like C#. On the other hand, F# promotes quite different coding style - there are only 2 for loops in our whole codebase (the framework and the sample combined).