Chapter 11. Application
-----------------------

The Application layer is the area that separates the Domain Model from the clients that query or change its state. Application Services are the building blocks for such a layer. As Vaughn Vernon [says](https://www.amazon.com/Implementing-Domain-Driven-Design-Vaughn-Vernon/dp/0321834577): "Application Services are the direct clients of the domain model." You could think about an Application Service as a point of contact between the outside world (HTML forms, API clients, the command line, frameworks, UI, and so on.) and the Domain Model itself. It might help by thinking about the top-level use cases that your system exposes to the world, example: "as guest, I want to register," "as a logged user, I want to purchase a product," and so on.

In this chapter, we'll explore how to implement Application Services, understand the role of the Command pattern, and establish the responsibilities of an Application Service. To do this, let's consider the use case of _signing up a new user_.

Conceptually, in order to register a new user, we need to:

> *   Get an email and password from the client
> *   Check if the email is already in use
> *   Create a new user
> *   Add this new user to the existing user set
> *   Return the user we've just created

Let's go for it.


Requests
--------

* * *

We need to send the `email` and `password` to the Application Service. There are many ways of doing such a thing from the client (HTML form, API client, or even the command line). We could just send standard parameters (email and password) through the method signature or build and send a data structure with this information. The latter approach, sending a [DTO](http://martinfowler.com/eaaCatalog/dataTransferObject.html), brings some interesting features to the table. By sending an object, it'll be possible to serialize and queue it over a Command Bus. It'll also be possible to add type safety and some IDE help, too.

### Note

**`Data Transfer Object`** A DTO is a data structure that carries information between processes. Don't mistake it for a full-featured object. A DTO doesn't have any behavior except for storage and retrieval of its own data (accessors and mutators). DTOs are simple objects that shouldn't contain any business logic that would require testing.

As Vaughn Vernon [says](https://www.amazon.com/Implementing-Domain-Driven-Design-Vaughn-Vernon/dp/0321834577):

Application Service method signatures use only primitive types (`int`, `strings`, and so on.), and possibly DTOs. As an alternative to these approaches, however, a better approach may be to design Command objects instead. There is not necessarily a right or wrong way. It mostly depends on your tastes and goals.

The implementation for a DTO that holds the data required for the Application Service could be something like this:

    namespace Lw\Application\Service\User;
    
    class SignUpUserRequest
    {
        private $email;
        private $password;
    
        public function __construct($email, $password)
        {
            $this->email = $email;
            $this->password = $password;
        }
    
        public function email()
        {
            return $this->email;
        }
    
        public function password()
        {
            return $this->password;
        }
    }

As you see, `SignUpUserRequest` has no behavior, only data. This could have come from an HTML form or an API endpoint, though we don't care which.

### Building Application Service Requests

Creating a request from the delivery mechanism, your favorite framework, should be pretty straightforward. On the web, you could pick up parameters from the controller request and pass them down to the Service inside a DTO. The same principle applies for a CLI command: read input parameters and send them down again.

With Symfony, we can extract the data we need from Request object from the `HttpFoundation` component:

    // ...
    class UsersController extends Controller
    {
        /**
         * @Route('/signup', name = 'signup')
         * @param Request $request
         * @return Response
         */
        public function signUpAction(Request $request)
        {
            // ...
            $signUpUserRequest = new SignUpUserRequest(
                $request->get('email'),
                $request->get('password')
            );
            // ...
        }
    // ...

On a more elaborate Silex application that uses the `Form` component to capture and validate parameters, it would look like this:

    // ...
    $app->match('/signup', function (Request $request) use ($app) {
        $form = $app['sign_up_form'];
        $form->handleRequest($request);
    
        if ($form->isValid()) {
            $data = $form->getData();
    
            try {
                $app['sign_in_user_application_service']->execute(
                    new SignUpUserRequest(
                         $data['email'],
                         $data['password']
                    )
                );
    
                return $app->redirect(
                    $app['url_generator']->generate('login')
                );
            } catch (UserAlreadyExistsException $e) {
                $form
                    ->get('email')
                    ->addError(
                        new FormError(
                            'Email is already registered by another user'
                        )
                    );
            } catch (Exception $e) {
                $form
                    ->addError(
                        new FormError(
                          'There was an error, please get in touch with us'
                        )
                    );
            }
        }
    
        return $app['twig']->render('signup.html.twig', [
            'form' => $form->createView(),
        ]);
    });

### Request Design

When designing your request objects, you should always follow these principles: use primitives, design for serialization, and don't include business logic inside. This way, you'll be able to save unit testing dollars.

#### Use Primitives

We recommend using basic types to build up your request objects — that means strings, integers, booleans, and so on. We're just abstracting away input parameters. You should be able to consume Application Services independently from the delivery mechanism. Even pretty complicated HTML forms get translated into basic types all the time at the controller level. You don't want to mix up your framework and your business logic.

With certain scenarios, it's tempting to use Value Objects directly. Don't do it. Updates on the Value Object definition will affect all clients, and you'll be coupling clients with your Domain logic.

#### Serializable

A cool side effect of using basic types is that any request object can easily be serialized into a string, sent through the wire, and stored in a messaging system or database.

#### No Business Logic

Avoid putting any business logic — even validation — inside your request objects. Validation should take place inside your Domain — this is inside your Entities, Value Objects, Domain Services, etc. Validation is a way of enforcing business invariants and Domain constraints.

#### No Tests

Application requests are data structures, not objects. Unit testing data structures is like testing getters and setters. There's no behavior to test, so there isn't much value in trying to unit test request objects and DTOs. These structures will be covered as a side effect of more elaborate tests, such as Integration or Acceptance tests.

Commands are an alternative to request objects. We could design a Service with multiple Application methods, and each one of them with the parameters you'd put inside the Request. This is OK for simple applications, but we'll worry about this topic later.


Anatomy of an Application Service
---------------------------------

* * *

Once we have the data encapsulated in a request, it's time for the business logic. As Vaughn Vernon [says](https://www.amazon.com/Implementing-Domain-Driven-Design-Vaughn-Vernon/dp/0321834577): "Keep Application Services thin, using them only to coordinate tasks on the model."

The first thing to do is to extract the necessary information from the request, That is, the `email` and `password`. At a high level, we need to check if there's an existing user with a particular email. If this isn't the case, then we create and add the user to the `UserRepository`. In the special case of finding a user with the same email, we raise an exception so the client can treat it their own way — by displaying an error, retrying, or just ignoring it:

    namespace Lw\Application\Service\User;
    
    use Ddd\Application\Service\ApplicationService;
    use Lw\Domain\Model\User\User;
    use Lw\Domain\Model\User\UserAlreadyExistsException;
    use Lw\Domain\Model\User\UserRepository;
    
    class SignUpUserService
    {
        private $userRepository;
    
        public function __construct(UserRepository $userRepository) 
        {
            $this->userRepository = $userRepository;
        }
    
        public function execute(SignUpUserRequest $request)
        {
            $email = $request->email();
            $password = $request->password();
    
            $user = $this->userRepository->ofEmail($email);
            if ($user) {
                throw new UserAlreadyExistsException();
            }
    
            $this->userRepository->add(
                new User(
                    $this->userRepository->nextIdentity(),
                    $email ,
                    $password
                )
            );
        }
    }

Nice! If you're wondering what this `UserRepository` thing is doing in the constructor, we'll show you that next.

> ### Note
> **`**Handling Exceptions**`**Exceptions raised by Application Services are a way of communicating unusual cases and flows to the client. Exceptions on this layer are related to business logic (like not finding a user), and not implementation details (like `PDOException`, `PredisException`, or `DoctrineException`).

### Dependency Inversion

Handling users is not the responsibility of the Service. As we saw in [Chapter 10](../chapters/10%20Repositories.md), _Repositories_, there's a specialized class that deals with `User` collections: the `User` Repository. This is a dependency from the Application Service to the Repository. We don't want to couple the Application Service with a concrete implementation of the Repository, as then we'd be coupling our Service with Infrastructure details. So we depend on the contract (interface) that concrete implementations depend on, the `UserRepository`.

A specific implementation of the `UserRepository` will be built and passed in at runtime — for example, with `DoctrineUserRepository`, a specific implementation that uses Doctrine. Passing a specific implementation will also work when testing. For example, `NotAvailableUserRepository` can be a specific implementation that will throw exceptions each time an operation is performed. This way, we can test all Application Service behaviors, including _sad_ paths, which is when the application must behave properly, even if something goes wrong.

Application Services could depend on Domain Services like `GetBadgesByUser` too. At runtime, the implementation for such a Service could be quite elaborate. Imagine an `HttpGetBadgesByUser` for integrating a Bounded Context through HTTP protocol.

Depending on abstractions, we'll make our Application Service immune to low-level Infrastructure changes.

### Instantiating Application Services

Instantiating just your Application Service is easy, but building the dependency tree might be tricky, depending on how complicated the dependencies are to build. For such a purpose, most frameworks come with a Dependency Injection Container. Without one, you'll end up with something like the following code somewhere in your controller:

    $redisClient = new Predis\Client([
        'scheme' => 'tcp',
        'host' => '10.0.0.1',
        'port' => 6379
    ]);
    
    $userRepository = new RedisUserRepository($redisClient);
    $signUp = new SignUpUserService($userRepository);
    $signUp->execute(new SignUpUserRequest(
        'user@example.com',
        'password'
    ));

We decided to use the [Redis](http://redis.io/) implementation for the `UserRepository`. In the previous code example, we built all dependencies needed for building a Repository that uses Redis internally. Those dependencies are: a [Predis](https://github.com/nrk/predis) client, and all parameters to connect to our Redis server. This is not only inefficient, but it also spreads duplication across controllers.

You could refactor the construction logic into a Factory, or you could use a Dependency Injection Container — most modern frameworks come with it.

> ### Note
> **`Is It Bad to Use a Dependency Injection Container?`**Not at all. Dependency Injection Containers are just a tool. They help by abstracting away the complexities of building your dependencies. They come in handy for building Infrastructure artifacts. Symfony offers a complete solution.

> ### Note
> Take into account the fact that passing the entire container as a whole to one of the Services is a bad practice. That would be like coupling the entire context of your application with the Domain. If a Service needs specific objects, build them from your framework and pass them as dependencies into the Service, but don't make that Service aware of the entire context.

Let's see how would we build dependencies in Silex:

    $app = new \Silex\Application();
    $app['redis_parameters'] = [
         'scheme' => 'tcp',
         'host' => '127.0.0.1',
         'port' => 6379
    ];
    
    $app['redis'] = $app->share(function ($app) {
        return new Predis\Client($app['redis_parameters']);
    });
    
    $app['user_repository'] = $app->share(function($app) {
        return new RedisUserRepository(
            $app['redis']
        );
    });
    
    $app['sign_up_user_application_service'] = $app->share(function($app) {
        return new SignUpUserService(
            $app['user_repository']
        );
    });
    
    // ...
    
    $app->match('/signup' ,function (Request $request) use ($app) {
        // ...
        $app['sign_up_user_application_service']->execute(
            new SignUpUserRequest(
                $request->get('email'),
                $request->get('password')
            )
        );
        // ...
    });

As you can see, `$app` is used as the Service Container. We register all the components needed, along with their dependencies. `sign_up_user_application_service` depends on the definitions made above. Changing the implementation for the `user_repository` is as easy as returning something else (MySQL, MongoDB, and so on.), so we don't need to change the Service code at all.

The equivalent for a Symfony application looks like this:

    <?xml version=" 1.0" ?>
    <container xmlns="http://symfony.com/schema/dic/services"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="
            http://symfony.com/schema/dic/services
            http://symfony.com/schema/dic/services/services-1.0.xsd">
        <services>
            <service
                id="sign_up_user_application_service"
                class="SignUpUserService">
                <argument type="service" id="user_repository" />
            </service>
    
            <service
                id="user_repository"
                class="RedisUserRepository">
                <argument type="service">
                    <service class="Predis\Client" />
                </argument>
            </service>
        </services> 
    </container>

Now that you have the definition of your Application Service in the Symfony Service Container, getting it later is pretty straightforward. All delivery mechanisms — Web Controllers, REST Controllers, and even Console Commands — share the same definition. The Service is available on any class implementing the `ContainerAware` interface. Getting the Service is as easy as calling `$this->get('sign_up_user_application_service'`.

To summarize, how you build your Services (adhoc, using Service Containers, using Factories, and so on.) doesn't matter. However, it's important to keep your Application Services setup out of the Infrastructure boundary.

#### Customize an Application Service

The main way to customize your Application Service is by choosing which dependencies you're passing in. Depending on your Service Container capabilities, that could be a bit tricky, so you can also add a setter to change the dependency on the fly. For example, you may need to change an output dependency so that you can set up a default one and then change it afterward. If logic gets too complicated, you can create an Application Service Factory that will handle this situation for you.

### Execution

There are two different approaches for invoking Application Services: a dedicated class per use case with a single execution method, and multiple Application Services and use cases inside the same class.

#### One Class Per Application Service

This is our preferred approach, and probably the one that fits all scenarios:

    class SignUpUserService 
    { 
        // ...
        public function execute(SignUpUserRequest $request)
        {
           // ...
        }
    }

Using a dedicated class per Application Service makes the code more robust against external changes (Single Responsibility Principle). There are fewer reasons to change the class, as the Service does one and only one thing. The Application Service will be easier to test, seeing as it does less things. It's easier to implement a common Application Service contract, making class decoration easier (check out Sub section _Transactions_ of [Chapter 10](../chapters/10%20Repositories.md),  _Repositories_ ). This will also result in higher cohesion, as all dependencies are exclusively dedicated to a single use case.

The `execution` method could have a more expressive name, like `signUp`. However, the execute [Command pattern](http://martinfowler.com/bliki/DecoratedCommand.html) format standardizes a common contract across Application Services, thereby enabling easy decoration, which comes in handy for transactions.

#### Multiple Application Service Methods per Class

Sometimes it might be a good idea to group cohesive Application Services under the same class:

    class UserService
    {
        // ...
        public function signUp(SignUpUserRequest $request)
        {
            // ...
        }
    
        public function signIn(SignUpUserRequest $request)
        {
            // ...
        }
    
        public function logOut(LogOutUserRequest $request)
        {
            // ...
        }
    }

We don't recommend such an approach, as not all Application Services are 100 percent cohesive. Some Services will require different dependencies, and you'll end up with Application Services depending on things they don't need. Another issue is that this kind of class grows fast. As it violates the Single Responsibility Principle, there will be multiple reasons to change and maybe even break it.

### Returning Values

After signing up, we might be thinking about redirecting the user to a profile page. The natural way of passing the required information back to the controller is to return the User Entity directly from the Service:

    class SignUpUserService
    {
        // ...
    
        public function execute(SignUpUserRequest $request)
        {
            $user = new User(
                $this->userRepository->nextIdentity(),
                $email,
                $password
            );
    
            $this->userRepository->add($user);
    
            return $user;
        }
    }

Then, from the controller, we would pick up the `id` field and redirect to some other place. However, think twice about what we've just done. We returned a full-featured Entity to the controller, which will allow the delivery mechanism to bypass the Application Layer and interact directly with the Domain.

Imagine the `User` Entity offers an `updateEmailAddress` method. You could try to prevent it, but at some point in the future, somebody might think about using it:

    $app-> match( '/signup' , function (Request $request) use ($app) {
       // ...
       $user = $app['sign_up_user_application_service']->execute(
           new SignUpUserRequest(
               $request->get('email'),
               $request->get('password'))
       );
       $user->updateEmailAddress('shouldnotupdate@email.com');
       // ...
    });

Not only that, but the data that the presentation layer needs is not the same that the Domain manages. We don't want to evolve and couple the Domain layer around the presentation layer. Instead, we want them to evolve freely.

To do this, we need a flexible way of decoupling both layers.

#### DTO from Aggregate Instances

We could return sterile data structures with the information the presentation layer needs. As we've seen before, DTOs fit with this scenario. We just need to compose them in the Application Service and return them to the client:

    class UserDTO
    {
        private $email ;
        // ...
    
        public function __construct(User $user)
        {
            $this->email = $user->email ();
            // ...
        }
    
        public function email ()
        {
            return $this->email ;
        }
    }

The `UserDTO` will expose whatever read-only data we need from the `User` Entity on the presentation layer, thereby avoiding exposing behavior:

    class SignUpUserService
    {
        public function execute(SignUpUserRequest $request)
        {
            // ...
    
            $user = // ...
    
            return new UserDTO($user);
        }
    }

Mission accomplished. Now we could pass parameters to the template engine and transform them into widgets, tags, or subtemplates, or do whatever we want with the data on the presentation side:

    $app->match('/signup' , function (Request $request) use ($app) {
        /**
         * @var UserDTO $user
         */
        $userDto=$app['sign_up_user_application_service']->execute(
            new SignUpUserRequest(
                $request->get('email'),
                $request->get('password')
            )
        );
    
        // ...
    });

However, letting the Application Service decide how to build the DTO reveals another limitation. As building the DTO depends exclusively on the Application Service, adapting the DTO to different clients will be very difficult. Consider the data needed for a redirect on a Web Controller and the data needed for a REST response for the same use case. Not the same data at all.

Let's allow the client to define how to build the DTO by passing a specific DTO Assembler:

    class SignUpUserService
    {
        private $userDtoAssembler;
    
        public function __construct(
            UserRepository $userRepository,
            UserDTOAssembler $userDtoAssembler
        ) {
            $this->userRepository = $userRepository;
            $this->userDtoAssembler = $userDtoAssembler;
        }
    
        public function execute(SignUpUserRequest $request)
        {
            $user = // ...
    
            return $this->userDtoAssembler->assemble($user);
        }
    }

Now the client can customize the response by passing a specific `UserDTOAssembler`.

#### Data Transformers

There are some cases where generating intermediate DTOs for more complex responses like JSON, XML, CSV, and iCAL Contact could be seen as an unnecessary overhead. We could output the representation in a buffer and ask for it later on the delivery side.

Transformers help reduce this overhead by transforming high-level Domain concepts into low-level client details. Let's see an example:

    interface UserDataTransformer
    {
        public function write(User $user);
    
        /**
         * @return mixed
         */
        public function read();
    }

Consider the case of generating different data representations for a given product. Usually, the product information is served through a web interface (HTML), but we might be interested in offering other formats, like XML, JSON, or CSV. This might enable integrations with other Services.

Consider a similar case for a blog. We might expose our potential as writers in HTML to the world, but some people will be interested in consuming our articles through RSS. The use cases — Application Services — remain the same. The representation doesn't.

DTOs are a clean and simple solution that could be passed to template engines for different representations, but this might complicate the logic of this last step of data transformation, as the logic for such templates could become a problem to maintain, test, and understand.

Data Transformers might be a better approach on specific cases. These are just black boxes with Domain concepts (Aggregates, Entities, and so on.) as inputs and read-only representations (XML, JSON, CSV, and so on.) as outputs. These transformers could be really easy to test:

    class JsonUserDataTransformer implements UserDataTransformer
    {
        private $data;
    
        public function write(User $user)
        {
            // More complex logic could be placed here
            // As using JMSSerializer, native json, etc.
            $this->data = json_encode($user);
        }
    
        /**
         * @return string
         */
        public function read()
        {
            return $this->data;
        }
    }

That was easy. Wondering how the XML or CSV one would look? Let's see how to integrate the Data Transformer with our Application Service:

    class SignUpUserService
    {
        private $userRepository;
        private $userDataTransformer;
    
        public function __construct(
            UserRepository $userRepository,
            UserDataTransformer $userDataTransformer
        ) {
            $this->userRepository = $userRepository;
            $this->userDataTransformer = $userDataTransformer;
        }
    
        public function execute(SignUpUserRequest $request)
        {
            $user = // ...
            $this->userDataTransformer()->write($user);
        }
    
        /**
         * @return UserDataTransformer
         */
        public function userDataTransformer()
        {
            return $this->userDataTransformer;
        } 
    }

That's similar to the DTO Assembler approach, but this time without returning a concrete value. The Data Transformer is being used to hold and interact with the data.

The main issue with DTOs is the overhead of writing them. Most of the time, your Domain concepts and DTO representations will present the same structure. Most of the time, you'll feel it's not worth your time to make such a mapping. That said, the relationship between representations and Aggregates is not 1:1. You can represent two Aggregates together in a single representation. You can also represent the same Aggregate in multiple ways. How you do it always depends on your use cases.

However, according to [Martin Fowler](http://www.martinfowler.com/books/eaa.html):

One case where it is useful to use something like a DTO is **when you have a significant mismatch between the model in your presentation layer and the underlying domain model**. In this case it makes sense to make presentation specific facade/gateway that maps from the domain model and presents an interface that's convenient for the presentation. It fits in nicely with Presentation Model. This is worth doing, but it is only worth doing for screens that have this mismatch (in this case it isn't extra work, since you'd have to do it in the screen anyway.)

We think the long-term vision will be worth the investment. On medium to big projects, interface representations and Domain concepts change at very different rhythms. You might want to decouple them from each other to lower the friction for updates. Using DTOs or Data Transformers allows you to evolve your model freely without having to think about breaking the layout all the time.

### Multiple Application Services on Compound Layouts

Most of the time, no layout is as simple as a single Application Service. Our projects have pretty complicated interfaces.

Consider the homepage of a specific project. How can we render so many pieces and use cases? There are a few options, so let's check them out.

#### AJAX Content Integration

You could let the browser ask for different endpoints directly and combine the data in the layout right after through AJAX or [Hijax](https://en.wikipedia.org/wiki/Hijax). This will avoid mixing a lot of Application Services in your controllers, but it might have a performance penalty, depending on the number of requests triggered.

#### ESI Content Integration

**Edge Side Includes** (**ESI**) is a tiny markup language similar to the previous approach, but on the server side. It requires additional effort configuring extra middleware, like NGINX or Varnish, to make it work. Includes (ESI) is a tiny markup language similar to the previous approach, but on the server side. It requires additional effort configuring extra middleware, like NGINX or [Varnish](https://en.wikipedia.org/wiki/Edge_Side_Includes), to make it work.

#### Symfony Sub Requests

If you use Symfony, Sub Requests could be an interesting option. According to the [Symfony Documentation](http://symfony.com/doc/current/components/http_kernel/introduction.html#sub-requests):

In addition to the main request that's sent into `HttpKernel::handle`, you can also send so-called sub request. A sub request looks and acts like any other request, but typically serves to render just one small portion of a page instead of a full page. You'll most commonly make sub-requests from your controller (or perhaps from inside a template, that's being rendered by your controller). This creates another full request-response cycle where this new Request is transformed into a Response. The only difference internally is that some listeners (Example: security) may only act upon the master request. Each listener is passed some sub-class of `KernelEvent`, whose `isMasterRequest()` can be used to check if the current request is a master or sub request.

This is great, as you'll get the benefits of invoking separate Application Services without AJAX penalties or complicated ESI configurations.

#### One Controller, Multiple Application Services

One last option could be managing multiple Application Services within the same controller, though the controller logic could get a little bit dirty, as it'll handle and merge the responses to pass to the view.


Testing Application Services
----------------------------

* * *

As you're interested in testing the behavior of the Application Service itself, there's no need to turn it into an integration test with complicated setups going against a real database. You're not interested in testing the low-level details, so most of the time, a unit test will be enough:

    class SignUpUserServiceTest extends \PHPUnit_Framework_TestCase
    {
        /**
         * @var \Lw\Domain\Model\User\UserRepository
         */
        private $userRepository;
    
        /**
         * @var SignUpUserService
         */
        private $signUpUserService;
    
        public function setUp()
        {
            $this->userRepository = new InMemoryUserRepository();
            $this->signUpUserService = new SignUpUserService(
                $this->userRepository
            );
        }
    
        /**
         * @test
         * @expectedException   
         *     \Lw\Domain\Model\User\UserAlreadyExistsException
         */
        public function alreadyExistingEmailShouldThrowAnException()
        {
            $this->executeSignUp();
            $this->executeSignUp();
        }
    
        private function executeSignUp()
        {
            return $this->signUpUserService->execute(
                new SignUpUserRequest(
                    'user@example.com',
                    'password'
                )
            );
        }
    
        /**
         * @test
         */
        public function afterUserSignUpItShouldBeInTheRepository()
        {
            $user = $this->executeSignUp();
    
            $this->assertSame(
                $user,
                $this->userRepository->ofId($user->id())
            );
        }
    }

We've used an in-memory implementation for the `User` Repository. This is what is called a Fake: a fully functional implementation for the Repository that will make our test work as a unit. We don't need to go to the database to test the behavior of this class. That would make our test slow and fragile.

Checking for a Domain Events submission might be interesting too. If creating a user fires a user registered event, ensuring it's been triggered might be a good idea:

    class SignUpUserServiceTest extends \PHPUnit_Framework_TestCase
    {
        // ...
    
        /**
         * @test
         */
        public function itShouldPublishUserRegisteredEvent()
        {
            $subscriber = new SpySubscriber();
            $id = DomainEventPublisher::instance()->subscribe($subscriber);
    
            $user = $this->executeSignUp();
            $userId = $user->id();
    
            DomainEventPublisher::instance()->unsubscribe($id);
            $this->assertUserRegisteredEventPublished(
                $subscriber, $userId
            );
        }  
    
        private function assertUserRegisteredEventPublished(
            $subscriber, $userId
        ) {
            $this->assertInstanceOf(
                'UserRegistered', $subscriber->domainEvent
            );
            $this->assertTrue(
                $subscriber->domainEvent->userId()->equals($userId)
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



Transactions
------------

* * *

Transactions are an implementation detail related to the persistence mechanism. The Domain layer shouldn't be aware of this low-level implementation detail. Thinking about beginning, committing, or rolling back a transaction at this level is a big smell. This level of detail belongs to the Infrastructure layer.

The best way of handling transactions is to not handle them at all. We could wrap our Application Services with a Decorator implementation for handling the transaction session automatically.

We've implemented a solution to this problem in one of our repositories, and you can check it out [here](https://github.com/dddinphp/ddd):

    interface TransactionalSession
    {
        /**
         * @return mixed
         */
        public function executeAtomically(callable $operation);
    }

This contract takes a piece of code and executes it atomically. Depending on your persistence mechanism, you'll end up with different implementations.

Let's see how we could do it with Doctrine ORM:

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

This is how a client would use the previous code:

    /** @var EntityManager $em */
    $nonTxApplicationService = new SignUpUserService(
        $em->getRepository('BoundedContext\Domain\Model\User\User')
    );
    
    $txApplicationService = new TransactionalApplicationService(
        $nonTxApplicationService,
        new DoctrineSession($em)
    );
    
    $response = $txApplicationService->execute(
        new SignUpUserRequest(
            'user@example.com',
            'password'
        )
    );

Now that we have the Doctrine implementation for transactional sessions, it would be great to create a Decorator for our Application Services. With this approach, we make transactional requests transparent to the Domain:

    class TransactionalApplicationService implements ApplicationService
    {
        private $session;
        private $service;
    
        public function __construct(
            ApplicationService $service, TransactionalSession $session
        ) {
            $this->session = $session;
            $this->service = $service;
        }
    
        public function execute(BaseRequest $request)
        {
            $operation = function () use ($request) {
                return $this->service->execute($request);
            };
    
            return $this->session->executeAtomically($operation);
        }
    }

A nice side effect of using Doctrine Session is that it automatically manages the flush method, so you don't need to add the flush inside your Domain or Infrastructure.


Security
--------

* * *

In case you're wondering how to manage and handle user credentials and security in general, unless it's the responsibility of your Domain, we recommend letting the framework handle it. The user session is a concern of the delivery mechanism. Polluting the Domain with such concepts will make it harder to develop.


Domain Events
-------------

* * *

Domain Event listeners have to be configured before the Application Service gets executed, or nobody will be noticed. There are situations where you'll have to be explicit and configure the listener before executing the Application Service:

    // ...
    $subscriber = new SpySubscriber();
    DomainEventPublisher::instance()->subscribe($subscriber);
    
    $applicationService = // ...
    $applicationService->execute(...);

Most of the time, this will be done by configuring the Dependency Injection Container.


Command Handlers
----------------

* * *

An interesting way of executing Application Services is through a Command Bus library. A good one is [Tactician](https://tactician.thephpleague.com/). From the Tactician website:

What is a Command Bus? The term is mostly used when we combine the [Command pattern](https://en.wikipedia.org/wiki/Command_pattern) with a [service layer](http://martinfowler.com/eaaCatalog/serviceLayer.html). Its job is to take a Command object (which describes what the user wants to do) and match it to a Handler (which executes it). This can help structure your code neatly.

— our Application Services are the Service Layer, and our Request objects look pretty much like Commands.

Fair enough — our Application Services are the Service Layer, and our Request objects look pretty much like Commands. Wouldn't it be great if we had a mechanism to link all the Application Services, and then based on the Request, execute the correct one? Well, that's actually what a Command Bus is.

### Tactician Library and Other Options

Tactician is a Command Bus library, which allows you to use the Command pattern for your Application Services. It's especially convenient for Application Services, but you could use any kind of input.

Let's see an example from the [Tactician](http://tactician.thephpleague.com/) website:

    // You build a simple message object like this:
    class PurchaseProductCommand
    {
        protected $productId;
        protected $userId;
    
        // ...and constructor to assign those properties...
    }
    
    // And a Handler class that expects it:
    class PurchaseProductHandler
    {
        public function handle(PurchaseProductCommand $command)
        {
            // use command to update your models, etc
        }
    }
    // And then in your Controllers, you can fill in the command using your favorite
    // form or serializer library, then drop it in the CommandBus and you're done!
    $command = new PurchaseProductCommand(42, 29);
    $commandBus->handle($command);

That's it. Tactician is the `$commandBus` Service. It does all the plumbing for finding the right handler and method, which can avoid a lot of boilerplate code. Here, Commands and Handlers are just normal classes, but you can configure whichever one fits your app better.

In summary, we can conclude that Commands are just Request objects, and Command Handlers are just Application Services.

A cool thing about Tactician (and Command Buses in general) is that they're really easy to extend. Tactician provides plug-ins for common tasks, like logging and database transactions. That way, you can forget about setting up the wiring on every handler.

Another interesting plug-in for [Tactician is Bernard](http://bernard.readthedocs.org/) integration. Bernard is an asynchronous job queue that allows you to leave some tasks for later processing. Heavy processes block the response. Most of the time, we can branch and delay their execution for later. For the best experience, answer the customer as fast as possible and let them know once the branched processes are done.

Matthias Noback has developed another similar project, called [SimpleBus](http://simplebus.github.io/MessageBus/), that can be used as an alternative to Tactician. The main difference is that `SimpleBus` Command Handlers don't have a return value.


Wrap-Up
-------

* * *

Application Services represent the Application layer of your Bounded Context. These high-level use cases should be relatively simple and skinny, as their purpose evolves around Domain coordination. Application Services are the entry point for Domain logic interaction. We've seen that Requests and Commands keep things organized; that DTOs and Data Transformers allow us to decouple data representation from Domain conceptualization; that building Application Services is pretty straightforward with Dependency Injection Containers; and that we have plenty of options for combining Application Services in complex layouts.
