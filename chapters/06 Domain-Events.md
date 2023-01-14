Chapter 6. Domain-Events
------------------------

Software Events are things that happened in the system that other components might be interested in knowing about. PHP developers are generally not used to working with Events, which are not a feature in the language. However, it's more common to see how new frameworks and libraries embrace them to provide new ways of decoupling, reusing, and speeding up code.

Domain Events are Events related to Domain changes. Domain Events are things that happen in our Domain that Domain Experts care about.

In Domain-Driven Design, Domain Events are fundamental building blocks that help:

*   Communicate with other Bounded Contexts.
*   Improve performance and scalability, pushing for eventual consistency.
*   Serve as historical checkpoints.

Domain Events represent the essence of asynchronous communication. For more on this topic, we recommend the book _Enterprise Integration Patterns:_  [Designing, Building, and Deploying Messaging Solutions](http://www.amazon.com/Enterprise-Integration-Patterns-Designing-Deploying/dp/0321200683) by Gregor Hohpe and Bobby Woolf.

Introduction
------------

* * *

Think about a JavaScript 2D platform game. There are tons of different components interacting with each other on the screen, all at the same time. There's a component that indicates the number of lives remaining, another one that shows all the points scored, and another one counting down the time remaining to finish the current level. Each time your character jumps on an enemy, the points increase. When your scoring goes higher than a certain number of points, you get an extra life. When your character picks up a key, a door usually opens. But how do all these components interact with one another? What's the optimal architecture for this scenario?

There are probably two main options: the first one is to couple each component with the ones it's connected to. However, in the above example, that means a lot of components are coupled together, with each new addition requiring the developer to modify the code. But do you remember the **Open Closed Principle** (**OCP**)? Adding a new component shouldn't make it so the first component has to be updated; this would be too much work to maintain.

The second — and better — approach is to connect all the components to a single object that handles all the important Events in the game. It receives Events from each component and forwards them to specific components. For example, the scoring component would be interested in an `EnemyKilled` Event, while the `LifeCaptured` Event is quite useful for the player Entity and the remaining lives component. In this way, all components are coupled to a single component that manages all the notifications. With this approach, adding or removing components doesn't affect the existing ones.

When developing a single application, Events come in handy for decoupling components. When developing a whole Domain in a distributed way, Events are very useful for decoupling each service or application that plays a role in the Domain. The key points are the same, but on a different scale.

Definition
----------

* * *

Domain Events are one specific type of Event used for notifying local or remote Bounded Contexts of Domain changes.

Vaughn Vernon [defines](http://www.amazon.com/Implementing-Domain-Driven-Design-Vaughn-Vernon-ebook/dp/B00BCLEBN8) a Domain Event as:

An occurrence of something that happened in the domain.

Eric Evans [defines](https://domainlanguage.com/ddd/patterns/DDD_Reference_2011-01-31.pdf) a Domain Event as:

A full-fledged part of the Domain Model, a representation of something that happened in the Domain. Ignore irrelevant Domain activity while making explicit the events that the Domain Experts want to track or be notified of, or which are associated with state change in the other Model objects.

Martin Fowler [defines](http://martinfowler.com/eaaDev/DomainEvent.html) a Domain Event as something that:

Captures the memory of something interesting which affects the Domain.

Examples of Domain Events in a web application are `UserRegistered`, `OrderPlaced`, `UserRelocated`, and `ProductAdded`.

### Short Story

In a ticket sales agency, a content manager decides to increase the price of a U2 show. Using her back office, she edits the show. A `ShowPriceChanged` Domain Event is published and persisted into the database with the new show price in the same transaction.

A batch process takes the Domain Event and queues it into RabbitMQ. The Domain Event gets distributed in two queues: one for the same local Bounded Context, and another remote one for Business Intelligence purposes.

In the first queue, a worker fetches the corresponding Show using the ID in the Event and pushes it into an Elasticsearch server so that the user can see the new price when searching. It could also update the new price in a different database table.

In the second queue, a worker inserts the information into a Logs Server or a Data Lake, where reporting or Data Mining processes can be run.

An external application that can't be integrated using Domain Events could access all the `ShowPriceChanged` Events using a REST API that the local Bounded Context provides.

As you can see, Domain Events are useful for dealing with eventual consistency and integrating different Bounded Contexts. Aggregates create Events and publish them. Subscribers may store Events and then forward them to remote subscribers.

### Metaphor

We go to Babur's for a meal on Tuesday and pay by credit card. This might be modeled as an Event with an Event type of `PurchasePlaced`, a subject of my credit card, and a date of occurrence of Tuesday. If Babur's uses an old manual system and doesn't transmit the transaction until Friday, the transaction would be effective on Friday.

Things happen. Not all of them are interesting, and some may be worth recording but don't provoke a reaction. However, the most interesting things cause a reaction. Many systems need to react to interesting Events. Often you need to know why a system reacts in the way it did.

By funneling inputs to a system into streams of Domain Events, you can keep a record of all the inputs to a system. This helps you organize your processing logic, and it also allows you to keep an audit log of the inputs to the system.

### Note

**`Exercise`** Try to locate examples of potential Domain Events in your current Domain.

### Real-World Example

Before going into detail about Domain Events, let's see a real example of Domain Events and how they can help us in our application and our whole Domain.

Let's consider a simple Application Service that will register a new user — for example, in an e-commerce context. Application Services will be explained in another chapter, so don't worry too much about the interface. Instead, just focus on the execute method:

    class SignUpUserService implements ApplicationService
    {
        private $userRepository;
        private $userFactory;
        private $userTransformer;
    
        public function __construct(
            UserRepository $userRepository,
            UserFactory $userFactory,
            UserTransformer $userTransformer
        ) {
            $this->userRepository = $userRepository;
            $this->userFactory = $userFactory;
            $this->userTransformer = $userTransformer;
        }
    
        /**
         * @param SignUpUserRequest $request
         * @return User
         * @throws UserAlreadyExistsException
         */
        public function execute(SignUpUserRequest $request)
        {
            $email = $request->email();
            $password = $request->password();
    
            $user = $this->userRepository->userOfEmail($email );
            if ($user) {
                throw new UserAlreadyExistsException();
            }
    
            $user = $this->userFactory->build(
                $this->userRepository->nextIdentity(),
                $email,
                $password
            );
    
            $this->userRepository->add($user);
            $this->userTransformer->write($user);
        }
    }

As shown, the _Application Service_ section checks if the user already exists. If not, it creates a new User and adds it to the `UserRepository`.

Now consider an additional requirement: a new user must be notified by email when registered. Without thinking about it too much, the first approach that comes to mind is to update our Application Service to include a piece of code that would do the job — probably some sort of `EmailSender` that would be run after the add method. However, let's consider another approach.

What about firing a `UserRegistered` Event so that another component listening can react and send that email? There are some cool benefits to this new approach. First of all, we don't need to update the code of our Application Service every time a new action must be performed when a new user is registered.

Second, it's easier to test. The Application Service remains simpler, and each time a new action is developed, we just write the tests for the action.

Later in the same e-commerce project, we're told to integrate an open source gamification platform not written in PHP. Each time users place purchases or review products in our e-commerce Bounded Context, they can get badges that can be shown on their e-commerce user profile pages or be notified by email. How could we model the problem?

Following the first approach, we would update the Application Service to integrate with the new platform in a fashion similar to the previous email confirmation approach. With the Domain Event approach, we could create another listener for the `UserRegistered` Event, which will connect directly, by REST or SOA, to the gamification platform. Even better, it could communicate the Event to a messaging system like RabbitMQ so that the gamification Bounded Context can subscribe and get notified automatically. Our e-commerce Bounded Context doesn't need to know about the gamification Bounded Context at all.

Characteristics
---------------

* * *

Domain Events are ordinarily **immutable**, as they're a record of something in the past. In addition to a description of the Event, a Domain Event typically contains a timestamp for the time the Event occurred and the identity of Entities involved in the Event. Additionally, a Domain Event often has a separate timestamp indicating when the Event was entered into the system, along with the identity of the person who entered it. When useful, an identity for the Domain Event can be based on some set of these properties. So, for example, if two instances of the same Event arrive at a node, they can be recognized as the same.

The essence of a Domain Event is that you use it to capture things that can trigger a change to the state of the application you're developing or to another application in your Domain that's interested in those changes. These Event objects are then processed to cause changes to the system and stored to provide an audit log.

### Naming Conventions

All Events should be represented as verbs in the past tense, as they're things that have been completed in the past — for example, `CustomerRelocated`, `CargoShipped`, or `InventoryLossageRecorde`d. There are interesting examples in the English language where one may be tempted to use nouns as opposed to verbs in the past tense; an example of this would be _Earthquake_ or _Capsize_ as relevant Events for a congressperson interested in natural disasters. We suggest avoiding the temptation of using names like those for Domain Events and sticking with verbs in the past tense.

### Domain Events and Ubiquitous Language

Consider the differences in the Ubiquitous Language when we discuss the side effects of relocating a customer. The Event makes the concept explicit, whereas previously, the changes that occurred within an Aggregate or between multiple Aggregates were left as an implicit concept that needed to be explored and defined. As an example, in most systems, when a side effect occurs on a library like Hibernate or the Entity Framework, it doesn't affect the Domain. These Events are implicit and transparent from the client point of view. The introduction of the Event makes the concept explicit and part of the Ubiquitous Language. Relocating a customer doesn't just change some stuff; rather it produces a `CustomerRelocatedEvent` that is explicitly defined within the language.

### Immutability

As we mentioned already, Domain Events talk about the past and describe changes in your Domain that have already occurred. By definition, it's impossible to change the past, unless you're Marty McFly and have a DeLorean, which is probably not the case. So just remember that Domain Events are immutable.

### Note

**`Symfony Event Dispatcher`** Some PHP frameworks support Events. However, don't confuse those Events with Domain Events; they're different in characteristics and goals. For example, Symfony has the Event Dispatcher component, and if you need to implement an Event system for a state machine, you can rely on it. In Symfony, the transformation from requests to responses is handled by Events too. However, Symfony Events are mutable, and each of the listeners is capable of modifying, adding to, or updating the information in the Event.

Modeling Events
---------------

* * *

In order to describe your business Domain accurately, you'll have to work closely with Domain Experts and define the Ubiquitous Language. This requires crafting Domain concepts using Domain Events, Entities, Value Objects, and so on. When modeling Events, name them and their properties according to the Ubiquitous Language, in the Bounded Context where they originated. If an Event is the result of executing a command operation on an Aggregate, the name is usually derived from the command that was executed. It's important that the Event name reflects the past nature of the occurrence.

Let's consider our user registration feature; the Domain Event needs to represent it. The following code shows a minimal interface for a base Domain Event:

    interface DomainEvent 
    { 
        /** 
         * @return DateTimeImmutable
         */
         public function occurredOn(); 
    }

As you can see, the minimum information required is a `DateTimeImmutable`, which is necessary in order to know when the Event happened.

Now let's model the new user registration Event using the following code. As we mentioned above, the name should be a verb in the past tense, so `UserRegistered` is probably a good choice:

    class UserRegistered implements DomainEvent 
    { 
        private $userId;
    
        public function __construct(UserId $userId)
        {
            $this->userId = $userId;
            $this->occurredOn = new \DateTimeImmutable();
        }
    
        public function userId()
        {
            return $this->userId;
        }
    
        public function occurredOn()
        {
            return $this->occurredOn;
        }
    }

The minimum amount of information required to notify subscribers about the creation of new users is the `UserId`. With this information, any process, command, or Application Service — from either the same Bounded Context or a different one — may react to this Event.

### Note

**`As a Rule of Thumb`**

*   Domain Events are usually designed as immutable
*   The Constructor will initialize the full state of the Domain Event.
*   Domain Events will have getters to access their attributes
*   Include the identity of the Aggregate that performs the action
*   Include other Aggregate identities related to the first one
*   Include parameters that caused the Event (if useful)

But what happens if your Domain experts from the same Bounded Context or a different one need more information? Let's see the same Domain Event modeled with more information — for example, the email address:

    class UserRegistered implements DomainEvent
    {
        private $userId;
        private $userEmail ;
    
        public function __construct(UserId $userId, $userEmail)
        {
            $this-> userId = $userId;
            $this->userEmail = $userEmail ;
            $this->occurredOn = new DateTimeImmutable();
        }
    
        public function userId()
        {
            return $this->userId;
        }
    
        public function userEmail ()
        {
            return $this->userEmail ;
        }
    
        public function occurredOn()
        {
            return $this->occurredOn;
        }
    }

Above, we've added the email address. Adding more information to a Domain Event can help improve performance or simplify the integration between different Bounded Contexts. Thinking from the point of view of another Bounded Context could help modeling Events. When a new user is created in the upstream Bounded Context, the downstream one would have to create its own user. Adding the user email could possibly save a sync request to the upstream Bounded Context in the case the downstream one needed it.

Do you remember the gamification example? In order to create a user of the gamification platform, probably called Player, the UserId from the e-commerce Bounded Context was probably enough. But what happens if the gamification platform has to notify the users by email about being rewarded? In this case, the email address is also mandatory. So if the email address is included in the original Domain Event, we're done. If that's not the case, the gamification Bounded Context needs to request this information from the e-commerce Bounded Context via REST or SOA integration.

### Note

**`Why Not the Whole User Entity?`** Wondering if you should include the whole User Entity from your Bounded Context in the Domain Event? Our suggestion is that you don't. Domain Events might be used to communicate messages internally to a given Bounded Context or externally to other Bounded Contexts. In other words, what can be a `Seller` in a C2C e-commerce product catalog Bounded Context can be an `Author` of a product review in a product feedback one. Both can share the same ID or email, but `Seller` and `Author` are different concepts representing different Entities from different Bounded Contexts. So Entities from one Bounded Context have no meaning or a totally different one in another Bounded Context.

Doctrine Events
---------------

* * *

Domain Events are not just for doing batch jobs such as sending emails or communicating to other Bounded Contexts; they're also interesting for performance and scalability improvements. Let's see an example.

Consider the following scenario. You have an e-commerce application. Your main persistence mechanism is MySQL, but for browsing and filtering your catalog, you're using a better approach, such as Elasticsearch or Solr. On Elasticsearch, you'll end up with a subset of the information stored in your full database. How do you keep the data in sync? What happens when the Content Team updates the catalog from the back office tool?

There have been people re-indexing the entire catalog from time to time. This is very expensive and slow. A smarter approach may be updating one or some of the documents related to the Product that has been updated. How can we do that? Using Domain Events.

However, if you've been working with Doctrine, this is likely not something that's new to you. According to the [Doctrine 2 ORM 2 Documentation](http://docs.doctrine-project.org/projects/doctrine-orm/en/latest/reference/events.html#events):

Doctrine 2 features a lightweight event system that is part of the Common package. Doctrine uses it to dispatch system events, mainly life cycle events. You can also use it for your own custom events.

Furthermore, it [states](http://docs.doctrine-project.org/projects/doctrine-orm/en/latest/reference/events.html#lifecycle-callbacks) that:

Life cycle Callbacks are defined on an entity class. They allow you to trigger callbacks whenever an instance of that entity class experiences a relevant life cycle event. More than one callback can be defined for each life cycle event. Life cycle Callbacks are best used for simple operations specific to a particular entity class's life cycle.

Let's see an example from the [Doctrine Events Documentation](http://doctrine-orm.readthedocs.io/projects/doctrine-orm/en/latest/reference/events.html):

    /** @Entity @HasLifecycleCallbacks */
    class User
    {
        // ...
    
       /**
        * @Column(type="string", length=255)
        */
        public $value;
    
        /** @Column(name="created_at", type="string", length=255) */
        private $createdAt;
    
        /** @PrePersist */
        public function doStuffOnPrePersist()
        {
            $this->createdAt = date('Y-m-d H:i:s');
        }
    
        /** @PrePersist */
        public function doOtherStuffOnPrePersist()
        {
            $this-> value = 'changed from prePersist callback!';
        }
    
        /** @PostPersist */
        public function doStuffOnPostPersist()
        {
            $this->value = 'changed from postPersist callback!';
        }
    
        /** @PostLoad */
        public function doStuffOnPostLoad()
        {
            $this->value = 'changed from postLoad callback!';
        }
    
        /** @PreUpdate */
        public function doStuffOnPreUpdate()
        {
            $this->value = 'changed from preUpdate callback!';
        }
    }

You can hook specific tasks on each different important moment in the Doctrine Entity life cycle. For example, on `PostPersist`, you can generate the JSON document of your Entity and put it into Elasticsearch. That way, it's easy to keep data consistent between different persistence mechanisms.

Doctrine Events are a good example of the benefits of Events around your Entities. But you may be wondering what the problem with them is. This is because they're coupled to a framework, they're synchronous, and they act on your application level, but not for communication purposes. So that's why Domain Events, despite being a bit more difficult to implement and handle, are so much more interesting.

Persisting Domain Events
------------------------

* * *

Persisting Events is always a good idea. Some of you may be wondering why you shouldn't publish Domain Events directly to a messaging or logging system. This is because persisting them has interesting benefits:

*   You can expose your Domain Events to other Bounded Contexts through a REST interface.
*   You can persist the Domain Event and the Aggregate changes in the same database transaction before pushing them to RabbitMQ. (You don't want to send notifications about something that didn't happen, just as you don't want to miss a notification about something that did happen.)
*   Business Intelligence can use this data to analyze, forecast, or trend.
*   You can audit your Entity changes.
*   For Event Sourcing, you can reconstitute Aggregates from Domain Events.

### Event Store

Where do we persist Domain Events? In an Event Store. An Event Store is a Domain Event Repository that lives in our Domain space as an abstraction (interface or abstract class). Its responsibility is to append Domain Events and query them. A possible basic interface could be the following:

    interface EventStore
    {
        public function append(DomainEvent $aDomainEvent);
        public function allStoredEventsSince($anEventId);
    }

However, depending on the usage of your Domain Events, the previous interface can have more methods to query your Events.

In terms of implementation, you can decide to use a Doctrine Repository, a DBAL one, or a plain PDO. Because Domain Events are immutable, using a Doctrine Repository adds an unnecessary performance penalty, though for a small to medium application, Doctrine is probably OK. Let's look at a possible implementation with Doctrine:

    class DoctrineEventStore extends EntityRepository implements EventStore
    {
        private $serializer;
    
        public function append(DomainEvent $aDomainEvent)
        {
            $storedEvent = new StoredEvent(
                get_class($aDomainEvent),
                $aDomainEvent->occurredOn(),
                $this->serializer()->serialize($aDomainEvent, 'json')
            );
    
            $this->getEntityManager()->persist($storedEvent);
         }
    
         public function allStoredEventsSince($anEventId)
         {
             $query = $this->createQueryBuilder('e');
             if ($anEventId) {
                 $query->where('e.eventId > :eventId');
                 $query->setParameters(['eventId' => $anEventId]);
             }
             $query->orderBy('e.eventId');
    
             return $query->getQuery()->getResult();
         }
    
         private function serializer()
         {
             if (null === $this->serializer) {
                 /** \JMS\Serializer\Serializer\SerializerBuilder */
                 $this->serializer = SerializerBuilder::create()->build();
             }
    
             return $this->serializer;
         }
     }


`StoredEvent` is the Doctrine Entity needed to map to the database. As you may have seen, when appending and after persisting the `Store`, there's no `flush` call. If this operation is inside a Doctrine transaction, it's not needed. So, let's leave it without the call and we'll go into more details when talking about Application Services.

Now let's see the `StoredEvent` implementation:

    class StoredEvent implements DomainEvent
    {
        private $eventId;
        private $eventBody;
        private $occurredOn;
        private $typeName;
    
        /**
         * @param string $aTypeName
         * @param \DateTimeImmutable $anOccurredOn
         * @param string $anEventBody
         */
        public function __construct(
            $aTypeName, \DateTimeImmutable $anOccurredOn, $anEventBody
        ) {
            $this->eventBody = $anEventBody;
            $this->typeName = $aTypeName;
            $this->occurredOn = $anOccurredOn;
        }
    
        public function eventBody()
        {
            return $this->eventBody;
        }
    
        public function eventId()
        {
            return $this->eventId;
        }
    
        public function typeName()
        {
            return $this->typeName;
        }
    
        public function occurredOn()
        {
            return $this->occurredOn;
        }
    }

And here is its mapping:

    Ddd\Domain\Event\StoredEvent:
        type: entity
        table: event
        repositoryClass:
            Ddd\Infrastructure\Application\Notification\DoctrineEventStore
        id:
            eventId:
                type: integer
                column: event_id
                generator:
                strategy: AUTO
        fields:
            eventBody:
                column: event_body
                type: text
            typeName:
                column: type_name
                type: string
                length: 255
            occurredOn:
                column: occurred_on
                type: datetime

In order to persist Domain Events with different fields, we'll have to join those fields as a serialized string. `typeName` identifies the Domain-wide Domain Event. An Entity or Value Object makes sense inside a Bounded Context, but Domain Events define a communication protocol between Bounded Contexts.

In distributed systems, shit happens. You'll have to deal with Domain Events that aren't published, are lost somewhere in the chain, or are published more than once. That's why it's important to persist a Domain Event with an ID, so that it's easy to track which Domain Events have been published and which are missing.

Publishing Events from the Domain Model
---------------------------------------

* * *

Domain Events should be published when the fact they represent occurs. For instance, when a new user has been registered, a new `UserRegistered` Event should be published.

Following the newspaper metaphor:

*   **Modeling** a Domain Event is like writing a news article
*   **Publishing** a Domain Event is like printing the article in the paper
*   **Spreading** a Domain Event is like delivering the newspaper so everyone can read the article

The recommended approach for publishing Domain Events is to use a simple Listener-Observer pattern to implement a `DomainEventPublisher`.

### Publishing a Domain Event from an Entity

Continuing with the example of a new user who has been registered in our application, let's see how the corresponding Domain Event can be published:

    class User
    {
        protected $userId;
        protected $email ;
        protected $password;
    
        public function __construct(UserId $userId, $email, $password)
        {
            $this->setUserId($userId);
            $this->setEmail($email);
            $this->setPassword($password);
    
            DomainEventPublisher::instance()->publish(
                new UserRegistered($this->userId)
            );
        }
    
        // ...
    }

As seen in the example, when the `User` is created, a new `UserRegistered` Event is published. It's done in the Entity constructor and not outside because, with this approach, it's easier to keep our Domain consistent; any client who creates a new `User` will publish its corresponding Event. On the other hand, this makes it a bit more complex to use an infrastructure that needs to create a `User` Entity without using its constructor. For example, Doctrine uses the `serialize` and `unserialize` technique that recreates an object without calling its constructor. However, if you have to create your own, this isn't going to be as easy as in Doctrine.

In general, constructing an object from plain data such as an array is called **hydration**. Let's see an easy approach to building a new `User` fetched from a database. First of all, let's extract the Domain Event publication to its own method by applying the [Factory Method pattern](http://en.wikipedia.org/wiki/Template_method_pattern).

According to [Wikipedia](https://en.wikipedia.org/wiki/Template_method_pattern):

### Note

The template method pattern is a behavioral design pattern that defines the program skeleton of an algorithm in an operation, deferring some steps to subclasses:

    class User
    {
        protected $userId;
        protected $email ;
        protected $password;
    
        public function __construct(UserId $userId, $email, $password)
        {
            $this->setUserId($userId);
            $this->setEmail($email);
            $this->setPassword($password);
            $this->publishEvent();
    
        }
    
        protected function publishEvent()
        {
            DomainEventPublisher::instance()->publish(
                new UserRegistered($this->userId)
            );
        }
    
        // ...
    }

Now, let's extend our current `User` with a new infrastructure Entity that will do the job for us. The trick here is make `publishEvent` do nothing so that the Domain Event isn't published:

    class CustomOrmUser extends User
    {
        protected function publishEvent()
        {
    
        }
    
        public static function fromRawData($data)
        {
            return new self(
                new UserId($data['user_id']),
                $data['email'],
                $data['password']
            );
        }
    }

Remember to be careful with this approach; you might fetch invalid objects from the persistence mechanism, as Domain rules change all the time. Another approach without using the parent constructor could be the following:

    class CustomOrmUser extends User
    {
        public function __construct()
        {
        }
    
        public static function fromRawData($data)
        {
            $user = new self();
            $user->userId = new UserId($data['user_id']);
            $user->email = $data['email'];
            $user->password = $data['password'];
    
            return $user;
        }
    }

With this approach, the parent constructor isn't called and User attributes must be protected. Other alternatives are Reflection, passing flags in the constructor, using a proxy library like [Proxy-Manager](https://packagist.org/packages/ocramius/proxy-manager), or using an ORM like Doctrine.

### Note

**`Other Strategy for Publishing Domain Events`** As you can see in the previous example, we're using a static class for publishing our Domain Events. Other people, as an alternative, and especially when using [Event Sourcing](http://martinfowler.com/eaaDev/EventSourcing.html), will suggest that Entities hold all the fired Events internally within a field. In order to access all the Events, a getter is used in the Aggregate. This is also a valid approach. However, sometimes it's a bit difficult to keep track of which Entities have fired an Event. It can also be difficult to fire Events from places that aren't just Entities, example: Domain Services. On the plus side, testing if an Entity has fired an Event is much easier.

### Publishing your Domain Events from Domain or Application Services

You should struggle to publish Domain Events from deeper in the chain. The closer to the inside of the Entity or the Value Object, the better. As we saw in the previous section, sometimes this isn't easy, but the final result is simpler for the clients. We've seen developers publishing Domain Events from the Application Services or Domain Services. This looks easier to do, but it will eventually lead to an Anemic Domain Model. This is not unlike when pushing business logic in Domain Services instead of placing it into your Entities.

### How the Domain Event Publisher Works

A Domain Event Publisher is a Singleton class available from our Bounded Context needed to publish Domain Events. It also has support to attach listeners — Domain Event Subscribers — that will be listening for any Domain Event they're interested in. This isn't much different from subscribing to an Event with jQuery using the on method:

    class DomainEventPublisher
    {
        private $subscribers;
        private static $instance = null;
    
        public static function instance()
        {
            if (null === static::$instance) {
                static::$instance = new static();
            }
    
            return static::$instance;
        }
    
        private function __construct()
        {
            $this->subscribers = [];
        }
    
        public function __clone()
        {
            throw new BadMethodCallException('Clone is not supported');
        }
    
        public function subscribe(
            DomainEventSubscriber $aDomainEventSubscriber
        ) {
            $this->subscribers[] = $aDomainEventSubscriber;
        }
    
        public function publish(DomainEvent $anEvent)
        {
            foreach ($this->subscribers as $aSubscriber) {
               if ($aSubscriber->isSubscribedTo($anEvent)) {
                   $aSubscriber->handle($anEvent);
               }
            }
        }
    }

The `publish` method goes through all the possible subscribers, checking if they're interested in the published Domain Event. If that's the case, the `handle` method of the subscriber is called.

The `subscribe` method adds a new `DomainEventSubscriber` that will be listening to specific Domain Event types:

    interface DomainEventSubscriber
    {
        /**
         * @param DomainEvent $aDomainEvent
         */
        public function handle($aDomainEvent);
    
        /**
         * @param DomainEvent $aDomainEvent
         * @return bool
         */
        public function isSubscribedTo($aDomainEvent);
    }


As we've already discussed, persisting all the Domain Events is a great idea. We can easily persist all the Domain Events published in our app by using a specific subscriber. Let's create a `DomainEventSubscriber` that will listen to all Domain Events, no matter what type, and persist them using our `EventStore`:

    class PersistDomainEventSubscriber implements DomainEventSubscriber
    {
        private $eventStore;
    
        public function __construct(EventStore $anEventStore)
        {
            $this->eventStore = $anEventStore;
        }
    
        public function handle($aDomainEvent)
        {
            $this->eventStore->append($aDomainEvent);
        }
    
        public function isSubscribedTo($aDomainEvent)
        {
            return true;
        }
    }

`$eventStore` could be a custom Doctrine Repository, as already seen, or any other object capable of persisting `DomainEvents` into a database.

### Setting Up DomainEventListeners

Where's the best place to set up the subscribers to the `DomainEventPublisher`? It depends. For global subscribers that will potentially affect the entire request cycle, the best place might be on the `DomainEventPublisher` initialization itself. For subscribers affected by a specific Application Service, the service instantiation might be a better place. Let's see an example using Silex.

In [Silex](http://silex.sensiolabs.org/), the best way to register a Domain Event Publisher that will persist all Domain Events is by using an Application Middleware. According to the [Silex 2.0 Documentation](http://silex.sensiolabs.org/doc/master/middlewares.html):

A _before_ application middleware allows you to tweak the Request before the controller is executed.

This is the correct place to subscribe the listener responsible for persisting to the database those Events that will be sent to RabbitMQ later:

    // ...
    $app['em'] = $app-> share(function () {
        return (new EntityManagerFactory())->build();
    });
    
    $app['event_repository'] = $app->share(function ($app) {
        return $app['em']->getRepository(
            'Ddd\Domain\Model\Event\StoredEvent'
        );
    });
    
    $app['event_publisher'] = $app->share(function($app) {
        return DomainEventPublisher::instance();
    });
    
    $app->before(
        function(Symfony\Component\HttpFoundation\Request $request)
            use($app) {
    
            $app['event_publisher']->subscribe(
                new PersistDomainEventSubscriber(
                    $app['event_repository']
                )
            );
        }
    );

With this setup, each time an Aggregate publishes a Domain Event, it will get persisted into the database. Mission accomplished.

### Note

**`Exercise`** If you're working with Symfony, Laravel, or another PHP framework, find a way to subscribe globally specific subscribers for performing tasks around your Domain Events.

In case you want to perform any action on all Domain Events when the request is about to finish, you can create a Listener that will store all published Domain Events in memory. If you add a getter to that Listener to return all Domain Events, you can then decide what to do. This can be useful if you don't want to or if you can't persist the Events in the same transaction, as suggested before.

### Testing Domain Events

You already know how to publish Domain Events, but how can you unit test this and ensure that `UserRegistered` is really fired? The easiest way we suggest is to use a specific `EventListener` that will work as a [Spy](http://www.martinfowler.com/bliki/TestDouble.html) to record whether or not the Domain Event was published. Let's see an example of the `User` Entity unit test:

    use Ddd\Domain\DomainEventPublisher;
    use Ddd\Domain\DomainEventSubscriber;
    
    class UserTest extends \PHPUnit_Framework_TestCase
    {
        // ...
    
        /**
         * @test
         */
        public function itShouldPublishUserRegisteredEvent()
        {
            $subscriber = new SpySubscriber();
            $id = DomainEventPublisher::instance()->subscribe($subscriber);
    
            $userId = new UserId();
            new User($userId, 'valid@email.com', 'password');
            DomainEventPublisher::instance()->unsubscribe($id);
    
            $this->assertUserRegisteredEventPublished($subscriber,$userId);
        }
    
        private function assertUserRegisteredEventPublished(
            $subscriber, $userId
        ) {
            $this->assertInstanceOf(
                'UserRegistered', $subscriber->domainEvent
            );
            $this->assertTrue(
                $subscriber->domainEvent->serId()->equals($userId)
            );
        }
    }
    
    class SpySubscriber implements DomainEventSubscriber
    {
        public $domainEvent;
    
        public function handle($aDomainEvent)
        {
            $this->domainEvent = $aDomainEvent;
        }
    
        public function isSubscribedTo($aDomainEvent)
        {
            return true;
        }
    }

There are some alternatives to the above. You could use a static setter for the `DomainEventPublisher` or some reflection framework to detect the call. However, we think the approach we've shared is more natural. Last but not least, remember to clean up the Spy subscription so it won't affect the execution of the rest of the unit tests.

Spreading the news to Remote Bounded Contexts
---------------------------------------------

* * *

In order to communicate a set of Domain Events to local or remote Bounded Contexts, there are two main strategies: messaging and a REST API. The first plans to use a messaging system such as RabbitMQ to transmit the Domain Events. The second plans to create a REST API for accessing the Domain Events of a specific Bounded Context.

### Messaging

With all Domain Events persisted into the database, the only thing remaining to spread the news is to push them to our favorite messaging system. We prefer [RabbitMQ](https://www.rabbitmq.com), but any other system, such as ActiveMQ or ZeroMQ, will do the job. For integrating with RabbitMQ using PHP, there aren't many options, but _[`php-amqplib`](https://packagist.org/packages/php-amqplib/php-amqplib)_ will do the work.

First of all, we need a service capable of sending persisted Domain Events to RabbitMQ. You may want to query EventStore for all the Events and send each one, which isn't a bad idea. However, we could push the same Domain Event more than once, and generally speaking, _we need to minimize the number of Domain Events republished_. If the number of Domain Events republished is 0, that's even better. In order to not republish Domain Events, we need some sort of component to track which Domain Events have already been pushed and which ones are remaining. Last but not least, once we know which Domain Events we have to push, we send them and keep track of the last one published into our messaging system. Let's see a possible implementation for this service:

    class NotificationService
    {
        private $serializer;
        private $eventStore;
        private $publishedMessageTracker;
        private $messageProducer;
    
        public function __construct(
            EventStore $anEventStore,
            PublishedMessageTracker $aPublishedMessageTracker,
            MessageProducer $aMessageProducer,
            Serializer $aSerializer
        ) {
            $this->eventStore = $anEventStore;
            $this->publishedMessageTracker = $aPublishedMessageTracker;
            $this->messageProducer = $aMessageProducer;
            $this->serializer = $aSerializer;
        }
    
        /**
         * @return int
         */
        public function publishNotifications($exchangeName)
        {
            $publishedMessageTracker = $this->publishedMessageTracker();
            $notifications = $this->listUnpublishedNotifications(
                $publishedMessageTracker
                    ->mostRecentPublishedMessageId($exchangeName)
            );
    
            if (!$notifications) {
                return 0;
            }
    
            $messageProducer = $this->messageProducer();
            $messageProducer->open($exchangeName);
            try {
                $publishedMessages = 0;
                $lastPublishedNotification = null;
                foreach ($notifications as $notification) {
                    $lastPublishedNotification = $this->publish(
                        $exchangeName,
                        $notification,
                        $messageProducer
                    );
                    $publishedMessages++;
                }
            } catch (\Exception $e) {
                // Log your error (trigger_error, Monolog, etc.)
            }
    
            $this->trackMostRecentPublishedMessage(
                $publishedMessageTracker,
                $exchangeName,
                $lastPublishedNotification
            );
    
            $messageProducer->close($exchangeName);
    
            return $publishedMessages;
        }
    
        protected function publishedMessageTracker()
        {
            return $this->publishedMessageTracker;
        }
    
        /**
         * @return StoredEvent[]
         */
        private function listUnpublishedNotifications(
            $mostRecentPublishedMessageId
        ) {
            return $this
                ->eventStore()
                ->allStoredEventsSince($mostRecentPublishedMessageId);
        }
    
        protected function eventStore()
        {
            return $this->eventStore;
        }
    
        private function messageProducer()
        {
            return $this->messageProducer;
        }
    
        private function publish(
            $exchangeName,
            StoredEvent $notification,
            MessageProducer $messageProducer
        ) {
            $messageProducer->send(
                $exchangeName,
                $this->serializer()->serialize($notification, 'json'),
                $notification->typeName(),
                $notification->eventId(),
                $notification->occurredOn()
            );
    
            return $notification;
        }
    
        private function serializer()
        {
           return $this->serializer;
        }
    
        private function trackMostRecentPublishedMessage(
            PublishedMessageTracker $publishedMessageTracker,
            $exchangeName,
            $notification
        ) {
            $publishedMessageTracker->trackMostRecentPublishedMessage(
                $exchangeName, $notification
            );
        }
    }

`NotificationService` depends on three interfaces. We've already seen `EventStore`, which is responsible for appending and querying Domain Events. The second one is `PublishedMessageTracker`, which is responsible for keeping track of pushed messages. The third one is `MessageProducer`, an interface representing our messaging system:

    interface PublishedMessageTracker
    {
        /**
         * @param string $exchangeName
         * @return int
         */
        public function mostRecentPublishedMessageId($exchangeName);
    
        /**
         * @param string $exchangeName
         * @param StoredEvent $notification
         */
        public function trackMostRecentPublishedMessage(
            $exchangeName, $notification
        );
    }

The `mostRecentPublishedMessageId` method returns the ID of last `PublishedMessage`, so that the process can start from the next one. `trackMostRecentPublishedMessage` is responsible for tracking which message was sent last, in order to be able to republish messages in case you need to. `$exchangeName` represents the communication channel we're going to use to send out our Domain Events. Let's see a Doctrine implementation of `PublishedMessageTracker`:

    class DoctrinePublishedMessageTracker extends EntityRepository\
    implements PublishedMessageTracker
    {
        /**
         * @param $exchangeName
         * @return int
         */
        public function mostRecentPublishedMessageId($exchangeName)
        {
            $messageTracked = $this->findOneByExchangeName($exchangeName);
            if (!$messageTracked) {
                return null ;
            }
    
            return $messageTracked->mostRecentPublishedMessageId();
        }
    
        /**
         *@param $exchangeName
         * @param StoredEvent $notification
         */
        public function trackMostRecentPublishedMessage(
            $exchangeName, $notification
        ) {
            if(!$notification) {
                return;
            }
    
            $maxId = $notification->eventId();
    
            $publishedMessage= $this->findOneByExchangeName($exchangeName);
            if(null === $publishedMessage){
                $publishedMessage = new PublishedMessage(
                    $exchangeName,
                    $maxId
                );
            }
    
            $publishedMessage->updateMostRecentPublishedMessageId($maxId);
    
            $this->getEntityManager()->persist($publishedMessage);
            $this->getEntityManager()->flush($publishedMessage);
        }
    }

This code is quite straightforward. The only edge case we have to consider is when no Domain Event has already been published.

### Note

**`Why an Exchange Name?`** We'll see this in more detail in the [Chpater 12](#integrating-bounded-contexts-chapter), _Integrating Bounded Contexts_. However, when a system is running and a new Bounded Context comes into play, you might be interested in resending all the Domain Events to the new Bounded Context. So keeping track of the last Domain Event published and the channel where it was sent might come in handy later.

In order to keep track of published Domain Events, we need an exchange name and a notification ID. Here's a possible implementation:

    class PublishedMessage
    {
        private $mostRecentPublishedMessageId;
        private $trackerId;
        private $exchangeName;
    
        /**
         * @param string $exchangeName
         * @param int $aMostRecentPublishedMessageId
         */
        public function __construct(
            $exchangeName, $aMostRecentPublishedMessageId
        ) {
            $this->mostRecentPublishedMessageId =
                $aMostRecentPublishedMessageId;
            $this->exchangeName = $exchangeName;
        }
    
        public function mostRecentPublishedMessageId()
        {
            return $this->mostRecentPublishedMessageId;
        }
    
        public function updateMostRecentPublishedMessageId($maxId)
        {
            $this->mostRecentPublishedMessageId = $maxId;
        }
    
        public function trackerId()
        {
            return $this->trackerId;
        }
    }

And here is its corresponding mapping:

    Ddd\Domain\Event\PublishedMessage:
        type: entity
        table: event_published_message_tracker
        repositoryClass:
            Ddd\Infrastructure\Application\Notification\
                DoctrinePublished\MessageTracker
        id:
            trackerId:
                column: tracker_id
                type: integer
                generator:
                strategy: AUTO
        fields:
            mostRecentPublishedMessageId:
                column: most_recent_published_message_id
                type: bigint
            exchangeName:
                type: string
                column: exchange_name

Now let's see what the `MessageProducer` interface is used for, along with its implementation details:

    interface MessageProducer 
    { 
        public function open($exchangeName);
    
        /**
         * @param $exchangeName
         * @param string $notificationMessage
         * @param string $notificationType
         * @param int $notificationId
         * @param \DateTimeImmutable $notificationOccurredOn
         * @return
         */
        public function send(
            $exchangeName,
            $notificationMessage,
            $notificationType,
            $notificationId,
            \DateTimeImmutable $notificationOccurredOn
        );
    
        public function close($exchangeName);
    }

Easy. The open and close methods open and close a connection with the messaging system. send takes a message body — message name and message ID — and sends them to our messaging engine, whatever it is. Because we've chosen RabbitMQ, we need to implement the connection and sending process:

    abstract class RabbitMqMessaging
    {
        protected $connection;
        protected $channel ;
    
        public function __construct(AMQPConnection $aConnection)
        {
            $this->connection =$aConnection;
            $this->channel = null ;
        }
    
        private function connect($exchangeName)
        {
            if (null !== $this->channel ) {
                return;
            }
    
            $channel = $this->connection->channel();
            $channel->exchange_declare(
                $exchangeName, 'fanout', false, true, false
            );
            $channel->queue_declare(
                $exchangeName, false, true, false, false
            );
            $channel->queue_bind($exchangeName, $exchangeName);
    
            $this->channel = $channel ;
        }
    
        public function open($exchangeName)
        {
    
        }
    
        protected function channel ($exchangeName)
        {
            $this->connect($exchangeName);
    
            return $this->channel;
        }
    
        public function close($exchangeName)
        {
            $this->channel->close();
            $this->connection->close();
        }
    }
    
    class RabbitMqMessageProducer
        extends RabbitMqMessaging
        implements MessageProducer
    {
        /**
         * @param $exchangeName
         * @param string $notificationMessage
         * @param string $notificationType
         * @param int $notificationId
         * @param \DateTimeImmutable $notificationOccurredOn
         */
        public function send(
            $exchangeName,
            $notificationMessage,
            $notificationType,
            $notificationId,
            \DateTimeImmutable $notificationOccurredOn
        ) {
            $this->channel ($exchangeName)->basic_publish(
                new AMQPMessage(
                    $notificationMessage,
                    [
                      'type'=>$notificationType,
                      'timestamp'=>$notificationOccurredOn->getTimestamp(),
                      'message_id'=>$notificationId
                    ]
                ),
                $exchangeName
            );
        }
    }

Now that we have a `DomainService` for pushing Domain Events into a messaging system like RabbitMQ, it's time to execute them. We need to choose a delivery mechanism to run the service. We personally suggest creating a [Symfony Console](http://symfony.com/doc/current/components/console/introduction.html) Command:

    class PushNotificationsCommand extends Command
    {
        protected function configure()
        {
            $this
                ->setName('domain:events:spread')
                ->setDescription('Notify all domain events via messaging')
                ->addArgument(
                    'exchange-name',
                    InputArgument::OPTIONAL,
                    'Exchange name to publish events to',
                    'my-bc-app'
                );
        }
    
        protected function execute(
            InputInterface $input, OutputInterface $output
        ) {
            $app = $this->getApplication()->getContainer();
    
            $numberOfNotifications =
                $app['notification_service']
                    ->publishNotifications(
                        $input->getArgument('exchange-name')
                    );
    
            $output->writeln(
                sprintf(
                    '<comment>%d</comment>' .
                    '<info>notification(s) sent!</info>',
                    $numberOfNotifications
                )
            );
        }
    }

Following the Silex example, let's see the definition of the `$app['notification_service']` defined in the [Silex Pimple Service Container](http://silex.sensiolabs.org/doc/services.html#id1):

     // ...
     $app['event_store']=$app->share( function ($app) {
         return $app['em']->getRepository('Ddd\Domain\Event\StoredEvent');
     });
    
    $app['message_tracker'] = $app->share(function($app) {
        return $app['em']
            ->getRepository('Ddd\Domain\Event\Published\Message');
    });
    
    $app['message_producer'] = $app->share(function () {
        return new RabbitMqMessageProducer(
           new AMQPStreamConnection('localhost', 5672, 'guest', 'guest')
        );
    });
    
    $app['message_serializer'] = $app->share(function () {
        return SerializerBuilder::create()->build();
    });
    
    $app['notification_service'] = $app->share(function ($app) {
        return new NotificationService(
           $app['event_store'],
           $app['message_tracker'],
           $app['message_producer'],
           $app['message_serializer']
        );
    });
    //...

### Syncing Domain Services with REST

With the `EventStore` already implemented in the messaging system, it should be easy to add some pagination capabilities, query for Domain Events, and render a JSON or XML representation publishing a REST API. Why is that interesting? Well, distributed systems using messaging have to face many different problems, such as messages that don't arrive, messages that arrive duplicated, or messages that arrive in an unexpected order. That's why it's nice to provide an API to publish your Domain Events so that other Bounded Contexts can ask for some missing information. Just as an example, consider that you make an HTTP request to an `/events` endpoint. A possible result would be the following:

    [
        {
            "id": 1,
            "version": 1,
            "typeName": "Lw\\Domain\\Model\\User\\UserRegistered",
            "eventBody": {
                "user_id": {
                    "id": "459a4ffc-cd57-4cf0-b3a2-0f2ccbc48234"
                }
            },
            "occurredOn": {
                "date": "2016-05-26 06:06:07.000000",
                "timezone_type": 3,
                "timezone": "UTC"
            }
        },
        {
            "id": 2,
            "version": 2,
            "typeName": "Lw\\Domain\\Model\\Wish\\WishWasMade",
            "eventBody": {
                "wish_id": {
                    "id": "9e90435a-395c-46b0-b4c4-d4b769cbf201"
                },
                "user_id": {
                    "id": "459a4ffc-cd57-4cf0-b3a2-0f2ccbc48234"
                },
                "address": "john@example.com",
                "content": "This is my new wish!"
            },
            "occurredOn": {
                "date": "2016-05-26 06:06:27.000000",
                "timezone_type": 3,
                "timezone": "UTC"
            },
            "timeTaken": "650"
        },
        //...
    ]

As you can see in the previous example, we're exposing a set of Domain Events in a JSON REST API. In the output example, you can see a JSON representation of each of the Domain Events. There are some interesting points. First, the `version` field. Sometimes your Domain Events will evolve: they'll include more fields, they'll change the behavior of some existing fields, or they'll remove some existing fields. That's why it's important to add a version field in your Domain Events. If other Bounded Contexts are listening to such Events, they can use the version field to parse the Domain Event in different ways. You may have faced the same problem when versioning REST APIs.

Another point is the name. If you want to use the `classname` of the Domain Event, it may work in most cases. The problem is when a team decides to change the name of the class because of a refactoring. In this case, all Bounded Contexts listening to that name would stop working. This problem only occurs if you publish different Domain Events in the same queue. If you publish each Domain Event type in a different queue, it's not a real problem, but if you choose this approach, you'll face a different set of problems, such as receiving unordered events. Like in many other instances, there's a tradeoff involved. We strongly recommend you read _Enterprise Integration Patterns:_ [Designing, Building, and Deploying Messaging Solutions](http://www.amazon.com/Enterprise-Integration-Patterns-Designing-Addison-Wesley-ebook/dp/B007MQLL4E). In this book, you'll learn different patterns for integrating multiple applications using asynchronous methods. Because Domain Events are messages sent in an integration channel, all messaging patterns also apply to them.

### Note

**`Exercise`** Think about the pros and cons of having a REST API for Domain Events. Consider Bounded Context coupling. You can also try to implement a REST API for your current application.

Wrap-Up
-------

* * *

We've seen the tricks to model a proper `DomainEvent` with a base interface, we've seen where to publish the `DomainEvent` (the nearer to the Entities the better), and we've seen the strategies for spreading those `DomainEvents` to local and remote Bounded Contexts. Now, the only thing remaining is listening for a notification in the messaging system, reading it, and executing the corresponding Application Service or Command. We'll see how to do this in [Chapter 12](https://subscription.packtpub.com/book/application-development/9781787284944/12), _Integrating Bounded Contexts_ and [Chapter 5](https://subscription.packtpub.com/book/application-development/9781787284944/5), _Services_.
