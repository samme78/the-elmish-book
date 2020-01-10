# Splitting Simple Programs

In this section, we will take a detailed look into splitting an Elmish program into multiple simple programs. The key is that in this example, we are working with *simple* programs: without commands in the definition of `init` and `update`. Splitting Elmish programs with commands will be tackled in the next section.

### Counter and Text Input

Consider the following application

<div style="margin-top: 40px; margin-bottom:40px; width:100%">
  <div style="margin: 0 auto; width:100%;">
    <resolved-image source="/images/scaling/multiple-simple-programs.gif" />
  </div>
</div>

This is an application that switches between two views: one for a counter with increment and decrement buttons and another view that shows an input text box which reflects the text onto a header below it when you start typing in that text input. Now, before I show you anything from the code, pause for a second and try to figure out the pieces of information that this program has to keep track of and which events are triggered from the user interface.

For the first view showing the counter, the application has to keep track of the count and have the associated "increment" and "decrement" events to change the count. As for the other input text view, the application keeps track of the text from the input and reacts when that text changes. There is also a checkbox that toggles the reflected text into uppercase mode and back. Finally, the application keeps track of information that is *irrelevant* to both the counter and input text: which view is currently shown on screen with the button that toggles the view from the counter to the input text and back.

Putting this together, we get the following `State` type that includes all these pieces in one place. You can follow along this section while reading the initial source code of this program from the repository [Zaid-Ajaj/multiple-simple-programs](https://github.com/Zaid-Ajaj/multiple-simple-programs).

The definition of the `State` is pretty straightforward:
```fsharp
type Page =
  | Counter
  | TextInput

type State =
  { Count: int
    InputText: string
    IsUpperCase: bool
    CurrentPage : Page }
```
We have the `Count` field which is used in the counter view. As for `InputText` and `IsUpper`, they are used in the text input view. Finally we have the `CurrentPage` field which is of type `Page` defined earlier that keeps track of view is currently shown on screen.

The associated `Msg` type and its cases are also straightforward:
```fsharp
type Msg =
  | Increment
  | Decrement
  | InputTextChanged of string
  | UppercaseToggled of bool
  | SwitchPage of Page
```
We have `Increment` and `Decrement` for the counter, the `InputTextChanged` and `UppercaseToggled` for the input text and finally a `SwitchPage` case for, well, switching between the counter and input text. The `update` function is implemented as follows
```fsharp
let update (msg: Msg) (state: State): State =
  match msg with
  | Increment -> { state with Count = state.Count + 1 }
  | Decrement -> { state with Count = state.Count - 1 }
  | InputTextChanged text -> { state with InputText = text }
  | UppercaseToggled upperCase -> { state with IsUpperCase = upperCase }
  | SwitchPage page -> { state with CurrentPage = page }
```
Finally the `render` function is where both views, the counter and text input view are put together by switching over the `CurrentPage` field from the state:
```fsharp {highlight: [5,6,7,8, 27, 28, 29, 30]}
let render (state: State) (dispatch: Msg -> unit) =
  match state.CurrentPage with
  | Page.Counter ->
      Html.div [
        Html.button [
          prop.text "Show Text Input"
          prop.onClick (fun _ -> dispatch (SwitchPage Page.TextInput))
        ]

        divider

        Html.button [
          prop.onClick (fun _ -> dispatch Increment)
          prop.text "Increment"
        ]

        Html.button [
          prop.onClick (fun _ -> dispatch Decrement)
          prop.text "Decrement"
        ]

        Html.h1 state.Count
      ]

  | Page.TextInput ->
      Html.div [
        Html.button [
          prop.text "Show counter"
          prop.onClick (fun _ -> dispatch (SwitchPage Page.Counter))
        ]

        divider

        Html.input [
          prop.valueOrDefault state.InputText
          prop.onChange (InputTextChanged >> dispatch)
        ]

        divider

        Html.input [
          prop.id "uppercase-checkbox"
          prop.type'.checkbox
          prop.isChecked state.IsUpperCase
          prop.onChange (UppercaseToggled >> dispatch)
        ]

        Html.label [
          prop.htmlFor "uppercase-checkbox"
          prop.text "Uppercase"
        ]

        Html.h3 (if state.IsUpperCase then state.InputText.ToUpper() else state.InputText)
      ]
```
The highlighted parts shows two buttons that dispatch the `SwitchPage` messages that causes the application to switch from the counter view to the text input and vice-versa.

At this point, you must be thinking: "Zaid, why are we going through this simple stuff, we have seen this before! Just get to the compostion techniques already." The thing is, I want to show you that composing larger programs or in this case, splitting a bigger program into smaller ones, naturally arises from *refactoring* the common parts into separate type definitions and separate functions that handle these types. Let us try to refactor the counter view and text input view such that the implementation of either views is entirely separate from one another.

### Refactoring The State

As we have seen before, when the `render` function becomes too big, we split it into smaller functions. In the case of the function above, it makes sense to put the counter view as well as the text input view into separate functions, let us call them `renderCounter` and `renderTextInput` where they have the same definition as the `render` function. The `render` function passes the state, as well as the dispatch function down to these functions and you end up with the same result:
```fsharp {highlight: [1, 3, 15, 26]}
let renderCounter (state: State) (dispatch: Msg -> unit) = (*...*)

let renderInputText (state: State) (dispatch: Msg -> unit) = (*...*)

let render (state: State) (dispatch: Msg -> unit) =
  match state.CurrentPage with
  | Page.Counter ->
      Html.div [
        Html.button [
          prop.text "Show Text Input"
          prop.onClick (fun _ -> dispatch (SwitchPage Page.TextInput))
        ]

        divider
        renderCounter state dispatch
      ]

  | Page.TextInput ->
      Html.div [
        Html.button [
          prop.text "Show counter"
          prop.onClick (fun _ -> dispatch (SwitchPage Page.Counter))
        ]

        divider
        renderInputText state dispatch
      ]
```
However, these functions have a parameter of type `State` but the `renderCounter` is only using the `Count` field and `renderInputText` is using the fields `InputText` and `IsUppercase`. It doesn't make sense for `renderInputText` to have access to the `Count` field since it doesn't do anything with it. Let us refactor the code such that these functions only get the information that they actually use. We can start by putting the fields into separate records where each record holds the data required by a certain view:
```fsharp {highlight: [9, 10]}
type CounterState =
  { Count: int }

type InputTextState =
  { InputText: string
    IsUpperCase: bool }

type State =
  { Counter: CounterState
    InputText: InputTextState
    CurrentPage : Page }
```
Now the render functions for both views can have access only to the information thay actually require:
```fsharp {highlight: [1, 3, 15, 26]}
let renderCounter (state: CounterState) (dispatch: Msg -> unit) = (*...*)

let renderInputText (state: InputTextState) (dispatch: Msg -> unit) = (*...*)

let render (state: State) (dispatch: Msg -> unit) =
  match state.CurrentPage with
  | Page.Counter ->
      Html.div [
        Html.button [
          prop.text "Show Text Input"
          prop.onClick (fun _ -> dispatch (SwitchPage Page.TextInput))
        ]

        divider
        renderCounter state.Counter dispatch
      ]

  | Page.TextInput ->
      Html.div [
        Html.button [
          prop.text "Show counter"
          prop.onClick (fun _ -> dispatch (SwitchPage Page.Counter))
        ]

        divider
        renderInputText state.InputText dispatch
      ]
```
But now there is another problem: the dispatch function. As things are right now, the `dispatch` function passed down to `renderCounter` and `renderInputText` is able to dispatch any `Msg` case which means that the `renderInputText` is able to dispatch `Increment` or `Decrement` events at will even though they have nothing to do with that part of the application when the input text view is being rendered on screen. Likewise, `renderCounter` should only be able to dispatch `Increment` and `Decrement`, nothing else.

We can further restrict the `Msg` type into smaller discriminated unions such that `renderCounter` and `renderInputText` can dispatch only the events that are relevant to their parts of the application:
```fsharp {highlight: [10, 11]}
type CounterMsg =
  | Increment
  | Decrement

type InputTextMsg =
  | InputTextChanged of string
  | UppercaseToggled of bool

type Msg =
  | CounterMsg of CounterMsg
  | InputTextMsg of InputTextMsg
  | SwitchPage of Page
```
Now the two smaller functions can have proper types such that they only access the data they need and can only dispatch events that are related to their data:
```fsharp
let renderCounter (state: CounterState) (dispatch: CounterMsg -> unit) = (*...*)
let renderInputText (state: InputTextState) (dispatch: InputTextMsg -> unit) = (*...*)
```

### Composing Dispatch Functions
