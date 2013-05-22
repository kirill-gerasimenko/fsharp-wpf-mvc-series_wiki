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

Being novice in type provider development, I decided to start from pilot version. There are two kinds of F# type providers: erased types and generated types. First is ubiquitously popular and has better development support. As matter fact, I don't know any serious "generated types" based implementation to this point.