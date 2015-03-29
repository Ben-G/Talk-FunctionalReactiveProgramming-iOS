# (1) What is this talk about?

Throughout this talk I want to introduce you to the principals of FRP and to some practical examples of how to use Reactive Cocoa in your iOS Apps. 

I want to start by answering why we should use FRP in the first place. This can be done easiest with an example app that I prepared for this talk.

# (2) State

Let's take a look at this example app. It allows you to keep track of people you met, by adding them via their twitter handle + adding personal notes.

Here's the view that allows users to add people. It has 3 high level states:

- **No Entry**: the user hasn't entered any text, the add button is disabled
- **Default**: the user has entered text, the add button is enabled
- **Loading**: the user has hit the add button, with a username entered. Textfield and add button get disabled until request completes

Let's look at how one would typically implement this functionality on iOS. I've created a diagram that illustrates a potential implementation.

# (3) Manual state handling

Typically we use a mix of callbacks, delegate methods and the target/action pattern to respond to events and establish the relevant state.

The *first* event is the button tap. When the button is tapped, we kick off the network request. Then we disable the textfield and disable te button.

The *second* relevant event is the network request. When the network request returns an error, for example because the user could not be found, we need to enable textfield and button again so that the user can correct the entry and try again.

Finally, the *third* relevant input is provided by the UITextfieldDelegate methods. When the text in the textfield changes we need to decide wether the add button is enabled or not.

So for so good. What's the conclusion here?

# (4)State handling by mutating variables

Usually we perform state propagation manually by adding code to callbacks, delegate methods, notification handlers, etc. 

What do I mean by *manual state propagation*? I mean the task of setting up one of these 3 high level states that we've seen earlier by directly modifying the state of the involved UI components. 

So what is the problem with this approach?


