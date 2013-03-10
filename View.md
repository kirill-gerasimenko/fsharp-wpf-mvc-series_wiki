The second constituent of MVC - View plays two roles: 
  * the role of **Event source** – the key abstraction for this architecture 
  * the role of **Data binding target** – the design constraint (reminder: without data binding our architecture doesn't make much sense) 

What's the best abstraction nowadays to represent an event source on .NET platform? Right, [IObservable<'T>](http://msdn.microsoft.com/en-us/library/dd990377.aspx). 
  F# users may choose [IEvent<'T>](http://msdn.microsoft.com/en-us/library/ee340349.aspx) as event source abstraction, but it's not the best choice (see an [excellent explanation](http://msdn.microsoft.com/en-us/library/hh273073.aspx#alert_note) by Tomas Petricek). Also, by picking `IObservable` we get at our disposal the full combinatorial power of Rx library, which becomes indispensable once you need to solve some concurrency problems or even generate event sequences for unit testing purposes. There is no need for frequently used in C# [Observable.FromEventPattern](http://msdn.microsoft.com/en-us/library/hh242978.aspx) factory because by F# compiler magic all .NET events are represented as `IEvent<'T >`, which is inherited from `IObservable<'T>`. Going forward Rx will get more support resources and hopefully one day will make it to BCL. Still [kudos to Don and F# team](http://channel9.msdn.com/Shows/Going+Deep/E2E-Erik-Meijer-and-Wes-Dyer-Reactive-Framework-Rx-Under-the-Hood-1-of-2#time=30m05s) for initial inspiration and being one of the first [to adopt](http://msdn.microsoft.com/en-us/library/ee370313.aspx) `IObservable` and of course to [Erik Meijer](http://twitter.com/headinthebox/) and [Rx team](http://blogs.msdn.com/b/rxteam/) for bringing us such a wonderful library.
Here is the interface definition:
```ocaml
type IView<'Events> =
    inherit IObservable<'Events>
    
    abstract SetBindings : obj -> unit
```
The interface is very generic - none of the underlying platform details is bleeding through. In that sense `View` can be thought of as an isolation layer. `View` is an example of both  _[Parametric](http://en.wikipedia.org/wiki/Parametric_polymorphism)_ polymorphism (by `'Events` type, which will allow to write a generic controller) and  _[Subtype](http://en.wikipedia.org/wiki/Subtype_polymorphism)_ polymorphism (a way to create `View` implementations). 

### Basic View implementation

Base class for `IView` implementation may look like the one below: 
```ocaml
[<AbstractClass>]
type View<'Events, 'Window when 'Window :> Window and 'Window : (new : unit -> 'Window)>() = 

    let window = new 'Window()
    member this.Window = window
    
    interface IView<'Events> with
        member this.Subscribe observer = 
            let xs = this.EventStreams |> List.reduce Observable.merge 
            xs.Subscribe observer
        member this.SetBindings model = 
            window.DataContext <- model
            this.SetBindings model

    abstract EventStreams : IObservable<'Events> list
    abstract SetBindings : obj -> unit
```
Let's create a sample view:

[[Images/SampleView.png]]

It looks quite ugly but serves the purpose. So, we have two text boxes X and Y to input operands, `TextBlock` to show the result and two buttons: Add for performing addition, and Clear for zeroing inputs.

First, we need WPF `Window` class. We will be building it manually for now:
```ocaml
type SampleWindow() as this = 
    inherit Window(Title = "Simple view", Width = 200., Height = 200.)
    
    let x = TextBox()
    let y = TextBox()
    let result = TextBlock()
    let add = Button(Content = "Add")
    let clear = Button(Content = "Clear")
    //code to add controls to Window
    member this.X = x
    member this.Y = y
    member this.Result = result
    member this.Add = add
    member this.Clear = clear
```
Secondly, we need to implement `SetBindings` (we use `SampleModel` from Model chapter) and `EventStreams` property. `SetBindings` is easy but `EventStreams` is challenging. Typical form in GUI has many potential event sources: button clicks, text boxes changed, grids filled, etc. How can we combine all of the above into a single stream of one type `'Events`? Moreover, some event sources like Add button click and Clear button click have same types but certainly should be distinguished. The answer is F# [Discriminated Unions](http://msdn.microsoft.com/en-us/library/dd233226.aspx). Each event source should be mapped to a single case of DU. 
```ocaml
type SampleEvents = 
    | Add
    | Clear
...
type SampleView() =
    inherit View<SampleEvents, SampleWindow>()
    
    override this.EventStreams = 
        [
            this.Window.Add.Click |> Observable.map(fun _ -> Add)
            this.Window.Clear.Click |> Observable.map(fun _ -> Clear)
        ]
    
    override this.SetBindings model = 
        this.Window.X.SetBinding(TextBox.TextProperty, "X") |> ignore
        this.Window.Y.SetBinding(TextBox.TextProperty, "Y") |> ignore
        this.Window.Result.SetBinding(TextBlock.TextProperty, "Result") |> ignore
```
Here is the main function of the sample application : 
```ocaml
[<STAThread>] 
do 
    let model : SampleModel = Model.Create()
    let view = SampleView()
    let iview = view :> IView<_> 
    
    iview.SetBindings model
    iview.Add(callback = function 
        | Add -> model.Result <- model.X + model.Y
        | Clear -> 
            model.X <- 0
            model.Y <- 0
            model.Result <- 0
    )
        
    Application().Run view.Window |> ignore
```
Surrounded by empty lines is a mini-prototype of future Controller. Special attention should be paid to the event callback that is implemented as [Pattern matching function](http://msdn.microsoft.com/en-us/library/dd233242.aspx). Discriminated unions and pattern matching always go hand in hand. If a DU has been defined this means that somewhere in the code there is a pattern matching over a value of this DU type. Moreover, if you try to comment out one of the case handlers, like Add in callback, you will be immediately warned by compiler "Incomplete pattern matches ...". F# compiler ensures that all events are handled! At first it may seem that Discriminated unions can be replaced by Enums. This is not true. DU has an ability to carry state which is important for UI development. This will be demonstrated by examples later in the series. 

### XAML View 

Of course, no one in sound mind will build WPF window manually for production code. That's what WPF Designer/Blend is for. Let's add Window.xaml file to F# project mimicking the same window. 
  Often XAML designer has to be reloaded to show window properly in design mode because XAML files are not officially supported for F# projects in VS 2010. This issue has been resolved in VS 2012. 

Access to individually named (using "Name" attribute) controls is provided by F# dynamic operator (?) – the [ idea](http://msdn.microsoft.com/en-us/library/hh273074.aspx) first presented by Tomas Petricek. 
```ocaml
[<AbstractClass>]
type View<'Events, 'Window when 'Window :> Window and 'Window : (new : unit -> 'Window)>(?window) = 
    
    let window = defaultArg window (new 'Window())
    member this.Window = window
    ...
        
[<AbstractClass>]
type XamlView<'Events>(resourceLocator) = 
    inherit View<'Events, Window>(resourceLocator |> Application.LoadComponent |> unbox)
    
    static member (?) (view : View<_, _>, name) = 
        match view.Window.FindName name with
        | null -> invalidArg "Name" ("Cannot find control with name: " + name)
        | control -> unbox control 
```

There are only small changes to sample view. Instead of explicitly creating child controls and adding them manually to the window we pull them off the loaded XAML `Window`. 
```ocaml
type SampleView() as this =
    inherit XamlView<SampleEvents>(resourceLocator = Uri("/Window.xaml", UriKind.Relative))
    
    let add : Button = this ? Add
    let clear : Button = this ? Clear
    let x : TextBox = this ? X
    let y : TextBox = this ? Y
    let result : TextBlock = this ? Result
    ...
```

### Statically typed View

XAML View is not bad at all but we can do better by further leveraging the static type system, which is especially beneficial in F#. We 'll explore a hybrid approach where XAML definition along with the generated `Window` class resides in C# project and is referenced by F#. To make this design most effective additional constraints are placed on controls that need to be accessed. Aside from having `Name` attribute in XAML it should have `x:FieldModifier="public"` too. It instructs code generator to place public modifier on control field. This is not a strict requirement. These approaches - static and dynamic - can be mixed and matched. To support it, let's move up dynamic lookup operator to `View` base class and extend it to access resources as well (for example, it can be helpful for implementing [DataTemplateSelector](http://msdn.microsoft.com/en-us/library/system.windows.controls.datatemplateselector.aspx)) 
```ocaml
[<AbstractClass>]
type View<'Events, 'Window when 'Window :> Window and 'Window : (new : unit -> 'Window)>(?window) = 
    
    let window = defaultArg window (new 'Window())
    
    member this.Window = window
    static member (?) (view : View<_, _>, name) = 
        match view.Window.FindName name with
        | null -> 
            match view.Window.TryFindResource name with
            | null -> invalidArg "Name" ("Cannot find child control or resource named: " + name)
            | resource -> unbox resource 
        | control -> unbox control
    
    interface IView<'Events> with
        member this.Subscribe observer = 
            let xs = this.EventStreams |> List.reduce Observable.merge 
            xs.Subscribe observer
        member this.SetBindings model = 
            window.DataContext <- model; 
            this.SetBindings model
    
    abstract EventStreams : IObservable<'Events> list
    abstract SetBindings : obj -> unit
    
[<AbstractClass>]
type XamlView<'Events>(resourceLocator) = 
    inherit View<'Events, Window>(resourceLocator |> Application.LoadComponent |> unbox)
    ...
type SampleView() as this =
    inherit View<SampleEvents, SampleWindow>()
    ...    
    override this.EventStreams = 
        [
            this.Window.Add.Click |> Observable.map(fun _ -> Add)
            this.Window.Clear.Click |> Observable.map(fun _ -> Clear)
        ]

    override this.SetBindings model = 
        ...
        this.Window.Result.SetBinding(TextBlock.TextProperty, "Result") |> ignore
```
Buttons "Add", "Clear", and `TextBlock` "Result" are statically typed. Now any renamed or deleted control will cause compilation error unless view's code fixed too. Also, control types are derived from the generated code. Both "pure F#" and hybrid solutions have own pros and cons. In VS 2010 setting, I  prefer the hybrid one, and that's the approach we'll take going forward in the series. But I will certainly give "pure F#" solution another chance in the special chapter dedicated to VS 2012 improvements, when we'll get a chance to try [XAML type provider](http://www.navision-blog.de/2012/03/22/wpf-designer-for-f/). 
  Sometimes application start-up code needs a part of C# project too. For example, if you want to have [ClickOnce](http://msdn.microsoft.com/en-us/library/t71a733d) deployment option, which is not supported by F# projects out of the box.