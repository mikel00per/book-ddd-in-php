Chapter 4. Entities
-------------------

We've talked about the benefits of trying to first model everything in the Domain as a Value Object. But when modeling the Domain, there will probably be situations where you'll find that some concept in the Ubiquitous Language demands a thread of Identity.

Introduction
------------

* * *

Clear examples of objects requiring an Identity include:

> *   A **person**. A person always has an Identity and it's always the same in terms of their name or identification card.
> *   An **order** in an e-commerce system. In such a context, every new order created has its own Identity and it's the same over time.

These concepts have an Identity that endures over time. No matter how many times data in the concepts changes, their Identities remain the same. That's what makes them Entities and not Value Objects. In terms of PHP implementation, they would be plain old classes. For example, consider the following in the case of a person:

    namespace Ddd\Identity\Domain\Model;
    
    class Person
    {
        private $identificationNumber;
        private $firstName;
        private $lastName;
    
        public function __construct(
            $anIdentificationNumber, $aFirstName, $aLastName
        ) {
            $this->identificationNumber = $anIdentificationNumber;
            $this->firstName = $aFirstName;
            $this->lastName  = $aLastName;
        }
    
        public function identificationNumber()
        {
            return $this->identificationNumber;
        }
    
        public function firstName()
        {
            return $this->firstName;
        }
    
        public function lastName()
        {
            return $this->lastName;
        }
     }


Or, consider the following in the case of an order:

    namespace Ddd\Billing\Domain\Model\Order;
    
    class Order
    {
        private $id;
        private $amount;
        private $firstName;
        private $lastName;
    
        public function __construct(
            $anId, Amount $amount, $aFirstName, $aLastName
        ) {
            $this->id = $anId;
            $this->amount = $amount;
            $this->firstName = $aFirstName;
            $this->lastName = $aLastName;
        }
    
        public function id()
        {
            return $this->id;
        }
    
        public function firstName()
        {
            return $this->firstName;
        }
    
        public function lastName()
        {
            return $this->lastName;
        }
    }



Objects Vs. Primitive Types
---------------------------

* * *

Most of the time, the Identity of an Entity is represented as a primitive type — usually a string or an integer. But using a Value Object to represent it has more advantages:

> *   Value Objects are immutable, so they can't be modified.
> *   Value Objects are complex types that can have custom behaviors, something which primitive types can't have. Take, as an example, **the equality operation**. With Value Objects, equality operations can be modeled and encapsulated in their own classes, making concepts go from implicit to explicit.

Let's see a possible implementation for `OrderId`, the `Order` Identity that has evolved into a Value Object:

    namespace Ddd\Billing\Domain\Model;
    
    class OrderId
    {
        private $id;
    
        public function __construct($anId)
        {
            $this->id = $anId;
        }
    
        public function id()
        {
            return $this->id;
        }
    
        public function equalsTo(OrderId $anOrderId)
        {
            return $anOrderId->id === $this->id;
        }
    }


There are different implementations you can consider for implementing the `OrderId`. The example shown above is quite simple. As explained in the [Chapter 3](../chapters/03%20Value%20Objects.md), _Value Objects_, you can make the `__constructor` method private and use static factory methods to create new instances. Talk with your team, experiment, and agree. Because Entity Identities are not complex Value Objects, our recommendation is that you shouldn't worry too much here.

Going back to the `Order`, it's time to update references to `OrderId`:

     class Order
     {
         private $id;
         private $amount;
         private $firstName;
         private $lastName;
    
         public function __construct(
             OrderId $anOrderId, Amount $amount, $aFirstName, $aLastName
         ) {
             $this->id = $anOrderId;
             $this->amount = $amount;
             $this->firstName = $aFirstName;
             $this->lastName = $aLastName;
         }
    
         public function id()
         {
             return $this->id;
         }
    
         public function firstName()
         {
             return $this->firstName;
         }
    
         public function lastName()
         {
             return $this->lastName;
         }
    
         public function amount()
         {
             return $this->amount;
         }
    }


Our Entity has an Identity modeled using a Value Object. Let's consider different ways of creating an `OrderId`.



Identity Operation
------------------

* * *

As stated before, the Identity of an Entity is what defines it. So then, handling it is an important aspect of the Entity. There are usually four ways to define the Identity of an Entity: the persistence mechanism provides the Identity, a client provides the Identity, the application itself provides the Identity, or another Bounded Context provides the Identity.

### Persistence Mechanism Generates Identity

Usually, the simplest way of generating the Identity is to delegate it to the persistence mechanism, because the vast majority of persistence mechanisms support some kind of Identity generation — like MySQL's `AUTO_INCREMENT` attribute or Postgres and Oracle sequences. This, although simple, has a major drawback: we won't have the Identity of the Entity until we persist it. So to some degree, if we're going with persistence mechanism-generated Identities, we'll couple the Identity operation with the underlying persistence store:

    CREATE TABLE `orders` (
        `id` int(11) NOT NULL auto_increment,
        `amount` decimal (10,5) NOT NULL,
        `first_name` varchar(100) NOT NULL,
        `last_name` varchar(100) NOT NULL,
        PRIMARY KEY (`id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;


And then we might consider this code:

    namespace Ddd\Identity\Domain\Model;
    
    class Person
    {
        private $identificationNumber;
        private $firstName;
        private $lastName;
    
        public function __construct(
            $anIdentificationNumber, $aFirstName, $aLastName
        ) {
            $this->identificationNumber = $anIdentificationNumber;
            $this->firstName = $aFirstName;
            $this->lastName = $aLastName;
        } 
    
        public function identificationNumber()
        {
            return $this->identificationNumber;
        }
    
        public function firstName()
        {
            return $this->firstName;
        }
    
        public function lastName()
        {
            return $this->lastName;
        }
    }

If you've ever tried to build your own ORM, you've already experienced this situation. What's the approach for creating a new Person? If the database is going to generate the Identity, do we have to pass it in the constructor? When and where is the magic that will update the Person with its Identity? What happens if we end up not persisting the Entity?

#### Surrogate Identity

Sometimes when using an ORM to map Entities to a persistence store, some constraints are imposed — for example, Doctrine demands an integer field if an `IDENTITY` generator strategy is used. This can conflict with the Domain Model if it requires another kind of Identity.

The simplest way to handle such a situation is by using a [Layer Supertype](http://martinfowler.com/eaaCatalog/layerSupertype.html), where the Identity field created for the persistence store is placed:

    namespace Ddd\Common\Domain\Model;
    
    abstract class IdentifiableDomainObject
    {
        private $id;
    
        protected function id()
        {
            return $this->id;
        }
    
        protected function setId($anId)
        {
            $this->id = $anId;
        }
    }
    
    namespace Acme\Billing\Domain;
    
    use Acme\Common\Domain\IdentifiableDomainObject;
    
    class Order extends IdentifiableDomainObject
    {
        private $orderId;
    
        public function orderId()
        {
            if (null === $this->orderId) {
               $this->orderId = new OrderId($this->id());
            }
    
            return $this->orderId;
        }
     }

#### Active Record Vs. Data Mapper for Rich Domain Models

Every project always faces the decision of which ORM should be used. There are a lot of good ORMs for PHP out there: Doctrine, Propel, Eloquent, Paris, and many more.

Most of them are [Active Record](http://www.martinfowler.com/eaaCatalog/activeRecord.html) implementations. An Active Record implementation is fine mostly for CRUD applications, but it's not the ideal solution for Rich Domain Models for the following reasons:

> *   The Active Record pattern assumes a one-to-one relation between an Entity and a database table. So it couples the design of the database to the design of the object system**.** And in a Rich Domain Model, sometimes Entities are constructed with information that may come from different data sources.
> *   Advanced things like collections and inheritance are tricky to implement.
> *   Most of the implementations force the use, through inheritance, of some sort of constructions that impose several conventions. This can lead to persistence leakage into the Domain Model by coupling the Domain Model with the ORM. The only Active Record implementation we've seen that doesn't impose inheriting from a base class is [Castle ActiveRecord](http://docs.castleproject.org/Active%20Record.MainPage.ashx) from  [Castle Project](http://www.castleproject.org/), a .NET framework. While this leads to some degree of separation between persistence and Domain concerns in the produced Entities, it doesn't decouple the low-level persistence details from high-level Domain design.

As mentioned in the previous chapter, currently the best ORM for PHP is [Doctrine](http://doctrine-project.org) , which is an implementation of the [Data Mapper pattern](http://www.martinfowler.com/eaaCatalog/dataMapper.html). Data Mapper decouples persistence concerns from Domain concerns, leading to persistence-free Entities. This makes the tool the best for someone wanting to build a Rich Domain Model.

### Client Provides Identity

Sometimes, when dealing with certain Domains, the Identities come naturally, with the client consuming the Domain Model. This is likely the ideal case, because the Identity can be modeled easily. Let's take a look at the book-selling market:

    namespace Ddd\Catalog\Domain\Model\Book;
    
    class ISBN
    {
        private $isbn;
    
        private function __construct($anIsbn)
        {
            $this->setIsbn($anIsbn);
        }
    
        private function setIsbn($anIsbn)
        {
            $this->assertIsbnIsValid($anIsbn, 'The ISBN is invalid.');
    
            $this->isbn = $anIsbn;
        }
    
        public static function create($anIsbn)
        {
            return new static($anIsbn);
        }
    
        private function assertIsbnIsValid($anIsbn, $string)
        {
            // ... Validates an ISBN code
        }
    }


According to [Wikipedia](https://en.wikipedia.org/wiki/International_Standard_Book_Number): The **International Standard Book Number** (**ISBN**) is a unique numeric commercial book identifier. An ISBN is assigned to each edition and variation (except re-printings) of a book. For example, an e-book, a paperback and a hardcover edition of the same book would each have a different ISBN. The ISBN is 13 digits long if assigned on or after 1 January 2007, and 10 digits long if assigned before 2007. The method of assigning an ISBN is nation-based and varies from country to country, often depending on how large the publishing industry is within a country.

The cool thing about the ISBN is that it's already defined in the Domain, it's a valid identifier because it's unique, and it can be easily validated. This is a good example of an Identity provided by the client:

    class Book
    {
       private $isbn;
       private $title;
    
       public function __construct(ISBN $anIsbn, $aTitle)
       {
           $this->isbn  = $anIsbn;
           $this->title = $aTitle;
       }
    } 

Now, it's just a matter of using it:

     $book = new Book(
         ISBN::create('...'),
         'Domain-Driven Design in PHP'
     ); 

> ### Note
> **`Exercise`** Think about other Domains where Identities are built in and model one.

### Application Generates Identity

If the client can't provide the Identity generally, the preferred way to handle the Identity operation is to let the application generate the Identities, usually through a UUID. This is our recommended approach in the case that you don't have a scenario as shown in the previous section.

According to [Wikipedia](https://en.wikipedia.org/wiki/Universally_unique_identifier):

The intent of UUIDs is to enable distributed systems to uniquely identify information without significant central coordination. In this context the word unique should be taken to mean _practically unique_ rather than _guaranteed unique_. Since the identifiers have a finite size, it is possible for two differing items to share the same identifier. This is a form of hash collision. The identifier size and generation process need to be selected so as to make this sufficiently improbable in practice. Anyone can create a UUID and use it to identify something with reasonable confidence that the same identifier will never be unintentionally created by anyone to identify something else. Information labeled with  UUIDs can therefore be later combined into a single database without needing to resolve identifier (ID) conflicts.

> ### Note
> There are several libraries in PHP that generate UUIDs, and they can be found at Packagist: [https://packagist.org/search/?q=uuid](https://packagist.org/search/?q=uuid). The best recommendation is the one developed by Ben Ramsey at the following  link: [https://github.com/ramsey/uuid](https://github.com/ramsey/uuid) because it has tons of watchers on GitHub and millions of installations on Packagist.

The preferred place to put the creation of the Identity would be inside a Repository (we'll go deeper into this in the [Chapter 10](../chapters/10%20Repositories.md), _Repositories_:

    namespace Ddd\Billing\Domain\Model\Order;
    
    interface OrderRepository
    {
        public function nextIdentity();
        public function add(Order $anOrder);
        public function remove(Order $anOrder);
    }


When using Doctrine, we'll need to create a custom Repository that implements such an interface. It will basically create the new Identity and use the `EntityManager` in order to persist and delete Entities. A small variation is to put the `nextIdentity` implementation into the interface that will become an abstract class:

    namespace Ddd\Billing\Infrastructure\Domain\Model\Order;
    
    use Ddd\Billing\Domain\Model\Order\Order;
    use Ddd\Billing\Domain\Model\Order\OrderId;
    use Ddd\Billing\Domain\Model\Order\OrderRepository;
    
    use Doctrine\ORM\EntityRepository;
    
    class DoctrineOrderRepository
        extends EntityRepository
        implements OrderRepository
    {
        public function nextIdentity()
        {
            return OrderId::create();
        }
    
        public function add(Order $anOrder)
        {
            $this->getEntityManager()->persist($anOrder);
        }
    
        public function remove(Order $anOrder)
        {
           $this->getEntityManager()->remove($anOrder);
        }
    }

Let's quickly review the final implementation of the `OrderId` Value Object:

    namespace Ddd\Billing\Domain\Model\Order;
    
    use Ramsey\Uuid\Uuid;
    
    class OrderId
    {
        private $id;
    
        private function __construct($anId = null)
        {
            $this->id = $id ? :Uuid::uuid4()->toString();
        }
    
        public static function create($anId = null )
        {
            return new static($anId);
        }
    }


The main concern about this approach, as you'll see in the following sections, is how easy it is to persist Entities that contain Value Objects. However, mapping embedded Value Objects that are inside an Entity can be tricky, depending on the ORM.

### Other Bounded Context Generates Identity

This is likely the most complex Identity generation strategy because it forces a local Entity to be dependent not only on local Bounded Context events, but also on external Bounded Contexts events. So in terms of maintenance, the cost would be high.

The other Bounded Context provides an interface to select the Identity from the local Entity. It can take some of the exposed properties as its own.

When synchronization is needed between the Entities of the Bounded Contexts, it can usually be achieved with an Event-Driven Architecture on each of the Bounded Contexts that need to be notified when the original Entity is changed.


Persisting Entities
-------------------

* * *

Currently, as discussed earlier in the chapter, the best tool for saving Entity state to a persistent store is Doctrine ORM. Doctrine has several ways to specify Entity metadata: by annotations in Entity code, by XML, by YAML, or by plain PHP. In this chapter, we'll discuss in depth why annotations are not the best thing to use when mapping Entities.

### Setting Up Doctrine

First of all, we need to require Doctrine through Composer. At the root folder of the project, the command below has to be executed:

     > php composer.phar require "doctrine/orm=^2.5"


Then, these lines will allow you to set up Doctrine:

    require_once '/path/to/vendor/autoload.php';
    
    use Doctrine\ORM\Tools\Setup;
    use Doctrine\ORM\EntityManager;
    
    $paths = ['/path/to/entity-files'];
    $isDevMode = false;
    
    // the connection configuration
    $dbParams = [
        'driver'   => 'pdo_mysql',
        'user'     => 'the_database_username',
        'password' => 'the_database_password',
        'dbname'   => 'the_database_name',
    ];
    
    $config = Setup::createAnnotationMetadataConfiguration($paths, $isDevMode);
    $entityManager = EntityManager::create($dbParams, $config);

### Mapping Entities

By default, Doctrine's documentation presents the code examples using annotations. So we begin the code example using annotations and discussing why they should be avoided whenever possible.

To do so, we'll bring back the `Order` class discussed earlier in this chapter.

#### Mapping Entities Using Annotated Code

When Doctrine was released, a catchy way of showing how to map objects in the code examples was by using annotations.

> ### Note
> *`What's an annotation?`** An annotation is a special form of metadata. In PHP, it's put under source code comments. For example, _PHPDocumentor_ makes use of annotations to build API information, and `PHPUnit` uses some annotations to specify data providers or to provide expectations about exceptions thrown by a piece of code:

`class SumTest extends PHPUnit_Framework_TestCase {`    `/** @dataProvider aMethodName */`    `public function testAddition() {`        `//...`     `}` `}`

In order to map the `Order` Entity to the persistence store, the source code for the `Order` should be modified to add the Doctrine annotations:

    use Doctrine\ORM\Mapping\Entity;
    use Doctrine\ORM\Mapping\Id;
    use Doctrine\ORM\Mapping\GeneratedValue;
    use Doctrine\ORM\Mapping\Column;
    
    /** @Entity */
    class Order
    {
        /** @Id @GeneratedValue(strategy="AUTO") */
        private $id;
    
        /** @Column(type="decimal", precision="10", scale="5") */
        private $amount;
    
        /** @Column(type="string") */
        private $firstName;
    
        /** @Column(type="string") */
        private $lastName;
    
        public function __construct(
            Amount $anAmount,
            $aFirstName,
            $aLastName
        ) {
            $this->amount = $anAmount;
            $this->firstName = $aFirstName;
            $this->lastName = $aLastName;
        }
    
        public function id()
        {
            return $this->id;
        }
    
        public function firstName()
        {
            return $this->firstName;
        }
    
        public function lastName()
        {
            return $this->lastName;
        }
    
        public function amount()
        {
            return $this->amount;
        }
    }


Then, to persist the Entity to the persistent store, it's just as easy to do the following:

    $order = new Order(
        new Amount(15, Currency::EUR()),
        'AFirstName',
        'ALastName'
    );
    $entityManager->persist($order);
    $entityManager->flush();


At first glance, this code looks simple, and this can be an easy way to specify mapping information. But it comes at a cost. What's odd about the final code?

First of all, Domain concerns are mixed with Infrastructure concerns. Order is a Domain concept, whereas Table, Column, and so on are infrastructure concerns.

As a result, this Entity is tightly coupled to the mapping information specified by the annotations in the source code. If the Entity were required to be persisted using another Entity manager and with different mapping metadata, this wouldn't be possible.

Annotations tend to lead to side effects and tight coupling, so it would be better to not use them.

So what's the best way to specify mapping information? The best way is the one that allows you to separate the mapping information from the Entity itself. This can be achieved by using XML mapping, YAML mapping, or PHP mapping. In this book, we're going to cover XML mapping.

#### Mapping Entities Using XML

To map the `Order` Entity using the XML mapping, the setup code of Doctrine should be altered slightly:

    require_once '/path/to/vendor/autoload.php';
    
    use Doctrine\ORM\Tools\Setup;
    use Doctrine\ORM\EntityManager;
    
    $paths = ['/path/to/mapping-files'];
    $isDevMode = false;
    
    // the connection configuration
    $dbParams = [
        'driver'   => 'pdo_mysql',
        'user'     => 'the_database_username',
        'password' => 'the_database_password',
        'dbname'   => 'the_database_name',
    ];
    
    $config = Setup::createXMLMetadataConfiguration($paths, $isDevMode);
    $entityManager = EntityManager::create($dbParams, $config);


The mapping file should be created on the path where Doctrine will search for the mapping files, and the mapping files should be named after the fully qualified class name, replacing the backslashes `\` with dots. Consider the following:

    Acme\Billing\Domain\Model\Order

The preceding illustration would have the mapping file named like this:

    Acme.Billing.Domain.Model.Order.dcm.xml

Additionally, it's convenient that all the mapping files use a special XML Schema created specially for specifying mapping information:

    https://raw.github.com/doctrine/doctrine2/master/doctrine-mapping.xsd

#### Mapping Entity Identity

Our Identity, `OrderId`, is a Value Object. As seen in the previous chapter, there are different approaches for mapping a Value Object using Doctrine, embeddables, and custom types. When Value Objects are used as Identities, the best option is custom types.

An interesting new feature in _Doctrine 2.5_ is that it's now possible to use Objects as identifiers for Entities, so long as they implement the magic method `__toString()`. So we can add  `__toString` to our Identity Value Objects and use them in our mappings:

    namespace Ddd\Billing\Domain\Model\Order;
    
    use Ramsey\Uuid\Uuid;
    
    class OrderId
    {
        // ...
    
        public function __toString()
        {
            return $this->id;
        }
    }


Check the implementation of the Doctrine custom types. They inherit from `GuidType`, so their internal representation will be a UUID. We need to specify the database native translation. Then we need to register our custom types before we use them. If you need help with these steps, [Custom Mapping Types](http://doctrine-orm.readthedocs.io/projects/doctrine-orm/en/latest/cookbook/custom-mapping-types.html) is a good reference.

    use Doctrine\DBAL\Platforms\AbstractPlatform;
    use Doctrine\DBAL\Types\GuidType;
    
    class DoctrineOrderId extends GuidType
    {
        public function getName()
        {
            return 'OrderId';
        }
    
        public function convertToDatabaseValue(
            $value, AbstractPlatform $platform
        ) {
            return $value->id();
        }
    
        public function convertToPHPValue(
            $value, AbstractPlatform $platform
        ) {
            return new OrderId($value);
        }
    }

Lastly, we'll set up the registration of custom types. Again, we have to update our bootstrapping:

    require_once '/path/to/vendor/autoload.php';
    
    // ...
    
    \Doctrine\DBAL\Types\Type::addType(
         'OrderId',
         'Ddd\Billing\Infrastructure\Domain\Model\DoctrineOrderId'
    );
    
    $config = Setup::createXMLMetadataConfiguration($paths, $isDevMode);
    $entityManager = EntityManager::create($dbParams, $config);


#### Final Mapping File

With all the changes, we're finally ready, so let's take a look at the final mapping file. The most interesting detail is to check how the id gets mapped with our defined custom type for `OrderId`:

    <?xml version="1.0" encoding="UTF-8"?>
    <doctrine-mapping
        xmlns="http://doctrine-project.org/schemas/orm/doctrine-mapping"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="
            http://doctrine-project.org/schemas/orm/doctrine-mapping
        https://raw.github.com/doctrine/doctrine2/master/doctrine-mapping.xsd">
    
        <entity
            name="Ddd\Billing\Domain\Model\Order"
            table="orders">
    
            <id name="id" column="id" type="OrderId" />
    
            <field
                name="amount"
                type="decimal"
                nullable="false"
                scale="10"
                precision="5"
            />
            <field
                name="firstName"
                type="string"
                nullable="false"
            />
            <field
                name="lastName"
                type="string"
                nullable="false"
            />
        </entity>
    </doctrine-mapping>




Testing Entities
----------------

* * *

It's relatively easy to test Entities, simply because they're plain old PHP classes with actions derived from the Domain concept they represent. The focus of the test should be the invariants that the Entity protects, because the behavior on the Entities will likely be modeled around those invariants.

For example, and for the sake of simplicity, suppose a Domain Model for a blog is needed. A possible one could be this:

    class Post
    {
        private $title;
        private $content;
        private $status;
        private $createdAt;
        private $publishedAt;
    
        public function __construct($aContent, $title)
        {
            $this->setContent($aContent);
            $this->setTitle($title);
    
            $this->unpublish();
            $this->createdAt(new DateTimeImmutable());
        }
    
        private function setContent($aContent)
        {
            $this->assertNotEmpty($aContent);
    
            $this->content = $aContent;
        }
    
        private function setTitle($aPostTitle)
        {
            $this->assertNotEmpty($aPostTitle);
    
            $this->title = $aPostTitle;
        }
    
        private function setStatus(Status $aPostStatus)
        {
            $this->assertIsAValidPostStatus($aPostStatus);
    
            $this->status = $aPostStatus;
        }
    
        private function createdAt(DateTimeImmutable $aDate)
        {
            $this->assertIsAValidDate($aDate);
    
            $this->createdAt = $aDate;
        }
    
        private function publishedAt(DateTimeImmutable $aDate)
        {
            $this->assertIsAValidDate($aDate);
    
            $this->publishedAt = $aDate;
        }
    
        public function publish()
        {
            $this->setStatus(Status::published());
            $this->publishedAt(new DateTimeImmutable());
        }
    
        public function unpublish()
        {
            $this->setStatus(Status::draft());
            $this->publishedAt = null ;
        }
    
        public function isPublished()
        {
            return $this->status->equalsTo(Status::published());
        }
    
        public function publicationDate()
        {
            return $this->publishedAt;
        }
    }
    
    class Status
    {
        const PUBLISHED = 10;
        const DRAFT = 20;
    
        private $status;
    
        public static function published()
        {
            return new self(self::PUBLISHED);
        }
    
        public static function draft()
        {
            return new self(self::DRAFT);
        }
    
        private function __construct($aStatus)
        {
            $this->status = $aStatus;
        }
    
        public function equalsTo(self $aStatus)
        {
            return $this->status === $aStatus->status;
        }
    }


In order to test this Domain Model, we must ensure the test covers all the `Post` invariants:

    class PostTest extends PHPUnit_Framework_TestCase
    {
         /** @test */
         public function aNewPostIsNotPublishedByDefault()
         {
              $aPost = new Post(
                  'A Post Content',
                  'A Post Title'
              );
    
              $this->assertFalse(
                  $aPost->isPublished()
              );
    
              $this->assertNull(
                  $aPost->publicationDate()
              );
          }
    
        /** @test */
        public function aPostCanBePublishedWithAPublicationDate()
        {
            $aPost = new Post(
                'A Post Content',
                'A Post Title'
            );
    
            $aPost->publish();
    
            $this->assertTrue(
                $aPost->isPublished()
            );
    
            $this->assertInstanceOf(
                'DateTimeImmutable',
                $aPost->publicationDate()
            );
        }
    }

### DateTimes

Because `DateTimes` are widely used in Entities, we think it's important to point out specific approaches on unit testing Entities that have fields with date types. Consider that a `Post` is new if it was created within the last 15 days:

    class Post
    {
        const NEW_TIME_INTERVAL_DAYS = 15;
    
        // ...
        private $createdAt;
    
        public function __construct($aContent, $title)
        {
            // ...
            $this->createdAt(new DateTimeImmutable());
        }
    
        // ...
    
        public function isNew()
        {
            return
                (new DateTimeImmutable())
                     ->diff($this->createdAt)
                     ->days <= self::NEW_TIME_INTERVAL_DAYS;
        }
    }


The `isNew()` method needs to compare two `DateTimes;` it's a comparison between the date when the Post was created and today's date. We compute the difference and check if it's less than the specified amount of days. How do we unit test the `isNew()` method? As we demonstrated in the implementation, it's difficult to reproduce specific flows in our test suites. What options do we have?

#### Passing All Dates as Parameters

One possible option could be passing all the dates as parameters when needed:

    class Post
    {
        // ...
    
        public function __construct($aContent, $title, $createdAt = null)
        {
            // ...
            $this->createdAt($createdAt ?: new DateTimeImmutable());
        }
    
        // ...
    
        public function isNew($today = null)
        {
            return
                ($today ? :new DateTimeImmutable())
                    ->diff($this->createdAt)
                    ->days <= self::NEW_TIME_INTERVAL_DAYS;
        }
    }


This is the easiest approach for unit testing purposes. Just pass different pairs of dates to test all possible scenarios with 100 percent coverage. However, if you consider the client code that's creating and asking for the `isNew()` method result, things don't look so nice. The resulting code can be a bit weird because of always passing today's `DateTime`:

    $aPost = new Post(
        'Hello world!',
        'Hi',
        new DateTimeImmutable()
    );
    
    $aPost->isNew(
        new DateTimeImmutable()
    );


#### Test Class

Another alternative is to use the Test Class pattern. The idea is to extend the `Post` class with a new one that we can manipulate to force specific scenarios. This new class is going to be used only for unit testing purposes. The bad news is that we have to modify the original `Post` class a bit, extracting some methods and changing some fields and methods from `private` to `protected`. Some developers may worry about increasing the visibility of class properties just because of testing reasons. However, we think that in most cases, it's worth it:

    class Post
    {
        protected $createdAt;
    
        public function isNew()
        {
            return
                ($this->today())
                    ->diff($this->createdAt)
                    ->days <= self::NEW_TIME_INTERVAL_DAYS;
        }
    
        protected function today()
        {
            return new DateTimeImmutable();
        }
    
        protected function createdAt(DateTimeImmutable $aDate)
        {
            $this->assertIsAValidDate($aDate);
    
            $this->createdAt = $aDate;
        }
    }


As you can see, we've extracted the logic for getting today's date into the `today()` method. This way, by applying the Template Method pattern, we can change its behavior from a derived class. Something similar happens with the `createdAt` method and field. Now they're protected, so they can be used and overridden in derived classes:

    class PostTestClass extends Post
    {
        private $today;
    
        protected function today()
        {
           return $this->today;
        }
    
        public function setToday($today)
        {
           $this->today = $today;
        }
    }


With these changes, we can now test our original `Post` class through testing `PostTestClass`:

    class PostTest extends PHPUnit_Framework_TestCase
    {
        // ...
    
        /** @test */
        public function aPostIsNewIfIts15DaysOrLess()
        {
            $aPost = new PostTestClass(
                'A Post Content' ,
                'A Post Title'
            );
    
            $format = 'Y-m-d';
            $dateString = '2016-01-01';
            $createdAt = DateTimeImmutable::createFromFormat(
                $format,
                $dateString
            );
    
            $aPost->createdAt($createdAt);
            $aPost->setToday(
                $createdAt->add(
                    new DateInterval('P15D')
                )
            );
    
            $this->assertTrue(
                $aPost->isNew()
            );
    
            $aPost->setToday(
                $createdAt->add(
                   new DateInterval('P16D')
                )
            );
    
            $this->assertFalse(
                $aPost->isNew()
            );
        }
    }

Just one last small detail: with this approach, it's impossible to achieve 100 percent coverage on the `Post` class, because the `today()` method is never going to be executed. However, it can be covered by other tests.

#### External Fake

Another option is to wrap calls to the `DateTimeImmutable` constructor or named constructors using a new class and some static methods. In doing so, we can statically change the result of those methods to behave differently based on specific testing scenarios:

    class Post
    {
        // ...
        private $createdAt;
    
        public function __construct($aContent, $title)
        {
            // ...
            $this->createdAt(MyCustomDateTimeBuilder::today());
        }
    
        // ...
    
        public function isNew()
        {
            return
                (MyCustomDateTimeBuilder::today())
                    ->diff($this->createdAt)
                    ->days <= self::NEW_TIME_INTERVAL_DAYS;
        }
    }


For getting today's `DateTime`, we now use a static call to `MyCustomDateTimeBuilder::today()`. This class also has some setter methods to fake the result to return in the next calls:

    class PostTest extends PHPUnit_Framework_TestCase
    {
        // ...
    
        /** @test */
        public function aPostIsNewIfIts15DaysOrLess()
        {
            $createdAt = DateTimeImmutable::createFromFormat(
                'Y-m-d',
                '2016-01-01'
            );
    
            MyCustomDateTimeBuilder::setReturnDates(
                [
                    $createdAt,
                    $createdAt->add(
                        new DateInterval('P15D')
                    ),
                    $createdAt->add(
                        new DateInterval('P16D')
                    )
                ] 
            );
    
            $aPost = new Post(
                'A Post Content' ,
                'A Post Title'
            );
    
            $this->assertTrue(
                $aPost->isNew()
            );
    
            $this->assertFalse(
                $aPost->isNew()
            );
        } 
    }

The main problem with this approach is it's coupled statically with an object. Depending on your use case, it'll also be tricky to create a flexible fake object.

#### Reflection

You can also use reflection techniques for building a new `Post` class with custom dates. Consider [Mimic](https://github.com/keyvanakbary/mimic), a simple functional library for object prototyping, data hydration, and data exposition:

    namespace Domain;
    
    use mimic as m;
    
    class ComputerScientist {
        private $name;
        private $surname;
    
        public function __construct($name, $surname) 
        {
            $this->name = $name;
            $this->surname = $surname;
        }
    
        public function rocks() 
        {
            return $this->name . ' ' . $this->surname . ' rocks!';
        }
    }
    
    assert(m\prototype('Domain\ComputerScientist')
        instanceof Domain\ComputerScientist);
    
    m\hydrate('Domain\ComputerScientist', [
        'name'   =>'John' ,
        'surname'=>'McCarthy'
    ])->rocks(); //John McCarthy rocks!
    
    assert(m\expose(
        new Domain\ComputerScientist('Grace', 'Hopper')) ==
        [
            'name'    => 'Grace' ,
            'surname' => 'Hopper'
        ]
    )

> ### Note
> **`Share and Discuss`** Discuss with your workmates how to properly unit test your Entities with fixed `DateTimes` and come up with additional alternatives.

If you want to know more about testing patterns and approaches, take a look at the book _xUnit Test Patterns: Refactoring Test Code_ by Gerard Meszaros.


Validation
----------

* * *

Validation is a highly important process in our Domain Model. It checks not only for the correctness of attributes, but also for that of entire objects and the composition of those objects. Different levels of validation are required in order to keep this Model in a valid state. Just because an object consists of valid attributes (on a per basis) doesn't necessarily mean the object (as a whole) is valid. And the opposite is true: valid objects don't necessarily equal valid compositions.

### Attribute Validation

Some people understand validation as the process whereby a service validates the state of a given object. In this case, the validation conforms to a [Design-by-contract](http://en.wikipedia.org/wiki/Design_by_contract) approach, which consists of preconditions, postconditions, and invariants. One such way to protect a single attribute is by using [Chapter 3](../chapters/03%20Value%20Objects.md), _Value Objects_. In order to make our design more flexible for change, we focus only on asserting Domain preconditions that must be met. Here, we'll be using guards as an easy way of validating the preconditions:

    class Username
    {
        const MIN_LENGTH = 5;
        const MAX_LENGTH = 10;
        const FORMAT = '/^[a-zA-Z0-9_]+$/';
    
        private $username;
    
        public function __construct($username)
        {
            $this->setUsername($username);
        }
    
        private setUsername($username)
        {
            $this->assertNotEmpty($username);
            $this->assertNotTooShort($username);
            $this->assertNotTooLong($username);
            $this->assertValidFormat($username);
            $this->username = $username;
        }
    
        private function assertNotEmpty($username)
        {
            if (empty($username)) {
                throw new InvalidArgumentException('Empty username');
            }  
        }
    
        private function assertNotTooShort($username)
        {
            if (strlen($username) < self::MIN_LENGTH) {
                throw new InvalidArgumentException(sprintf(
                    'Username must be %d characters or more',
                    self::MIN_LENGTH
                ));
            }
        }
    
        private function assertNotTooLong($username)
        {
            if (strlen( $username) > self::MAX_LENGTH) {
                throw new InvalidArgumentException(sprintf(
                    'Username must be %d characters or less',
                    self::MAX_LENGTH
                ));
            }
        }
    
        private function assertValidFormat($username)
        {
            if (preg_match(self:: FORMAT, $username) !== 1) {
                throw new InvalidArgumentException(
                    'Invalid username format'
                );
            }
        }
    }

As you can see in the example above, there are four preconditions that must be satisfied in order to build a Username Value Object. It:

*   Must not be empty
*   Must be at least 5 characters
*   Must be less than 10 characters
*   Must follow a format of alphanumeric characters or underscores

If all the preconditions are met, the attribute will be set and the object will be successfully built. Otherwise, an `InvalidArgumentException` will be raised, execution will be halted, and the client will be shown an error.

Some developers may consider this kind of validation defensive programming. However, we're not checking that the input is a string or that nulls are not permitted. We can't avoid people using our code incorrectly, but we can control the correctness of our Domain state. As seen in the [Chapter 3](../chapters/03%20Value%20Objects.md), _Value Objects_, validation can help us with security too.

[Defensive programming](https://en.wikipedia.org/wiki/Defensive_programming) isn't a bad thing. In general, it makes sense when developing components or libraries that are going to be used as a third party in other projects. However, when developing your own Bounded Context, those extra paranoid checks (nulls, basic types, type hinting, and so  on.) can be avoided to increase development speed by relying on the coverage of your unit test suite.

### Entire Object Validation

There are times when an object composed of valid properties, as a whole, can still be deemed invalid. It can be tempting to add this kind of validation to the object itself, but generally this is an anti-pattern. Higher-level validation changes at a rhythm different than that of the object logic itself. Also, it's good practice to separate these responsibilities.

The validation informs the client about any errors that have been found or collects the results to be reviewed later, as sometimes we don't want to stop the execution at the first sign of trouble.

An `abstract` and reusable `Validator` could be something like this:

    abstract class Validator
    {
        private $validationHandler;
    
        public function __construct(ValidationHandler $validationHandler)
        {
            $this->validationHandler = $validationHandler;
        }
    
        protected function handleError($error)
        {
            $this->validationHandler->handleError($error);
        }
    
        abstract public function validate();
    }

As a concrete example, we want to validate an entire `Location` object, composed of valid Country, City, and Postcode Value Objects. However, these individual values might be in an invalid state at the time of validation. Maybe the city doesn't form part of the country, or maybe the postcode doesn't follow the city format:

    class Location
    {
        private $country;
        private $city;
        private $postcode;
    
        public function __construct(
            Country $country, City $city, Postcode $postcode
        ) {
            $this->country = $country;
            $this->city = $city;
            $this->postcode = $postcode;
        }
    
        public function country()
        {
            return $this->country;
        }
    
        public function city()
        {
            return $this->city;
        }
    
        public function postcode()
        {
            return $this->postcode;
        }
    }


The validator checks the state of the `Location` object in its entirety, analyzing the meaning of the relationships between properties:

    class LocationValidator extends Validator
    {
        private $location;
    
        public function __construct(
            Location $location, ValidationHandler $validationHandler
        ) {
            parent:: __construct($validationHandler);
            $this->location = $location;
        }
    
        public function validate()
        {
            if (!$this->location->country()->hasCity(
                $this->location->city()
            )) {
                $this->handleError('City not found');
            }
    
            if (!$this->location->city()->isPostcodeValid(
                $this->location->postcode()
            )) {
                $this->handleError('Invalid postcode');
            }
        }
    }

Once all the properties have been set, we're able to validate the Entity, most likely after some described process. On the surface, it looks as if the Location validates itself. However, this isn't the case. The  `Location` class delegates this validation to a concrete validator instance, splitting these two clear responsibilities:

    class Location
    {
        // ...
    
        public function validate(ValidationHandler $validationHandler)
        {
         $validator = new LocationValidator($this, $validationHandler);
         $validator->validate();
        }
    }


#### Decoupling Validation Messages

With some minor changes to our existing implementation, we're able to decouple the validation messages from the validator:

    class LocationValidationHandler implements ValidationHandler
    {
        public function handleCityNotFoundInCountry();
    
        public function handleInvalidPostcodeForCity();
    }
    
    class LocationValidator
    {
        private $location;
        private $validationHandler;
    
        public function __construct(
            Location $location,
            LocationValidationHandler $validationHandler
        ) {
            $this->location = $location;
            $this->validationHandler = $validationHandler;
        }
    
        public function validate()
        {
            if (!$this->location->country()->hasCity(
                $this->location->city()
            )) {
                $this->validationHandler->handleCityNotFoundInCountry();
            } 
    
            if (! $this->location->city()->isPostcodeValid(
                $this->location->postcode()
            )) {
                $this->validationHandler->handleInvalidPostcodeForCity();
            }
        }
    }


We also need to change the signature of the validation method to the following:

    class Location
    {
       // ...
    
        public function validate(
            LocationValidationHandler $validationHandler
        ) {
            $validator = new LocationValidator($this, $validationHandler);
            $validator->validate();
        }
    }


### Validating Object Compositions

Validating object compositions can be complicated. As such, the preferred way of achieving this is through a Domain Service. The service then communicates with Repositories in order to retrieve the valid Aggregate. Due to the likely complex object graphs that can be created, an Aggregate could be in an intermediate state, requiring other Aggregates to be validated beforehand. We can use Domain Events to notify other parts of the system that a particular element has been validated.


Entities and Domain Events
--------------------------

* * *

We'll explore [Chapter 6](../chapters/06%20Domain-Events.md), _Domain-Events_ in future chapters; however, it's important to highlight that operations performed on Entities can fire Domain Events. This approach is used to communicate the Domain change to other parts of the Application, or even to other Applications, as you'll see in [Chapter 12](../chapters/12%20Integrating%20Bounded%20Contexts.md), _Integrating Bounded Contexts_:

    class Post
    {
       // ...
    
        public function publish()
        {
            $this->setStatus(
                Status::published()
            );
    
            $this->publishedAt(new DateTimeImmutable());
    
            DomainEventPublisher::instance()->publish(
                new PostPublished($this->id)
            );
        }
    
        public function unpublish()
        {
            $this->setStatus(
                Status::draft()
            );
    
            $this-> publishedAt = null;
    
            DomainEventPublisher::instance()->publish(
                new PostUnpublished($this->id)
            );
        }
    
        // ...
    }

Domain Events can even be fired when a new instance of our Entity is created:

    class User
    {
        // ...
    
        public function __construct(UserId $userId, $email, $password)
        {
            $this->setUserId($userId);
            $this->setEmail($email);
            $this->setPassword($password);
    
            DomainEventPublisher::instance()->publish(
                new UserRegistered($this->userId)
            );
        }
    }




Wrap-Up
-------

* * *

Some concepts in the Domain demand Identity — that is, changes to their internal states don't change their own unique identities. We've seen how modeling Identity as a Value Object brings benefits like immutability, in addition to logic for operating the Identity itself. We've also shown several ways of providing Identity, restated in the following pointers:

*   Persistence mechanism: Easy to implement, but you won't have the Identity before persisting the object, which delays and complicates event propagation.
*   Surrogate ID: Some ORMs require an extra field on your Entity to map the Identity with the persisting mechanism.
*   Provided by the client: Sometimes the Identity fits a Domain concept and you can model it inside your Domain.
*   Generated by the application: You can use a library to generate IDs.
*   Generated by a Bounded Context: Probably the most complex strategy. Other Bounded Contexts provide an interface for generating Identities.

We've seen and discussed Doctrine as a persistence mechanism, we've looked at the drawbacks of using the Active Record pattern, and finally, we've checked different levels of Entity validation:

*   Attribute validation: Check for specifics inside the object state through preconditions, postconditions, and invariants.
*   Entire object validation: Look for consistency of an object as a whole. Extracting the validation to an external service is a good practice.
*   Object compositions: Complex object compositions could be validated through Domain Services. A good way of communicating this to the rest of the application is through Domain Events.
