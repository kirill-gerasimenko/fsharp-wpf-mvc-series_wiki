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

In order for this to work [MultiBinding](http://blogs.mscommunity.net/blogs/borissevo/archive/2009/01/20/wpf-trick-3-multibinding-and-stringformat.aspx) support for  has to be introduced and this is not simple at all. We expressed everything in terms of [Binding](http://msdn.microsoft.com/en-us/library/system.windows.data.binding.aspx) class. This class along with [MultiBinding](http://msdn.microsoft.com/en-us/library/ms613634.aspx) are siblings sharing the common parent [BindingBase](http://msdn.microsoft.com/en-us/library/system.windows.data.bindingbase.aspx). Realistically, to make this work we should reformulate everything in terms of `BindingBase` and then use downcast extensively. This is going to be ugly. To a certain extent it's a case of [Expression problem](http://en.wikipedia.org/wiki/Expression_problem) for functional languages - it's easy to add new operations, but really hard to add new types. Fortunately, there is a much better solution. 

Let's write down the ideal final solution we're looking for: 
