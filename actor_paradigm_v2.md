# A Look at Actors

Concurrency has become more and more relevant over the last few years.
Smartphones makers have continuously increased the number of cores in their package, firmly cementing *multitasking* and *multi-processing* in our everyday life.
I may only be in my mid-twenties, but my smartphone has more processing power and four times as many cores as first Pentium IV 2.6Ghz.

From a programmers perspective, concurrency is one the hardest things to program for.
The challenge lies both in making the most of the cores available to us and to make sure that programs do not crash or corrupt our users data just because we did not anticipate the octo-core that he has.

Inherently, concurrency is hard for us to reason about.
We humans are not as multitasking capable as we think we are.
Listening to music while you browse the web is not multi-tasking.
Even though you can probably hum along to your Spotify playlist, you are not concentrating on it.
Its pure 'audible' memory.

There are multiple approaches to handling concurrency being used today.
I'd like to give you an overview of the **Actor Model**, which has come and gone out of fashion but is undergoing a resurgence with Akka on the JVM.
The most interesting aspect of the **Actor Model** is that its core ideas allow us to *think* about concurrency in a different way than before.

## The approaches so far...

Different approaches have developed to handle situations where many things happen concurrently.
The object-oriented paradigm offers inherent solution to write concurrent programs.
It has resorted to creating abstractions for locking primitives such as mutexes, semaphores and cyclic barriers and relying on programmers to use these carefully.

Functional programming on the other hand is a viable option when working on concurrent programs as the the fundamental premise is that functions should be state and side-effect free.
Hence multiple calls to the same function should yield in the same result, no-matter the order or context.

## ...and the Actor Model.

A third approach which has had mixed popularity is the **Actor Model**.
The **Actor Model** is an interesting combination of functional- and object-oriented programming: From object-orientation it borrowed encapsulation and abstraction, from functional programming it took the principle of statelessness and immutability.

At the very core of the **Actor model** are the actors themselves and the messages they can send and receive.
Actors are intended to be stateless and the only perceivable side-effect should be messages as their side-effect.
They are long-lived objects whereas messages live just long enough to be consumed by an actor.
To overcome this disparity, actors have a *mailbox* which is managed by the underlying system.
An actor is guaranteed to only process a single message at a time.
This means that within an actor, there is no concurrency at all.
One important property that the messages have to fulfil to make this viable, is that they are to be immutable.
Once a message is out, there is no going back or changing it mid-flight.
You can imagine an Actor system like a little factory where co-workers send each other sealed envelops with orders or information.
Each works in isolation based on the information he has.

## Actor conversations for beginners

So much for the basic ideas of the **Actor Model**.
Let's look at some code using Scala/Javas framework **Akka**.

One striking feature of Akkas implementation of the **Actor Model** is its clean, minimal abstraction.
Below you can see a simple Akka Actor named `Bob` that will send a `Greeting` to an `Anna` Actor when he gets a `SayHello` message:

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
        case Greeting(message) => log.info(s"I was greeted with '$message'" )
        case _ => unhandled(message)
      }
    }


All work happens in the `receive(message: Any)` method.
It takes `Any` as a parameter, which means you can send your actors any message you want!

To send an actor a message, you first must get a reference to it.
In this case we create the `anna` Actor on the fly during construction.
An important aspect here is maintaining abstraction.
Even though we the `actorOf` method takes the class `AnnaActor` it will only return an `ActorRef` (*hidden a little bit behind Scalas type inference*)! Once we have an `ActorRef` pointing to an `AnnaActor`, all we have to do is *tell* it something.
In Scala we are allowed to have fairly arbitrary methods names, so *tell* is shortened to a simple `!`.
Here, `BobActor` creates a new instance of the immutable case-object `Greeting(message: String)` with the greeting `"Hi Anna!"` and sends it on its way.

`BobActor` and `AnnaActor` model persons and as such should be able to almost have a conversation.
Lets extend `BobActor` to *ask* `AnnaActor` if she knows of any current acting jobs:

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


And this is how `AnnaActor` would respond to that message using the `sender` method to get an `ActorRef` of the current message:

    class AnnaActor extends Actor {
      val random = new Random
      def receive = {
        case Greeting(message) => log.info(s"I was greeted with '$message'" )
        case ActingGig => {
           if random.nextBoolean {
             sender ! Answer("Sure, there is an opening for the Globe Theatre")
           } else {
             sender ! Answer("Sorry, I have no acting jobs right now.")
           }
        }
        case _ => unhandled(message)
      }
    }


That completes our brief overview of how to use Actors within Akka.
Let's have now move on to a topic where the **Actor Model** has a refreshing new approach: failure.

## Letting go

> Improve the wording in this section.
Clean up the end...
possibly add an example

Error handling cases is hard.
Handling errors elegantly is harder.
Doing it in a distributed, concurrent application is among the hardest.

Akka has a very interesting approach to error handling: let it fail.
From time to time actors will encounter problems, like an unplugged network cable when talking to a 3.
party or a failing database that that won't take any new connections.
Usually, such problems are solved locally, by catching some kind of exception, logging and more often than not, re-throwing an exception.
After all, what should we do?

Akkas approach on the other hand introduces clear semantics on what is supposed to happen: each actor has a supervisor that gets to decide what to do upon failure of a sibling.
The action to be taken can be one of four:

*   Escalate: *This* supervisor can't decide, so escalate the issue (*akin to re-throwing an exception*).
*   Restart: Discard the actor and create a fresh one to replace him.
*   Resume: You can't win them all. Drop the message and proceed with the next one in the mailbox.
*   Stop: Discontinue processing and remove the actor.

Further more, there are to *SupervisionStrategies* that are to be applied: `OneForOneStrategy` or `AllForOneStrategy`.
The difference in these strategies is in how far an *action* is going to reach:

*   `OneForOneStrategy` will apply the action only to the failing actor.
*   `AllForOneStrategy` will apply to **all** of the supervised siblings, whether they failed themselves or not.

What makes this approach so interesting is that it makes error handling explicit yet concise.
The wording `Escalate`, `Resume`, `Supervisor`, `OneForOneStrategy` perfectly fits into the domain.
It also allows you to handle all failures relevant to an actor in a single place due to the messaging nature of the model you can get retries out of the box.

## On the Scenic Route

Lets assume for a moment that `BobActor` and `AnnaActor` or not the only actors in our system.
Let's go as far as assuming that we have 5 `BobActor` and `AnnaActors` each.
They are perfect clones, so we don't care which of them takes up our acting gig.

Each of them could have their own agent that gives them `JobMessages`, but that would mean that some of the Actors go underutilized depending on whoever tells their agent about new acting jobs.

This is an area where the **Actor Model** shines.
Since actors don't hold state there is no difference in sending a message to an actor A or actor B as long as both fulfil the same job.
This allows us to have an entire swarm of actors just waiting to receive a message! As far as the sender is concerned, he sent the message to *any* of the right actors.
Which one ultimately does the work is of no relevance.

Akka provides `Routes` for this.
You can write your own, but the ones that come out of the box cover most of the cases.
The nature of these routes allows you to adapt dispatching messages to your actors.
Here are some of the more interesting `Routes`:

*   `RoundRobinRoutingLogic`: each of the actors in the pool just takes turns. Good for jobs that will roughly take the same time distributing the load evenly.
*   `SmallestMailboxRoutingLogic`: The time for the task may vary so assign the message to the actor with the least pending messages to keep the load approximately even.
*   `ScatterGatherFirstCompletedRouteLogic`: Your task is idempotent and latency critical, so have all actors deal with it and the original sender gets the first response that comes through.

In our example such a `Route` would be the agent.
Were we to use a `RoundRobinRoutingLogic` then all our actors would be treated equally.
In most cases this probably safe initial approach.

Were we to use a `SmallestMailboxRoutingLogic` then our agent would take into account that some `JobMessages` (*acting jobs*) such as "Filming Avatar" take longer than filming a Kelloggs commercial.
Faster actors with a shorter mailbox will thus receive messages with a higher priority.
This should reduce the latency in a system with variable task runtime.

Using the `ScatterGatherFirstCompleteRouteLogic` makes no sense for individual human actors.
But think about an actor being a production company.
A client such as Kelloggs will put in a request for a finished ad, and multiple `ProductionCompanyActors` will get the task.
Whoever finishes first will see his ad aired during halftime at the Super Bowl.
Though fairly inefficient from a resource perspective, this promise the lowest latency for the original sender of the messages.

Akka makes it possible to configure such `Routes` in a configuration file, which allows you to adapt to changes in latency requirements with a simple change in configuration.

# The Take-away

We barely skimmed the surface of the **Actor Model** and one of its implementation for the JVM, Akka.
This post was not meant to be an exhaustive introduction into either of those two.
We briefly touched on three elements of the **Actor Model** I'd like you take into consideration the next time you have to build concurrent system:

From *Actor Conversations For Beginners* I'd like you to take the idea of isolating concurrency to the smallest possible unit and rely on immutable messages as a means of communication.
Even just translating *immutable messages* into other languages like Java will help you in the future.

From *Letting Go* I'd like you to think about your concurrent sections of code as little, idempotent tasks that could be just restarted or dropped if need be.
Writing in that code in a way that you don't try to compensate failure by some complicated roll-back mechanism but just *roll-forward* with a new attempt is not always easy and at time scary.
We are so used fixing things we break, rather than just trying to mask our mistake by retrying.
but once that is a possibility, code becomes easier and failure is just another regular code-path.

Finally, from *On The Scenic Route* I'd like you realise that sometimes, doing a little extra work is not waste but has real user value.
During normal programming, we try to get the exact solution with the minimum amount of resources as fast as possible.
We will got to great lengths when trying to *get it right*.
The beauty of the `Routes` is that we can embrace spilling some resources for sake of latency.
The task your actor has to accomplish might require a low latency, so instead of tweaking the actor the nth degree, you could just hand over the task to one of 20 and just see who answers first.
Sure, you will have wasted 95% of the computing power, but you got the fastest result.

I recommend everyone to take a little time and study the **Actor Model**.
If Akka and the JVM are not your thing, there is a whole buffet of frameworks for different languages ([Pulsar] for both Python and Clojure, [SObjectizer] for C++ or [Celluloid] for Ruby).
For those who like to work on the classics first, I advise you to look into [Erlang] and [Elixir] first.
