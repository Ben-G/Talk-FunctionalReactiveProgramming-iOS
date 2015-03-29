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



