# Look at Actors

Concurrency has become more and more relevant over the last few years.
From a programmers perspective, concurrency is one the hardest things to program for.
The challenge lies both in making the most of the available processing power and ensuring that programs do not crash or corrupt our users data just because we did not anticipate his octa-core.

Inherently, concurrency is hard for us to reason about.
We humans are not as multitasking capable as we think we are.
Listening to music while you browse the web is not multi-tasking.
Even though you can probably hum along to your Spotify playlist, you are not concentrating on it.
Its pure 'muscle' memory.
Hence, its just as hard to imagine how the code we just wrote *sequentially* will run *concurrently*.

There are multiple approaches to handling concurrency in use today.
The **Actor Model** is a very interesting approach to concurrency as it makes you *think* about concurrency very differently.

## The approaches so far...

Different approaches have developed to handle situations where many things happen at the same time.
The object-oriented paradigm offers no inherent solution to write concurrent code.
It has resorted to creating abstractions for locking primitives such as mutexes, semaphores and cyclic barriers and relying on programmers to use these carefully.

Functional programming on the other hand is a viable option when working on concurrent programs as the fundamental premise is that functions should be state and side-effect free.
Hence multiple calls to the same function should yield in the same result, no matter the order or context.

## ...and the Actor Model.

The **Actor Model** is an interesting combination of functional and object-oriented programming: it borrows encapsulation and abstraction from object-orientation and takes the principles of statelessness and immutability from functional programming.

At the very core of the **Actor model** are the actors themselves and the immutable messages they can send and receive.
Actors are intended to be stateless and the only perceivable side-effect should be the messages they send to other actors.
They are long-lived objects, whereas messages live just long enough to be consumed by an actor.
To overcome this disparity, actors have a *mailbox* which is managed by the underlying system.
An actor is guaranteed to only process a single message at a time.
This means that within an actor, there is no concurrency at all.
The fact that the messages are immutable ensures that two actors will never have a shared resource.
You can imagine an actor system like a little factory where co-workers send each other sealed envelopes with orders or information.
Each one works in isolation based on the information he has.

## Actor conversations for beginners

So much for the basic ideas of the **Actor Model**.
Let's look at some code using Scala/Java's framework [Akka][1].

One striking feature of Akka's implementation of the **Actor Model** is its clean, minimal interface.
Below you can see a simple Akka actor named `Bob` that will send a `Greeting` to an `Anna` actor when he receives a `SayHello` message:

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
It takes `Any` as a parameter, which means you can send your actors any message you want.

To send an actor a message, you first must get a reference to it.
In this case we create the `anna` actor on the fly during initialisation.
An important aspect here is maintaining abstraction.
The method `actorOf` will always return an instance of an `ActorRef`, rather than the concrete type of the actor.
Once we have an `ActorRef` pointing to an `AnnaActor`, all we have to do is *tell* it something, which is done with the handy `!` method (*Scala allows arbitrary method names*).
In the above example `BobActor` creates a new instance of the immutable case-object `Greeting(message: String)` with the greeting `"Hi Anna!"` and sends it on its way.

`BobActor` and `AnnaActor` model persons and as such should be able to almost have a conversation.
Let's extend `BobActor` to *ask* `AnnaActor` if she knows of any current jobs.
The difference between *tell* and *ask* is that we expect an answer from *ask*.
Since we don't know when we'll get the answer, the method returns a `Future` onto which we can attach a callback:

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
           case Failure(t) => log.error("Something broke...")
         }
      }
    }


That completes our brief overview of how to use actors within Akka.
Let's now move on to a topic where the **Actor Model** has a refreshing new approach: failure.

## Letting go

Handling errors is hard.
Doing so in a distributed, concurrent application is among the hardest.

Akka has a very interesting approach to error handling: let it fail.
From time to time actors will encounter problems, like an unplugged network cable or a failing database that won't accept any new connections.
Such failures are usually solved locally by logging and possibly rethrowing an exception After all, what else should we do?

Akkas approach on the other hand introduces clear semantics on what is supposed to happen: each actor has a supervisor that decides how to react to the failure of a sibling.
The action to be taken can be one of four:

*   Escalate: *This* supervisor can't decide, so escalate the issue (*akin to re-throwing an exception*).
*   Restart: Discard the actor and create a fresh one to replace him.
*   Resume: You can't win them all. Drop the message and proceed with the next one in the mailbox.
*   Stop: Discontinue processing and remove the actor.

Furthermore, there are two *SupervisionStrategies* that can be applied: `OneForOneStrategy` or `AllForOneStrategy`.
The `OneForOneStrategy` means that an action will only be applied to the failing actor.
`AllForOneStrategy` is interesting when restarting actors.
This means that the failure of one actor will lead to restarting the entire group of actors under supervision.

This is interesting because it makes error handling explicit yet concise.
The words `Escalate`, `Resume`, `Supervisor`, `OneForOneStrategy` perfectly fit into the domain.
It also allows you to handle all failures relevant to an actor in a single place and due to the messaging nature of the model you get retries out of the box.
Lastly, it allows you to express that some tasks may fail without your entire application having to suffer.
Sometimes a simple restart of your actors is just enough to solve the problem.

## The Scenic Route

Let's assume for a moment that `BobActor` and `AnnaActor` are not the only actors in our system.
Let's go as far as imagining that we have 5 `BobActor` and `AnnaActors` each.
They are perfect clones, so we don't care which of them picks up an acting gig.
How do we make the most of these actor clones?

This is an area where the **Actor Model** shines.
Since actors don't hold state, there is no difference in sending a message to an actor A or actor B as long as both fulfil the same job.
We can even combine multiple actors under a virtual address.
After all, we are just sending messages, not calling methods on specific instances of actors.
This allows us to have an entire swarm of actors just waiting to receive messages! As far as the sender is concerned, he sent the message to *any* of the right actors.
Which one ultimately does the work is of no relevance.

Akka provides `Routes` for this.
The nature of these routes allows you to adpat how messages are dispatched to your actors.
Here are some of the more interesting `Routes`:

*   `RoundRobinRoutingLogic`: each of the actors receives messages in turns. This is good for jobs that will roughly take the same time leading to an even load distribution.
*   `SmallestMailboxRoutingLogic`: The time for the task may vary so assign the message to the actor with the least pending messages to keep the load approximately even.
*   `ScatterGatherFirstCompletedRouteLogic`: Your task is idempotent and latency critical, so have all actors deal with it and the original sender gets the first response that comes through.

In our example such a `Route` would be the agent.
If we use a `SmallestMailboxRoutingLogic` then our agent will take into account that some `JobMessages` (*acting jobs*) such as "Filming Avatar" take longer than filming a Kellogg's commercial.
Faster actors with a shorter mailbox will thus receive messages with a higher priority.
This should reduce the latency in a system with variable task runtime.

Using the `ScatterGatherFirstCompleteRouteLogic` makes no sense for individual human actors.
But think about an actor being a production company.
A client such as Kellogg's will put in a request for a finished ad, and multiple `ProductionCompanyActors` will get the task.
Whoever finishes first will see his ad aired at the Super Bowl halftime show.
Though fairly inefficient from a resource perspective, this promises the lowest latency for the original sender of the messages.

Akka makes it possible to configure such `Routes` in a configuration file, which allows you to adapt to changes in latency requirements with a simple change in configuration.

# The Take-away

We briefly touched on three elements of the **Actor Model** I'd like you take into consideration the next time you have to build concurrent system:

*   Isolate concurrency to the smallest possible unit and rely on immutable messages as a means of communication.
Even just translating *immutable messages* into other languages like Java will help you in the future.

*   Think about your concurrent sections of code as little, idempotent tasks that could be just restarted or dropped if need be. Rolling back broken data is not necessary if you can retry the operation multiple times.

*   Realise that sometimes, doing a little extra work is not waste but has real user value. We will got to great lengths when trying to *get the performance just right*. The beauty of the `Routes` is that we can embrace spilling some resources for sake of overall performance. Furthermore, it expemplifies that have a clean abstraction, such as the `Actor` and `ActorRef` allow us to slot in such routing mechanisms wihtout a lot of fuzz.

# Reading material

I recommend everyone to take a little time and study the **Actor Model**.
If Akka and the JVM are not your thing, there is a whole buffet of frameworks for different languages ([Pulsar][2] for Python and [Puslar][3] forClojure, [CAF][4] for C++ or [Celluloid][5] for Ruby).
For those who like to work on the classics first, I advise you to look into [Erlang][6] and [Elixir][7] first.
Finally, for the academicians among you that want to read up on the underlying theory, have a look at Carl Hewitts papers from [1976][8] and [1977][9].

 [1]: http://akka.io/
 [2]: http://pythonhosted.org/pulsar/
 [3]: https://github.com/puniverse/pulsar
 [4]: http://actor-framework.org/
 [5]: https://celluloid.io/
 [6]: http://www.erlang.org/
 [7]: http://elixir-lang.org/
 [8]: https://www.cypherpunks.to/erights/history/actors/AIM-410.pdf
 [9]: http://publications.csail.mit.edu/lcs/pubs/pdf/MIT-LCS-TR-194.pdf
