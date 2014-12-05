Programming paradigms have a tendency to come and go into trend, depending on the kind of problems programmers face and what hardware is available to them.
While we started out with very procedural code on (comparably) slow and expensive machines (the era of COBOL, Pick and firends), we have moved through object-oriented laguages (Smalltalk, Eiffel, Java and C# to name a few) and have met a resurgence in functional programming with Lips, F# and Clojure.


Programming paradigms have evolved over time to solve problems of a generation of take advantage of the latest hardware.
Functional programming was born out from academia based on the lambda calculus.
Its focus is in composing applications out of functions that have no state and no sideeffects.
Using such side-effect free functions promised easier reasoning and testing.
Object orientated programming on the other hand composes applications out of little objects that collaborate.
These objects hide their specific implementation behind an abstraction, which in turn makes each object isolated from changes in another. Modeling an application with the "things" or objects from a business domain was supposed to improve reasoning and make large scale systems easier to maintain.

The actor paradigm builds upon two inseparable entities: Actors and Messages.
Actors are the smallest unit of work in the application. They are to be small, decoupled and most of all stateless for the application to scale.


Let's have a look at the basic building blocks behind the Actor paradigm and then we can move on to see what it possible with it.
At the very core of the Actor paradigm are the actor themselves and the messages they can send and receive.
Actors are the small unit of work.
They are intenden to be stateless and only send messages as their side-effect.
Actors are long-lived objects whereas messages live just long enough to be consumed by an actor. To overcome this disparity, actors have a _mailbox_ which managed by the underlying system.
An actor is guaranteed to only process a single message at a time. This means that within an actor, there is no concurrency at all.
One important property that the messages have to fulfil to make this viable, is that they are to be immutable. Once a message is out, there is no going back or changing it mid-flight.
You can imagine a system of small actors to work a like a little factory where co-workers send eachother sealed envelops with orders or information.


