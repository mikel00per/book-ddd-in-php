Chapter 5. Services
-------------------

You've already seen what Entities and Value Objects are. As basic building blocks, they should contain most of the business logic of any application. However, there are some scenarios where Entities and Value Objects aren't the best solutions. Let's take a look at what Eric Evans has to say about this in his book, [Domain-Driven Design: Tackling Complexity in the Heart of Software](http://www.amazon.com/Domain-Driven-Design-Tackling-Complexity-Software/dp/0321125215):

When a significant process or transformation in the domain is not a natural responsibility of an Entity or Value Object, add an operation to the model as standalone interface declared as a Service. Define the interface in terms of the language of the model and make sure the operation name is part of the Ubiquitous Language. Make the Service stateless.

So when there are operations that need to be represented, but Entities and Value Objects aren't the best place, you should consider modeling these operations as Services. In Domain-Driven Design, there are typically three different types of Services you'll encounter:

> *   **Application Services**: Operate on scalar types, transforming them into Domain types. A scalar type can be considered any type that's unknown to the Domain Model. This includes primitive types and types that don't belong to the Domain. We'll provide an overview in this chapter, but for a deeper look at this topic, check out the [Chapter 11](../chapters/11%20Application.md), _Application_.
> *   **Domain Services**: Operate only on types belonging to the Domain. They contain meaningful concepts that can be found within the Ubiquitous Language. They hold operations that don't fit well into Value Objects or Entities.
> *   **Infrastructure Services**: Are operations that fulfill infrastructure concerns, such as sending emails and logging meaningful data. In terms of Hexagonal Architecture, they live outside the Domain boundary.



Application Services
--------------------

* * *

Application Services are the middleware between the outside world and the Domain logic. The purpose of such a mechanism is to transform commands from the outside world into meaningful Domain instructions.

Let's consider the _User signs up to our platform_ use case. Starting with an outside-in approach: from the delivery mechanism, we need to compose the input request for our Domain operation. Using a framework like Symfony as the delivery mechanism, the code would look something like this:

    class SignUpController extends Controller
    {
        public function signUpAction(Request $request)
        {
            $signUpService = new SignUpUserService(
                $this->get('user_repository')
            );
    
            try {
                $response = $signUpService->execute(new SignUpUserRequest(
                    $request->request->get('email'),
                    $request->request->get('password')
                ));
            } catch (UserAlreadyExistsException $e) {
                return $this->render('error.html.twig', $response);
            }
    
            return $this->render('success.html.twig', $response);
        }
    }

As you can see, we create a new instance of our Application Services, passing all dependencies needed — in this case, a `UserRepository`. `UserRepository` is an interface that can be implemented with any specific technology (Example: MySQL, Redis, Elasticsearch). Then, we build a request object for our Application Service in order to abstract the delivery mechanism — in this example, a web request — from the business logic. Last, we execute the Application Service, get the response, and use that response for rendering the result. On the Domain side, let's check a possible implementation for the Application Service that coordinates the logic that fulfills the _User signs up_ use case:

    class SignUpUserService
    {
        private $userRepository;
    
        public function __construct(UserRepository $userRepository)
        {
            $this->userRepository = $userRepository;
        }
    
        public function execute(SignUpUserRequest $request)
        {
            $user = $this->userRepository->userOfEmail($request->email);
            if ($user) {
                throw new UserAlreadyExistsException();
            }
    
            $user = new User(
                $this->userRepository->nextIdentity(),
                $request->email,
                $request->password
            );
    
            $this->userRepository->add($user);
    
            return new SignUpUserResponse($user);
        }
    }

Everything in the code is about the Domain problem we want to solve, and not about the specific technology we're using to solve it. With this approach, we can decouple the high-level policies from the low-level implementation details. The communication between the delivery mechanism and the Domain is carried by data structures called DTOs, which we introduced in the [Chapter 2](../chapters/02%20Architectural%20Styles.md), _Architectural Styles_:

    class SignUpUserRequest
    {
        public $email;
        public $password;
    
        public function __construct($email, $password)
        {
            $this->email = $email;
            $this->password = $password;
        }
    }

There are different strategies for returning content, but for now, consider that we shouldn't return our Entities so that they can't be modified from outside our Application Services. That's why it's common to return another DTO with information, rather than the whole Entity. Let's see a simple example:

    class SignUpUserResponse
    {
        public $id;
        public $email;
    
        public function __construct(User $user)
        {
            $this->id = $user->id();
            $this->email = $user->email();
        }
    }



For creating your responses, you can use getters or public instance variables. Application Services should take care with transaction scopes and security. However, you'll delve into more detail about these and other things related to Application Services in the [Chapter 11](../chapters/11%20Application.md), _Application_.



Domain Services
---------------

* * *

Throughout conversations with Domain Experts, you'll come across concepts in the Ubiquitous Language that can't be neatly represented as either an Entity or a Value Object, such as:

> *   Users being able to sign into systems by themselves
> *   A shopping cart being able to become an order by itself

The preceding example are two concrete concepts, neither of which can naturally be bound to either an Entity or a Value Object. Further highlighting this oddity, we can attempt to model the behavior as follows:

    class User
    {
        public function signUp($aUsername, $aPassword)
        {
            // ...
        }
    }
    
    class Cart
    {
        public function createOrder()
        {
            // ...
        }
    }

In the case of the first implementation, we're not able to know that the given username and password relate to the invoked-upon user instance. Clearly, this operation doesn't suit this Entity; instead, it should be extracted out into a separate class, making its intention explicit.

With this in mind, we could create a Domain Service with the sole responsibility of authenticating users:

    class SignUp
    {
        public function execute($aUsername, $aPassword)
        {
            // ...
        }
    }


Similarly, as in the case of the second example, we could create a Domain Service specialized in creating orders from a supplied cart:

    class CreateOrderFromCart
    {
        public function execute(Cart $aCart)
        {
            // ...
        }
    }


A Domain Service can be defined as an operation that fulfills a Domain task and naturally doesn't fit into either an Entity or a Value Object. As concepts that represent operations in the Domain, Domain Services should be used by clients regardless of their run history. Domain Services don't hold any kind of state by themselves, so Domain Services are stateless operations.



Domain Services and Infrastructure Services
-------------------------------------------

* * *

It's common to encounter infrastructural dependencies when modeling a Domain Service  — for example, in the case where an authentication mechanism that handles password hashing is required. In this instance, you could use a [Separated Interface](http://martinfowler.com/eaaCatalog/separatedInterface.html), which allows for multiple hashing mechanisms to be defined. Using this pattern still provides you with a clear Separation of Concerns between the Domain and the Infrastructure:

    namespace Ddd\Auth\Domain\Model;
    
    interface SignUp
    {
        public function execute($aUsername, $aPassword);
    }


Using the preceding interface found in the Domain, we could create an implementation in the Infrastructure layer, like the following:

    namespace Ddd\Auth\Infrastructure\Authentication;
    
    class DefaultHashingSignUp implements Ddd\Auth\Domain\Model\SignUp
    {
        private $userRepository;
    
        public function __construct(UserRepository $userRepository)
        {
            $this->userRepository = $userRepository;
        }
    
        public function execute($aUsername, $aPassword)
        {
            if (!$this->userRepository->has($aUsername)) {
                throw UserDoesNotExistException::fromUsername($aUsername);
            }
    
            $aUser = $this->userRepository->byUsername($aUsername);
    
            if (!$this->isPasswordValidForUser($aUser, $aPassword)) {
                throw new BadCredentialsException($aUser, $aPassword);
            }
    
            return $aUser;
        }
    
        private function isPasswordValidForUser(
            User $aUser, $anUnencryptedPassword
        ) {
            return password_verify($anUnencryptedPassword,$aUser->hash());
        }
    }


Here is another implementation based instead on the MD5 algorithm:

    namespace Ddd\Auth\Infrastructure\Authentication;
    
    use Ddd\Auth\Domain\Model\SignUp
    
    class Md5HashingSignUp implements SignUp
    {
        const SALT = 'S0m3S4lT' ;
    
        private $userRepository;
    
        public function __construct(UserRepository $userRepository)
        {
            $this->userRepository = $userRepository;
        }
    
        public function execute($aUsername, $aPassword)
        {
            if (!$this->userRepository->has($aUsername)) {
                throw new InvalidArgumentException(
                    sprintf('The user "%s" does not exist.', $aUsername)
                );
            }
    
            $aUser = $this->userRepository->byUsername($aUsername);
    
            if ($this->isPasswordInvalidFor($aUser, $aPassword)) {
                throw new BadCredentialsException($aUser, $aPassword);
            }
    
            return $aUser;
        }
    
        private function salt()
        {
            return md5(self::SALT);
        }
    
        private function isPasswordInvalidFor(
            User $aUser, $anUnencryptedPassword
        ) {
            $encryptedPassword = md5(
                $anUnencryptedPassword . '_' .$this->salt()
            );
    
            return $aUser->hash() !== $encryptedPassword;
        }
    }

Opting for this choice allows us to have multiple implementations of the Domain Service interface at the Infrastructure layer. In other words, we end up with several Infrastructure Domain Services. Each Infrastructure service will be responsible for handling a different hash mechanism. Depending on the implementation, the use can easily be managed through a Dependency Injection container — for example, through Symfony's Dependency Injection component:

    <?xml version="1.0"?>
    <container
        xmlns="http://symfony.com/schema/dic/services"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="
            http://symfony.com/schema/dic/services
            http://symfony.com/schema/dic/services/services-1.0.xsd">
    
        <services>
    
            <service id="sign_in" alias="sign_in.default" />
    
            <service id="sign_in.default"
                class="Ddd\Auth\Infrastructure\Authentication
                \DefaultHashingSignUp">
                <argument type="service" id="user_repository"/>
            </service>
    
            <service id="sign_in.md5"
                class="Ddd\Auth\Infrastructure\Authentication
                    \Md5HashingSignUp">
                <argument type="service" id="user_repository"/>
            </service>
    
        </services>
    </container>

If, in the future, we wish to handle a new type of hashing, we can simply start by implementing the Domain Service interface. Then it's a matter of declaring the service in the Dependency Injection container and replacing the service alias dependency with the newly created one.

### An Issue of Code Reuse

Although the implementation described previously clearly defines the Separation of Concerns, we're required to repeat the password verification algorithm every time we wish to implement a new hashing mechanism. An alternative for solving this problem, which improves code reuse, is by separating out these two responsibilities. We could instead extract the password hashing logic out into a specialized class, using the [Strategy Pattern](http://en.wikipedia.org/wiki/Strategy_pattern) for all defined hashing algorithms. This leaves the design open for extension and closed for modification:

    namespace Ddd\Auth\Domain\Model;
    
    class SignUp 
    {
        private $userRepository;
        private $passwordHashing;
    
        public function __construct(
            UserRepository $userRepository, PasswordHashing $passwordHashing
        ) {
            $this->userRepository = $userRepository;
            $this->passwordHashing = $passwordHashing;
        }
    
        public function execute($aUsername, $aPassword)
        {
            if (!$this->userRepository->has($aUsername)) {
                throw new InvalidArgumentException(
                    sprintf('The user "%s" does not exist.', $aUsername)
                );
            }
    
            $aUser = $this->userRepository->byUsername($aUsername);
    
            if ($this->isPasswordInvalidFor($aUser, $aPassword)) {
                throw new BadCredentialsException($aUser, $aPassword);
            }
    
            return $aUser;
        }
    
        private function isPasswordInvalidFor(User $aUser, $plainPassword)
        {
            return !$this->passwordHashing->verify(
                $plainPassword,
                $aUser->hash()
            );
        }
    }
    
    interface PasswordHashing 
    {
        /**
         * @param string $password
         * @param string $hash 
         * @return boolean 
         */
        public function verify($plainPassword, hash);
    }

Defining different hashing strategies is as easy as implementing the `PasswordHashing` interface:

    namespace Ddd\Auth\Infrastructure\Authentication;
    
    class BasicPasswordHashing
        implements \Ddd\Auth\Domain\Model\PasswordHashing
    {
        public function verify($plainPassword, $hash)
        {
            return password_verify($plainPassword, $hash);
        }
    }
    
    class Md5PasswordHashing
        implements Ddd\Auth\Domain\Model\PasswordHashing
    {
        const SALT = 'S0m3S4lT' ;
    
        public function verify($plainPassword, $hash)
        {
            return $hash === $this-> calculateHash($plainPassword);
        }
    
        private function calculateHash($plainPassword)
        {
            return md5($plainPassword . '_' .$this-> salt());
        }
    
        private function salt()
        {
            return md5(self::SALT);
        }
    }




Testing Domain Services
-----------------------

* * *

Given the user authentication example from multiple Domain Service implementations, it's extremely beneficial to be able to easily test the service. Typically, however, testing the Template Method implementations can be tricky. As a result, we'll be using a plain password hashing implementation for testing purposes:

    class PlainPasswordHashing implements PasswordHashing
    {
        public function verify($plainPassword, $hash)
        {
            return $plainPassword === $hash;
        }
    }

Now we can test all cases in the Domain Service:

    class SignUpTest extends PHPUnit_Framework_TestCase
    {
        private $signUp;
        private $userRepository;
    
        protected function setUp()
        {
            $this->userRepository = new InMemoryUserRepository();
            $this->signUp = new SignUp(
                $this->userRepository,
                new PlainPasswordHashing()
            );
        }
    
        /**
         * @test
         * @expectedException InvalidArgumentException
         */
        public function itShouldComplainIfTheUserDoesNotExist()
        {
            $this->signUp->execute('test-username', 'test-password');
        }
    
        /**
         * @test
         * @expectedException BadCredentialsException
         */
        public function itShouldTellIfThePasswordDoesNotMatch()
        {
            $this->userRepository->add(
                new User(
                    'test-username',
                    'test-password'
                )
            );
    
            $this->signUp->execute('test-username', 'no-matching-password')
        }
    
        /**
         * @test
         */
        public function itShouldTellIfTheUserMatchesProvidedPassword()
        {
            $this->userRepository->add(
                new User(
                    'test-username',
                    'test-password'
                )
            );
    
            $this->assertInstanceOf(
                'Ddd\Domain\Model\User\User',
                $this->signUp->execute('test-username', 'test-password')
            );
        }
    }



Anemic Domain Models Vs Rich Domain Models
------------------------------------------

* * *

You must be cautious not to overuse Domain Service abstractions within your system. Following this path can lead to Entities and Value Objects being stripped of all behavior and becoming mere data containers. This is contrary to the goal of Object-Oriented Programming, which can be thought of as the gathering of both data and behavior into semantic units called objects, with the intent of expressing real-world concepts and problems. Domain Service overuse can be considered an anti-pattern and is referred to as the Anemic Domain Model.

Typically, when starting a new project or feature, it's easy to fall into the trap of modeling the data first. This commonly includes thinking that each database table has a direct one-to-one object form representation. However, this thinking may or may not be the exact case all the time.

Suppose we're tasked with modeling an order processing system. If we start by modeling the data first, we could end up with an SQL script like this:

    CREATE TABLE `orders` (
        `ID` INTEGER NOT NULL AUTO_INCREMENT,
        `CUSTOMER_ID` INTEGER NOT NULL,
        `AMOUNT` DECIMAL(17, 2) NOT NULL DEFAULT '0.00',
        `STATUS` TINYINT NOT NULL DEFAULT 0,
        `CREATED_AT` DATETIME NOT NULL,
        `UPDATED_AT` DATETIME NOT NULL, 
        PRIMARY KEY (`ID`)
    ) ENGINE=INNODB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

From this, it's relatively easy to create an `Order` class representation. This representation includes the required accessor methods, which are used to set or get data to and from the underlying orders database table:

    class Order 
    { 
        const STATUS_CREATED   = 10;
        const STATUS_ACCEPTED  = 20;
        const STATUS_PAID      = 30;
        const STATUS_PROCESSED = 40;
    
        private $id;
        private $customerId;
        private $amount;
        private $status;
        private $createdAt;
        private $updatedAt;
    
        public function __construct(
            $customerId,
            $amount,
            $status,
            DateTimeInterface $createdAt,
            DateTimeInterface $updatedAt
        ) {
            $this->customerId = $customerId;
            $this->amount = $amount;
            $this->status = $status;
            $this->createdAt = $createdAt;
            $this->updatedAt = $updatedAt;
        }
    
        public function setId($id)
        {
            $this->id = $id;
        }
    
        public function getId()
        {
            return $this->id;
        }
    
        public function setCustomerId($customerId)
        {
            $this->customerId = $customerId;
        }
    
        public function getCustomerId()
        {
            return $this->customerId;
        }
    
        public function setAmount($amount)
        {
            $this->amount = $amount;
        }
    
        public function getAmount()
        {
            return $this->amount;
        }
    
        public function setStatus($status)
        {
            $this->status = $status;
        }
    
        public function getStatus()
        {
            return $this->status;
        }
    
        public function setCreatedAt(DateTimeInterface $createdAt)
        {
            $this->createdAt = $createdAt;
        }
    
        public function getCreatedAt()
        {
            return $this->createdAt;
        }
    
        public function setUpdatedAt(DateTimeInterface $updatedAt)
        {
            $this->updatedAt = $updatedAt;
        }
    
        public function getUpdatedAt()
        {
            return $this->updatedAt;
        }
    }

An example use case for this implementation could be to update the order status as follows:

    // Fetch an order from the database
    $anOrder = $orderRepository->find( 1 );
    
    // Update order status
    $anOrder->setStatus(Order::STATUS_ACCEPTED);
    
    // Update updatedAt field
    $anOrder->setUpdatedAt(new DateTimeImmutable());
    
    // Save the order to the database
    $orderRepository->save($anOrder);

With regard to code reuse, this code has a problem similar to the initial user authentication solution. To resolve this issue, defenders of such a practice suggest the use of a [Service layer](http://martinfowler.com/eaaCatalog/serviceLayer.html), thereby making the operations explicit and reusable. This preceding implementation could now instead be encapsulated into a separate class:

    class ChangeOrderStatusService
    {
        private $orderRepository;
    
        public function __construct(OrderRepository $orderRepository)
        {
            $this->orderRepository = $orderRepository;
        }
    
        public function execute($anOrderId, $anOrderStatus)
        {
            // Fetch an order from the database
            $anOrder = $this->orderRepository->find($anOrderId);
    
            // Update order status
            $anOrder->setStatus($anOrderStatus);
    
            // Update updatedAt field
            $anOrder->setUpdatedAt(new DateTimeImmutable());
    
            // Save the order to the database
            $this->orderRepository->save($anOrder);
        }
    }

Or, in the case of updating an order amount, consider this:

    class UpdateOrderAmountService
    {
        private $orderRepository;
    
        public function __construct(OrderRepository $orderRepository)
        {
            $this->orderRepository = $orderRepository;
        }
    
        public function execute( $orderId, $amount)
        {
            $anOrder = $this->orderRepository->find(1);
    
            $anOrder->setAmount($amount);
            $anOrder->setUpdatedAt(new DateTimeImmutable());
            $this->orderRepository->save($anOrder);
        }
    }

The client code would be drastically reduced following a clearly intentioned operation:

    $updateOrderAmountService = new UpdateOrderAmountService(
        $orderRepository
    );
    
    $updateOrderAmountService->execute(1, 20.5);

Implementing this approach can result in a large degree of code reusability. Someone who wishes to update the order amount simply has to retrieve an instance of `UpdateOrderAmountService` and invoke the execute method with the appropriate parameters.

However, choosing this path breaks the discussed Object-Oriented Design principles and incurs the costs of building a Domain Model without taking advantage of any of the benefits.

### Anemic Domain Model Breaks Encapsulation

If we revisit the code used to define the services within our Service layer, we can see that as a client making use of the Order Entity, we're required to know every detail of its internal representation. This finding goes against the fundamental rule of Object-Oriented Programming, which is combining data with subsequent behavior

### Anemic Domain Model Brings a False Sense of Code Reuse

Say there's an instance where a client bypasses `UpdateOrderAmountService` and instead fetches, updates, and persists directly to `OrderRepository`. Then, all the extra business logic that the `UpdateOrderAmountService` service might have won't be executed. This could lead to the order being stored in an inconsistent state. As such, invariants should be correctly guarded, and the best way to do this is to let the true Domain Model handle it. In the case of this example, the Order Entity would be the best place to ensure this:

    class Order 
    { 
        // ...
        public function changeAmount($amount)
        {
            $this->amount = $amount;
            $this->setUpdatedAt(new DateTimeImmutable());
        }
    }

Note that by pushing this action down into the Entity and naming it in terms of the Ubiquitous Language, the system achieves great code reuse. Anyone who now wishes to change the amount of the order has to invoke the `Order::changeAmount` method directly.

This leads to far richer classes, where behavior is the goal for code reuse. This is commonly referred to as a Rich Domain Model.

### How to Avoid Anemic Domain Models

The way to avoid falling into an Anemic Domain Model is to, when starting a new project or feature, think of the behavior first. Databases, ORMs, and so on are just implementation details, and we should strive to push the decision to use these tools as late in the development process as we can. In doing so, we can focus on the one true attribute that matters: the behavior.

Just as is the case with Entities, Domain Services can also fire [Chapter 6](../chapters/06%20Domain-Events.md), _Domain-Events_. However, when events are mostly fired by Domain Services and not Entities, it's again an indicator that you may be creating an Anemic Domain Model.



Wrap-Up
-------

* * *

As we've seen, Services represent operations inside our system, and we can differentiate between three versions of them:

> *   **Application Services**: Help coordinate requests from the outside world into the Domain. These Services should not contain Domain logic. Transactions are handled in the application level; wrapping your services inside transnational decorators will make your code transaction agnostic.
> *   **Domain Services**: Operate with Domain concepts only, which are expressed by the Ubiquitous Language. Remember to postpone implementation details and think of behavior first, as abuse of Domain Services will lead to Anemic Domain Models and bad Object-Oriented Design.
> *   **Infrastructure Services**: Operate over Infrastructure, doing things like sending emails or logging information.

Our most important recommendation is that you should consider all your options before deciding on creating a Domain Service. First try to move your business logic inside an Entity or Value. Check with some workmates. Review again. If, after different approaches, the best option is creating a Domain Service, go for it.
