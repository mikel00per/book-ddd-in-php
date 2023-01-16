Chapter 12. Integrating Bounded Contexts
----------------------------------------

Every enterprise application is typically composed of several areas in which the company operates. Areas such as _billing_, _inventory_, _shipping management_, _catalog_, and so on are common examples. The easiest manner in which to manage all these concerns may seem to lean toward a **monolithic system**. But, you might wonder, does it have to be this way? What if any friction garnered between teams working on these separate areas could be reduced by splitting this big monolithic application into smaller, independent chunks? In this chapter, we'll explore how to do this, so be prepared for insights and heuristics around **strategical design**.

> ### Note
> **`Dealing with Distributed Systems`** Dealing with distributed systems is **hard**. Breaking a system into independent autonomous parts has its benefits, but it also increases complexity. For example, the coordination and synchronization of distributed systems is not trivial, and as a result, should be considered carefully. As Martin Fowler said in the [PoEAA](https://www.martinfowler.com/books/eaa.html) book, the first law of distributed systems is always: **Don't distribute**.


Integration Through the Data Store
----------------------------------

* * *

One of the most commonly used techniques to integrate different parts of an application has always been to share the same data store, along with the same code base. This is usually known as a monolithic application, and it often ends up with a single data store that hosts the data related to all the concerns within the application.

Consider an e-commerce application. A shared data store would contain all concerns (Example: tables within a relational database) surrounding the catalog, billing, inventory, and so on. There's nothing wrong with this approach per se—for example, in small linear applications where the complexity is not too high. However, within complex Domains, some issues can arise. If you share data across many tables touching multiple application concerns, transactions will have a big impact on performance.

Another less technical problem that could develop is in regard to the Ubiquitous Language. The main advantage of the separation of Bounded Contexts is having **a single Ubiquitous Language for each one**. In doing so, models will be separated into their own Contexts. Mixing all models together within the same Context can lead to ambiguity and confusion.

Going back to the e-commerce system, imagine we want to introduce the concept of a t-shirt. Within the catalogue Context, a t-shirt would be a _product_ with properties like _color_, _size_, _material_, and maybe some fancy _pictures_. In the _inventory_ system, however, we don't really want to concern ourselves with these things. Here, a _product_ has a different meaning, where we care about different properties like _weight_, _location in the warehouse_, or _dimensions_. Mixing both Contexts together will tangle concepts and complicate the design. In Domain-Driven Design terms, mixing concepts in this manner is what is called a Shared Kernel.

> ### Note
> **`Shared Kernel`** Designate some subset of the domain model that the teams agree to share. Of course this includes, along with this subset of the model, the subset of code or of the database design associated with that part of the model. This explicitly shared stuff has special status, and shouldn't be changed without consultation with the other team. Integrate a functional system frequently, but somewhat less often than the pace of CONTINUOUS INTEGRATION within the teams. At these integrations, run the tests of both teams.  Eric Evans - [Domain-Driven Design: Tackling Complexity in the Heart of Software](https://www.amazon.com/Domain-Driven-Design-Tackling-Complexity-Software/dp/0321125215)

We don't recommend using a Shared Kernel, as multiple teams can collide within the development of it, which not only results in maintenance issues but also becomes a point of friction. However, if you opt to use a Shared Kernel, changes should be agreed upon beforehand and between all parties involved. Conceptually, this approach has other problems, such as people seeing it as a bag to place _stuff_ that doesn't belong anywhere else, and this grows indefinitely. A better way of dealing with the ever-growing complexity of the monolith is to break it up in different autonomous pieces, such as communicating through REST, RPC, or messaging systems. This requires drawing clear boundaries, with each Context likely ending up with its own Infrastructure—data stores, servers, messaging middleware, and so on — and even its own team.

As you might imagine, this could lead to some degree of duplication, but that's a tradeoff that we're willing to make in order to reduce complexity. In Domain-Driven Design, we call these independent pieces **Bounded Contexts**.


Integration Relationships
-------------------------

* * *

### Customer - Supplier

When there's a unidirectional integration between two Bounded Contexts, where one acts as a provider (**upstream**) and the other as a client (**downstream**), we'll end up with **Customer - Supplier Development Teams**.

> ### Note
> Establish a clear customer/supplier relationship between the two teams. In planning sessions, make the downstream team play the customer role to the upstream team. Negotiate and budget tasks for downstream requirements so that everyone understands the commitment and schedule. Jointly develop automated acceptance tests that will validate the interface expected. Add these tests to the upstream team's test suite, to be run as part of its' continuous integration. This testing will free the upstream team to make changes without fear of side effects downstream. Eric Evans - [Domain-Driven Design: Tackling Complexity in the Heart of Software](https://www.amazon.com/Domain-Driven-Design-Tackling-Complexity-Software/dp/0321125215).

Customer - Supplier Development Teams are the most common way of integrating Bounded Contexts and usually represent a win-win situation when teams work closely.

### Separate Ways

Continuing with the e-commerce example, think about reporting revenue to an old legacy retailer financial system. The integration could be incredibly expensive, resulting in it not being worth the effort to implement. In Domain-Driven Design strategic terms, this is known as **Separate Ways**.

> ### Note
> Integration is always expensive. Sometimes the benefit is small. So Declare a BOUNDED CONTEXT to have no connection to the others at all, allowing developers to find simple, specialized solutions within this small scope. Eric Evans - _Domain-Driven Design:_ [Tackling Complexity in the Heart of Software](https://www.amazon.com/Domain-Driven-Design-Tackling-Complexity-Software/dp/0321125215).

### Conformist

Consider again the e-commerce example and integration with a third-party shipping service. Both Domains differ in models, teams, and Infrastructure. The team responsible for maintaining the third-party shipping service will not participate in your product planning or provide any solutions to the e-commerce system. These teams don't have a close relationship. We could choose to accept and _conform_ to their Domain Model. In strategic design, this is what we call a **Conformist Integration**.

> ### Note
> Eliminate the complexity of translation between BOUNDED CONTEXTS by slavishly adhering to the model of the upstream team. Although this cramps the style of the downstream designers and probably does not yield the ideal model for the application, choosing CONFORMITY enormously simplifies integration. Also, you will share a UBIQUITOUS LANGUAGE with your supplier team. The supplier is in the driver's seat, so it is good to make communication easy for them. Altruism may be sufficient to get them to share information with you. Eric Evans - _Domain-Driven Design:_ [Tackling Complexity in the Heart of Software](https://www.amazon.com/Domain-Driven-Design-Tackling-Complexity-Software/dp/0321125215).


Implementing Bounded Context Integrations
-----------------------------------------

* * *

To make things easier, we'll assume Bounded Contexts have a Customer - Supplier relationship.

### Modern RPC

With modern RPC, we refer to RPC through RESTful resources. A Bounded Context reveals a clear interface to interact with to the outside world. It exposes resources that could be manipulated through HTTP verbs. We could say that the Bounded Context offers a set of services and operations. In strategical terms, this is what is called an **Open Host Service**.

> ### Note
> **`Open Host Service`** Define a protocol that gives access to your subsystem as a set of SERVICES. Open the protocol so that all who need to integrate with you can use it. Enhance and expand the protocol to handle new integration requirements, except when a single team has idiosyncratic needs. Then, use a one-off translator to augment the protocol for that special case so that the shared protocol can stay simple and coherent. Eric Evans - _Domain-Driven Design:_ [Tackling Complexity in the Heart of Software](https://www.amazon.com/Domain-Driven-Design-Tackling-Complexity-Software/dp/0321125215)_._

Let's explore an example provided within the [Last Wishes application](https://github.com/dddinphp/last-wishes) that comes with this book's GitHub organization.

The application is a web platform with the purpose of letting people save their last wills before they die. There are two Contexts: one responsible for handling wills—the Will Bounded Context—and one in charge of giving points to the users of the system—the [Gamification Context](https://github.com/dddinphp/last-wishes-gamify). In the Will Context, the user could have badges related to the number of points the user made on the Gamification Context. This means that we need to integrate both Contexts together in order to show the badges a user has on the Will Context.

The Gamification Context is a full-fledged event-driven application powered by a custom event sourcing engine. It's a full-stack Symfony application that uses [FOSRestBundle](http://symfony.com/doc/current/bundles/FOSRestBundle/index.html), [BazingaHateoasBundle](https://github.com/willdurand/BazingaHateoasBundle), [JMSSerializerBundle](https://github.com/schmittjoh/JMSSerializerBundle), [NelmioApiDocBundle](https://github.com/schmittjoh/JMSSerializerBundle), and [OngrElasticsearchBundle](https://github.com/schmittjoh/JMSSerializerBundle) to provide a level 3 and up REST API (commonly known as the Glory of REST), according to the [Richardson Maturity _Mo_del](https://martinfowler.com/articles/richardsonMaturityModel.html). All the Events triggered within this Context are projected against an Elasticsearch server, in order to produce the data needed for the views. We'll expose the number of points made for a given user through an endpoint like `http://gamification.context.host/api/users/{id}`.

We'll also fetch the user projection from Elasticsearch and serialize it to a format previously negotiated with the client:

```php
namespace AppBundle\Controller;

use FOS\RestBundle\Controller\Annotations as Rest;
use FOS\RestBundle\Controller\FOSRestController;
use Nelmio\ApiDocBundle\Annotation\ApiDoc;

class UsersController extends FOSRestController
{
    /**
     * @ApiDoc(
     *     resource = true,
     *     description = "Finds a user given a user ID",
     *     statusCodes = {
     *         200 = "Returned when the user have been found",
     *         404 = "Returned when the user could not be found"
     *     }
     * )
     *
     * @Rest\View(
     *     statusCode = 200
     * )
     */
    public function getUserAction($id)
    {
        $repo = $this->get('es.manager.default.user');
        $user = $repo->find($id);

        if (!$user) {
            throw $this->createNotFoundException(
                sprintf(
                    'A user with an ID of %s does not exist',
                    $id
                )
            );
        }
        return $user;
    }
}
```

As we explained in the [Chapter 2](../chapters/02%20Architectural%20Styles.md), _Architectural Styles_ reads are treated as an Infrastructure concern, so there's no need to wrap them inside a Command / Command Handler flow.

The resulting JSON+HAL representation of a user will be like this:

```json
{
    "id": "c3c587c6-610a-42df",
    "points": 0,
    "_links": {
        "self": {
            "href":
            "http://gamification.ctx/api/users/c3c587c6-610a-42df"
        }
    }
}
```

Now we're in a good position to integrate both Contexts. We just need to write the client in the Will Context for consuming the endpoint we've just created. Should we mix both Domain Models? Digesting the Gamification Context directly will mean adapting the Will Context to the Gamification one, resulting in a **Conformist** integration. However, separating these concerns seems worth the effort. We need a layer for guaranteeing the integrity and the consistency of the Domain Model within the Will Context, and we need to translate _points_ (Gamification) to _badges_ (Will). In Domain-Driven Design, this translation mechanism is what's called an **Anti-Corruption layer**.

> ### Note
> **`Anti-Corruption Layer`** Create an isolating layer to provide clients with functionality in terms of their own domain model. The layer talks to the other system through its existing interface, requiring little or no modification to the other system. Internally, the layer translates in both directions as necessary between the two models. Eric Evans - _Domain-Driven Design:_ [Tackling Complexity in the Heart of Software.](https://www.amazon.com/Domain-Driven-Design-Tackling-Complexity-Software/dp/0321125215)

So, what does the Anti-Corruption layer look like? Most of the time, Services will be interacting with a combination of Adapters and Facades. The Services encapsulate and hide the low-level complexities behind these transformations. Facades aid in hiding and encapsulating access details required for fetching data from the Gamification model. Adapters translate between models, often using specialized Translators.

Let's see how to define a User Service within the Will's model that will be responsible for retrieving the badges earned by a given user:

```php
namespace Lw\Domain\Model\User;

interface UserService
{
    public function badgesFrom(UserId $id);
}
```


Now let's look at the implementation on the Infrastructure side. We'll use an adapter for the transformation process:

```php
namespace Lw\Infrastructure\Service;

use Lw\Domain\Model\User\UserId;
use Lw\Domain\Model\User\UserService;

class TranslatingUserService implements UserService
{
    private $userAdapter;

    public function __construct(UserAdapter $userAdapter)
    {
        $this->userAdapter = $userAdapter;
    }

    public function badgesFrom(UserId $id)
    {
        return $this->userAdapter->toBadges($id);
    }
}
```

And here's the HTTP implementation for the `UserAdapter`:

```php
namespace Lw\Infrastructure\Service;

use GuzzleHttp\Client;

class HttpUserAdapter implements UserAdapter
{
    private $client;

    public function __construct(Client $client)
    {
        $this->client = $client;
    }

    public function toBadges( $id)
    {
        $response = $this->client->get(
            sprintf('/users/%s', $id),
            [
                'allow_redirects' => true,
                'headers' => [
                    'Accept' => 'application/hal+json'
                ]
            ]
        );

        $badges = [];
        if (200 === $response->getStatusCode()) {
            $badges = 
                (new UserTranslator())
                    ->toBadgesFromRepresentation(
                        json_decode(
                            $response->getBody(),
                            true
                        )
                    );
        }
        return $badges;
    }
}
```

As you can see, the Adapter acts as a **Facade to the Gamification Context** too. We did it this way, as fetching the User resource on the Gamification side is pretty straightforward. The Adapter uses the `UserTranslator` to perform the translation:

```php
namespace Lw\Infrastructure\Service;

use Lw\Infrastructure\Domain\Model\User\FirstWillMadeBadge;
use Symfony\Component\PropertyAccess\PropertyAccess;

class UserTranslator
{
    public function toBadgesFromRepresentation($representation)
    {
        $accessor = PropertyAccess::createPropertyAccessor();
        $points = $accessor->getValue($representation, 'points');
        $badges = [];
        if ($points > 3) {
            $badges[] = new FirstWillMadeBadge();
        }
        return $badges;
    }
}
```

The Translator specializes in transforming the points coming from the Gamification Context into badges.

We've shown how to integrate two Bounded Contexts where respective teams share a **Customer-Supplier** relationship. The Gamification Context exposes the integration through an **Open Host Service** implemented by a RESTful protocol. On the other side, the Will Context consumes the service through an **Anti-Corruption layer** responsible for translating the model from one Domain to the other, ensuring the Will Context's integrity.

### Message Queues

RESTful resources aren't the only way of enabling integrations between Bounded Contexts. As we'll see, messaging middleware enables decoupled integrations between different Contexts.

Continuing with the Last Wishes application, we've just implemented a unidirectional relationship between two teams to manage points and badges within their respective Contexts. However, we left an important functionality out of scope on purpose: **rewarding the user every time they make a wish**.

We could go for another Open Host Service with a pull strategy. The Will Context will be pulling the Gamification Context periodically to get badges on sync (Example: through an scheduler like Cron). This solution will impact the user's experience, and it'll waste a lot of unnecessary resources.

A better approach is to use a **messaging middleware**. With this solution, Contexts could push messages to a middleware (often a message queue). Interested parties will be able to subscribe, inspect, and consume information on demand in a decoupled fashion. In order to do this, we need a **specialized, shared, and common communication language**, so all the parties can understand the information transmitted. This is what's called the **Published Language**.

> ### Note
> **`Published Language`** Use a well-documented shared language that can express the necessary domain information as a common medium of communication, translating as necessary into and out of that language.  Eric Evans - _Domain-Driven Design:_ [Tackling Complexity in the Heart of Software](https://www.amazon.com/Domain-Driven-Design-Tackling-Complexity-Software/dp/0321125215).

In thinking about the format of these messages and looking closer at our Domain Model, we realize we already have what we need: [Chapter 6](../chapters/06%20Domain-Events.md), _Domain-Events_. It's not necessary to define a new way of communicating between Bounded Contexts. Instead, we can just use Domain Events to define a common language across Contexts. The definition of _something that Domain Experts care about that just happened_ fits perfectly with what we're looking for: a formal Published Language.

In our example, we could use RabbitMQ as a messaging middleware. This is probably one of the most reliable and robust messaging [AMQP](https://www.amqp.org/) protocols out there. We'll also incorporate the widely used PHP libraries [php-amqplib](https://github.com/php-amqplib/php-amqplib) and [RabbitMQBundle](https://github.com/php-amqplib/RabbitMqBundle).

Let's start with the Will Context, as it's the one that triggers Events when the user signs up or when making a wish. As we've already seen in the [Chapter 6](../chapters/06%20Domain-Events.md), _Domain-Events_, **it's a good idea to store Domain Events into a persistent mechanism**, so we'll assume that's what was done. We need a message publisher to fetch and publish stored Domain Events from the Event store to the messaging middleware. We already did the integration with RabbitMQ in the [Chapter 6](../chapters/06%20Domain-Events.md), _Domain-Events_, so we just need to implement the code in the Gamification Context. We'll listen for Events triggered by the Will Context. As we're using the Symfony Framework, we take advantage of a Symfony package called RabbitMQBundle.

We define two message consumers for the _User Registered_ and _Wish Was Made_ events:

```php
namespace AppBundle\Infrastructure\Messaging\PhpAmqpLib;

use Lw\Gamification\Command\SignupCommand;
use OldSound\RabbitMqBundle\RabbitMq\ConsumerInterface;
use PhpAmqpLib\Message\AMQPMessage;

class PhpAmqpLibLastWillUserRegisteredConsumer
    implements ConsumerInterface
{
    private $commandBus;

    public function __construct($commandBus)
    {
        $this->commandBus = $commandBus;
    }

    public function execute(AMQPMessage $message)
    {
        $type = $message->get('type');

        if('Lw\Domain\Model\User\UserRegistered' === $type) {
            $event = json_decode($message->body);
            $eventBody = json_decode($event->event_body);

            $this->commandBus->handle(
                new SignupCommand($eventBody->user_id->id)
            );
            return true;
        }
        return false;
    }
}
```

Note that in this case, we're only processing messages with the type of `Lw\Domain\Model\User\UserRegistered`:

```php
namespace AppBundle\Infrastructure\Messaging\PhpAmqpLib;

use Lw\Gamification\Command\RewardUserCommand;
use Lw\Gamification\Domain\Model\AggregateDoesNotExist; 
use OldSound\RabbitMqBundle\RabbitMq\ConsumerInterface;
use PhpAmqpLib\Message\AMQPMessage;

class PhpAmqpLibLastWillWishWasMadeConsumer implements ConsumerInterface
{
    private $commandBus;

    public function __construct($commandBus)
    {
        $this->commandBus = $commandBus;
    }

    public function execute(AMQPMessage $message)
    {
        $type = $message->get('type');

        if ('Lw\Domain\Model\Wish\WishWasMade' === $type) {
            $event = json_decode($message->body);
            $eventBody = json_decode($event->event_body);

            try {
                $points = 5;
                $this->commandBus->handle(
                    new RewardUserCommand(
                        $eventBody->user_id->id,
                        $points
                    )
                );
            } catch (AggregateDoesNotExist $e) {
                // Noop
            }

            return true;
        }

        return false;
    }
}
```

Again, we're only interested in tracking `Lw\Domain\Model\Wish\WishWasMade events`.

In both cases, we use a Command Bus, which we discussed in the Chapter, Application. However, we can summarize it as a highway that decouples the Command and Receiver. The **when** and **how** a Command is executed is independent from **who** triggered it.

The Gamification Context uses [Tactician](http://tactician.thephpleague.com/) (and [TacticianBundle](https://github.com/thephpleague/tactician-bundle)), a simple Command Bus that can be extended and adapted to your system. So now we're almost ready to start consuming Events from the Will Context.

The only thing we still need to do is define the RabbitMQBundle configuration in Symfony's `config.yml` file:

```yaml
services:
    last_will_user_registered_consumer:
        class:
            AppBundle\Infrastructure\Messaging\
                PhpAmqpLib\PhpAmqpLibLastWillUserRegisteredConsumer
        arguments:
            - @tactician.commandbus

    last_will_wish_was_made_consumer:
        class:
            AppBundle\Infrastructure\Messaging\
                PhpAmqpLib\PhpAmqpLibLastWillWishWasMadeConsumer
        arguments:
            - @tactician.commandbus

old_sound_rabbit_mq:
    connections:
         default:
              host: " %rabbitmq_host%"
              port: " %rabbitmq_port%"
              user: " %rabbitmq_user%"
              password: " %rabbitmq_password%"
              vhost: " %rabbitmq_vhost%"
              lazy: true

    consumers:
        last_will_user_registered:
            connection: default
            callback: last_will_user_registered_consumer

            exchange_options:
                name: last-will
                type: fanout

            queue_options:
                name: last-will

        last_will_wish_was_made:
            connection: default
            callback: last_will_wish_was_made_consumer

            exchange_options:
                name: last-will
                type: fanout

            queue_options:
                name: last-wil
```

The most convenient RabbitMQ configuration is probably the \[[Publish / Subscribe](https://www.rabbitmq.com/tutorials/tutorial-three-php.html)\] pattern. All messages published by the Will Context will be delivered to all connected consumers. This is called **fanout** in the RabbitMQ exchange configuration.

The exchange consists of an agent being in charge of delivering messages to the corresponding queues:

```shell
$ php app/console rabbitmq:consumer --messages=1000 last_will_user_registered
$ php app/console rabbitmq:consumer --messages=1000 last_will_wish_was_made
```

With those two commands, Symfony will execute both consumers and they'll start listening for Domain Events. We've specified a limit of 1,000 messages to consume, as PHP isn't the best platform for executing long-running processes. It also might be a good idea to use something like [Supervisor](http://supervisord.org/) to monitor and restart processes periodically.


Wrap-Up
-------

* * *

Although we've only seen a small part of it, strategical design is at the heart and soul of Domain-Driven Design. It's an essential part that aids in developing better and more semantic models. We recommend using messaging middleware to integrate Bounded Contexts, as this naturally leads to simpler, decoupled, and Event-driven architectures.
