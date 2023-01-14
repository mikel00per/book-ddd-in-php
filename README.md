
# Domain Driven Design in PHP

Libro de Carlos Buenosvinos, comprado y añadido aquí para consultas rápidas.

---
## Table of Contents
* [Chapter 1. Getting Started with Domain-Driven Design](#chapter-1-getting-started-with-domain-driven-design)
* [Chapter 2. Architectural Styles](#chapter-2-architectural-styles)
* [Chapter 3. Value Objects](#chapter-3-value-objects)
* [Chapter 4. Entities](#chapter-4-entities)
* [Chapter 5. Services](#chapter-5-services)
* [Chapter 6. Domain-Events](#chapter-6-domain-events)
* [Chapter 7. Modules](#chapter-7-modules)
* [Chapter 8. Aggregates](#chapter-8-aggregates)
* [Chapter 9. Factories](#chapter-9-factories)
* [Chapter 10. Repositories](#chapter-10-repositories)
* [Chapter 11. Application](#chapter-11-application)
* [Chapter 12. Integrating Bounded Contexts](#chapter-12-integrating-bounded-contexts)
* [Chapter 13. Hexagonal Architecture with PHP](#chapter-13-hexagonal-architecture-with-php)
* [Chapter 14. Bibliography](#chapter-14-bibliography)

## Chapter 1. Getting Started with Domain-Driven Design

So what is all the fuss about? If you've already read books on this topic by Vaughn Vernon and Eric Evans, you're probably familiar with what we're about to say, as we borrow heavily from their definitions and explanations. Domain-Driven Design (DDD), is an approach that helps us succeed in understanding and building software model designs. It provides us with strategic and tactical modeling tools to aid designing high-quality software that meets our business goals.

[Read more on the original file...](/chapters/01%20Getting%20Started.md)

## Chapter 2. Architectural Styles

In order to be able to build complex applications, one of the key requirements is having an architectural design that fits the application's needs. One advantage of Domain-Driven Design is that it's not tied to any particular architecture style. Instead, we're free to choose the architecture that best fits the needs of every Bounded Context inside the Core Domain, which offers a diverse set of architectural choices for every specific Domain problem.

[Read more on the original file...](/chapters/02%20Architectural%20Styles.md)

## Chapter 3. Value Objects 

By using the self keyword, we don't The Value Objects are a fundamental building block of Domain-Driven Design, and they're used to model concepts of your Ubiquitous Language in code. A Value Object is not just a thing in your Domain — it measures, quantifies, or describes something. Value Objects can be seen as small, simple objects — such as money or a date range — whose equality is not based on identity, but instead on the content held.

[Read more on the original file...](https://github.com/mikel00per/book-ddd-in-php/blob/dev/chapters/03%20Value%20Objects.md)

## Chapter 4. Entities

We've talked about the benefits of trying to first model everything in the Domain as a Value Object. But when modeling the Domain, there will probably be situations where you'll find that some concept in the Ubiquitous Language demands a thread of Identity.

[Read more on the original file...](https://github.com/mikel00per/book-ddd-in-php/blob/dev/chapters/04%20Entities.md)

## Chapter 5. Services

You've already seen what Entities and Value Objects are. As basic building blocks, they should contain most of the business logic of any application. However, there are some scenarios where Entities and Value Objects aren't the best solutions. Let's take a look at what Eric Evans has to say about this in his book, Domain-Driven Design: Tackling Complexity in the Heart of Software:

[Read more on the original file...](https://github.com/mikel00per/book-ddd-in-php/blob/dev/chapters/05%20Services.md)

## Chapter 6. Domain-Events

Software Events are things that happened in the system that other components might be interested in knowing about. PHP developers are generally not used to working with Events, which are not a feature in the language. However, it's more common to see how new frameworks and libraries embrace them to provide new ways of decoupling, reusing, and speeding up code.

[Read more on the original file...](https://github.com/mikel00per/book-ddd-in-php/blob/dev/chapters/06%20Domain-Events.md)

## Chapter 7. Modules

When you place some classes together in a Module, you are telling the next developer who looks at your design to think about them together. If your model is telling a story, the Modules are chapters. Domain-Driven Design:Tackling Complexity in the Heart of Software -Eric Evans

[Read more on the original file...](https://github.com/mikel00per/book-ddd-in-php/blob/dev/chapters/07%20Modules.md)

## Chapter 8. Aggregates

Aggregates are probably the most difficult building blocks of Domain-Driven Design. They're hard to understand, and they're even harder to properly design. But don't worry; we're here to help you. However, before jumping into Aggregates, there are some key concepts we need to go through first: transactions and concurrency strategies.

[Read more on the original file...](https://github.com/mikel00per/book-ddd-in-php/blob/dev/chapters/08%20Aggregates.md)

## Chapter 9. Factories

Factories are a powerful abstraction. They help decouple the client from the details of how to interact with the Domain. The client doesn't need to know how to build complex objects and Aggregates, so you can use Factories to create whole Aggregates, thereby enforcing their invariants.

[Read more on the original file...](https://github.com/mikel00per/book-ddd-in-php/blob/dev/chapters/09%20Factories.md)

## Chapter 10. Repositories

In order to interact with a Domain object, you need to hold a reference to it. One way of achieving this is by creation. Alternatively, you can traverse an association. In Object-Oriented programs, objects have links (references) to other objects, which makes them easily traversable, thereby contributing to the expressive power of our models. But here's the catch: you need a mechanism to retrieve the first object, the Aggregate Root.

[Read more on the original file...](https://github.com/mikel00per/book-ddd-in-php/blob/dev/chapters/10%20Repositories.md)

## Chapter 11. Application

The Application layer is the area that separates the Domain Model from the clients that query or change its state. Application Services are the building blocks for such a layer. As Vaughn Vernon says: "Application Services are the direct clients of the domain model." You could think about an Application Service as a point of contact between the outside world (HTML forms, API clients, the command line, frameworks, UI, and so on.) and the Domain Model itself. It might help by thinking about the top-level use cases that your system exposes to the world, example: "as guest, I want to register," "as a logged user, I want to purchase a product," and so on.

[Read more on the original file...](https://github.com/mikel00per/book-ddd-in-php/blob/dev/chapters/11%20Application.md)

## Chapter 12. Integrating Bounded Contexts

Every enterprise application is typically composed of several areas in which the company operates. Areas such as billing, inventory, shipping management, catalog, and so on are common examples. The easiest manner in which to manage all these concerns may seem to lean toward a monolithic system. But, you might wonder, does it have to be this way? What if any friction garnered between teams working on these separate areas could be reduced by splitting this big monolithic application into smaller, independent chunks? In this chapter, we'll explore how to do this, so be prepared for insights and heuristics around strategical design.

[Read more on the original file...](https://github.com/mikel00per/book-ddd-in-php/blob/dev/chapters/12%20Integrating%20Bounded%20Contexts.md)

## Chapter 13. Hexagonal Architecture with PHP

The following article was posted in php|architect magazine in June 2014 by Carlos Buenosvinos.

[Read more on the original file...](https://github.com/mikel00per/book-ddd-in-php/blob/dev/chapters/13%20Hexagonal%20Architecture%20with%20PHP.md)

## Chapter 14. Bibliography

Beck, Kent. Test-Driven Development: By Example. Addison-Wesley Professional, 2002.

[Read more on the original file...](https://github.com/mikel00per/book-ddd-in-php/blob/dev/chapters/14%20Bibliography.md)
