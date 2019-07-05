# Prototyping vs. building production-grade software

From the very get go, I set out not only to develop a concept for a potential service, but instead _build_ the service _for real_.

That means that in addition to just designing and coming up with the concept itself, I had to think about a huge amount of different challenges. From deciding on a suitable tech stack, to ensuring security with real users, to authenticating and validating user credentials — all those things don't exactly matter in a prototyping context. But if a piece of software is inteded to launch and be used out there, by real people, treating all of these challenges lightly will not only result in a service horribly failing, but can even put you in jail for leaking personal information.

In addition, a prototype runs in a very controlled environment and is not publicly accessible. And its goal is not to be used — its goal is to serve as a tool in testing and validating ideas. As a result of that, you don't need to think about edge cases or test and assure quality in any possible context. When building a real, production-ready application, you need to think about _exactly these_ edge cases. What happens if the user looses the connection in this very specific moment? Can my users perform an action a million times, resulting in insane costs for running the service? Could a user somehow "hack" the service by exploiting weak validations? Or maybe, could a user even cause harm to another user through one of these loopholes?

Real services are used by real people. That comes with a great responsibility for you as their designer and developer. Shoddy implementations can result in grave consequences, even if they are unlikely or don't seem all that grave from your point of view. Maybe some "rare" edge case doesn't seem worth fixing — but for that one user that is affected, unexpected behaviour of the service in the context of a large transaction might cause a mental breakdown.

In practice, what this meant for me: _My code had to be bullet-proof_. As a Product person who really is nothing more than a hobby programmer, this was a great challenge, and I learned a lot. I kept implementing functionality in that typical prototype-like fashion, just for one of my hardcore backend engineer colleagues to rip it to absolute shreds due to concerns I would not have had in my wildest dreams. Writing to a database as a result of a user interaction? Better make sure you _write-lock that document_ so that no other user could perform that same action in the same moment. Would this ever happen? It's highly unlikely. Can it? Most definitely. And if it does, you've got an unprocessable entity in your system, the clients most likely just crash and you probably just lost both of those users forever. 

# Designing a service

In _Service Thinking_, I write in detail about a service being more than just what the user sees. In fact, the underlying logic is what enables the service to function at all. In a standard API <> Client model, the user interface is nothing more than … a _client_. The _service_, really, is the centralized business logic that results from choreographed functionality running behind the scenes, exchanging data with potentially thousands of clients constantly and concurrently.

And so, at first, I set out to design a rough user journey. What I wanted to understand is exactly how a user will interact with a service, how they request items, and how those requests transition through different states during their lifecycle.

## Jobs, Actions and Job Relationships

Based on the idea, I established some terminology for myself.

- A __Job__ is an entity that can be created by users, holding information such as the user's location, a description of what they want, as well as the amount they want to tip.
- A __Job Owner__ is the user that created a specific job.
- An __Assignee__ is a user that __took__ (assigned themselves to) any given available job.

Next, I imagined how users interact with the app and drafted a number of __states__ that jobs can be in. The state represents the job's current position in its lifecycle — from being newly created to being delivered.

- First, a user creates a new job, indicating what they want, where they want it and how much they want to tip. The job is now in state __unassigned__.
- Users can see all unassigned jobs in their vincinity and may decide to assign themselves to one. As soon as a user assigns themselves to a job, its state becomes __assigned__.
- At any point during the lifecycle of a job, the owner may choose to cancel the job. This results in the job moving to the state __cancelled__.
- While assigned to a job, the assignee may _drop_ the job when they realize they actually cannot pick it up. This is akin to the owner cancelling it, and causes the job to move to the state __lost__.
- While assigned to a job, the assignee may _indicate delivery_. This moves the job to state __delivered__.
- The job's owner must now _confirm delivery_ in order to be ensure that assignees don't abuse the system. This action transitions the job to __delivery_confirmed__.

Resulting from this, I had a number of _open states_ and _terminal states_. Open states are those from which the job may transition to another state. Terminal states however are _outcomes_, meaning in those states, the job can not transition to any others. Thus, there had to be three _terminal states_: Delivery confirmed, cancelled and lost.

In a normal lifecycle, the job thus traverses through states like this: 

unassigned -> assigned -> delivered -> delivery_confirmed.

And at any point, it can go to `lost` or `cancelled`. Next, I established wording for the _transitions_ between those states in order to validate them.

- unassigned -> assigned: `assign`
- assigned -> delivered: `deliver`
- delivered -> delivery_confirmed: `confirm_delivery`
- * -> cancelled: `cancel`
- * -> lost: `drop`

Now, I was able to establish a rudimentary "permission system" — any of these actions may only be performed by one of the two parties interacting with a job:

- Assignee may `assign`.
- Assignee may `deliver`.
- Owner may `confirm_delivery`.
- Owner may `cancel`.
- Assignee may `drop`.


<img src='https://raw.githubusercontent.com/bingo-bango-corp/documentation/master/assets/State%20Machine.png' width=500>

_In the above illustrations, owner actions are represented in cyan, and assignee actions are red._


Later on, I encoded these definitions into a *state machine*. What that is, what it means, and how it works — later in this documentation. Stay tuned. Like and subscribe.

### Communication

Bingo Bango, as a community-driven platform, of course needs a way for owner and assignee to communicate in real-time over the course of the job's lifecycle. Thus, there had to be a chat attached to every job. My understanding of the "job view" at this point entailed a way to trigger the transitions available in the current context, as well as communicate with the other user in real-time.

At this point in the development process, I had an exact idea of what Bingo Bango _is_, what it _does_, and how it _works_. What I didn't know yet at all was how it was going to look like. Even without a finished interaction wireframe and only a very rough understanding of the client interface, I knew exactly what touch points and functionality I had to build to enable the service itself. And so the first part in the development process was building that very raw functionality itself: The Bingo Bango API.

# Designing a tech stack for hyperlocal, real-time interactions

In _Service Thinking_, I also mention how User Experience goals directly affects raw technical choices regarding the technology stack. One of the examples I gave was Google Docs, which is a product priding itself on its real-time processing capabilities. Interestingly, the real-time, hyperlocal aspect of Bingo Bango was one of the major decision influences when I chose the tech stack for it too.

I had no special requirements for my tech stack regarding the API. It had to be an API. It had to work. Nothing crazy. Any solution for hosting APIs out there in the market would do.

On the database side of things however, things became a little more interesting. I needed something that works with a large amount of data, quickly, and in real-time. I needed it to enable anything from a live chat to updating clients immediately as a job's state changed. And I needed something that I could run _geolocation queries_ on. That's a quite specific set of requirements.

Probably the most important requirement however was the ability to get up and running quickly. After all, I had about 3 months to build a whole service. The tech stack didn't just need to work and be viable, it also needed to be _easy to use_, especially since I am by no means a professional software engineer.


<img src='https://raw.githubusercontent.com/bingo-bango-corp/documentation/master/assets/Architecture.png' width=500>


After evaluating countless options, I decided to go with **Google Cloud** and **Firebase**. Google Cloud is one of the largest cloud infrastructure providers, used by the likes of Spotify and Netflix. Firebase is another Google service that wraps around some more advanced Google Cloud features and provides an easier way to get started, through beginner-friendly client APIs for services. It also offers an extremely easy to use authentication platform that makes integrating social logins a breeze. The biggest reason for using Google Cloud however: _Cloud Firestore_.

Firestore is a real-time noSQL database. That means: You can interact with documents in collections. And you can _subscribe_ to collections — and every time the data changes, every subscribed clients get notified immediately. This matched my "real-time" requirement perfectly.

## Modelling data

Now I moved on to the fun part: Finding the optimal data structure for solving my needs. When it comes to data modelling, you have to think about not only a structure that allows your clients to easily retrieve the data they need, but also about _how many reads_ and _how many writes_ are needed in order to perform frequent actions. As a noSQL database solution, Firestore bills by _every document retrieved_. Meaning if you run a query and it matches 250 jobs, you pay for 250 _reads_. That means: Data needs to be structured in a way that minimizes the amount of reads and writes the application uses. 

I already knew I had two types of entities in my system: `job` and `user`, and I already had a pretty good understanding of their attributes, so I went ahead and defined initial models for both of them.

A user:
- Stores the user's unique identifier. `uid`.
- Stores an optional URL to a profile picture. `photoURL`.
- Stores a freely editable display name. `displayName`.

However, I also had to store some sensitive information. I knew that clients would need to read non-sensitive information about other users, but only a user themselves may access their private information. For this reason, I added a `privateProfile` _subcollection_ on each _user_.

A privateProfile:
- Stores the real given name of the user. `accountName`.
- Stores the user's email address. `email`.
- Stores a user's phone number. `phoneNumber`.
- Stores the user's `messageToken`. This is a hashed identifier used to deliver push notifications. More on that later.

Now that the database knew about users, I moved on to jobs.

A job: 
- Stores a description of the requested item. `thing`.
- Stores the user's location. `point`.
- Stores an additional location description. `description`.
- Stores its state. `state`.
- Stores, if assigned, a reference to the `user` that is assigned. `assignee`.
- Stores a reference to the `user` that created the job. `owner`.
- Stores a timestamp of when it was created. `timestamp`.
- Stores the tip amount the owner selected. `tip`.

However, I also knew I had to provide real-time communication as part of each job. So, I added a subcollection to each job, called `chat`. Inside of this collection, each individual message is stored.

A message:
- Stores a reference to the user that sent it. `created_by`.
- Stores the actual message text. `message`.
- Stores a timestamp of when it was sent. `seconds`.

In addition, I had to find a way to store if one of the users is currently typing. For that reason, I governed another type of document inside the `chat` subcollection called `typing`. In this document, a client may simply write their user's user id, with a boolean value. If the value is `true`, the user is typing.

### Finding nearby jobs

Now that I knew roughly what entities I needed and what they looked like, I had to decide on some data formats. The biggest challenge: location.

Initially, I stored location as coordinates — latitude and longitude. I quickly realized this was a dead end, because my main use case for jobs was _finding all jobs within a certain radius of a user_. With the query types provided by Firestore, it is impossible query on lat and long directly, meaning the only possible way would have been to retrieve _all_ jobs and then perform complicated client-side arithmetic to filter those jobs outside a certain radius. Which was not an acceptable way of solving this.

When researching solutions, I stumbled upon the brilliant idea of [Geohashing](https://en.wikipedia.org/wiki/Geohash). With geohashing, you divide the world into 31 squares — one for each letter in the alphabet, plus the numbers 0 through 9. Each square is then divided into another 31 equal pieces. Each square is then divided into another 31 equal pieces. Each square is then divided into another 31 equal pieces. And so on and so forth.

For example, the cafe I'm sitting in right now has the geohash `u33d8wqe8jx3`. What's cool about this is that you can now take away letters from the end of the geohash, effectively reducing its _precision_. And what's even cooler is that you can write a query for Firestore that returns every `job` where the geohash `begins with` an arbitrary string of letters. And what's _even cooler_ is that someone else went through all of this before me and wrote an [open source library to solve this exact problem](https://github.com/codediodeio/geofirex). So, all I had to do was geohash a device's location and query Firestore accordingly, and the problem was solved: I could now find every job in a user's vincinity. Neat.

## Designing the API

Now, I had my data model and I knew what states my jobs could be in. So, the next logical steps was to implement the above-mentioned `transitions`, or `actions` that the user can perform on a job in a given context.

I chose to use _Google Cloud Functions_. As a typical serverless application environment, Cloud Functions allows you to define functionality and interact with that functionality from other places without worrying about server infrastructure and routing. As I was already using Firestore, which is a Google Cloud product, it was only natural to go with Google Cloud Functions too.

First, I defined what my actions actually _had to do_.

- `assign` should write the assignee to the job, set the job state to `assigned` and notify the owner.
- `deliver` should set the job state to `delievered` and notify the owner.
- `confirm_delivery` should set the job state to `delivery_confirmed` and notify the assignee.
- `cancel` should set the job state to `cancelled` and notify, if there is one, the assignee.
- `drop` should set the job state to `lost` and notify the owner.

Now I realized something: I need to keep track of these transitions if I want to properly understand, record and display to the user a history of each respective job. So, I decided to inject `event` objects into the `chat` subcollection of the job document. This turned out to be a great decision because it was now trivial to display a full, ordered history of both messages and events like "You marked this as delivered" to the user on a job view.

As I learned over and over again at work, _a real state machine validates transitions_. That means that it won't allow a job to go from a terminal state such as `cancelled` to `delivered` for example, in accordance with business logic. This is the only way to ensure that if a client for whatever reason behaves unexpectedly, it can not lead to an _unprocessable entity_ — a database entry that cannot be explained. For this reason, I added validations on each action, ensuring that the job is allowed to actually transition in the requested way, and that the user that triggered the action is actually allowed to do so.

# The light side

It was time to start working on the client application.

Again, first up, I had to make a decision on what technology I want to work with for "the app". My biggest competence is a JavaScript framework called Vue, so I naturally gravitated towards using it. However, I still strongly considered native technologies, or a cross-platform framework like Flutter.

The web is no longer what it used to be — more and more "app-like" functionality is enabled constantly by new APIs being standardized, and operating systems allowing deeper integration with their system for websites. A lot, probably most, usecases can these days be handled by a web application, with almost native feel. Both on iOS and Android, it's now even possible to install so-called "PWAs" (Progressive Web Applications) straight onto the system, offering behavior very similar to a native app. Websites can even work offline and send push notifications as well as access many device APIs such as geolocation.

Because of this and my highly limited time budget, I decided to implement a PWA in Vue. However, because I always want to learn something new with every new project, I went with TypeScript on top, which is a transpiler that enriches JavaScript with strongly-typed syntax and fantastic tooling.

## Back to the drawing board

This is the point at which some good old UI & UX Design comes into play.

My colleague and dear friend Brandon helped me immensely with designing the UI for Bingo Bango, and I'd like to thank him a lot for that!

What became very clear very early on is that Bingo Bango was going to have 3 main views: Make Money, Get Things and You. Make Money is where you make money. Get Things is where you get things. And You is a profile settings kind of screen. By splitting the application into Make Money and Get Things, the top level views are immediately straight to the point. Those are our two usecases: Deliver things for others and get tips, or request things and have them delivered. 

What also became very clear very early on is that Bingo Bango does not take itself all that seriously. The visual style may be described as a little bit crazy, it makes heavy use of Emojis for UI elements, and the "logo" is a crude drawing of a wizard that we laughed tears at when Brandon first sketched it in a few minutes. The service is not supposed to be serious or sleek — instead, it presents itself as invitingly nonchalant and fun.

## Implementing a design system

While Brandon was sketching out some screens, I decided to set up infrastructure that allows sharing UI components between different applications. I described a "design system" in Service Thinking already — a library of components that can be re-used globally. Bingo Bango's design system is [Simsalabim Design](https://github.com/bingo-bango-corp/simsalabim-design).

In Simsalabim, I also implemented a [simple Design Token system](https://github.com/bingo-bango-corp/simsalabim-design/blob/master/src/components/ThemeProvider/themes.ts). Using the [Theme Provider component](https://github.com/bingo-bango-corp/simsalabim-design/blob/master/src/components/ThemeProvider/ThemeProvider.vue), a number of theming CSS variables can be injected into anything nested inside. That made it a breeze to offer the three themes Bingo Bango has — Dark, Light, and Insane.

## Setting up the route structure

For routes, I chose a pretty crazy approach. The first component I built, even before settling on top level routes, was the [BottomNav](https://github.com/bingo-bango-corp/simsalabim-design/blob/master/src/components/BottomNav/BottomNav.vue), and with it, I defined the [BingoRoute interface](https://github.com/bingo-bango-corp/simsalabim-design/blob/master/src/components/BottomNav/interfaces.ts). BottomNav simply parses a `routes` object passed to it as a prop and displays top level navigation targets automatically. It even allows passing an instance of `i18n-vue` for translations. This way, all that needs to be done in the parent application consuming this component is to pass the correctly configured `routes` object - which of course can be passed as-is to `vue-router` as well.

After building BottomNav, I set up some skeleton routes and [configured vue-router](https://github.com/bingo-bango-corp/app/blob/master/src/router.ts). Naturally, my next step was setting up the authentication system.

## Handling Authentication and Permissions

As already mentioned, I chose Firebase Authentication as my authentication provider, due to its ease of use and support for all major oAuth services. Implementing it in Vue was pretty easy. Naturally, I first [set up Firebase initialization](https://github.com/bingo-bango-corp/app/blob/master/src/initFirebase.ts). After that, I changed my `main.ts` so that after Firebase is initialized, a callback from `onAuthStateChanged` is awaited [before the Vue app itself gets initialized](https://github.com/bingo-bango-corp/app/blob/master/src/main.ts). This way, I was sure that the app always knows its auth state before any route guards run.

Now, I set up a simple sync of my authentication data with the `vuex` store. This way, anywhere in the application components may access profile information of the currently logged-in user from a centralized location. This store is immediately populated with Firebase Authentication information right in the `initializeApp` function.

The natural next step was [implementing route guards](https://github.com/bingo-bango-corp/app/blob/master/src/router.ts#L109) so that non logged-in users would simply be forwarded to the login screen. Here, I realized another challenge: Making sure that the app has all permissions that it requires to run. Bingo Bango in order to work properly needs location and notification permissions. There was little sense in allowing the app to run without those, so we decided to require users to grant permissions in a blocking manner. For this, I implemented a [permissions vuex module](https://github.com/bingo-bango-corp/app/blob/master/src/store/modules/permissions.ts) that keeps track of permissions and also handles asking for them. Together with the [permissions view](https://github.com/bingo-bango-corp/app/blob/master/src/views/permissions/permissions.vue) and the [appropriate route guards](https://github.com/bingo-bango-corp/app/blob/master/src/router.ts#L109), the app now automatically asked for missing permissions.

## Representing Jobs and Job States

One of the most interesting challenges with this frontend application was handling jobs. Depending on your relationship with the job (owner or assignee), as well as the current state of the job, we wanted to display different information as well as allow different actions.

Fist, I implemented a [generic, almost purely representational `JobCard` in Simsalabim](https://github.com/bingo-bango-corp/simsalabim-design/blob/master/src/components/JobCard/JobCard.vue). JobCard consumes an array of `BingoActions`. Each action corresponds to a button shown on the card and includes representational configuration, such as the button's title, its color and the `onClick` handler. When the action is clicked, the `onClick` handler for this action is emitted with an event to the parent component. This way, the parent can take care of calling actions, and the `JobCard` stays representational.

In the app itself, I then wrapped this component into a [new one called `JobCardWithActions`](https://github.com/bingo-bango-corp/app/blob/master/src/components/JobCardWithActions/JobCardWithActions.vue). Depending on if the user is the owner or assignee for a given job, this component maps [actions](https://github.com/bingo-bango-corp/app/blob/master/src/components/JobCardWithActions/assigneeActionsForStates.ts) and [properties](https://github.com/bingo-bango-corp/app/blob/master/src/components/JobCardWithActions/assigneePropsForStates.ts) for each possible `state`. After writing this component, I could drop it in at any possible point in the app and it just worked - always displaying exactly the right information and actions.

## The Chat View

Building a real-time chat definitely wasn't easy. But it was certainly fun and I learned a lot.

The [Chat component](https://github.com/bingo-bango-corp/app/blob/master/src/components/Chat/Chat.vue) takes an ID of a job (as chats are always connected with a job) and then initializes a realtime connection to the `chat` collection. There are a number of utility functions in the Chat component, such as [otherPersonsRole](https://github.com/bingo-bango-corp/app/blob/master/src/components/Chat/Chat.vue#L102). Using these, the component renders a `ChatMessage` for each message in the `chat` collection.

One interesting thing was implementing the "typing" indicator. As soon as the user starts typing, the Chat component sets a property named after their user id in a document called `typing` to `true`. Both clients write to this same document, and both have two-way data binding set up with it. Then, a [simple debounce mechanism](https://github.com/bingo-bango-corp/app/blob/master/src/components/Chat/Chat.vue#L131) sets typing to false after a few seconds of inactivity. With this approach, displaying the typing indicator becomes just a matter of a simple conditional render.