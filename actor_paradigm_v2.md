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

## Actor conversations for beginners
So much for the basic ideas of the **Actor Model**. Let's look at some code using Scala/Javas framework **Akka.**

One striking feature of Akkas implementation of the   **Actor Model** is its clean, minimal abstraction.
Below you can see a simple Akka Actor named `Bob` that will send a `Greeting` to an `Anna` Actor when he gets a `SayHello` message:

```scala
case object SayHello
case object Greeting(message: String)

class BobActor extends Actor {

  val anna = context.actorOf(Props[AnnaActor], name = "anna")
  def receive = {
    case SayHello => anna ! Greeting("Hi Anna!")
    case _ => unhandled(message)
  }
}

class AnnaActor extends Actor {
  def receive = {
    case Greeting(message) => log.info(s"I was greeted with '$mesasge'" )
    case _ => unhandled(message)
  }
}
```

All work happens in the `receive(message: Any)` method.
It takes `Any` as a parameter, which means you can send your actors any message you want!

To send an actor a message, you first must get a reference to it.
In this case we create the `anna` Actor on the fly during construction.
An important aspect here is maintaining abstraction. Even though we the `actorOf` method takes the class `AnnaActor` it will only return an `ActorRef` (_hidden a little bit behind Scalas type inference_)!
Once we have an `ActorRef` pointing to an `AnnaActor`, all we have to do is _tell_ it something.
In Scala we are allowed to have fairly arbitrary methods names, so _tell_ is shortened to a simple `!`.
Here, `BobActor` creates a new instance of the immutable case-object `Greeting(message: String)` with the greeting `"Hi Anna!"` and sends it on its way.

`BobActor` and `AnnaActor` model persons and as such should be able to almost have a conversation. Lets extend `BobActor` to _ask_ `AnnaActor` if she knows of any current acting giggs:


```scala
case object SayHello
case object Greeting(message: String)
case object ActingGig
case object Answer(message: String)

class BobActor extends Actor {

  val anna = context.actorOf(Props[AnnaActor], name = "anna")
  def receive = {
    case SayHello => anna ! Greeting("Hi Anna!")
    case Fired  => askAnnAboutGig()
    case _ => unhandled(message)
  }

  def askAnnaAboutGig = {
     val response = (anna ? ActingGig).mapTo[Answer]

     response onComplete {
       case Success(answer) => log.info(s"Anna said $answer.message")
       case Failure(t) => println("Something broke...")
     }
  }
}
```

And this is how `AnnaActor` would respond to that message using the `sender` method to get an `ActorRef` of the current message:
```scala
class AnnaActor extends Actor {
  val random = new Random
  def receive = {
    case Greeting(message) => log.info(s"I was greeted with '$mesasge'" )
    case ActingGig => {
       if random.nextBoolean {
         sender ! Answer("Sure, there is an opening for the Globe Theater")
       } else {
         sender ! Answer("Sorry, I have no acting gigs.")
       }
    }
    case _ => unhandled(message)
  }
}
```

That completes our brief overview of how to use Actors within Akka.
Let's have now move on to a topic where the **Actor Model** has a refreshing new approach: failure.

## Letting go
Handling error cases is hard. Handling them elegantly is harder.
Doing it in a distributed, concurrent application is among the hardest.

Akka has a very interesting approach to error handling: let it fail.
From time to time actors will encounter problems, like an unplugged network cable when talking to a 3. party or a failing database that that won't take any new connections.
Usually, such problems are solved locally, by catching some kind of exception, logging and more often than not, re-throwing an exception.
After all, what should we do?

Akkas approach on the other hand introduces clear semantics on what is supposed to happen: each actor has a supervisor that gets to decide what to do.
The action to be taken can be one of four:
  * Escalate: _This_ supervisor can't decide, so escalate the issue (_akin to re-throwing an exception_)
  * Restart: Discard the actor and create a fresh one to replace him
  * Resume: You can't win them all. Drop the message and proceed with the next one in the mailbox.
  * Stop: Discontinue processing and remove the actor.

Further more, there are to _SupervisionStrategies_ that are to be applied: `OneForOneStrategy` or `AllForOneStrategy`.
The difference in these strategies is in how far an _action_ is going to reach:
  * `OneForOneStrategy` will apply the action only to the failing actor
  * `AllForOneStrategy` will apply to **all** of the supervised sibblings, whether they failed themselves or not
