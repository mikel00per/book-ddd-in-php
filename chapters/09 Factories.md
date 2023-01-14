Chapter 9. Factories
--------------------

Factories are a powerful abstraction. They help decouple the client from the details of how to interact with the Domain. The client doesn't need to know how to build complex objects and Aggregates, so you can use Factories to create whole Aggregates, thereby enforcing their invariants.



Factory Method on Aggregate Root
--------------------------------

* * *

The [Factory Method](https://en.wikipedia.org/wiki/Factory_method_pattern) pattern, as defined in the classic, [Gang of Four](http://wiki.c2.com/?GangOfFour), is a creational pattern that:

Defines an interface for creating an object, but leaves the choice of its type to the subclasses, creation being deferred at run-time.

Adding a Factory Method in the Aggregate Root hides the internal implementation details of creating Aggregates from any external client. This also moves the responsibility for the integrity of the Aggregate back to the root.

In a Domain Model where we have a `User` Entity and a `Wish` Entity, the `User` acts as the Aggregate root. There's no `Wish` without `User`. The `User` Entity should manage its Aggregates.

The way to move the control of `Wish` back to the `User` Entity is by placing a Factory method in the Aggregate root:

    class User
    {
        // ...
    
        public function makeWish(WishId $wishId, $email, $content)
        {
            $wish = new WishEmail(
                $wishId,
                $this->id(),
                $email,
                $content
            );
    
            DomainEventPublisher::instance()->publish(
                new WishMade($wishId)
            );
    
            return $wish;
        }
    }


The client doesn't need to know the internal details of how the Aggregate Root handles the creation logic:

     $wish = $aUser->makeWish(
         $wishRepository->nextIdentity(),
         'user@example.com',
         'I want to be free!'
     );


### Forcing Invariants

Factory Methods in the Aggregate Root are also a good place for invariants.

In a Domain Model with `Forum` and `Post` Entities, where `Post` is an aggregated part of the Aggregate Root `Forum`, publishing a `Post` could look something like this:

    class Forum
    {
        // ...
    
        public function publishPost(PostId $postId, $content)
        {
            $post = new Post($this->id, $postId, $content);
    
            DomainEventPublisher::instance()->publish(
                new PostPublished($postId)
            );
    
            return $post;
        }
     }


After talking with a Domain Expert, we came to the conclusion that a `Post` shouldn't be published when the `Forum` is closed. This is an invariant, and we could force it directly on `Post` creation, thereby preventing an inconsistent Domain state:

    class Forum
    {
         // ...
    
        public function publishPost(PostId $postId, $content)
        {
            if ($this->isClosed()) {
                throw new ForumClosedException();
            }
    
            $post = new Post($this->id, $postId, $content);
    
            DomainEventPublisher::instance()->publish(
                new PostPublished($postId)
            );
    
            return $post;
        }
    }



Factory on Service
------------------

* * *

Decoupling creation logic also comes in handy in our Services.

### Building Specifications

Using Specifications in our Services might be the best example to illustrate how to use Factories within our Services.

Consider the following Service example. Given a request from the outside world, we want to build a feed based on the latest `Posts` added to the system:

    namespace Application\Service;
    
    use Domain\Model\Post;
    use Domain\Model\PostRepository;
    
    class LatestPostsFeedService 
    {
        private $postRepository;
    
        public function __construct(PostRepository $postRepository) 
        {
            $this->postRepository = $postRepository;
        }
    
        /**
         * @param LatestPostsFeedRequest $request
         */
        public function execute($request) 
        {
            $posts = $this->postRepository->latestPosts($request->since);
    
            return array_map(function(Post $post) {
                return [
                    'id' => $post->id()->id(),
                    'content' => $post->body()->content(),
                    'created_at' => $post-> createdAt()
                ];
            }, $posts);
        }
    }

Finder methods in Repositories like `latestPosts` have some limitations, as they keep adding complexity to our Repositories indefinitely. As we discuss in the [Chapter 10](/chapters/10%20Repositories.md), _Repositories_ Specifications are a better approach.

Lucky for us, we have a nice `query` method in our `PostRepository` that works with Specifications:

    class LatestPostsFeedService 
    {
        // ...
    
        public function execute($request) 
        {
            $posts = $this->postRepository->query($specification);
        }
    }

Using a concrete implementation for the Specification is a bad idea:

    class LatestPostsFeedService
    {
    
        public function execute($request)
        {
            $posts = $this->postRepository->query(
                new SqlLatestPostSpecification($request->since)
            );
        }
    }

Coupling our high-level application Service with a low-level Specification implementation mixes layers and breaks the Separation of Concerns. In addition, this is a pretty bad way of coupling our Service to a concrete Infrastructure implementation. There's no way you could use this Service outside of the SQL persistence solution. What if we want to test our Service with an in-memory implementation?

The solution to this problem is to decouple Specification creation from the Service itself by using the [Abstract Factory pattern](https://en.wikipedia.org/wiki/Abstract_factory_pattern). According to [OODesign.com](http://www.oodesign.com/abstract-factory-pattern.html):

Abstract Factory offers the interface for creating a family of related objects, without explicitly specifying their classes.

As we might have multiple Specification implementations, we first need to create an interface for the Factory:

    namespace Domain\Model;
    
    interface PostSpecificationFactory
    {
        public function createLatestPosts(DateTimeImmutable $since);
    }

Then we need to create Factories for each `PostRepository` implementation. As an example, a Factory for the in-memory `PostRepository` implementation could look like this:

    namespace Infrastructure\Persistence\InMemory;
    
    use Domain\Model\PostSpecificationFactory;
    
    class InMemoryPostSpecificationFactory
        implements PostSpecificationFactory
    {
        public function createLatestPosts(DateTimeImmutable $since)
        {
            return new InMemoryLatestPostSpecification($since);
        }
    }

Once we have a centralized place for the creation logic, it's easy to decouple it from the Service:

    class LatestPostsFeedService
    {
        private $postRepository;
        private $postSpecificationFactory;
    
        public function __construct(
            PostRepository $postRepository,
            PostSpecificationFactory $postSpecificationFactory
        ) {
            $this->postRepository = $postRepository;
            $this->postSpecificationFactory = $postSpecificationFactory;
        }
    
        public function execute($request)
        {
            $posts = $this->postRepository->query(
                $this->postSpecificationFactory->createLatestPosts(
                    $request->since
                )
            );
        }
    }

Now, unit testing our Service through an in-memory `PostRepository` implementation is pretty easy:

    namespace Application\Service;
    
    use Domain\Model\Body;
    use Domain\Model\Post;
    use Domain\Model\PostId;
    use Infrastructure\Persistence\InMemory\InMemoryPostRepositor;
    
    class LatestPostsFeedServiceTest extends PHPUnit_Framework_TestCase
    {
        /**
         * @var \Infrastructure\Persistence\InMemory\InMemoryPostRepository
         */
        private $postRepository;
    
        /**
         * @var LatestPostsFeedService
         */
        private $latestPostsFeedService;
    
        public function setUp()
        {
            $this->latestPostsFeedService = new LatestPostsFeedService(
                $this->postRepository = new InMemoryPostRepository()
            );
        }
    
       /**
        * @test
        */
        public function shouldBuildAFeedFromLatestPosts()
        {
            $this->addPost(1, 'first', '-2 hours');
            $this->addPost(2, 'second', '-3 hours');
            $this->addPost(3, 'third', '-5 hours');
    
            $feed = $this->latestPostsFeedService->execute(
                new LatestPostsFeedRequest(
                     new \DateTimeImmutable('-4 hours')
                )
            );
    
            $this->assertFeedContains([
                ['id' => 1, 'content' => 'first'],
                ['id' => 2, 'content' => 'second']
            ], $feed);
        }
    
        private function addPost($id, $content, $createdAt)
        {
            $this->postRepository->add(new Post(
                new PostId($id),
                new Body($content),
                new \DateTimeImmutable($createdAt)
            ));
        }
    
        private function assertFeedContains($expected, $feed)
        {
            foreach ($expected as $index => $contents) {
                $this->assertArraySubset($contents, $feed[$index]);
                $this->assertNotNull($feed[$index]['created_at']);
            }
        }
    }

### Building Aggregates

Entities are agnostic to the persistence mechanism. You don't want to couple and pollute your Entities with persistence details. Take a look at the next Application Service:

    class SignUpUserService
    {
        private $userRepository;
    
        public function __construct(UserRepository $userRepository)
        {
            $this->userRepository = $userRepository;
        }
    
        /**
         * @param SignUpUserRequest $request
         */
        public function execute( $request)
        {
            $email = $request->email();
            $password = $request->password();
    
            $user = $this->userRepository->userOfEmail($email);
            if (null !== $user) {
                throw new UserAlreadyExistsException();
            }
    
            $this->userRepository->persist(new User(
                $this->userRepository->nextIdentity(),
                $email,
                $password
            ));
    
            return $user;
        }
    }

Imagine a `User` Entity like the following one:

    class User
    {
        private $userId;
        private $email;
        private $password;
    
        public function __construct(UserId $userId, $email, $password)
        {
            // ...
        }
    
        // ...
     }

Imagine we want to use Doctrine as our Infrastructure persistence mechanism. Doctrine requires having an `id` as a plain string instance variable in order to work properly. In our Entity, `$userId` is a `UserId` Value Object. Adding an additional `id` to our `User` Entity just because of Doctrine would couple our persistence mechanism with our Domain Model. We saw in the [Chapter 4](/chapters/04%20Entities.md), _Entities_ that we could solve this problem with a Surrogate ID by creating a wrapper around our `User` Entity in the Infrastructure layer:

    class DoctrineUser extends User
    {
        private $surrogateUserId;
    
        public function __construct(UserId $userId, $email, $password)
        {
            parent:: __construct($userId, $email, $password);
            $this->surrogateUserId = $userId->id();
        }
    }

As creating the `DoctrineUser` in our Application Service would again couple the persistence layer with our Domain, we need to decouple the creation logic out of the Service with an Abstract Factory.

We could do this by creating an interface in our Domain:

    interface UserFactory
    {
        public function build(UserId $userId, $email, $password);
    }

Then, we place the implementation of it inside our Infrastructure layer:

    class DoctrineUserFactory implements UserFactory
    {
        public function build(UserId $userId, $email, $password)
        {
            return new DoctrineUser($userId, $email, $password);
        }
    }

Once decoupled, we only need to inject the Factory into our Application Service:

    class SignUpUserService
    {
        private $userRepository;
        private $userFactory;
    
        public function __construct(
            UserRepository $userRepository,
            UserFactory $userFactory
        ) {
            $this->userRepository = $userRepository;
            $this->userFactory = $userFactory;
        }
    
        /**
         * @param SignUpUserRequest $request
         */
        public function execute($request)
        {
            // ...
            $user = $this->userFactory->build(
                $this->userRepository->nextIdentity(),
                $email,
                $password
            );
            $this->userRepository->persist($user);
            return $user;
        }
    }



Testing Factories
-----------------

* * *

You'll see a common pattern while writing your tests. This is because building Entities and complex Aggregates can be a very tedious and repetitive process. Inevitably, complexity and duplication will start creeping into your test suite. Consider the following Entity:

    class Author
    {
        private $username;
        private $email ;
        private $fullName;
    
        public function __construct(
            Username $aUsername,
            FullName $aFullName,
            Email $anEmail
        ) {
            $this->username = $aUsername;
            $this->email = $anEmail ;
            $this->fullName = $aFullName;
        }
    
        // ...
    }

Somewhere in your system, you'll end up with a test looking like this:

    class MyTest extends PHPUnit_Framework_TestCase
    {
        /**
         * @test
         */
        public function itDoesSomething()
        {
            $author = new Author(
                new Username('johndoe'),
                new FullName('John', 'Doe' ),
                new Email('john@doe.com' )
            );
    
            //do something with author
        }
    }

Services inside boundaries share concepts like Entities, Aggregates, and Value Objects. Imagine the clutter of repeating the same building logic over and over across your tests. As we'll see, extracting the building logic out of tests comes in handy and prevents duplication.

### Object Mother

An [Object Mother](https://martinfowler.com/bliki/ObjectMother.html) is a catchy name for a Factory that creates fixed fixtures for your tests. Similar to the previous example, we could extract the duplicated logic to an Object Mother so it could be reused across tests:

    class AuthorObjectMother
    {
        public static function createOne()
        {
            return new Author(
                new Username('johndoe'),
                new FullName('John', 'Doe'),
                new Email('john@doe.com )
            );
        }
    }
    
    class MyTest extends PHPUnit_Framework_TestCase
    {
        /**
         * @test
         */
        public function itDoesSomething()
        {
            $author = AuthorObjectMother::createOne();
        }
    }

You'll notice that the more tests and situations you have, the more methods the Factory will have.

As Object Mothers aren't very flexible, they tend to grow in complexity quickly. Luckily, there's a more flexible alternative for your tests.

### Test Data Builder

Test Data Builders are just normal Builders with default values used exclusively in your test suites so that you don't have to specify irrelevant parameters on specific test cases:

    class AuthorBuilder
    {
        private $username;
        private $email ;
        private $fullName;
    
        private function __construct()
        {
            $this->username = new Username('johndoe');
            $this->email = new Email('john@doe.com');
            $this->fullName = new FullName('John', 'Doe');
        }
    
        public static function anAuthor()
        {
            return new self();
        }
    
        public function withFullName(FullName $aFullName)
        {
            $this->fullName = $aFullName;
    
            return $this;
        }
    
        public function withUsername(Username $aUsername)
        {
            $this->username = $aUsername;
    
            return $this;
        }
    
        public function withEmail(Email $anEmail)
        {
            $this->email = $anEmail ;
    
            return $this;
        }
    
        public function build()
        {
            return new Author($this->username, $this->fullName, $this->email);
        }
    }

    class MyTest extends PHPUnit_Framework_TestCase
    {
        /**
         * @test
         */
        public function itDoesSomething()
        {
            $author = AuthorBuilder::anAuthor()
                ->withEmail(new Email('other@email.com'))
                ->build();
        }
    }

We could even combine Test Data Builders to build more complicated Aggregates, like a `Post`:

    class Post
    {
        private $id;
        private $author;
        private $body;
        private $createdAt;
    
        public function __construct(
            PostId $anId, Author $anAuthor, Body $aBody
        ) {
            $this->id = $anId;
            $this->author = $anAuthor;
            $this->body = $aBody;
            $this->createdAt = new DateTimeImmutable();
        }
    }

Let's see the corresponding Test Data Builder for our `Post`. We could reuse the `AuthorBuilder` for building a default `Author`:

    class PostBuilder
    {
        private $postId;
        private $author;
        private $body;
    
        private function __construct()
        {
            $this->postId = new PostId();
            $this->author = AuthorBuilder::anAuthor()->build();
            $this->body = new Body('Post body');
        }
    
        public static function aPost()
        {
            return new self();
        }
    
        public function withAuthor(Author $anAuthor)
        {
            $this->author = $anAuthor;
    
            return $this;
        }
    
        public function withPostId(PostId $aPostId)
        {
            $this->postId = $aPostId;
    
            return $this;
        }
    
        public function withBody(Body $body)
        {
            $this->body = $body;
    
            return $this;
        }
    
        public function build()
        {
            return new Post($this->postId, $this->author, $this->body);
        }
    }

This solution is now flexible enough to cover any test case, including the possibility of building inner Entities:

    class MyTest extends PHPUnit_Framework_TestCase
    {
        /**
         * @test
         */
        public function itDoesSomething()
        {
            $post = PostBuilder::aPost()
                ->withAuthor(AuthorBuilder::anAuthor()
                ->withUsername(new Username('other'))
                    ->build())
                ->withBody(new Body('Another body'))
                    ->build();
    
            //do something with the post
        }
    }




Wrap-Up
-------

* * *

Factories are a powerful tool for decoupling construction logic from our business logic. The Factory Method pattern not only helps by moving creation responsibility to the Aggregate Root, but it could also force Domain invariants. Using the Abstract Factory pattern in our Services allows us to separate our Domain logic from Infrastructure creation details. A common use case is that of Specifications and their respective persistence implementations. We've seen that Factories come in handy on our test suites too. While we could extract building logic into Object Mother Factories, Test Data Builders provide more flexibility for our tests.
