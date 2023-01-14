Chapter 7. Modules
------------------

> _When you place some classes together in a Module, you are telling the next developer who looks at your design to think about them together. If your model is telling a story, the Modules are chapters.               [Domain-Driven Design:Tackling Complexity in the Heart of Software](https://www.amazon.com/Domain-Driven-Design-Tackling-Complexity-Software/dp/0321125215)                                                                                                                                  -Eric Evans_                                                                                                                                       

A common concern when building an Application following Domain-Driven Design is where to place the code. Specifically if you're using a PHP framework, it's important to know the recommended way to place the code, where to place Infrastructure code, and how the different concepts inside the model should be structured.

In Domain-Driven Design, there's a tactical pattern for this: **modules**. Nowadays, everyone structures code in modules. All languages have some sort of tool to group classes and language definitions together. Java has packages. Ruby has modules. **PHP has namespaces**.

Domain-Driven Design goes one step further toward packaging and grouping your classes together and gives semantic meaning to these building blocks. Indeed, **it treats modules as a part of the model**. As part of the model, it's important to find the best naming, group together Domain objects that are close to each other, and keep the Domain objects that aren't related decoupled. **Modules should not be treated as a way to separate code but as a way to separate meaningful concepts in the model**.



General Overview
----------------

* * *

As explained in the [chapter 1](/chapters/01%20Getting%20Started.md), _Getting Started with Domain-Driven Design_, our Domain is organized internally into Subdomains. Each Subdomain is ideally modeled and implemented by one Bounded Context, but sometimes more than one is needed. If well designed, each Bounded Context is an independent system that will be developed and managed by a team. Our suggestion is to implement each Bounded Context with a whole Application. This means that two Bounded Contexts won't live in the same code Repository. As such, they can be deployed independently, have a different development cycle, or even be developed using different languages. Inside your Bounded Contexts, you'll use modules to group Domain objects that hold a strong relation to one another.



Leverage Modules in PHP
-----------------------

* * *

Until PHP 5.3, modules weren't fully supported. But since the introduction of PHP 5.3, we can use PHP namespaces to implement the module pattern. For historical reasons, we're going to present how namespaces were used before PHP 5.3, but you should strive to use a PHP version that supports PHP namespaces. The best choice is always going to be the latest stable version of PHP.

### First-Level Namespacing

A common approach is to use a first-level namespace that identifies your company. This will help in avoiding conflicts with third-party libraries. If you're using PSR-0, you'll have a real folder for the namespace; if you're using PSR-4, you don't need it. We'll go deeper into this shortly. But first, let's take a look at the PHP namespacing conventions.

### PEAR-Style Namespacing

Before PHP 5.3, due to the lack of a namespace construction, PEAR-style namespaces were used. PEAR is an acronym for PHP Extension and Application Repository, and in the good old days, it was a Repository of reusable components. It's still active, but it's not very convenient, and not many people use it anymore — particularly since Composer and Packagist were introduced. PEAR, as a source of reusable components, needed a way to avoid class name collisions, so contributors started prefixing class names with namespaces. There are still projects that use this form of namespaces (_PHPUnit_ and _Zend_ Framework 1, to name a couple). An example of PEAR-style namespaces:

The following would be an example of PEAR-style namespaces:

![](https://static.packt-cdn.com/products/9781787284944/graphics/1.png)

The class name for the Bill entity, using PEAR-style namespaces, would become `BuyIt_Billing_Domain_Model_Bill_Bill`. However, this is a bit ugly, and it doesn't follow one of the main Domain-Driven Design mantras: every class name should be named in terms of the Ubiquitous Language. For this reason, we strongly discourage its use.

#### PSR-0 and PSR-4 Namespacing

Namespaces entered the scene when _PHP 5.3_ was introduced, along with other important features. This was a major shift, and a group of the most important framework collaborators emerged with [PHP-FIG](http://www.php-fig.org/), an acronym of PHP Framework Interop Group, in an attempt to standardize and unify common aspects of the framework and library creation. The first **PHP Standard Recommendation** (**PSR**) the group released was an autoloading standard that, in short, proposed a one-to-one relation between a class and a PHP file using namespaces. Today, [PSR-4](http://www.php-fig.org/psr/psr-4/) — a simplification of [PSR-0](http://www.php-fig.org/psr/psr-0/) that still maintains the relation between classes and physical PHP files — is the preferred and recommended way to structure code. We believe that this should be the one used to implement modules in a project.

Referring back to the same folder structure shown in the previous section, let's see what changes with PSR-0. The class name for the Bill Entity, using namespaces and PSR-0, would simply become Bill, and the fully qualified class name would be `BuyIt\Billing\Domain\Model\Bill\Bill`.

As you can see, this enables us to name Domain objects in terms of the Ubiquitous Language, and this is the preferred way to structure and organize code. If you're using Composer, as you should be doing, you need to set some autoloading configurations in your `composer.json` file:

    ...
    "autoload": {
        "psr-0": {
            "BuyIt\\": "src/BuyIt/"
        }
    },
    "autoload-dev": {
        "psr-0": {
            "BuyIt": "tests/BuyIt/"
        }
    },
    ...

If you're not using PSR-4 or you haven't migrated from PSR-0 yet, we strongly recommend doing so. You can get rid of the first-level namespace folder, and your code structure will better match the Ubiquitous Language:

![](https://static.packt-cdn.com/products/9781787284944/graphics/2.png)

However, in order to avoid the collision with third-party libraries, it's still recommended to add the first-level namespace in your `composer.json` file:

    ...
    "autoload": {
        "psr-4": {
            "BuyIt\\": "src/"
        }
    },
    "autoload-dev": {
        "psr-4": {
            "BuyIt\\": "tests/"
        }
    },
    ...

If you prefer to have a first-level namespace but use PSR-4, there are some small changes to make:

![](https://static.packt-cdn.com/products/9781787284944/graphics/New3.png)

    ...
    "autoload": {
        "psr-4": {
            "BuyIt\\": "src/BuyIt/"
        }
    },
    "autoload-dev": {
        "psr-4": {
            "BuyIt\\": "tests/BuyIt/"
        }
    },
    ...

As you may have noticed in the examples, we split the `src` and `tests` folders. This was done in order to optimize the autoloading file generated by Composer, and it will reduce the memory needed to store the classmap. It will also help you set up whitelisting and blacklisting options when generating your unit testing code coverage reports. If you want to know more about Composer's autoloading configuration, take a look at the [documentation](https://getcomposer.org/doc/04-schema.md#autoload).

### Note

**`What about PHAR files?`** They could also be used, however, we don't recommend this. As an exercise, make a list of pros and cons for using PHAR files to model modules.



Bounded Contexts and Applications
---------------------------------

* * *

If we take the example of a fictional company called `BuyIt`, which deals with an e-commerce Domain, it may make sense to create a different application for each of the different Bounded Contexts solving specific Domain areas.

If some of the different Bounded Contexts are Order Management, Payment Management, Catalog Management, and Inventory Management, we recommend having an application for each one:

![](https://static.packt-cdn.com/products/9781787284944/graphics/3.png)

Each application exposes any set of delivery mechanisms needed. With the microservices trend, more and more people build Bounded Contexts that end up exposing REST APIs to the outside world. However, a Bounded Context is more than just an API. Remember that an API is just one of many delivery mechanisms; a Bounded Context can provide a web interface to interact with too.

### Note

**`Can Two Bounded Contexts Be in the Same Application? What about the Other Way Around?`** The best option is one Subdomain, one Bounded Context, and one application. If we have a Bounded Context implemented with two applications, the maintenance and the deployment get a bit tricky. And in the case of an application implementing two Bounded Contexts, the deployment process, the time for running the tests, and merging issues can slow down the development.

Beware that each Bounded Context name represents a meaningful concept in our e-commerce Domain and is named in terms of the Ubiquitous Language:

*   **Catalog** to hold all the code related to the product descriptions, product combinations, and so on.
*   **Inventory** to hold all the code related to the management of product stocks.
*   **Orders** to hold all the code related to the order processing systems. It will contain the finite-state machine in charge of processing orders.
*   **Payments** to hold all the code related to payments, bills, and waybills.



Structuring Code in Modules
---------------------------

* * *

Let's dig a bit further into one of the Bounded Contexts. Take, for example, the Orders context and examine the structure details. As its name suggests, this Bounded Context is responsible for representing all the flows that an order passes — from its creation up to delivering to the customer who has purchased it. Furthermore, it's an independent Application, so it contains a source code folder and a tests folder. The source code folder contains all the code necessary for this Bounded Context to work: the Domain code, the Infrastructure code, and the Application layer.

The following diagram should illustrate the organization:

![](https://static.packt-cdn.com/products/9781787284944/graphics/4.png)

All the code is prefixed with a vendor namespace named in terms of the organization name (`BuyIt`, in this case), and contains two subfolders: **Domain** holds all the Domain code, and **Infrastructure** holds the Infrastructure layer, thereby isolating all the Domain logic from the details of the Infrastructure layer. Following this structure, we're making it clear that we're going to use Hexagonal Architecture as a foundational architecture. Below is an example of an alternative structure that could be used:

![](https://static.packt-cdn.com/products/9781787284944/graphics/5.png)

The above style of structure uses an additional subfolder to store the Services defined inside the Domain Model. While this organization may make sense, our preference here is to not use it, since this way of separating code tends to be more focused on the architectural elements rather than the relevant concepts in the model. We believe this style could easily lead to some sort of Service layer on top of the Domain Model, which isn't necessarily a bad thing. Remember that Domain Services are used to describe operations in the Domain that don't belong to Entities or Value Objects. So from now on, we'll stick with the previous code organization.

It's possible to place code directly inside the `Domain/Model` subfolder. For example, it may be customary to place common interfaces and Services, like the `DomainEventPublisher` or the `DomainEventSubscriber`, in it.

If we had to model an Order Management context, we'd probably have an `Order` Entity with its Repository and all the state information. So our first attempt would be to place all those elements directly into the `Domain/Model` subfolder. At first glance, this may seem like the simplest way:

![](https://static.packt-cdn.com/products/9781787284944/graphics/6.png)

### Design Guidelines

Consider some basic rules and typical issues to pay attention to when implementing modules:

*   Namespaces should be named in terms of Ubiquitous Language.
*   Don't name your namespaces based on patterns or building blocks (Value Objects, Services, Entities, and so on).
*   Create namespaces so that what's inside is as loosely coupled with other namespaces as possible.
*   Refactor namespaces the same way as your code. Move them, rename them, group them, extract them, and so on.
*   Don't use commercial product names, as they can change. Stick to the Ubiquitous Language.

We've placed the Order and the `OrderLine` Entities, the `OrderLineWasAdded` and the `OrderWasCreated` Event, and the `OrderRepository` into the same subfolder `Domain/Model`. This structure may be fine, but that's because we still have a simple model. What about the `Bill` Entity and its Repository? Or the `Waybill` Entity and its respective Repository? Let's add all those elements and see how they fit into the actual code structure:

![](https://static.packt-cdn.com/products/9781787284944/graphics/7-1.png)

While this style of code organization could be fine, it can become non-practical and rather unmaintainable in the long run. Every time we iterate and add new features, the model will become even bigger, and the subfolder will be eating even more code. We need to split the code in a way that give us a perspective of the model at a glance. No technical concerns, just Domain concerns. To reach this, we can split the model using the Ubiquitous Language, by finding meaningful concepts that help us group elements logically in terms of the Domain.

To do this, we could try the following approach:

![](https://static.packt-cdn.com/products/9781787284944/graphics/8.png)

This way, the code is more organized, conceptually speaking. And as Eric Evans points out in [the Blue Book](http://www.amazon.com/Domain-Driven-Design-Tackling-Complexity-Software/dp/0321125215), modules are a way to communicate, as they provide us with insights about how the Domain Model works internally, along with helping us increase the cohesion and decrease the coupling between the concepts. If we look at the previous example, we can see that the concepts `Order` and `OrderLine` are strongly related, so they live in the same module. On the other hand, Order and Waybill, although sharing the same context, are different concepts, so they live in different modules. Modules are not just a way to group related concepts in the model, but also a way to express part of the design of the model.

### Note

**`Should We Place Repositories, Factories, Domain Events, and Services in Their Own Subfolders?`** Effectively, they could be placed into their own subfolders, but it's strongly discouraged. In doing so, we would be mixing technical concerns and Domain concerns — remember that the module's main interest is to group related concepts from the Domain model and decouple them from non-related concepts. Modules don't separate code but instead separate meaningful concepts.

### Modules in the Infrastructure Layer

Thus far, we've been discussing how we structure and organize code in the Domain layer, but we've said almost nothing about the Infrastructure layer. And since we're using Hexagonal Architecture to inverse the dependency between the Domain layer and the Infrastructure layer, we'll need a place where we can put all the implementations of the interfaces defined in the Domain layer. Returning to the example of the billing context, we need a place for the implementations of `BillRepository`, `OrderRepository`, and `WaybillRepository`.

It's clear that they should be placed into the Infrastructure folder, but where? Suppose we decided to use Doctrine ORM to implement the persistence layer. How do we put the Doctrine implementations of our Repositories into the Infrastructure folder? Let's do it directly and see how it looks:

![](https://static.packt-cdn.com/products/9781787284944/graphics/9.png)

We could leave this as is, but as we saw in the Domain layer, this structure and organization will rot fast and become a mess within a few model iterations. Each time the model grows, it'll probably need even more Infrastructure, and we'll end up mixing different technical concerns such as persistence, messaging, logging, and more. Our first attempt to avoid a tangled mess of Infrastructure implementations is to define a module for each technical concern in the Bounded Context:

![](https://static.packt-cdn.com/products/9781787284944/graphics/10.png)

This looks much better and is a lot more maintainable in the long term than our first attempt. However, our namespaces are lacking some sort of relation to the Ubiquitous Language. Let's consider a variation:

![](https://static.packt-cdn.com/products/9781787284944/graphics/11.png)

Much better. It matches our Domain Model organization, but inside the Infrastructure layer — plus everything seems easier to find. If you know beforehand that you'll always have a single persistence mechanism, you can stick with this structure and organization. It's rather simple and easy to maintain.

But what about when you have to play with several persistence mechanisms? Nowadays, it's quite common to have a relational persistence mechanism and some kind of shared in-memory persistence like Redis or Riak, or to have some sort of local in-memory implementation to be able to test the code. Let's see how this fits into the actual approach:

![](https://static.packt-cdn.com/products/9781787284944/graphics/12.png)

We recommend the above. However, all the Repository implementations are living in the same module. This could seem a bit odd when having so many different technologies. In case you find it interesting, you can create an additional module in order to group the related implementations by their underlying technology:

![](https://static.packt-cdn.com/products/9781787284944/graphics/13.png)

This approach is similar to the unit testing organization. However, there are classes, configurations, templates, and so on. that can't be matched with the Domain Model. That's why you may have additional modules inside the Infrastructure one that are related to specific technologies.

Where should you place Doctrine mapping files or Twig templates?

![](https://static.packt-cdn.com/products/9781787284944/graphics/14.png)

As you can see, in order to make Doctrine work, we need an `EntityManagerFactory` and all the mapping files. We may also include any other Infrastructure objects needed as base classes. Because they're not directly related to our Domain Model, it's better to place these resources in a different module. The same things happen with the Delivery Mechanisms (API, Web, Console Commands, and so on.). In fact, you can be using different PHP frameworks or libraries for each delivery mechanism:

![](https://static.packt-cdn.com/products/9781787284944/graphics/15.png)

In the previous example, we were using the Laravel Framework for serving the API, the Symfony Console Component as the entry point for the command line, and Silex and Slim for the web delivery mechanism. Regarding the `User` Interface, you should place it inside each delivery mechanism. However, if there's any chance to share the UI between different delivery mechanisms, you can create a module called UI at the same level as Persistence or Delivery. In general, our suggestion is struggling with how the frameworks tell you to organize your code. Frameworks should obey you, and not the other way around.

### Mixing Different Technologies

In large business-critical applications, it's quite common to have a mix of several technologies. For example, in read-intensive web applications, you usually have some sort of denormalized data source (Solr, Elasticsearch, Sphinx, and so on.) that provides all the reads of the application, while a traditional RDBMS like MySQL or Postgres is mainly responsible for handling all the writes. When this occurs, one of the concerns that normally arises is whether we can have read operations go with the search engine and write operations go with the traditional RDBMS data source. Our general advice here is that these kind of situations are a smell for CQRS, since we need to scale the reads and the writes of the application independently. So if you can go with CQRS, that's likely the best choice.

But if for any reason you can't go with CQRS, an alternative approach is needed. In this situation, the use of the Proxy pattern from the _Gang of Four_ comes in handy. We can define an implementation of a Repository in terms of the Proxy pattern:

    namespace BuyIt\Billing\Infrastructure\FullTextSearching\Elastica;
    
    use BuyIt\Billing\Domain\Model\Order\OrderRepository;
    use BuyIt\Billing\Infrastructure\Domain\Model\Order\Doctrine\
        DoctrineOrderRepository;
    
    class ElasticaOrderRepository implements OrderRepository 
    { 
        private $client; 
        private $baseOrderRepository;
    
        public function __construct(
            Client $client,
            DoctrineOrderRepository $baseOrderRepository
        ) {
            $this->client = $client;
            $this->baseOrderRepository = $baseOrderRepository;
        }
    
        public function find($id) 
        {
            return $this->baseOrderRepository->find($id);
        }
    
        public function findBy(array $criteria)
        {
            $search = new \Elastica\Search($this->client);
            // ...
            return $this->toOrder($search->search());
        }
    
        public function add($anOrder)
        {
            // First we attempt to add it to the Elastic index
            $ordersIndex = $this->client->getIndex('orders');
            $orderType = $ordersIndex->getType('order');
            $orderType->addDocument(
                new \ElasticaDocument(
                    $anOrder->id(),
                    $this->toArray($anOrder)
                )
            );
    
            $ordersIndex->refresh();
    
            // When it is done, we attempt to add it to the RDBMS store
            $this->baseOrderRepository->add($anOrder);
        }
    }

This example provides a naive implementation using the `DoctrineOrderRepository` and the Elastica client, a client to interact with an Elasticsearch server. Note that for some operations, we're using the RDBMS datasource, and for others, we're using the Elastica client. Also note that the add operation consists of two parts. The first one attempts to store the Order to the Elasticsearch index, and the second one attempts to store the Order into the relational database, delegating the operation to the Doctrine implementation. Take into account that this is just an example and a way to do it. It can probably be improved — for example, now the whole add operation is synchronous. We could instead enqueue the operation to some sort of messaging middleware that stores the Order in Elasticsearch, for example. There are a lot of possibilities and improvements, depending on your needs.

### Modules in the Application Layer

We've seen Domain and Infrastructure modules, so now let's take a look at the Application layer. In Domain-Driven Design, we suggest using Application Services as a way of decoupling the client from both the Domain Model and the necessary knowledge on how to interact with it. As you'll see in [Chapter 11](/chapters/11%20Application.md),  _Application_, an Application Service is built with its dependencies, is executed with a DTO request, and returns a DTO response.

It can also use an output dependency to return the result:

![](https://static.packt-cdn.com/products/9781787284944/graphics/16.png)

Our suggestion is to create modules around Application Services. Each module will hold its request and response. If you're using the Data Transformer as an output dependency, follow the Infrastructure approach as you would with the UI.



Wrap-Up
-------

* * *

Modules are a way of grouping and separating concepts in our application. Modules should be named following the Ubiquitous Language. We shouldn't forget that modules are a way to communicate high-level concepts, which aids us in keeping coupling low and cohesion high. We've seen that we could create meaningful modules even in old versions of PHP by using prefixes. Nowadays, it's easy to build our modules following the PSR-0 and PSR-4 namespacing conventions.
