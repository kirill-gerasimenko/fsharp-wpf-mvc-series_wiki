GUI programming is inherently hard but modern languages and new paradigms make these experiences more palatable. 

There are many ways to [ organize GUI architecture](http://martinfowler.com/eaaDev/OrganizingPresentations.html). We're looking for one that provides the least coupling to the underlying GUI framework infrastructure. A loosely-coupled architecture that follows good, old, but still important [Single Responsibility Principle](http://www.objectmentor.com/resources/articles/srp.pdf) should lead to simple, readable and testable code.

Event processing is at the heart of any GUI framework. F# embraced [first class events](http://blogs.msdn.com/b/dsyme/archive/2006/03/24/fsharpcompositionalevents.aspx) long time ago, followed by [.NET Reactive Extensions (Rx)](http://channel9.msdn.com/Blogs/Charles/Erik-Meijer-Rx-in-15-Minutes). Both offer unprecedented composability, which enables scenarios that were either impossible or very hard to implement in the past.

Design of the framework will be mostly based on Martin Fowler's [Supervising Controller pattern](http://martinfowler.com/eaaDev/SupervisingPresenter.html) with a twist of functional programming techniques.

Here is the diagram of main players:

[[/Images/Mvc1.png]]

Let's have a closer look at the individual components:

Component | Dependencies | Responsibilities | Notes 
----|------|----|---------
Model | - | Holds state | To be more precise, Model carries the state.<br>It can be thought of as in-memory snapshot of the View.<br>Data binding is a key component responsible for synchronizing Model with View 
View | Model | Event source<br>Data binding target | Never mind that View also draws actual interface - this function is well encapsulated inside implementation.<br>Dependency on Model is run-time instance-level only because the binding is usually implemented in terms of the object type.||
Controller | View, Model | Event processor | Controller performs the processing of events and delivers results by updating Model.
Ideally a functional type definition of Controller may look like one below:
```ocaml
type Controller<'Event, 'Model> = 'Event -> 'Model -> 'Model
```
Controller takes an event, executes event-specific logic while reading data from the Model and returns a new version of the Model - not the mutated original one. However, that would only be possible in an ideal pure functional world. In reality we heavily rely on data binding and returning new Model means re-binding it every time, which can be slow and thus impractical. Let's change the definition to a more practical one:
```ocaml
type Controller<'Event,'Model> = 'Event -> 'Model -> unit
```
The type signature indicates that it relies on mutation because its return type is unit. This version might be even better than the initial one because after all any GUI framework is about screen manipulation being side-effectful by its nature. But we still can benefit a lot from the functional way of thinking. Let's re-write Controller type definition slightly differently:
```ocaml
type Controller<'Event,'Model> = 'Event -> ('Model -> unit)
```
Do you see the difference? No? Then how about this?
```ocaml
type EventHandler<'Model> = 'Model -> unit
type Controller<'Event,'Model> = 'Event -> EventHandler<'Model>
```

Obviously, Controller is an event handler factory. Once it's partially applied to the first argument (i.e. an event) it returns the handler that contains this particular event-specific logic. We'll see how this definition of controller plays out very well later in the series.

A fair question would be why not to use ever so popular [MVVM ] (http://msdn.microsoft.com/en-us/magazine/dd419663.aspx) pattern? The reason is that MVVM, which is a variation of Martin Fowler's [PresentationModel] (http://martinfowler.com/eaaDev/PresentationModel.html) pattern, follows the classic OOP approach where state and behavior are smashed together in one monolithic object. Once `ViewModel` grows big, which is inevitable for real-world GUI applications, it's getting hard to distinguish between methods and internal state (like DB connection) on one side and visual state (Model) manipulated by Controller on the other. Functional approach where [declarative by nature](http://channel9.msdn.com/Blogs/Charles/JAOO-2007-Joe-Armstrong-On-Erlang-OO-Concurrency-Shared-State-and-the-Future-Part-2#time=22m06s) data structures and functions/methods are separated is more favorable. Also, having Model type defined separately gives us a way to automate the implementation of tedious [INotifyPropertyChanged](http://msdn.microsoft.com/en-us/library/ms229614.aspx) interface.

The goals of these series are:
 * Explain the architecture of GUI framework. 
 * Make the series a sort of language tutorial by demonstrating some advanced code techniques available in F#.
 * [Don Syme](http://blogs.msdn.com/b/dsyme/), designer and architect of the F# programming language, often positions F# as a language mostly suitable for [processing, analysis, algorithms etc.](http://skillsmatter.com/podcast/scala/practical-functional-first-programming-with-f data). I would like to show that F# can be superior to C# even when it comes to more common tasks like GUI development. 