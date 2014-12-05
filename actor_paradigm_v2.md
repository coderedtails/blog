# Looking at concurrency through the lens of Actors
Concurrency has become more and more relevant over the last few years.
Smartphones makers have continuously increased the number of cores in their package,
firmly cementing _multitasking_ and _multi-processing_ in our everyday life.
I may only be in my mid-twenties, but my smartphone has more processing power and four times as many cores as first Pentium IV 2.6Ghz.

From a programmers perspective, concurrency is one the hardest things to programm for.
The challenge lies both in making the most of the cores available to us and to make sure that programs do not crash or corrupt our users data just because we did not anticipate the octo-core that he has.

Inherently, concurrency is hard for us to reason about.
We humans are not as multitasking capable as we think we are.
Listening to musik while you browse the web is not multi-tasking.
Even though you can probably hum along to your Spotify playlist, you are not concentrating on it.
Its pure 'audible' memory.

There are multiple approaches to handling concurrency being used today.
I'd like to give you an overview of the **Actor Model**, which has come and gone out of fashion but is undergoing a resurgance with Akka on the JVM.
The most interesting aspect of the **Actor Model** is that its core ideas allow us to *think* about concurrency in a different way than before.

## The approaches so far...
Different approaches have developed to handle situations where many things happen concurrently.
The object-oriented paradigm offers inherent solution to write concurrent programs.
It has resorted to creating abstractions for locking primitives such as mutexes, semaphores and cyclic barriers and relying on programmers to use these carefully.

Functional programming on the other hand is a viable option when working on concurrent programs as the
the fundamental premise is that functions should be state and side-effect free.
Hence multiple calls to the same function should yield in the same result, no-matter the order or context.

## ...and the Actor Model.
A third approach which has had mixed popularity is the **Actor Model**.
The **Actor Model** is an interesting combination of functional- and object-oriented programming:
From object-orientation it borrowed encapsulation and abstraction, from functional programming it took the
principle of statelessness and immutability.

At the very core of the **Actor model** are the actors themselves and the messages they can send and receive.
Actors are intended to be stateless and the only perceivable side-effect should be messages as their side-effect.
They are long-lived objects whereas messages live just long enough to be consumed by an actor. To overcome this disparity, actors have a _mailbox_ which is managed by the underlying system.
An actor is guaranteed to only process a single message at a time. This means that within an actor, there is no concurrency at all.
One important property that the messages have to fulfil to make this viable, is that they are to be immutable. Once a message is out, there is no going back or changing it mid-flight.
You can imagine an Actor system like a little factory where co-workers send each other sealed envelops with orders or information.
Each works in isolation based on the information he has.

## Simple abstractions
So much for the basic ideas of the **Actor Model**. Let's look at some code using Scala/Javas framework **Akka.**

One striking feature of Akkas implementation of the   **Actor Model** is its clean, minimal abstraction.
Below you can see a simple Akka Actor named `Bob` that will send a `Greeting` to an `Anna` Actor when he gets a `SayHello` message:

```scala
class BobActor extends Actor {

  val anna = context.actorOf(Props[AnnaActor], name = "anna")
  def receive = {
    case SayHello => anna ! Greeting("Hi Anna!")
    case _ => unhandled(message)
  }
}
```

* messages are sent with !
* "questions", where an anser is exected are sent with ? and return a Future
* Sending can take any object.
* Receiving is exahaustive or will result in an undhandled message
* the recipient of the message is either created on the fly with actorFor(...)...
* ...for looked-up using a ActorSelector (URI) in the ActorSystem

## Letting go
Handling failure in a distributed, concurrent system is hard.
The Actor
