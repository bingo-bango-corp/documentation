## It works, but it's nowhere near perfect

What did I want to make? A fully functional web service. Did I do that? Yeah. Is it perfect? Not at all.

It rarely makes sense to build a jewel, perfect application right away. What I learned time and time again at work is that continous delivery usually wins against attempting to build something perfect for months at a time. The challenge with this approach however is: What should I build first?

In the case of Bingo Bango, the core functionality is pretty straight-forward. During the process, I tried to think in a minimum viable way and avoid bloating up the project too much. However, there is a vast amount of potential upcoming features and improvements to be made.

I also couldn't avoid introducing some early tech debt into the project:

- Types are not shared across backend and frontend, instead being duplicated.
- There is a high amount of code duplication in state machine actions.
- The frontend application has a little bit too much logic in it that I would like to decouple into the backend.

In the future, I am going to solve all of these issues.

We are actually aiming to launch this thing out to the public at some point in the future. So, while this is a major milestone, it's just the very beginning.