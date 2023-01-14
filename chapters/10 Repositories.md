Chapter 10. Repositories
------------------------

In order to interact with a Domain object, you need to hold a reference to it. One way of achieving this is by creation. Alternatively, you can traverse an association. In Object-Oriented programs, objects have links (references) to other objects, which makes them easily traversable, thereby contributing to the expressive power of our models. But here's the catch: you need a mechanism to retrieve the first object, the Aggregate Root.

Repositories act as storage locations, where a retrieved object is returned in the exact same state it was persisted in. In Domain-Driven Design, every Aggregate type typically has a unique associated Repository, which is used for its persistence and fetching needs. However, in the case where it's required to share an Aggregate object hierarchy, the types might share a Repository.

Once you've successfully retrieved the Aggregate from the Repository, every change you make is persisted, which removes the need to go back to the Repository.



Definition
----------

* * *

Martin Fowler defines a Repository as:

The mechanism between the domain and data mapping layers, acting like an in-memory domain object collection. Client objects construct query specifications declaratively and submit them to Repository for satisfaction. Objects can be added to and removed from the Repository, as they can from a simple collection of objects, and the mapping code encapsulated by the Repository will carry out the appropriate operations behind the scenes. Conceptually, a Repository encapsulates the set of objects persisted in a data store and the operations performed over them, providing a more object-oriented view of the persistence layer. Repository also supports the objective of achieving a clean separation and one-way dependency between the domain and [data mapping layers](http://martinfowler.com/eaaCatalog/repository.html).



Repositories Are Not DAOs
-------------------------

* * *

**Data Access Objects** (**DAOs**) are a common pattern for persisting Domain objects into the database. It's easy to confuse the DAO pattern with a Repository. The significant difference is that Repositories represent collections, while DAOs are closer to the database and are often far more table-centric. Typically, a DAO would contain CRUD methods for a particular Domain object. Let's see how a common interface for a DAO might look:

    interface UserDAO 
    { 
        /** 
         * @param string $username 
         * @return User 
         */ 
        public function get($username);
    
        public function create(User $user);
    
        public function update(User $user);
    
        /**
         * @param string $username
         */
        public function delete($username);
    }

A DAO interface could have multiple implementations, which could range from using ORM constructions to using plain SQL queries. The main problem with DAOs is that their responsibilities are not clearly defined. DAOs are usually perceived as gateways to the database, so it's relatively easy to greatly decrease cohesion with many specific methods in order to query the database:

    interface BloatUserDAO 
    { 
        public function get($username);
    
        public function create(User $user);
    
        public function update(User $user);
    
        public function delete($username);
    
        public function getUserByLastName($lastName);
    
        public function getUserByEmail($email);
    
        public function updateEmailAddress($username, $email);
    
        public function updateLastName($username, $lastName);
    }

As you can see, the more we add new methods to implement, the harder it becomes to unit test the DAO, and it becomes increasingly coupled to the User object. This problem will grow over time, with many other contributors collaborating in making the Big Ball of Mud even bigger.



Collection-Oriented Repositories
--------------------------------

* * *

Repositories mimic a collection by implementing their common interface characteristics. As a collection, a Repository shouldn't leak any intentions of persistence behavior, such as the notion of saving to a store.

The underlying persistence mechanism has to support this need. You shouldn't be required to handle changes to the objects over their lifetime. The collection references the most recent changes to the object, meaning that upon each access, you get the latest object state.

Repositories implement a concrete collection type, the Set. A Set is a data structure with an invariant that doesn't contain duplicate entries. If you try to add an element that's already present to a Set, it won't be added. This is useful in our use case, as each Aggregate has a unique identity that's associated with the Root Entity.

Consider, for example, that we have the following Domain Model:

    namespace Domain\Model;
    
    class Post
    {
        const EXPIRE_EDIT_TIME = 120; // seconds
    
        private $id;
        private $body;
        private $createdAt;
    
        public function __construct(PostId $anId, Body $aBody) 
        {
            $this->id = $anId;
            $this->body = $aBody;
            $this->createdAt = new \DateTimeImmutable();
        }
    
        public function editBody(Body $aNewBody)
        {
            if($this->editExpired()) {
                throw new RuntimeException('Edit time expired');
            }
    
            $this->body = $aNewBody;
        }
    
        private function editExpired()
        {
            $expiringTime= $this->createdAt->getTimestamp() +
                self::EXPIRE_EDIT_TIME;
    
            return $expiringTime < time();
        }
    
        public function id()
        {
            return $this->id;
        }
    
        public function body()
        {
           return $this->body;
        }
    
        public function createdAt()
        {
           return $this->createdAt;
        }
    }
    
    class Body
    {
        const MIN_LENGTH = 3;
        const MAX_LENGTH = 250;
    
        private $content;
    
        public function __construct($content)
        {
            $this->setContent(trim($content));
        }
    
        private function setContent($content)
        {
            $this->assertNotEmpty($content);
            $this->assertFitsLength($content);
    
            $this->content = $content;
        }
    
        private function assertNotEmpty($content)
        {
            if(empty($content)) {
                throw new DomainException('Empty body');
            }
        }
    
        private function assertFitsLength($content)
        {
            if(strlen($content) < self::MIN_LENGTH) {
                throw new DomainException('Body is too short');
            }
    
            if(strlen($content) > self::MAX_LENGTH) {
                throw new DomainException('Body is too long');
            }
        }
    
        public function content()
        {
            return $this->content;
        }
    }
    
    class PostId
    {
        private $id;
    
        public function __construct($id = null)
        {
            $this->id = $id ?: uniqid();
        }
    
        public function id()
        {
            return $this->id;
        }
    
        public function equals(PostId $anId)
        {
           return $this->id === $anId->id();
        }
    }

If we wanted to persist this `Post` Entity, a simple in-memory `Post` Repository could be created like this:

    class SimplePostRepository 
    {   
        private $post = [];
    
        public add(Post $aPost)
        {
            $this->posts[(string) $aPost->id()] = $aPost;
        }
    
        public function postOfId(PostId $anId)
        {
            if (isset($this->posts[(string) $anId])) {
                return $this->posts[(string) $anId];
            }
    
            return null;
        }
    }

And, as you would expect, it's handled as a collection:

    $id = new PostId();
    $repository = new SimplePostRepository();
    $repository->add(new Post($id, 'Random content'));
    
    // later ...
    $post = $repository->postOfId($id);
    $post->editBody('Updated content');
    
    // even later ...
    $post = $repository->postOfId($id);
    assert('Updated content' === $post->body());

As you can see, from the collection's point of view, there's no need for a save method in the Repository. Changes affecting the object are correctly handled by the underlying persistence layer. Collection-oriented Repositories are the ones that don't need to add an Aggregate that was persisted before. This mainly happens with the Repositories that are memory based, but we also have ways to do this with the Persisted-Oriented Repositories. We'll look at this in a moment; additionally, we'll cover this more in depth in the [Chapter 11]( ../chapters/11%20Application.md), _Application_.

The first step to design a Repository is to define a collection-like interface for it. The interface needs to define the usual collection methods, like so:

    interface PostRepository 
    { 
        public function add(Post $aPost);
        public function addAll(array $posts); 
        public function remove(Post $aPost); 
        public function removeAll(array $posts); 
        // ... 
    }

For implementing such an interface, you could also use an abstract class. In general, when we talk about an interface, we refer to the general concept and not just the specific PHP interface. To keep your design simple, don't add methods you don't need; the Repository interface definition and its corresponding Aggregate should be placed in the same Module.

Sometimes `remove` doesn't physically delete the Aggregate from the database. This strategy - where the Aggregate has a status field that's updated to a _deleted_ value - is known as a _soft delete_. Why is this approach interesting? It can be interesting for auditing changes and performance. In those cases, you can instead mark the Aggregate as disabled or _logically removed_. The interface could be updated accordingly by removing the removal methods or providing disable behavior in the Repository.

Another important aspect of Repositories are the finder methods, like the following:

    interface PostRepository 
    { 
        // ... 
    
        /**
         * @return Post
         */
        public function postOfId(PostId $anId);
    
        /**
         * @return Post[]
         */
        public function latestPosts(DateTimeImmutable $sinceADate);
    }

As we suggested in [Chapter 4](../chapters/04%20Entities.md), _Entities_, we prefer Application-Generated Identities. The best place to generate a new Identity for an Aggregate is its Repository. So to retrieve the globally unique ID for a `Post`, a logical place to include it is in `PostRepository`:

    interface PostRepository
    { 
        // ...
    
        /**
         * @return PostId
         */
        public function nextIdentity();
    }

The code responsible for building up each `Post` instance calls `nextIdentity` to get a unique identifier, `PostId`:

    $post = newPost($postRepository->nextIdentity(), $body);

Some developers favor placing the implementation close to the interface definition as a subpackage of the Module. However, because we want a clear Separation of Concerns, we recommend instead placing it inside the Infrastructure layer.

### In-Memory Implementation

As Uncle Bob wrote in [Screaming Architecture](http://blog.8thlight.com/uncle-bob/2011/09/30/Screaming-Architecture.html):

A good software architecture allows decisions about frameworks, databases, web-servers, and other environmental issues and tools, to be deferred and delayed. A good architecture makes it unnecessary to decide on Rails, or Spring, or Hibernate, or Tomcat or MySql, until much later in the project. A good architecture makes it easy to change your mind about those decisions too. A good architecture emphasizes the use-cases and decouples them from peripheral concerns.

In the early stages of your application, a fast in-memory implementation could come in handy. It's something you could use to mature other parts of your system, allowing you to delay database decisions to the correct moment. An in-memory Repository is simple, fast, and easy to implement.

For our `Post` Repository, an in-memory hash map is enough to provide all the functionality we need:

    namespace Infrastructure\Persistence\InMemory;
    
    use Domain\Model\Post;
    use Domain\Model\PostId;
    use Domain\Model\PostRepository;
    
    class InMemoryPostRepository implements PostRepository
    {
        private $posts = [];
    
        public function add(Post $aPost)
        {
           $this->posts[$aPost->id()->id()] = $aPost;
        }
    
        public function remove(Post $aPost)
        {
            unset($this->posts[$aPost->id()->id()]);
        }
    
        public function postOfId(PostId $anId)
        {
            if (isset($this->posts[$anId->id()])) {
                return $this->posts[$anId->id()];
            }
    
            return null;
        }
    
        public function latestPosts(\DateTimeImmutable $sinceADate)
        {
            return $this->filterPosts(
               function (Post $post) use($sinceADate) {
                   return $post->createdAt() > $sinceADate;
               }
            );
        }
    
        private function filterPosts(callable $fn)
        {
            return array_values(array_filter($this->posts, $fn));
        }
    
        public function nextIdentity()
        {
             return new PostId();
        }
    }

### Doctrine ORM

We've talked about [Doctrine](http://www.doctrine-project.org/) in past chapters quite a bit. Doctrine is a set of libraries for database storage and object mapping. It comes bundled with the popular [Symfony2 web framework](http://symfony.com/) by default and, among other features, it allows you to easily decouple your application from the persistence layer, thanks to the [Data Mapper pattern](http://martinfowler.com/eaaCatalog/dataMapper.html).

Meanwhile, the ORM stands over a powerful database abstraction layer that enables database interaction through an SQL dialect called **Doctrine Query Language** (**DQL**), which is inspired by the famous Java Hibernate framework.

If we're going to use Doctrine ORM, the first task to complete is adding the dependencies to our project through [Composer](https://getcomposer.org/):

    composer require doctrine/orm 

#### Object Mapping

The mapping between your Domain objects and the database can be considered an implementation detail. The Domain lifecycle shouldn't be aware of these persistence details. As such, the mapping information should be defined as part of the Infrastructure layer, outside the Domain, and as the implementation for the Repositories.

##### Doctrine Custom Mapping Types

As our `Post` Entity is composed of Value Objects like `Body` or `PostId`, it's a good idea to make Custom Mapping Types or use Doctrine Embeddables for them, as seen in the Value Objects chapter. This will make the object mapping considerably easier:

    namespace Infrastructure\Persistence\Doctrine\Types;
    
    use Doctrine\DBAL\Types\Type;
    use Doctrine\DBAL\Platforms\AbstractPlatform;
    use Domain\Model\Body;
    
    class BodyType extends Type
    {
        public function getSQLDeclaration(
            array $fieldDeclaration, AbstractPlatform $platform
        ) {
            return $platform->getVarcharTypeDeclarationSQL(
                $fieldDeclaration
            );
        }
    
        /**
         * @param string $value
         * @return Body
         */
        public function convertToPHPValue(
            $value, AbstractPlatform $platform
        ) {
            return new Body($value);
        }
    
        /**
         * @param Body $value
         */
        public function convertToDatabaseValue(
            $value, AbstractPlatform $platform
        ) {
            return $value->content();
        }
    
        public function getName()
        {
            return 'body';
        }
    }

    namespace Infrastructure\Persistence\Doctrine\Types;
    
    use Doctrine\DBAL\Types\Type;
    use Doctrine\DBAL\Platforms\AbstractPlatform;
    use Domain\Model\PostId;
    
    class PostIdType extends Type
    {
        public function getSQLDeclaration(
            array $fieldDeclaration, AbstractPlatform $platform
        ) {
            return $platform->getGuidTypeDeclarationSQL(
                $fieldDeclaration
            );
        }
    
        /**
         * @param string $value
         * @return PostId
         */
        public function convertToPHPValue(
            $value, AbstractPlatform $platform
        ) {
            return new PostId($value);
        }
    
        /**
         * @param PostId $value
         */
        public function convertToDatabaseValue(
            $value, AbstractPlatform $platform
        ) {
           return $value->id();
        }
    
        public function getName()
        { 
           return 'post_id';
        }
    }

Don't forget to implement the `__toString` magic method on the `PostId` Value Object, as Doctrine requires this:

    class PostId 
    { 
        // ...   
        public function __toString()
        {
            return $this->id;
        }
    }

Doctrine offers multiple formats for the mapping, such as YAML, XML, or annotations. XML is our preferred choice, as it provides robust IDE autocompletion:

    <?xml version="1.0" encoding="UTF-8"?>
    <doctrine-mapping
        xmlns="http://doctrine-project.org/schemas/orm/doctrine-mapping"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="
            http://doctrine-project.org/schemas/orm/doctrine-mapping
        http://raw.github.com/doctrine/doctrine2/master/doctrine-mapping.xsd">
    
        <entity name="Domain\Model\Post" table="posts">
            <id name="id" type="post_id" column="id">
                <generator strategy="NONE" />
            </id>
            <field name="body" type="body" length="250" column="body"/>
            <field name="createdAt" type="datetime" column="created_at"/>
        </entity>
    
    </doctrine-mapping>



### Note

**`Exercise`**   Write down what the mapping would look like in the case of using the Doctrine Embeddables approach. Take a look at Chapter Value Objects or Chapter Entities if you need some help.

#### Entity Manager

The `EntityManager` is the central access point for the ORM functionality. Bootstrapping it is easy:

    use Doctrine\DBAL\Types\Type;
    use Doctrine\ORM\EntityManager;
    use Doctrine\ORM\Tools;
    
    Type::addType(
        'post_id',
        'Infrastructure\Persistence\Doctrine\Types\PostIdType'
    );
    
    Type::addType(
        'body',
        'Infrastructure\Persistence\Doctrine\Types\BodyType'
    );
    
    $entityManager = EntityManager::create(
        [
            'driver' => 'pdo_sqlite',
            'path'=> __DIR__ . '/db.sqlite',
        ],
        Tools\Setup::createXMLMetadataConfiguration(
             ['/Path/To/Infrastructure/Persistence/Doctrine/Mapping'],
             $devMode = true
        )
    );

Remember to configure it according to your needs and setup.

#### DQL Implementation

In the case of this Repository, we'll only need the `EntityManager` to retrieve Domain objects directly from the database:

    namespace Infrastructure\Persistence\Doctrine;
    
    use Doctrine\ORM\EntityManager;
    use Domain\Model\Post;
    use Domain\Model\PostId;
    use Domain\Model\PostRepository;
    
    class DoctrinePostRepository implements PostRepository
    {
        protected $em;
    
        public function __construct(EntityManager $em)
        {
            $this->em = $em;
        }
        public function add(Post $aPost)
        {
            $this->em->persist($aPost);
        }
    
        public function remove(Post $aPost)
        {
            $this->em->remove($aPost);
        }
    
        public function postOfId(PostId $anId)
        {
            return $this->em->find('Domain\Model\Post', $anId);
        }
    
        public function latestPosts(\DateTimeImmutable $sinceADate)
        {
           return $this->em->createQueryBuilder()
               ->select('p')
               ->from('Domain\Model\Post', 'p')
               ->where('p.createdAt > :since')
               ->setParameter(':since', $sinceADate)
               ->getQuery()
               ->getResult();
        } 
    
        public function nextIdentity()
        {
            return new PostId();
        }
    }

If you check some Doctrine examples out there, you may find that after running persist or remove, flush should be called. But as seen in our proposal, there's no call to `flush`. Flushing and dealing with transactions is delegated to the Application Service. That's why you can work with Doctrine, considering that flushing all the changes on Entities will happen at the end of the request. In terms of performance, one flush call is best.



Persistence-Oriented Repository
-------------------------------

* * *

There are times when collection-oriented Repositories don't fit well with our persistence mechanism. If you don't have a unit of work, keeping track of Aggregate changes is a difficult task. The only way to persist such changes is by explicitly calling `save`.

The interface definition for a persistence-oriented Repository is similar to how you would define a collection-oriented equivalent:

     interface PostRepository
     {
         public function nextIdentity();
         public function postOfId(PostId $anId);
         public function save(Post $aPost);
         public function saveAll(array $posts);
         public function remove(Post $aPost);
         public function removeAll(array $posts);
     }

In this case, we now have save and `saveAll` methods, which provide functionality similar to the previous add and `addAll` methods. However, the important difference is how the client uses them. Within a collection-oriented style, you use the add methods just once: when the Aggregate is created. In a persistence-oriented style, you'll not only use the `save` action after creating a new Aggregate, but also when an existing one is modified:

     $post = new Post(/* ... */);
     $postRepository->save($post);
    
     // later ...
     $post = $postRepository->postOfId($postId);
     $post->editBody(new Body('New body!'));
     $postRepository->save($post);

Other than this difference, the details are only in the implementation.

### Redis Implementation

The [Redis](http://redis.io/) is an in-memory key value that can be used as a cache or store.

Depending on the circumstances, we could consider using Redis as a store for our Aggregates.

To get started, make sure you have a PHP client to connect to Redis. A good one that we recommend is [Predis](https://github.com/nrk/predis):

    composer require predis/predis:~1.0
    namespace Infrastructure\Persistence\Redis;
    
    use Domain\Model\Post;
    use Domain\Model\PostId;
    use Domain\Model\PostRepository;
    use Predis\Client;
    
    class RedisPostRepository implements PostRepository
    {
        private $client;
    
        public function __construct(Client $client)
        {
            $this->client = $client;
        }
    
        public function save(Post $aPost)
        {
            $this->client->hset(
                'posts',
                (string) $aPost->id(), serialize($aPost)
            );
        }
    
        public function remove(Post $aPost)
        {
            $this->client->hdel('posts', (string) $aPost->id());
        }
    
        public function postOfId(PostId $anId)
        {
           if($data = $this->client->hget('posts', (string) $anId)) {
              return unserialize($data);
           }
    
           return null;
        }
    
        public function latestPosts(\DateTimeImmutable $sinceADate)
        {
            $latest = $this->filterPosts(
                function(Post $post) use ($sinceADate) {
                    return $post->createdAt() > $sinceADate;
                }
            );
    
            $this->sortByCreatedAt($latest);
    
            return array_values($latest);
        }
    
        private function filterPosts(callable $fn)
        {
            return array_filter(array_map(function ($data) {
                return unserialize($data);
            },
    
            $this->client->hgetall('posts')), $fn);
        }
    
        private function sortByCreatedAt(&$posts)
        {
            usort($posts, function (Post $a, Post $b) {
                if ($a->createdAt() == $b->createdAt()) {
                    return 0;
                }
                return ($a->createdAt() < $b->createdAt()) ? -1 : 1;
            });
        }
    
        public function nextIdentity()
        {
            return new PostId();
        }
     }

### SQL Implementation

In a classic example, we could create a simple [PDO](http://php.net/manual/en/book.pdo.php) implementation for our `PostRepository` just by using plain SQL queries:

    namespace Infrastructure\Persistence\Sql;
    
    use Domain\Model\Body;
    use Domain\Model\Post;
    use Domain\Model\PostId;
    use Domain\Model\PostRepository;
    
    class SqlPostRepository implements PostRepository
    {
        const DATE_FORMAT = 'Y-m-d H:i:s';
    
        private $pdo;
    
        public function __construct(\PDO $pdo)
        {
            $this->pdo = $pdo;
        }
    
        public function save(Post $aPost)
        {
            $sql ='INSERT INTO posts ' .
                '(id, body, created_at) VALUES ' .
                '(:id, :body, :created_at)';
    
            $this->execute($sql, [
                'id' => $aPost->id()->id(),
                'body' => $aPost->body()->content(),
                'created_at' => $aPost->createdAt()->format(
                    self::DATE_FORMAT
                )
            ]);
        }
    
        private function execute($sql, array $parameters)
        {
            $st = $this->pdo->prepare($sql);
    
            $st->execute($parameters);
    
            return $st;
        }
    
        public function remove(Post $aPost)
        {
            $this->execute('DELETE FROM posts WHERE id = :id', [
                'id' => $aPost->id()->id()
            ]);
        } 
    
        public function postOfId(PostId $anId)
        {
            $st =$this->execute('SELECT * FROM posts WHERE id = :id',[
                'id' => $anId->id()
            ]);
    
            if($row = $st->fetch(\PDO::FETCH_ASSOC)) {
                return $this->buildPost($row);
            }
    
            return null;
        }
    
        private function buildPost($row)
        {
            return new Post(
                new PostId($row['id']),
                new Body($row['body']),
                new \DateTimeImmutable($row['created_at'])
            );
        }
    
        public function latestPosts(\DateTimeImmutable $sinceADate)
        {
            return $this->retrieveAll(
                'SELECT * FROM posts WHERE created_at > :since_date', [
                    'since_date' => $sinceADate->format(self::DATE_FORMAT)
                ]
            );
        }
    
        private function retrieveAll($sql, array $parameters = [])
        {
            $st = $this->pdo->prepare($sql);
    
            $st->execute($parameters);
    
            return array_map(function ($row) {
                return $this->buildPost($row);
            }, $st->fetchAll(\PDO::FETCH_ASSOC));
        }
    
        public function nextIdentity()
        {
            return new PostId();
        }
    
        public function size()
        {
            return $this->pdo->query('SELECT COUNT(*) FROM posts')
                ->fetchColumn();
        }
    }

As we don't have any mapping configuration, it would be very useful to have an initialization method for the schema within the same class. **Things that change together should remain together**:

    class SqlPostRepository implements PostRepository
    {
        // ...
        public function initSchema()
        {
            $this->pdo->exec(<<<SQL
    DROP TABLE IF EXISTS posts;
    
    CREATE TABLE posts (
        id CHAR(36) PRIMARY KEY,
        body VARCHAR (250) NOT NULL,
        created_at DATETIME NOT NULL
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
            SQL);
        }
    }

### Extra Behavior

    interface PostRepository 
    {  
        // ... 
        public function size(); 
    }

The implementation could look like this:

    class DoctrinePostRepository implements PostRepository
    {
        // ...
    
        public function size()
        {
            return $this->em->createQueryBuilder()
                ->select('count(p.id)')
                ->from('Domain\Model\Post', 'p')
                ->getQuery()
                ->getSingleScalarResult();
        }
    } 

Adding additional behavior to a Repository can be very beneficial. An example of this is the ability to count all the items in a given collection. You might think to add a method with the name count; however, as we're trying to mimic a collection, a better name would instead be size:

You're also able to place specific calculations, counters, read-optimized queries, or complex commands (`INSERT`, `UPDATE`, or `DELETE`) into the Repository. However, all behavior should still follow the Repositories' collection characteristics. You're encouraged to move as much logic into Domain-specific stateless Domain Services as possible, instead of simply adding these responsibilities to the Repository.

In some instances, you won't require the entire Aggregate for simply accessing small amounts of information. To solve this, you can add Repository methods to access these as shortcuts. You should make sure to only access data that could be retrieved by navigating through the Aggregate Root. As such, you shouldn't allow access to the private and internal areas of the Aggregate Root, as this would violate the laid out contractual agreement.

For some use cases, you'll require very specific queries that are compositions of multiple Aggregate types, each returning specific information. These queries can be run and then returned as a single Value Object. It's very common for Repositories to return Value Objects.

If you find yourself creating many use case optimal finder methods, you may be introducing a common code smell. This could be an indication of a misjudged Aggregate boundary. If, however, you're confident that the boundaries are correct, it could be time to explore CQRS.



Querying Repositories
---------------------

* * *

Upon comparison, Repositories are different than a collection if we consider their querying ability. A Repository deals with a large set of objects that typically aren't in memory when the query is performed. It's not feasible to load all the instances of a Domain object in memory and perform a query over them.

A good solution is to pass a criterion and let the Repository handle the implementation details to successfully perform the operation. It might translate the criterion to SQL or ORM queries or iterate over an in-memory collection. However, it doesn't matter, because the implementation deals with it.

### Specification Pattern

A common implementation for the criterion object is the Specification pattern. A specification is a simple predicate that takes a Domain object and returns a boolean. Given a Domain object, it will return true if it specifies the specification, and false otherwise:

    interface PostSpecification
    {
        /**
         * @return boolean
         */
        public function specifies(Post $aPost);
    }

We just need to add a `query` method to our Repository:

    interface PostRepository
    {
        // ...
        public function query($specification);
    }

#### In-Memory Implementation

As an example, if we wanted to replicate the `latestPosts` query method in our `PostRepository` by using a Specification for an in-memory implementation, it would look like this:

    namespace Infrastructure\Persistence\InMemory;
    
    use Domain\Model\Post;
    
    interface InMemoryPostSpecification
    {
        /**
         * @return boolean
         */
         public function specifies(Post $aPost);
    }

The in-memory implementation for the `latestPosts` behavior could look like this:

    namespace Infrastructure\Persistence\InMemory;
    use Domain\Model\Post;
    
    class InMemoryLatestPostSpecification
        implements InMemoryPostSpecification
    {
        private $since;
    
        public function __construct(\DateTimeImmutable $since)
        {
           $this->since = $since;
        }
    
        public function specifies(Post $aPost)
        {
          return $aPost->createdAt() > $this->since;
        }
    }

The `query` method for our Repository implementation could look like this:

    class InMemoryPostRepository implements PostRepository
    {
        // ...
    
        /**
         * @param InMemoryPostSpecification $specification
         *
         * @return Post[]
         */
        public function query($specification)
        {
            return $this->filterPosts(
                function (Post $post) use($specification) {
                    return $specification->specifies($post);
                }
            );
        }
    }

Retrieving all the latest posts from the Repository is as simple as creating a tailored instance of the above implementation:

    $latestPosts = $postRepository->query(
        new InMemoryLatestPostSpecification(new \DateTimeImmutable('-24'))
    );

#### SQL Implementation

A standard specification works well for in-memory implementations. However, as we don't pre-load all the Domain objects in memory for an SQL implementation, we need a more specific specification for these cases:

    namespace Infrastructure\Persistence\Sql;
    
    interface SqlPostSpecification
    {
        /**
         * @return string
         */
        public function toSqlClauses();
    }

The SQL implementation for this specification could look like this:

    namespace Infrastructure\Persistence\Sql;
    
    class SqlLatestPostSpecification implements SqlPostSpecification
    {
        private $since;
    
        public function __construct(\DateTimeImmutable $since)
        {
            $this->since = $since;
        }
    
        public function toSqlClauses()
        {
            return "created_at >'" .
                $this->since->format('Y-m-d H:i:s') .
                "'";
        }
    }

And here's an example of how to query an `SQLPostRepository` implementation:

    class SqlPostRepository implements PostRepository
    {
        // ...
    
        /**
         * @param SqlPostSpecification $specification
         *
         * @return Post[]
         */
        public function query($specification)
        {
            return $this->retrieveAll(
                'SELECT * FROM posts WHERE ' .
                    $specification->toSqlClauses()
            );
        }
    
        private function retrieveAll($sql, array $parameters = [])
        {
            $st = $this->pdo->prepare($sql);
    
            $st->execute($parameters);
    
            return array_map(function ($row) {
                return $this->buildPost($row);
            }, $st->fetchAll(\PDO::FETCH_ASSOC));
        }
    }



Managing Transactions
---------------------

* * *

The Domain Model isn't the place to manage transactions. The operations applied over the Domain Model should be agnostic to the persistence mechanism. A common approach to solving this problem is placing a [Facade](http://en.wikipedia.org/wiki/Facade_pattern) in the Application layer, thereby grouping related use cases together. When a method of the Facade is invoked from the UI layer, the business method begins a transaction. Once complete, the Facade ends the interaction by committing the transaction. If anything goes wrong, the transaction is rolled back:

    use Doctrine\ORM\EntityManager;
    
    class SomeApplicationServiceFacade
    {
        private $em;
    
        public function __construct(EntityManager $em)
        {
            $this->em = $em;
        }
    
        public function doSomeUseCaseTask()
        {
            try {
                $this->em->getConnection()->beginTransaction();
                // Use domain model
    
                $this->em->getConnection()->commit();
            } catch (Exception $e) {
                 $this->em->getConnection()->rollback();
                 throw $e;
            }
        }
    } 

The problem introduced with Facades is that we have to repeat the same boilerplate code over and over. If we unify the way we execute use cases, we could wrap them in a transaction using the [Decorator pattern](http://en.wikipedia.org/wiki/Decorator_pattern):

    interface ApplicationService
    {
       /**
        * @param $request
        * @return mixed
        */
        public function execute(BaseRequest $request);
    }
    
    class SomeApplicationService implements ApplicationService
    {
        public function execute(BaseRequest $request)
        {
           // do something
        }
    }

We don't want to couple our Application layer with the concrete transactional procedure, so instead we can create a simple interface for it:

    interface TransactionalSession
    {
        /**
         * @param callable $operation
         * @return mixed
         */
        public function executeAtomically(callable $operation);
    }

A Decorator's pattern implementation that can make any Application Service transactional is as easy as this:

    class TransactionalApplicationService implements ApplicationService
    {
        private $session;
        private $service;
    
        public function __construct(
            ApplicationService $service,
            TransactionalSession $session
        ) {
            $this->session = $session;
            $this->service = $service;
        }
    
        public function execute(BaseRequest $request)
        {
            $operation = function() use($request) {
                return $this->service->execute($request);
            };
    
            return $this->session->executeAtomically(
                $operation->bindTo($this)
            );
        }
    }

Following this, we could alternatively create a Doctrine transactional session implementation:

    class DoctrineSession implements TransactionalSession
    {
        private $entityManager;
    
        public function __construct(EntityManager $entityManager)
        {
            $this->entityManager = $entityManager;
        }
    
        public function executeAtomically(callable $operation)
        {
            return $this->entityManager->transactional($operation);
        }
    }

Now we have everything to execute our use cases within a transaction:

    $useCase = new TransactionalApplicationService(
        new SomeApplicationService(
            // ...
        ),
        new DoctrineSession(
           // ...
        )
    );
    
    $response = $useCase->execute();



Testing Repositories
--------------------

* * *

In order to be sure that the Repository will work in production, we'll need to test its implementation. To do this, we have to test the boundaries of the system, making sure that our expectations are correct.

In the case of a Doctrine test, the setup will be a little bit more sophisticated:

    use Doctrine\DBAL\Types\Type;
    use Doctrine\ORM\EntityManager;
    use Doctrine\ORM\Tools;
    use Domain\Model\Post;
    
    class DoctrinePostRepositoryTest extends \PHPUnit_Framework_TestCase
    {
        private $postRepository;
    
        public function setUp()
        {
            $this->postRepository = $this->createPostRepository();
        }
    
        private function createPostRepository()
        {
            $this->addCustomTypes();
            $em = $this->initEntityManager();
            $this->initSchema($em);
    
            return new PrecociousDoctrinePostRepository($em);
        }
    
        private function addCustomTypes()
        {
            if (!Type::hasType('post_id')) {
                Type::addType(
                    'post_id',
                    'Infrastructure\Persistence\Doctrine\Types\PostIdType'
                );
            }
    
            if (!Type::hasType('body')) {
                Type::addType(
                    'body',
                    'Infrastructure\Persistence\Doctrine\Types\BodyType'
                );
            }
        }
    
        protected function initEntityManager()
        {
            return EntityManager::create(
                ['url' => 'sqlite:///:memory:'],
                Tools\Setup::createXMLMetadataConfiguration(
                    ['/Path/To/Infrastructure/Persistence/Doctrine/Mapping'],
                    $devMode = true
                )
            );
        }
    
        private function initSchema(EntityManager $em)
        {
            $tool = new Tools\SchemaTool($em);
            $tool->createSchema([
                $em->getClassMetadata('Domain\Model\Post')
            ]);
        }
    
        // ...
    }
    
    class PrecociousDoctrinePostRepository extends DoctrinePostRepository
    {
        public function persist(Post $aPost)
        {
            parent::persist($aPost);
    
            $this->em->flush();
        }
    
        public function remove(Post $aPost)
        {
            parent::remove($aPost);
    
            $this->em->flush();
        }
    }

Once we have this environment set up, we can continue to test the Repository's behavior:

    class DoctrinePostRepositoryTest extends \PHPUnit_Framework_TestCase
    {
        // ...
    
        /**
         * @test
         */
        public function itShouldRemovePost()
        {
            $post = $this->persistPost('irrelevant body');
    
            $this->postRepository->remove($post);
    
            $this->assertPostExist($post->id());
        }
    
        private function assertPostExist($id)
        {
            $result = $this->postRepository->postOfId($id);
            $this->assertNull($result);
        }
    
        private function persistPost(
            $body,
            \DateTimeImmutable $createdAt = null
        ) {
            $this->postRepository->add(
                $post = new Post(
                    $this->postRepository->nextIdentity(),
                    new Body($body),
                    $createdAt
                )
            );
    
            return $post;
        }
    }

Following our earlier assertion, if we save a `Post`, we expect to find it in the exact same state.

Now we can move on to test finding the latest posts by specifying a given date:

    class DoctrinePostRepositoryTest extends \PHPUnit_Framework_TestCase
    {
        // ...
    
        /**
         * @test
         */
        public function itShouldFetchLatestPosts()
        {
            $this->persistPost(
                'a year ago', new \DateTimeImmutable('-1 year')
            );
            $this->persistPost(
                'a month ago', new \DateTimeImmutable('-1 month')
            );
            $this->persistPost(
                'few hours ago', new \DateTimeImmutable('-3 hours')
            );
            $this->persistPost(
                'few minutes ago', new \DateTimeImmutable('-2 minutes')
            );
    
            $posts = $this->postRepository->latestPosts(
                new \DateTimeImmutable('-24 hours')
            );
    
            $this->assertCount(2, $posts);
            $this->assertEquals(
                'few hours ago', $posts[0]->body()->content()
            );
            $this->assertEquals(
                'few minutes ago', $posts[1]->body()->content()
            );
        }
    }



Testing Your Services with In-Memory Implementations
----------------------------------------------------

* * *

Setting up a fully persistent Repository implementation can be complex and result in slow execution. You should care about keeping your tests fast. Going through the whole database setup and then querying will slow you down enormously. Having an in-memory implementation could help delay persistence decisions until the end. We can test in the same manner as we did before, but this time, we'll use a full-featured fast and simple in-memory implementation:

    class MyServiceTest extends \PHPUnit_Framework_TestCase
    {
        private $service;
    
        public function setUp()
        {
            $this->service = new MyService(
                new InMemoryPostRepository()
            );
        }
    }



Wrap-Up
-------

* * *

A Repository is a mechanism that acts as a storage location. The difference between a DAO and a Repository is that a DAO follows a database-first approach, decreasing cohesion with many low-level methods to query the database. Depending on the underlying persistence mechanics, we've seen different Repository approaches:

*   **Collection-oriented Repositories** tend to be purer to the Domain model, even if they persist Entities. From the client's point of view, a collection-oriented Repository looks like a collection (Set). There's no need for explicit persistence calls on Entity updates, as the Repository tracks changes on the objects. We explored how to use Doctrine as the underlying persistence mechanism for this type of Repository.
*   **Persistence-oriented Repositories** require explicit persistence calls, as they don't track object changes. We explored Redis and plain SQL implementations.

Along the way, we discovered Specifications as a pattern that helps us query the database without sacrificing flexibility and cohesion. We also studied how to manage transactions and how to test our services with simple and fast in-memory Repository implementations.
