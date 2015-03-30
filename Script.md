#(1) What is this talk about?

Throughout this talk I want to introduce you to the principals of FRP and to some practical examples of how to use Reactive Cocoa in your iOS Apps. 

I want to start by answering why we should use FRP in the first place. This can be done easiest with an example app that I prepared for this talk.

#(2) State

Let's take a look at this example app. It allows you to keep track of people you met, by adding them via their twitter handle + adding personal notes.

Here's the view that allows users to add people. It has 3 high level states:

- **No Entry**: the user hasn't entered any text, the add button is disabled
- **Default**: the user has entered text, the add button is enabled
- **Loading**: the user has hit the add button, with a username entered. Textfield and add button get disabled until request completes

Let's look at how one would typically implement this functionality on iOS. I've created a diagram that illustrates a potential implementation.

#(3) Manual state handling

Typically we use a mix of callbacks, delegate methods and the target/action pattern to respond to events and establish the relevant state.

The *first* event is the button tap. When the button is tapped, we kick off the network request. Then we disable the textfield and disable te button.

The *second* relevant event is the network request. When the network request returns an error, for example because the user could not be found, we need to enable textfield and button again so that the user can correct the entry and try again.

Finally, the *third* relevant input is provided by the UITextfieldDelegate methods. When the text in the textfield changes we need to decide wether the add button is enabled or not.

So for so good. What's the conclusion here?

#(4)State handling by mutating variables

Usually we perform state propagation manually by adding code to callbacks, delegate methods, notification handlers, etc. 

What do I mean by *manual state propagation*? I mean the task of setting up one of these 3 high level states that we've seen earlier by directly modifying the state of the involved UI components. 

So what is the problem with this approach?

#(5) Problems with manual state handling

Using this classic apporoach of responding to user input, state handling code is dispersed which means it has poor code locality. This makes it harder to reason about it's functionality. Even in this simple example we have at least one delegate method, one callback block and an IBAction involved. Many views involve more logic, making this state propagation code very hard to read.

This in turn means that is hard to *see* which input sources drive the state of a certain component, or which potential states exist in the first place. Without further documentation it will be unclear which valid and invalid states exists.

That also means it is hard to change the code without introducing bugs.

There's another thing I'd like to point out...

#(6) Large amount of invalid states

You might think: this is a really trivial feature. Aren't we overthinking things when we're dissecting it so much?

In the example above, we have a total of 4 different states. The button can be enabled/disabled and the same is true for the textfield.

In each *high level* state we only have a *single* valid combination of *low level states*. By low level states I mean the state of each individual UI component. For example in the *No entry* state the only valid combination of states is:

 - textfield disabled
 - button disabled

This leaves us of with 3 invalid possible combinations. This is true for all three high level states which results in 9 possible invalid states. **That's a lot of room for error!**

Just to drive this point home...

#(7) A common example of state management gone wrong

I want to show this very common example of manual state management gone bad. I bet that every iOS user has seen a bug of this sort before.

Actually this specific example is from a Mac OS X app, but the underlying issue is the same.

The table view that you can see here recycles cells to avoid allocating huge amount of memory as a user scrolls through a long list.

Recycling a cell involves manual state management, at least if you aren't working with FRP tools.

**Many, many apps get this wrong.** In the left image you can see the table view as it should look, with my profile picture next to all of my commit messages.

In the right image you can see the same part of the table view after frantically scrolling for a few seconds. You can see that many cells do not display my profile image anymore. This is not a networking issue, the app is not waiting for the image to download; Instead manually resetting the state of this cell, and likely forgetting to cancel an asynchronous request that sets the image have caused this issue.

When working with manual state management, a developer **needs to remember all of the stateful information stored** in the view and reset it.

This puts a lot of burden onto the programmer!
I had this bug in my MovieLoggr app as well before using Reactive Cocoa :)

#(8) What is FRP?

I hope I was able to show some problems and pain points of manual state management; now let's look at what FRP is and how it can help us avoid these kind of issues.

#(9) Imperative vs. Declarative

One essential differentiator of FRP is, that allows us to replace a lot of *imperative* code with *declarative* one. 

What's the difference between these two, and why should declarative code be better?

Let's look at a classic example, the spreadsheet problem.

#(10) Imperative Spreadsheet

Assume you know how to derive the value of the Column **C** from the values in **A** and **B** and you want to describe it to someone. 

The imperative way would be breaking this down into a set of instructions.

1. Perform the following steps whenever A or B changes
2. Add 50 to value of A
3. Subtract 10 from value of B
4. Add the results from 1.) and 2.)
5. Write result from 3.)  into C

This is how we write the majority of our code these days.

What does the declarative approach look like?

#(11) Declarative Spreadsheet

The declarative way is to **describe the relationship** instead of explaining the steps of how the calculation can be performed.

When we are working with a spreadsheet program we can use such a **declaration** and the spreadsheet program will know how to implement the declared relationship.

*A program that describes what computation should be performed and not how to compute it.*

The declarative approach is a lot simpler and easier to understand by other developers. **The semantics of the relationship are visible from this formula.** 
In the imperative example we need to read the code and figure out the relationship between A, B and C.

So now that we've discussed **declarative** code and **imperative** code in the abstract, let's go back to our example app with the enabled/disabled textfields and buttons.

#(12) Imperative Code

What does the imperative implementation of our example look like? 

Exactly as we've illustrated it earlier. Using callbacks and delegation are examples of using imperative code.

#(13) Imperative State propagation

Looking at this diagram now, it is very similar to the **imperative** approach of solving the spreadsheet problem.

We have different events that trigger a sequence of instructions. The goal of these intructions sequences are to move the view from one state to another. Similar to the spreadsheet, where the relationship between the columns was not obvious when using the imperative approach, the relationship between inputs and state of individual components is not obvious here.

#(14) Declarative

What would a declarative solution look like?
What is the equivalent to describing the relationship between columns in a spreadsheet?

#(15) FRP in a nutshell

This is the basic structure of a code using principles of FRP.

Each row describes how the state of an individual component can be **derived**. On the right hand side we can see the different **event emitters** that influence the state of the component.

All of the events of these event emitters feed into a single **decision mechanism** that is responsible for deriving the state of the UI component based on the latest event. 

For example in the first row, if the last event delivered was a Network Error Callback, we want to enable the textfield so that the user can modify the entered username and perform a new request.

##Next animation step

The decide mechanism is **referentially transparent**. This means, given the same input it will always produce the same output. 

The mechanism doesn't consider any outside state. It derives the state of the UI component solely from the latest incoming event. This makes the behavior of the decide mechanism predictable. This also takes a lot of burden of the developer and reduces room for error. We no longer need to reset state manually. Once the relationship between event and state is declared the FRP framework will garantuee and implement that declared relationship. 

##Next animation step

This diagram illustrates the two main aspects of Functional Reactive Programming. 

The code is **reactive** because it responds to events. The state deriving mechanism is triggered whenever one of the event emitters produce a new event.

The code is **functional** because we use a referentially transparent function to derive state in our application. Later we will also see that FRP provides higher order functions, another aspect of FP, to transform emitted event values.

#(16) Derived State

This is very different from pure FP. Having state in FRP is absolutely fine as long as it is derived state.

#(17) Reactive Cocoa
For now he have covered enough of the theory of FRP to dive into a framework that brings FRP to iOS: Reactive Cocoa.

It provides the tools to implement the FRP architecture that we discussed.

I hope you aren't too disappointed, but throghout the remaining slides of this talk you will see Objective-C code.

As of today, Reactive Cocoa 2 is the latest supported major release. It is entirely written in Objective-C and has some compatibilty issues with Swift code.

Reactive Cocoa 3 is currently under development. The first alpha was released about 2 weeks ago (mid march). It provides a pure Swift API but it isn't ready for production use yet, and therefore wasn't an option for this talk.

RAC 3 will bring some semantical changes to the API, however I think at this point learning RAC 2 first makes a lot of sense. Similar as it makes sense to learn Objective-C along with Swift.

If you like RAC 2, I'm sure you will love RAC 3. It cleans up some parts of the API and it makes use of Swift features, most importantly Generics. As we'll go through the example code on the next slides, you will see how Generics will be very benefitial for this API.

Let's get to know the framework!

#(18) Signals

Signals are **the** core concept that is used for programming in Reactive Cocoa. They provide a unified interface for different types of *future values*. 

Signals can wrap callbacks, delegate methods, KVO.

Throughout the next slides you'll see why this is extremely useful. Now let's look at some API level details of Signals in Reactive Cocoa.

#(19) RACSignal

A signal has four different states:

- At first it is inactive
- Then it becomes active and sends values through the *sendNext* method
- The signal keeps sending values until:
   - it errors and calls *sendError*
   - it completes and calls *sendCompletes*
   
Signals by themselves aren't extremely interesting. They become intersting as soon as some *subscriber* subscribes to them. Whoever subscribes to a signal receives all emitted values and also gets informed when the signal errors or completes.
   
A signal could for example be an HTTP request that downloads 5 images. At some point the signal would become active, then it would retrieve one image at a time from a webservice, sending *sendNext:* for each of the images to each of its subscribers. Finally it would send the completed event.

*Signals send values over time until they complete or error out*.

Since the only the subscription makes a Signal useful, let's look at different ways to subscribe to and consume values emitted by Signals.

#(20) Binding and subscribing

There are two different ways to consume Signals. We can bind them OR we can subscribe to them. Understanding the difference is important, so let's dive right into the details.

#(21) Binding

Binding is the preferred way of consuming a Signal. The most typical form of binding in RAC is the binding of a signal to an object property.

In this example we have a Signal that represents the text entered into a textfield, and the changes thereof. We bind this signal to the *email* property of a User object. This means, whenever the text updates, and thus the Signal sends a new value, the *email* stored with the user is updated to this new value.

Using this type of binding we can avoid manually mutating variables and can instead let Signals drive these variable changes.

Let's take short look at the code involved. We are using to different parts of the RAC API to accomplish this binding.

Firstly, we are using the `rac_textSignal`. This an extension provided by RAC that wraps changes to the textfield's text into a Signal that we can consume. RAC provides many of these extensions that add signals to existing UIKit APIs.

Secondly we are using the `RAC` macro. The `RAC` macro allows us to define a target object and a keypath on which a signal should operate. It allows us to use a signal on the right hand side of the assignment and the affected object + keypath on the left hand side.

With this code in place, the text of the textfield is bound to the email property of the user in will automatically update it.

No manual mutating, less errors. Now that we discussed the essentials of bindings let's take a look at *subscriptions*.

#(22) Subscriptions

Explicitly subscribing is another way of consuming a signal. Instead of binding a signal value to a property, we provide a *block* that is executed whenever the signal emits a new value.

Subscriptions should only be used in rare cases as executing aribitrary code upon new signal values brings us back to imperative code. 

Explicit subscriptions can be used to inject side effects, e.g. logging values. Side effects in FRP should be minimized, therefore explicit subscriptions should be minimized to.

#(23) Prefer binding

A reminder slide in case you'll review this presentation later. Explicit subscriptions are the first step back into the imperative world, avoid them as much as possible.

Now that we have discussed Signals and Signal subcriptions in RAC let's go back to our demo app and let's take a look at some real world RAC code.

#(24) Model-View binding

Let's start with a simple example. One way bindings from the model to the view.

#(25) Model-View binding code

Here you can see a view from the example app. It displays the twitter profile information of a person that has been added to the user. 

With our knowledge of RAC we know that we can use bindings to tie all of these subviews to the relevant properties of the person model.

On this slide you can see a subset of the binding code for this view. In the first line we are binding the avatar property of the person to the avatarImageView. In the second line we are binding the person's name to a text label. The code involved here is pretty straightforward.

The only new API call is `RACObserve`. `RACObserve` turns KVO notifications into a signal. The first argument is the object we are observing, the second argument is the keypath. 

Looking at the first line, whenever the `model` or the `avatar` changes, the generated signal will emit a new value. Since the signal is bound to a view property, the view will update automatically.

Theres another important thing worth mentioning. We don't need any code to check wether the `model` already has been set in this method. Without bindings we typically need to check that both the model and the view have been set/ set up before we can perform any of these assignments.

Using bindings we can declare all the relationships *upfront* in the `awakeFromNib` and the view will be updated as soon as the model is available.

#(26) Model-View binding - check 

This was pretty straightforward. You could stop here and use RAC only for such kind of bindings and it already would be usfeul. But I want to dive into some more complex examples to show that entire apps can be built purely on FRP principles, not only small individual features.

#(27) Model <-> Binding

We have seen real world RAC code for simple, one way bindings without interactivity. As we've shown earlier FRP shines when it comes to implementing interactive features so let's look at some real world interactive code.

We will need to take a short detour and discuss to aspects of the RAC API before moving on.

#(28) Signal Operators

A very powerful feature of RAC that you will use a lot when implementing more complex features are signal operators. Signal operators allow us to modify Signals and the values they carry, using declarative code.

Throughout the code samples we will see a few operators and discuss how they work as we go.

On this slide you can see the `map` operator. The map operator allows us to `map` each value of a stream to a new value. In this specific diagram I'm coming back to our earlier example. When we have an empty textfield we want to disable a button, when the textfield is filled we want to enable it.

We could solve this problem with imperative code, but instead we can use the map operator to map the current value of the textfield to a corresponding boolean value. The mapping code is pretty simple, if the string is empty we return NO otherwise YES.

You'll see code samples with operators shortly. Operators provide us with a powerful way of transforming signal values into representations that are useful for a specific binding, making imperative code unecessary in almost all cases. In this example `button.enabled` is not interested in the text itself, but only if it is empty or not, so we use an operator to map to a boolean value.

#(29) RACCommand

The last important part of RAC's API that we haven't yet discussed is `RACCommand`. 

RACCommands are used to create Signals triggered by certain actions, in most cases user interactions. Not all Signals are set up as soon as the app or a view is instantiated.

A typical example is a button that triggers a network request. We want to start the request when the button is tapped, therefore we create the Signal that wraps the network request as soon as the button is tapped.

The `RACCommand` exposes two signals. One is the `Errors` signal, but I don't want to get into the details of error handling with `RACCommand` now. The important one for us is the `ExecutionSignals` Signal.

It's a Signal of Signals. Whenever a button is tapped and a new signal is spawned, that new Signal is sent on the exection signals signal.

Subscribing to RACCommand is interesting, because the subcriber can either subscribe to the Signal of Signals or to the inner Signals themselves. In most cases we only need to subscribe to the inner signal.

Bringing this back to our example: When the user taps a button, a Signal is created and a network request is triggered. The subscriber immediately subscribes to that network request and gets informed when it sends new values or completes.

I think the concept of `RACCommand` will become more obvious as we take a look at some code later on.

#(30) Model <-> View Binding

Now that we've discussed Signal Operators and RACCommands we have the toolset to understand interactive RAC code. 

So how do we typically architect interactive RAC code? It turns out that MVC is not the ideal architecture when working with RAC. 

One of the main tasks of the Controller in an MVC architecture is to provide different callbacks, delegate methods etc. to build a relationship between the view and the model. With RAC we are trying to move away from imperative code, therefore the Controller doesn't really fit into our app architecture. We don't need a heayweight class that mediates between View and Model.

So let's take a look at an alternative.

#(31) MVVM

Using FRP we can replace the Controller with a `ViewModel`. This brings us to a Model-View-ViewModel architecture. 

With this architecture the view is extremely lightweight. All it does is bind to properties of the ViewModel.

The ViewModel is the glue between the Model and the View. It stores the view state, e.g. which buttons are enabled or disabled. It tunnels the relevant properties of the model to the view. It also invokes methods on the business layer, e.g. network requests. 

It's important to note that in this architecture Views and ViewControllers are considered part of the *View* layer. So you can either create custom views or ViewControllers.

Let's not discuss this in theory all too long, instead let's look at the next feature of our example App that's been implemented on the MVVM architecture.

#(32) Enabling/Disabling add button

On this slide you can see the sequence of interactions for adding a new person in our example app. First, we'll focus on the first two steps. Allowing the user to enter a twitter handle and activating/deactivating the *add* button accordingly. We've already discussed this in theory, now let's take a look at the implementation.

#(33) PersonAddingViewModel

On this slide you can see how the View and the ViewModel for this feature are conencted. The View in this example is called the *PersonAddingView*.

For this feature we have two relevant UI components in the view. The textfield and the add button.

The textfield in the View is bound to a property called `usernameSearchText` in the ViewModel using a two-way-binding. This means that changes to either of the two are reflected to the other part of the binding. Updating the viewModel updates the textfield. Updating the textfield updates the ViewModel.

The add button gets a RACCommand assigned to it. That RACCommand is provided by the ViewModel. The RACCommand contains the code that kicks of the request to the twitter API and looks up the usernae that has been entered.

All of that code is implemented in the ViewModel, the view only sets the RACCommand on the UIButton.

Additionally the ViewModel provides an `addButtonEnabledSignal`. That signal is used to determine if the add button is enabled or not, we'll get into the details in a second.

Hopefully this example illustrates the role of the View and the ViewModel. The View is a very thin layer that binds to values and signals of the ViewModel.

#(34) PersonAddingView - Initialization

To show you just how thin the View layer is, here is some setup code from the `awakeFromNib` in the `PersonAddingView`. All we do here is take signals from the ViewModel and bind and assign them to UI components.

`rac_command` is another RAC UIKit extension that allows us to hook up a RACCommand that is executed whenever a button is tapped.

The code in the ViewModel is more interesting, since it's the component that provides the signals consumed by this view.

#(35) PersonAddingViewModel - Button enabled signal

Here you can see how the signal that enables/disables the button is impleted. 
Since this signal depends on the search text that is entered, we start with a signal that captures changes to `usernameSearchText` by using the `RACObserve` macro. Then we use the `map` operator to map each of the text values that we receive into a boolean value.

The latest search text get's handed into the map method as an argument. We check wether the text is empty or not and return an according boolean value.

Voila! We have a signal that reacts to the search text and sends boolean values that describe wether the `add` button should be enabled or not.

#(36) PersonAddingView - deactivated button

Here you can see how View and ViewModel communicate on a conceptual level. 
An empty text results in a deactivated button.

#(37) PersonAddingView - activated button

And a non-empty search text triggers our Signal to send a `YES` value to the view and enable the button.

Now we've seen how we can use the MVVM architecture to build interactive views with RAC. 

#(38) Networking with Reactive Cocoa

Now let's look how we can integrate network requests into this example, another area where RAC shines in my oppinion.

#(39) Example App Part 2

We've covered the transition between the first two states shown here. Now I want to extend our example and look how we can kick off the network request and how to display the profile details upon successful response.

#(40) PersonContainerView

For this step it's important to understand the view hierarchy that I chose for the example project. I've decided to add a `PersonContainerView` that has a `PersonAddingView` **or** a `PersonDetailView` as a child, depending on the current state.

So when the Twitter API request completes successfully the PersonContainerView needs to switch is child view from the `PersonAddingView` to the `PersonDetailView`.

Let's start discussing the implementation by looking at the `RACCommand` that is attached to the add button in `PersonAddingView`.

#(41) TwitterRACCommand

This `addTwitterButtonCommand` is defined in the PersonAddingViewModel and it has two interesting components.

#(42) TwitterRACCommand - Enabled

Let's first take a look at these two highlighted lines. We are initializing a `RACCommand` and as port of the initializer we are handing in a signal. This is a feature of `RACCommand`, it allows us to provide a signal that determiens wether the command is enabled or not. Wether or not the `RACCommand` is enabled will automatically be reflected on the UI component to which the `RACCommand` is assigned.

So *technically* this is the line of code that enables/disables the `twitterAddButtonCommand` and in turn the add button on the PersonAddingView.

We could also bind the `addButtonEnabledSignal` to the button directly, but semantically it makes more sense to enable/disable the command and let the RAC framework handle the button state accordingly.

#(43) TwitterRACCommand - Button Callback Code

Now let's shift our focus to the inner block. This block gets invoked whenever the button is tapped and it is responsible for returning a signal. Here we are returning a signal by calling `infoForUsername` on the `twitterClient`.

That method on the `twitterClient` kicks of an API request and returns a Signal that will emit values as soon as the API request finishes. We will discuss the implementation details of that method later on.

##Animation step

It might not be obvious, but it's amazing how little code is called when the add button is tapped. **We are doing exactly ONE thing, kicking of the network request**. Using traditional imperative code we would likely handle the callback in exactly the same place where we kick of the request.

This results in **huge** code locality issues. As a response to this API call we need to create/update a person model and we need to have the PersonContainerView switch from the current view to the ProfileDetailView.

Typically we would have the callback code here and we'd need to use delegation to communicate with the components that are *actually interested in the outcome of this request*. Using Signals we can focus on one thing here and let any component that needs to know about completed Twitter API requests subscribe. The `PersonAddingViewModel` doesn't need to know about the parties that are potentially interested, this component is decoupled a lot better than with a classic imperative approach.

Now let's see how this request can be consumed by the PersonContainerViewModel in order to display the Person Details View.

#(44) PersonContainerViewModel - UIState

We have two steps going on here. Firstly, subscribing to the Signals emitted by tapping the add button. Secondly binding that Signal to the UIState.

Let's look at how we subscribe to the Signal first.

#(45) PersonContainerViewModel - Subscription

Let's discuss this Signal step by step. We start with a signal that observers the `personAddingViewModel` property of the PersonContainerViewModel. The `PersonAddingViewModel` is only created when necessary, not as soon as the container is created. Therefore we can only subscribe to the add button once the AddingViewModel has been initialized.

On this Signal we use the `flattenMap` operator. Simplified we can say that `flattenMap` allows us to chain a signal to another, and send the values from the latest chained signal to the subscriber.

So whenever the `personAddingViewModel` is set, we subscribe to the inner signal of the `addTwitterButtonCommand`. That means we are subscribing to the Signal of the twitter API request. Earlier we discussed that the `executionSignals` is a Signal of Signals that are spawned by button taps. By using the `concat` operator we subscribe to the inner signal, which is the network request itself.

The result of all of this is that we have `twitterFetchSignal` that will send the latest values from the latest Twitter API request.

Now how do we consume this signal?

#(46) PersonContainerViewModel - UIState binding

In this second step we are creating a new signal `UIStateSignal` that we bind to the `UIState` property. This signal uses the `twitterFetchSignal`, and we apply the `startWith` operator.

The `startWith` operator creates a new Signal from a given Signal that returns the provided constant as it's first value. This means this Signal will always return `PersonCollectionReusableViewStateAddingTwitter` as first value, then all future values will be provided by the twitter fetch signal.

The `startWith` operator is another great example of how we can avoid imperative code. Instead of explictly mutating the `UIState` we can define an initial value as part of our signal.

After the signal is constructed we bind it to the `UIState` property.
Now we we will always start in the `Adding` state, then switch to the details state as soon as the twitter request completes.

There's additional code that handles the view replacing whenever the UIState changes, but we won't discuss it now.

Last but not least we are also updating the person model. The twitter request returns a new person and we bind the person property of this ViewModel to the twitterRequestSignal.

#(47) Network binding complete

Now we have seen a pretty advanced feature that can be solved with RAC. Using Signals we can handle responses of network requests *where they need to be handled.* The place where the network request is started doesn't have to be the place where all responses are handled.

Let's take a little closer look at the Twitter API request, RAC has some more advantages when it comes to networking that we haven't discussed yet.

#(48) Chaining network requests

Using the `flattenMap` operator we can chain operations without ending up in callback hell. Here you can see the `infoForUsername:` method that we are calling when the add button is tapped.

We are performing 4 distinct operations:
 
 1. Authenticate with Twitter API
 2. Get user info
 3. Download avatar
 4. Construct a Person object

This is much more readable than nested callbacks! It's possible because error handling happens in the subscriber. If any of the chained signals errors out, the error bubbles up to the subscriber. All errors can be handled in only one error block in one place.

Used this way, RACSignals work similar to promises, we return a `RACSignal` immediately, the `RACSignal` will send a `Person` object as soon as all chained operations have been completed.

Networking is another great use case for RAC and is the main reason I originally adopted the framework. To conclude the discussion of networking in RAC I want to show you how you wrap non RAC network requests into Signals, an important part of bridging Non-RAC code into the RAC world.

#(49) Wrapping network requests into Signals

Here's an example from the Demo project where I'm wrapping a `STTwitterAPI` call into a RACSignal. We create a Signal with the `createSignal` method. That gives us a reference to a subscriber. We are responsible for sending the `sendNext`, `sendCompleted` and `sendError` messages to that subscriber.
Here it means adding the according method calls to the callback blocks.

Additionally the `createSignal` requires us to return a `RACDisposable`. A `RACDisposable` allows us to provide a block of code that is executed when the Signal is cancelled in case the subscriber unsubscribes. In this specific case the `STTwitterAPI` doesn't provide an API to cancel the request, so we cannot create a Disposable that would cancel the request - instead we return `nil`.

This illustrates how you can bridge non-RAC code into the RAC World.

#(50) MMVM - Check

We have covered a lot now. We've discussed how to build a highly interactive RAC app that responds to User input and performs network requests. We've also seen how the MMVM architecture works in combination with bindings.

#(51) Testing

One last aspect I want to cover briefly is testing RAC code, because I think it's another are where FRP and MMVM makes your life easier.

#(52) Testing UI without UIKit

Here I've picked an example test case that involves the UI, a part of our codebase that is typically harder to test. 

Using RAC and MVVM we can almost always resort to testing the ViewModel instead of the View. This means we don't need to stage UIKit components or emulate touch events. 

Obviously this runs into limitations when you want to test sophisticated animations, but it should make the bulk of your tests easier.

Additionally Signals give you a good starting point for deciding what to test. Building an App that relies heavliy on RAC means that by testing all Signals you will have fantastic test coverage. And Signals are pretty easy to test since they do not rely on global state but only on the values emitted by events.

In this example we want to test that the twitterClient is called. So I create a twitter mock object that returns a very simple signal. Then I initialize the PersonAddingViewModel with the mock twitter client. Then we set a username search text and execute the `addTwitterButtonCommand`. 

All we need to verify is that `twitterMock` received `infoForUsername` call.
Staging this test is straightforward even though it involves UI and talking to the networking code. RAC drives the design of our app to be fairly easy testable.

#(53) Summary

Let's sum all of this up.

#(54) Reactive Derived State

After talking a lot about implementaiton details and features of the RAC API I want to come back to the original motivation for exploring FRP. Instead of having dispersed, imperative state handling code we can work with declarative code.

RAC provides us with tools to describe relationships between events, values and state, taking a lot of burden of the developer and reducing the likelyhood of bugs.

#(55)
#(56)
#(57) Dijkstra
I want to close with a quote from a very famous computer scientist. He obviously was not talking about Functional Reactive Programming, but instead was arguing that the use of GOTO statement was a bad practice.

However, this quote fits amazingly well to the comparison of FRP and imperative programming. FRP enables us to write declarative code with **static** relations. Imperative code requires us to **know** how state and codepaths evolve over time.
#(58) 
