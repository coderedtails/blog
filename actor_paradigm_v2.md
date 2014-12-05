# Looking at concurrency through the lens of Actors

Concurrency has become more and more relevant over the last few years.
Smartphones makers have continuously increased the number of cores in their package,
firmly cementing _multitasking_ and _multi-processing_ in our everyday life.
I may only be 25, but my smartphone has more processing power and four times as many cores as first Pentium IV 2.6Ghz.

From a programmers perspective, concurrency is one the hardest things to programm for.
The challenge lies both in making the most of the cores available to us and to make sure
that programs do not crash or corrupt our users data just because we did not anticipate
the octo-core that he has.

Inherently, concurrency is hard to reason about. We humans are not as multitasking capable as we think we are.
Listening to musik while you browse the web is not multi-tasking. Even though you can probably hum along to your
Spotify playlist, you are not concentrating on it. Its pure 'audible' memory.
