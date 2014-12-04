# Titles
* Akka - a look at the Actor Paradigm
* Acting up with Akka
* 

* possibly add a nice title image over the article

# Structure
* Give CONTEXT
    * What is the actor paradigm?
          * Historic context?
    * What is Akka
    * What will I talk about in this blog
    * Set the tone. Funny? Serious? Choose words carefull!

* The basics of the Actor paradigm
    * Actors,
        * Only unit of work
        * Stateless
        * The only sideeffect should be messages sent to other actors (aside from obvious exceptions at the boundaries of systesm, e.g. DB)
        * Should not contain long, blocking tasks
    * Messages
        * Have to be immutable. In Scala use CaseObjects, in Java be careful!
        * Only form of communication between systems
    * The ActorSystem

* Decoupling, Scaling and Configuration
    * Akkas 'receive' method on an Actor is as untyped as it gets
        * Any Actor can receive any message
        * They are rather indirectly coupled to one another.
            Sending the wrong mesage will not lead to a compile time error, but it wont have the 
            desired effect either.
        * Messages  
