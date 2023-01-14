Chapter 1. Getting Started with Domain-Driven Design
----------------------------------------------------

So what is all the fuss about? If you've already read books on this topic by Vaughn Vernon and Eric Evans, you're probably familiar with what we're about to say, as we borrow heavily from their definitions and explanations. **Domain-Driven Design** (**DDD**), is an approach that helps us succeed in understanding and building software model designs. It provides us with _strategic_ and _tactical_ modeling tools to aid designing high-quality software that meets our business goals.

### Note

The main goal of this book is to show you PHP code examples of the Domain-Driven Design tactical patterns. If you want to learn more about the strategic patterns and the main Domain-Driven Design, you should read [Domain Driven Design Distilled](https://www.amazon.com/Domain-Driven-Design-Distilled-Vaughn-Vernon/dp/0134434420) by _Vaughn Vernon_ or [Domain-Driven Design Reference: Definitions and Pattern Summaries](https://www.amazon.com/Domain-Driven-Design-Reference-Definitions-Summaries/dp/1457501198) by _Eric Evans_.

More importantly, _Domain-Driven Design is not about technology_. Instead, it's about developing knowledge around business and using technology to provide value. Only once you're capable of understanding the business your company works within will you be able to participate in the software model discovery process to produce a Ubiquitous Language.


Why Domain-Driven Design Matters
--------------------------------

* * *

Software is not just about code. If you think about it, code is rarely the end goal of our profession. Code is just the medium to solve business problems. So why does it have to talk a different language? Domain-Driven Design emphasizes making sure businesses and software speak the same language. Once broken the barrier, there is no need for translations or tedious syncing, information doesn't get lost. Everyone contributes to discovering the Business Domain, not just coders. The resulting software is the only truth for the common language.

Domain-Driven Design it also provides a framework for strategic and tactical design — strategic to pinpoint the most important areas to develop based on business value, and tactical to build a working Domain Model of battle-tested building blocks and patterns.


The Three Pillars of Domain-Driven Design
-----------------------------------------

* * *

Domain-Driven Design is an approach for delivering software, and it's focused on three pillars:

1.  **Ubiquitous Language**: Domain Experts and software developers work together to build a common language for the business areas being developed. There's no _us versus them_; it's always _us_. Developing software is a business investment and not just a cost. The effort involved in building the Ubiquitous Language helps spread deep Domain insight among all team members.
2.  **Strategic Design**: Domain-Driven Design addresses the strategy behind the direction of the business and not just the technical aspects. It helps define the internal relationships and early warning feedback systems. On the technical side, strategic design protects each business service by providing the motivation for how an service-oriented architecture should be achieved.
3.  **Tactical Design**: Domain-Driven Design provides the tools and the building blocks for iterative software deliverable. Tactical design tools produce software that is not only correct, but that is also testable and less error prone.

### Ubiquitous Language

Along with  [Chapter 12](../chapters/12%20Integrating%20Bounded%20Contexts.md), _Integrating_ _Bounded Contexts_, Ubiquitous Language is one of the main strengths of Domain-Driven Design.

### Note

**`In Terms of Context`** For now, consider that a Bounded Context is a conceptual boundary around a system. The Ubiquitous Language inside a boundary has a specific contextual meaning. Concepts outside of this context can have different meanings.

So, how to find, explore and capture this very special language, the following pointers would highlight the same:

*   Identify key business processes, their inputs, and their outputs
*   Create a glossary of terms and definitions
*   Capture important software concepts with some kind of documentation
*   Share and expand upon the collected knowledge with the rest of the team (Developers and Domain Experts)

Since Domain-Driven Design was born, new techniques for improving the process of building the Ubiquitous Language have emerged. The most important one, which is used regularly now, is Event Storming.

#### Event Storming

Alberto Brandolini explains Event Storming and its advantages in a [blog post](http://ziobrando.blogspot.com.es/2013/11/introducing-event-storming.html), and he does it far more succinctly than we could.Event Storming is a workshop format for quickly exploring complex business domains:

*   It is **powerful**: It has allowed me and many practitioners to come up with a comprehensive model of a complete business flow in hours instead of weeks.
*   It is **engaging**: The whole idea is to bring people with the questions and people who know the answer in the same room and to build a model together.
*   It is **efficient**: The resulting model is perfectly aligned with a Domain-Driven Design implementation style (particularly fitting an Event Sourcing approach), and allows for a quick determination of Context and Aggregate boundaries.
*   It is **easy**: The notation is ultra-simple. No complex UML that might cut off participants from the heart of the discussion.
*   It is **fun**: I always had a great time leading the workshops, people are energized and deliver more than they expected. The right questions arise, and the atmosphere is the right one.

If you want to know more about Event Storming, check out Brandolini's book, [Introducing EventStorming](https://leanpub.com/introducing_eventstorming).


Considering Domain-Driven Design
--------------------------------

* * *

Domain-Driven Design is not a silver bullet; as with everything in software, it depends on the context. As a rule of thumb, use it to simplify your Domain, but never to add more complexity.

If your application is data-centric and your use cases mainly manipulate rows in a database and perform CRUD operations — that is, Create, Read, Update, and Delete — you don't need Domain-Driven Design. Instead, the only thing your company needs is a fancy face in front of your database.

If your application has less than 30 use cases, it might be simpler to use a framework like Symfony or Laravel to handle your business logic.

However, if your application has more than 30 use cases, your system may be moving toward the dreaded [Big Ball of Mud](https://en.wikipedia.org/wiki/Big_ball_of_mud). If you know for sure your system will grow in complexity, you should consider using Domain-Driven Design to fight that complexity.

If you know your application is going to grow and is likely to change often, Domain-Driven Design will definitely help in managing the complexity and refactoring your model over time.

If you don't understand the Domain you're working on because it's new and nobody has invested in a solution before, this might mean it's complex enough for you to start applying Domain-Driven Design. In this case, you'll need to work closely with Domain Experts to get the models right.


The Tricky Parts
----------------

* * *

Applying Domain-Driven Design is not easy. It requires time and effort to get around the Business Domain, terminology, research, and collaboration with Domain Experts rather than coding jargon. You'll need to have the commitment of Domain Experts for getting involved in the process too. This will requires an open and healthy continuous conversation to model their spoken language into software. On top of that, we'll have to make an effort to avoid thinking technically, to think seriously about the behavior of objects and the Ubiquitous Language first.


Strategical Overview
--------------------

* * *

In order to provide a general overview of the strategical side of Domain-Driven Design, we'll use an approach from _Jimmy Nilsson's_ book, [Applying Domain-Driven Design and Patterns](https://www.amazon.com/Applying-Domain-Driven-Design-Patterns-Examples/dp/0321268202). Consider two different spaces: the problem space and the solution space.

In the problem space, Domain-Driven Design uses Domains and Subdomains to group and organize what companies want to solve. In the case of an **Online Travel Agency** (**OTA**), the problem is about dealing with things like flight tickets and booking hotels. Such a Domain can be organized into different Subdomains such as Pricing, Inventory, User Management, and so on.

In the solution space, Domain-Driven Design provides two patterns: Bounded Contexts and Context Maps. The goal is to define how to provide an implementation to all the identified Subdomains by defining their interactions and the details of those interactions. Continuing with the OTA example, each of the Subdomains will be solved with a Bounded Context implementation — for example, consider a custom Web Application developed by a team for the Pricing Management Subdomain, and an off-the-shelf solution for the User Management Subdomain. The Context Map will show how each Bounded Context is related to the rest. Inside the Context Map, we can see what type of relation two Bounded Contexts have (example: customer-supplier, partners). The ideal approach is to have each Subdomain implemented by one Bounded Context, but that's not always possible. In terms of implementation, when following Domain-Driven Design, you'll end up with distributed architectures. As you may already know, distributed architectures are more complex than monolithic ones, so why is this approach interesting, especially for big and complex companies? Is it really worth it? Well, it is.

Distributed architectures are proven to increase overall company productivity because they define boundaries for your product that can be developed by focused teams.

If your Domain — the problem you need to solve  — is not complex, applying the strategical part of Domain-Driven Design can add unnecessary overhead and slow down your development speed.

If you want to know more about the strategical part of Domain-Driven Design, you should take a look at the first three chapters of _Vaughn Vernon's_ book, [Implementing Domain-Driven Design](http://www.amazon.com/Implementing-Domain-Driven-Design-Vaughn-Vernon-ebook/dp/B00BCLEBN8), or the book [Domain-Driven Design: Tackling Complexity in the Heart of Software](http://www.amazon.com/Domain-Driven-Design-Tackling-Complexity-Software/dp/0321125215)) by _Eric Evans_, both of which specifically focus on this aspect.


Related Movements: Microservices and Self-Contained Systems
-----------------------------------------------------------

* * *

There are other movements promoting architectures that follow the same principles Domain-Driven Design is promoting. Microservices and Self-Contained Systems are good examples of this. _James Lewis_ and _Martin Fowler_ define Microservices in the [Microservices Resource Guide](http://martinfowler.com/microservices/):

The Microservice architectural style is an approach to developing a single application as a suite of small services, each running in its own process and communicating with lightweight mechanisms, often an HTTP resource API. These services are built around business capabilities and are also independently deployable using fully automated machinery. There is a bare minimum of centralized management of these services, which may be written in different programming languages and also use different data storage technologies.

If you want to know more about Microservices, their guide is a good place to start.How is this related to Domain-Driven Design? As explained in _Sam Newman's_ book, [Building Microservices](http://www.amazon.com/Building-Microservices-Sam-Newman/dp/1491950358), Microservices are implementations of Domain-Driven Design Bounded Contexts.

In addition to Microservices, another related movement is **Self-Contained Systems** (**SCS**). According to the [Self-Contained Systems](http://scs-architecture.org) website:

The Self-contained System approach is an architecture that focuses on a separation of the functionality into many independent systems, making the complete logical system a collaboration of many smaller software systems. This avoids the problem of large monoliths that grow constantly and eventually become unmaintainable. Over the past few years, we have seen its benefits in many mid-sized and large-scale projects. The idea is to break a large system apart into several smaller self-contained system, or SCSs, that follow certain rules.

The website also spells out seven characteristics of SCS:

Each SCS is an autonomous web application. For the SCS's domain all data, the logic to process that data and all code to render the web interface is contained within the SCS. An SCS can fulfill its primary use cases on its own, without having to rely on other systems being available.

Each SCS is owned by one team. This does not necessarily mean that only one team might change the code, but the owning team has the final say on what goes into the code base, for example by merging pull-requests.

Communication with other SCSs or 3rd party systems is asynchronous wherever possible. Specifically, other SCSs or external systems should not be accessed synchronously within the SCS's own request/response cycle. This decouples the systems, reduces the effects of failure, and thus supports autonomy. The goal is decoupling concerning time: An SCS should work even if other SCSs are temporarily offline. This can be achieved even if the communication on the technical level is synchronous, example by replicating data or buffering requests.

An SCS can have an optional service API. Because the SCS has its own web UI it can interact with the user — without going through a UI service. However, an API for mobile clients or for other SCSs might still be useful.

Each SCS must include data and logic. To really implement any meaningful features both are needed. An SCS should implement features by itself and must therefore include both.

An SCS should make its features usable to end-users by its own UI. Therefore the SCS should have no shared UI with other SCSs. SCSs might still have links to each other. However, asynchronous integration means that the SCS should still work even if the UI of another SCS is not available. To avoid tight coupling an SCS should share no business code with other SCSs. It might be fine to create a pull-request for an SCS or use common libraries, example: database drivers or oAuth clients.

### Note

**`Exercise`** Discuss the pros and cons of such distributed architectures with your workmates. Think about using different languages, deployment processes, infrastructure responsibilities, and so on.


Wrap-Up
-------

* * *

During this chapter you've learned:

*   Domain-Driven Design is not about technology; it's actually about providing value in the field you're working in by focusing on the model. Everyone takes part in the process of discovering the Domain, and developers and Domain Experts team up to build the knowledge base by sharing the same language, the Ubiquitous Language.
*   Domain-Driven Design provides tactical and strategic modeling tools to design high-quality software. Strategic design targets the business direction, helps in defining the internal relationships, and technically protects each business service by defining strong boundaries. Tactical design provides useful building blocks for iterative design.
*   Domain-Driven Design only makes sense in certain contexts. It's not a silver bullet for every problem in software, so whether or not you use it highly depends on the amount of complexity you're dealing with.
*   Domain-Driven Design is a long-term investment; it requires active effort. Domain Experts will be required to collaborate closely with developers, and developers will have to think in terms of the business. In the end, the business customer is the one who has to be pleased.

Implementing Domain-Driven Design requires effort. If it were easy, everybody would be writing high-quality code. Get ready, because you'll soon learn how to write code that, when read, will perfectly describe the business your company operates on. Enjoy this journey!