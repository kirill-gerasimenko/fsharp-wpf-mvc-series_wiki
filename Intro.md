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

```csharp
type Controller<'Event, 'Model> = 'Event -> 'Model -> 'Model
