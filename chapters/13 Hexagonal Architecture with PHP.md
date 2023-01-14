Chapter 13. Hexagonal Architecture with PHP
-------------------------------------------

> _The following article was posted in php|architect magazine in June 2014 by Carlos Buenosvinos._



Introduction
------------

* * *

With the rise of **Domain-Driven Design** (**DDD**), architectures promoting domain centric designs are becoming more popular. This is the case with **Hexagonal Architecture**, also known as **Ports and Adapters**, that seems to have being rediscovered just now by PHP developers. Invented in 2005 by Alistair Cockburn, one of the Agile Manifesto authors, the Hexagonal Architecture allows an application to be equally driven by users, programs, automated tests or batch scripts, and to be developed and tested in isolation from its eventual run-time devices and databases. This results into agnostic infrastructure web applications that are easier to test, write and maintain. Let's see how to apply it using real PHP examples.

Your company is building a brainstorming system called _Idy_. Users add and rate ideas so the most interesting ones can be implemented in a company. It is Monday morning, another sprint is starting and you are reviewing some user stories with your team and your Product Owner. **As a not logged in user, I want to rate an idea and the author should be notified by email**, that's a really important one, isn't it?



First Approach
--------------

* * *

As a good developer, you decide to divide and conquer the user story, so you'll start with the first part, _I want to rate an idea_. After that, you will face _the author should be notified by email_. That sounds like a plan.

In terms of business rules, rating an idea is as easy as finding the idea by its identifier in the ideas repository, where all the ideas live, add the rating, recalculate the average and save the idea back. If the idea does not exist or the repository is not available we should throw an exception so we can show an error message, redirect the user or do whatever the business asks us for.

In order to _execute_ this _UseCase_, we just need the idea identifier and the rating from the user. Two integers that would come from the user request.

Your company web application is dealing with a Zend Framework 1 legacy application. As most of companies, probably some parts of your app may be newer, more SOLID, and others may just be a big ball of mud. However, you know that it does not matter at all which framework you are using, it is all about writing clean code that makes maintenance a low cost task for your company.

You're trying to apply some Agile principles you remember from your last conference, how it was, yeah, I remember "make it work, make it right, make it fast". After some time working you get something like Listing 1.

    class IdeaController extends Zend_Controller_Action
    {
        public function rateAction()
        {
            // Getting parameters from the request
            $ideaId = $this->request->getParam('id');
            $rating = $this->request->getParam('rating');
    
            // Building database connection
            $db = new Zend_Db_Adapter_Pdo_Mysql([
                'host'     => 'localhost',
                'username' => 'idy',
                'password' => '',
                'dbname'   => 'idy'
            ]);
    
            // Finding the idea in the database
            $sql = 'SELECT * FROM ideas WHERE idea_id = ?';
            $row = $db->fetchRow($sql, $ideaId);
            if (!$row) {
                throw new Exception('Idea does not exist');
            }
    
            // Building the idea from the database
            $idea = new Idea();
            $idea->setId($row['id']);
            $idea->setTitle($row['title']);
            $idea->setDescription($row['description']);
            $idea->setRating($row['rating']);
            $idea->setVotes($row['votes']);
            $idea->setAuthor($row['email']);
    
            // Add user rating
            $idea->addRating($rating);
    
            // Update the idea and save it to the database
            $data = [
                'votes' => $idea->getVotes(),
                'rating' => $idea->getRating()
            ];
            $where['idea_id = ?'] = $ideaId;
            $db->update('ideas', $data, $where);
    
            // Redirect to view idea page
            $this->redirect('/idea/' . $ideaId);
        }
    }

I know what readers are thinking: _Who is going to access data directly from the controller? This is a 90's example!_, ok, ok, you're right. If you are already using a framework, it is likely that you are also using an ORM. Maybe done by yourself or any of the existing ones such as Doctrine, Eloquent, Zend, and so on. If this is the case, you are one step further from those who have some Database connection object but don't count your chickens before they're hatched.

For newbies, Listing 1 code just works. However, if you take a closer look at the Controller, you'll see more than business rules, you'll also see how your web framework routes a request into your business rules, references to the database or how to connect to it. So close, you see references to your **infrastructure**.

Infrastructure is the **detail that makes your business rules work**. Obviously, we need some way to get to them (API, web, console apps, and so on.) and effectively we need some physical place to store our ideas (memory, database, NoSQL, and so on.). However, we should be able to exchange any of these pieces with another that behaves in the same way but with different implementations. What about starting with the Database access?

All those `Zend_DB_Adapter` connections (or straight MySQL commands if that's your case) are asking to be promoted to some sort of object that encapsulates fetching and persisting Idea objects. They are begging for being a Repository.



Repositories and the Persistence Edge
-------------------------------------

* * *

Whether there is a change in the business rules or in the infrastructure, we must edit the same piece of code. Believe me, in CS, you don't want many people touching the same piece of code for different reasons. Try to make your functions do one and just one thing so it is less probable having people messing around with the same piece of code. You can learn more about this by having a look at the **Single Responsibility Principle** (**SRP**). For more information about this principle: [http://www.objectmentor.com/resources/articles/srp.pdf](http://www.objectmentor.com/resources/articles/srp.pdf)

Listing 1 is clearly this case. If we want to move to Redis or add the author notification feature, you'll have to update the `rateAction` method. Chances to affect aspects of the `rateAction` not related with the one updating are high. Listing 1 code is fragile. If it is common in your team to hear _If it works, don't touch it_, SRP is missing.

So, we must decouple our code and encapsulate the responsibility for dealing with fetching and persisting ideas into another object. The best way, as explained before, is using a Repository. Challenged accepted! Let's see the results in Listing 2:

    class IdeaController extends Zend_Controller_Action
    {
        public function rateAction()
        {
            $ideaId = $this->request->getParam('id');
            $rating = $this->request->getParam('rating');
    
            $ideaRepository = new IdeaRepository();
            $idea = $ideaRepository->find($ideaId);
            if (!$idea) {
                throw new Exception('Idea does not exist');
            }
    
            $idea->addRating($rating);
            $ideaRepository->update($idea);
    
            $this->redirect('/idea/' . $ideaId);
        }
    }

    class IdeaRepository
    {
        private $client;
    
        public function __construct()
        {
            $this->client = new Zend_Db_Adapter_Pdo_Mysql([
                'host' => 'localhost',
                'username' => 'idy',
                'password' => '',
                'dbname' => 'idy'
            ]);
        }
    
        public function find($id)
        {
            $sql = 'SELECT * FROM ideas WHERE idea_id = ?';
            $row = $this->client->fetchRow($sql, $id);
            if (!$row) {
                return null;
            }
    
            $idea = new Idea();
            $idea->setId($row['id']);
            $idea->setTitle($row['title']);
            $idea->setDescription($row['description']);
            $idea->setRating($row['rating']);
            $idea->setVotes($row['votes']);
            $idea->setAuthor($row['email']);
    
            return $idea;
        }
    
        public function update(Idea $idea)
        {
            $data = [
                'title' => $idea->getTitle(),
                'description' => $idea->getDescription(),
                'rating' => $idea->getRating(),
                'votes' => $idea->getVotes(),
                'email' => $idea->getAuthor(),
            ];
    
            $where = ['idea_id = ?' => $idea->getId()];
            $this->client->update('ideas', $data, $where);
        }
    }

The result is nicer. The `rateAction` of the `IdeaController` is more understandable. When read, it talks about business rules. `IdeaRepository` is a **business concept**. When talking with business guys, they understand what an `IdeaRepository` is: A place where I put Ideas and get them.

A Repository _mediates between the domain and data mapping layers using a collection-like interface for accessing domain objects_. as found in Martin Fowler's pattern catalog.

If you are already using an ORM such as Doctrine, your current repositories extend from an `EntityRepository`. If you need to get one of those repositories, you ask Doctrine `EntityManager` to do the job. The resulting code would be almost the same, with an extra access to the `EntityManager` in the controller action to get the `IdeaRepository`.

At this point, we can see in the landscape one of the edges of our hexagon, the _persistence_ edge. However, this side is not well drawn, there is still some relationship between what an `IdeaRepository` is and how it is implemented.

In order to make an effective separation between our _application boundary_ and the _infrastructure boundary_ we need an additional step. We need to explicitly decouple behavior from implementation using some sort of interface.



Decoupling Business and Persistence
-----------------------------------

* * *

Have you ever experienced the situation when you start talking to your Product Owner, Business Analyst or Project Manager about your issues with the Database? Can you remember their faces when explaining how to persist and fetch an object? They had no idea what you were talking about.

The truth is that they don't care, but that's ok. If you decide to store the ideas in a MySQL server, Redis or SQLite it is your problem, not theirs. Remember, from a business standpoint, **your infrastructure is a detail**. Business rules are not going to change whether you use Symfony or Zend Framework, MySQL or PostgreSQL, REST or SOAP, and so on.

That's why it is important to decouple our `IdeaRepository` from its implementation. The easiest way is to use a proper interface. How can we achieve that? Let's take a look at Listing 3.

    class IdeaController extends Zend_Controller_Action
    {
        public function rateAction()
        {
            $ideaId = $this->request->getParam('id');
            $rating = $this->request->getParam('rating');
    
            $ideaRepository = new MySQLIdeaRepository();
            $idea = $ideaRepository->find($ideaId);
            if(!$idea) {
                throw new Exception('Idea does not exist');
            }
    
            $idea->addRating($rating);
            $ideaRepository->update($idea);
    
            $this->redirect('/idea/' . $ideaId);
        }
    }
    
    interface IdeaRepository
    {
        /**
         * @param int $id
         * @return null|Idea
         */
        public function find($id);
    
        /**
         * @param Idea $idea
         */
        public function update(Idea $idea);
    }
    
    class MySQLIdeaRepository implements IdeaRepository
    {
        // ...
    }

Easy, isn't it? We have extracted the `IdeaRepository` behavior into an interface, renamed the `IdeaRepository` into `MySQLIdeaRepository` and updated the `rateAction` to use our `MySQLIdeaRepository`. But what's the benefit?

We can now exchange the repository used in the controller with any implementing the same interface. So, let's try a different implementation.



Migrating our Persistence to Redis
----------------------------------

* * *

During the sprint and after talking to some mates, you realize that using a NoSQL strategy could improve the performance of your feature. Redis is one of your best friends. Go for it and show me your Listing 4:

    class IdeaController extends Zend_Controller_Action
    {
        public function rateAction()
        {
            $ideaId = $this->request->getParam('id');
            $rating = $this->request->getParam('rating');
    
            $ideaRepository = new RedisIdeaRepository();
            $idea = $ideaRepository->find($ideaId);
            if (!$idea) {
                throw new Exception('Idea does not exist');
            }
    
            $idea->addRating($rating);
            $ideaRepository->update($idea);
    
            $this->redirect('/idea/' . $ideaId);
        }
    }
    
    interface IdeaRepository
    {
        // ...
    }
    
    class RedisIdeaRepository implements IdeaRepository
    {
        private $client;
    
        public function __construct()
        {
            $this->client = new Predis\Client();
        }
    
        public function find($id)
        {
            $idea = $this->client->get($this->getKey($id));
            if (!$idea) {
                return null;
            }
            return unserialize($idea);
        }
    
        public function update(Idea $idea)
        {
            $this->client->set(
                $this->getKey($idea->getId()),
                serialize($idea)
            );
        }
    
        private function getKey($id)
        {
            return 'idea:' . $id;
        }
    }

Easy again. You've created a `RedisIdeaRepository` that implements `IdeaRepository` interface and we have decided to use Predis as a connection manager. Code looks smaller, easier and faster. But what about the controller? It remains the same, we have just changed which repository to use, but it was just one line of code.

As an exercise for the reader, try to create the `IdeaRepository` for SQLite, a file or an in-memory implementation using arrays. Extra points if you think about how ORM Repositories fit with Domain Repositories and how ORM _@annotations_ affect this architecture.



Decouple Business and Web Framework
-----------------------------------

* * *

We have already seen how easy it can be to changing from one persistence strategy to another. However, the persistence is not the only edge from our Hexagon. What about how the user interacts with the application?

Your CTO has set up in the roadmap that your team is moving to Symfony2, so when developing new features in you current ZF1 application, we would like to make the incoming migration easier. That's tricky, show me your Listing 5:

    class IdeaController extends Zend_Controller_Action
    {
        public function rateAction()
        { 
            $ideaId = $this->request->getParam('id');
            $rating = $this->request->getParam('rating');
    
            $ideaRepository = new RedisIdeaRepository();
            $useCase = new RateIdeaUseCase($ideaRepository);
            $response = $useCase->execute($ideaId, $rating);
    
            $this->redirect('/idea/' . $ideaId);
        }
    }
    
    interface IdeaRepository
    {
        // ...
    }
    
    class RateIdeaUseCase
    {
        private $ideaRepository;
    
        public function __construct(IdeaRepository $ideaRepository)
        {
            $this->ideaRepository = $ideaRepository;
        }
    
        public function execute($ideaId, $rating)
        {
            try {
                $idea = $this->ideaRepository->find($ideaId);
            } catch(Exception $e) {
                throw new RepositoryNotAvailableException();
            }
    
            if (!$idea) {
                throw new IdeaDoesNotExistException();
            }
    
            try {
                $idea->addRating($rating);
                $this->ideaRepository->update($idea);
            } catch(Exception $e) {
                throw new RepositoryNotAvailableException();
            } 
    
            return $idea;
        }
    }

Let's review the changes. Our controller is not having any business rules at all. We have pushed all the logic inside a new object called `RateIdeaUseCase` that encapsulates it. This object is also known as Controller, Interactor or Application Service.

The magic is done by the `execute` method. All the dependencies such as the `RedisIdeaRepository` are passed as an argument to the constructor. All the references to an `IdeaRepository` inside our UseCase are pointing to the interface instead of any concrete implementation.

That's really cool. If you take a look inside `RateIdeaUseCase`, there is nothing talking about MySQL or Zend Framework. No references, no instances, no annotations, nothing. It is like your infrastructure does not mind. It just talks about business logic.

Additionally, we have also tuned the Exceptions we throw. Business processes also have exceptions. `NotAvailableRepository` and `IdeaDoesNotExist` are two of them. Based on the one being thrown we can react in different ways in the framework boundary.

Sometimes, the number of parameters that a UseCase receives can be too many. In order to organize them, it is quite common to build a _UseCase request_ using a **Data Transfer Object** (**DTO**) to pass them together. Let's see how you could solve this in Listing 6:

    class IdeaController extends Zend_Controller_Action
    {
        public function rateAction()
        {
            $ideaId = $this->request->getParam('id');
            $rating = $this->request->getParam('rating');
    
            $ideaRepository = new RedisIdeaRepository();
            $useCase = new RateIdeaUseCase($ideaRepository);
            $response = $useCase->execute(
                new RateIdeaRequest($ideaId, $rating)
            );
    
            $this->redirect('/idea/' . $response->idea->getId());
        }
    }
    
    class RateIdeaRequest
    {
        public $ideaId;
        public $rating;
    
        public function __construct($ideaId, $rating)
        {
            $this->ideaId = $ideaId;
            $this->rating = $rating;
        }
    }
    
    class RateIdeaResponse
    {
        public $idea;
    
        public function __construct(Idea $idea)
        {
            $this->idea = $idea;
        }  
    }
    
    class RateIdeaUseCase
    {
        // ...
    
        public function execute($request)
        {
            $ideaId = $request->ideaId;
            $rating = $request->rating;
    
            // ...
    
            return new RateIdeaResponse($idea);
        }
    }

The main changes here are introducing two new objects, a Request and a Response. They are not mandatory, maybe a UseCase has no request or response. Another important detail is how you build this request. In this case, we are building it getting the parameters from ZF request object.

Ok, but wait, what's the real benefit? it is easier to change from one framework to other, or execute our UseCase from another _delivery mechanism_. Let's see this point.



Rating an Idea Using the API
----------------------------

* * *

During the day, your Product Owner comes to you and says: _by the way, a user should be able to rate an idea using our mobile app. I think we will need to update the API, could you do it for this sprint?_. Here's the PO again. _No problem!_. Business is impressed with your commitment.

As Robert C. Martin says:

> The Web is a delivery mechanism \[...\] Your system architecture should be as ignorant as possible about how it is to be delivered. You should be able to deliver it as a console app, a web app, or even a web service app, without undue complication or any change to the fundamental architecture

.

Your current API is built using Silex, the PHP micro-framework based on the Symfony2 Components. Let's go for it in Listing 7:

    require_once __DIR__.'/../vendor/autoload.php';
    
    $app = new Silex\Application();
    
    // ... more routes
    
    $app->get(
        '/api/rate/idea/{ideaId}/rating/{rating}',
        function ($ideaId, $rating) use ($app) {
            $ideaRepository = new RedisIdeaRepository();
            $useCase = new RateIdeaUseCase($ideaRepository);
            $response = $useCase->execute(
                new RateIdeaRequest($ideaId, $rating)
            );
    
            return $app->json($response->idea);
        }
    );
    
    $app->run();

Is there anything familiar to you? Can you identify some code that you have seen before? I'll give you a clue:

    $ideaRepository = new RedisIdeaRepository();
    $useCase = new RateIdeaUseCase($ideaRepository);
    $response = $useCase->execute(
        new RateIdeaRequest($ideaId, $rating)
    );

_Man! I remember those 3 lines of code. They look exactly the same as the web application_. That's right, because the UseCase encapsulates the business rules you need to prepare the request, get the response and act accordingly.

We are providing our users with another way for rating an idea; another _delivery mechanism_. The main difference is where we created the `RateIdeaRequest` from. In the first example, it was from a ZF request and now it is from a Silex request using the parameters matched in the route.



Console App Rating
------------------

* * *

Sometimes, a UseCase is going to be executed from a Cron job or the command line. As examples, batch processing or some testing command lines to accelerate the development. While testing this feature using the web or the API, you realize that it would be nice to have a command line to do it, so you don't have to go through the browser.

If you are using shell scripts files, I suggest you to check the Symfony Console component. What would the code look like:

    namespace Idy\Console\Command;
    
    use Symfony\Component\Console\Command\Command;
    use Symfony\Component\Console\Input\InputArgument;
    use Symfony\Component\Console\Input\InputInterface;
    use Symfony\Component\Console\Output\OutputInterface;
    
    class VoteIdeaCommand extends Command
    {
        protected function configure()
        {
            $this
                ->setName('idea:rate')
                ->setDescription('Rate an idea')
                ->addArgument('id', InputArgument::REQUIRED)
                ->addArgument('rating', InputArgument::REQUIRED);
        }
    
        protected function execute(
            InputInterface $input,
            OutputInterface $output
        ) {
            $ideaId = $input->getArgument('id');
            $rating = $input->getArgument('rating');
    
            $ideaRepository = new RedisIdeaRepository();
            $useCase = new RateIdeaUseCase($ideaRepository);
            $response = $useCase->execute(
                new RateIdeaRequest($ideaId, $rating)
            );
    
            $output->writeln('Done!');
          }
     }

Again those 3 lines of code. As before, the UseCase and its business logic remain untouched, we are just providing a new _delivery mechanism._ Congratulations, you've discovered the _user side_ hexagon edge.

There is still a lot to do. As you may have heard, a real craftsman does TDD. We have already started our story so we must be ok with just testing after.



Testing Rating an Idea UseCase
------------------------------

* * *

Michael Feathers introduced a definition of legacy code as _code without tests_. You don't want your code to be legacy just born, do you?

In order to unit test this UseCase object, you decide to start with the easiest part, what happens if the repository is not available? How can we generate such behavior? Do we stop our Redis server while running the unit tests? No. We need to have an object that has such behavior. Let's use a _mock_ object in Listing 9:

    class RateIdeaUseCaseTest extends \PHPUnit_Framework_TestCase
    {
        /**
         * @test
         */
        public function whenRepositoryNotAvailableAnExceptionIsThrown()
        {
            $this->setExpectedException('NotAvailableRepositoryException');
            $ideaRepository = new NotAvailableRepository();
            $useCase = new RateIdeaUseCase($ideaRepository);
            $useCase->execute(
                new RateIdeaRequest(1, 5)
            );
        }
    }
    
    class NotAvailableRepository implements IdeaRepository
    {
        public function find($id)
        {
            throw new NotAvailableException();
        }
    
        public function update(Idea $idea)
        {
            throw new NotAvailableException();
        }
    }

Nice. `NotAvailableRepository` has the behavior that we need and we can use it with `RateIdeaUseCase` because it implements `IdeaRepository` interface.

Next case to test is what happens if the idea is not in the repository. Listing 10 shows the code:

    class RateIdeaUseCaseTest extends \PHPUnit_Framework_TestCase
    {
        // ...
    
        /**
         * @test
         */
        public function whenIdeaDoesNotExistAnExceptionShouldBeThrown()
        {
            $this->setExpectedException('IdeaDoesNotExistException');
            $ideaRepository = new EmptyIdeaRepository();
            $useCase = new RateIdeaUseCase($ideaRepository);
            $useCase->execute(
                new RateIdeaRequest(1, 5)
            );
        }
    }
    
    class EmptyIdeaRepository implements IdeaRepository
    {
        public function find($id)
        {
            return null;
        }
    
        public function update(Idea $idea)
        {
    
        }
    }

Here, we use the same strategy but with an `EmptyIdeaRepository`. It also implements the same interface but the implementation always returns null regardless which identifier the find method receives.

Why are we testing these cases?, remember Kent Beck's words: _Test everything that could possibly break_.

Let's carry on with the rest of the feature. We need to check a special case that is related with having a read available repository where we cannot write to. Solution can be found in Listing 11:

    class RateIdeaUseCaseTest extends \PHPUnit_Framework_TestCase
    {
        // ...
    
        /**
         * @test
         */
        public function whenRatingAnIdeaNewRatingShouldBeAdded()
        {
            $ideaRepository = new OneIdeaRepository();
            $useCase = new RateIdeaUseCase($ideaRepository);
            $response = $useCase->execute(
                new RateIdeaRequest(1, 5)
            );
    
            $this->assertSame(5, $response->idea->getRating());
            $this->assertTrue($ideaRepository->updateCalled);
        }
    }
    
    class OneIdeaRepository implements IdeaRepository
    {
        public $updateCalled = false;
    
        public function find($id)
        {
            $idea = new Idea();
            $idea->setId(1);
            $idea->setTitle('Subscribe to php[architect]');
            $idea->setDescription('Just buy it!');
            $idea->setRating(5);
            $idea->setVotes(10);
            $idea->setAuthor('john@example.com');
    
            return $idea;
        }
    
        public function update(Idea $idea)
        {
            $this->updateCalled = true;
        }
    }

Ok, now the key part of the feature is still remaining. We have different ways of testing this, we can write our own mock or use a mocking framework such as Mockery or Prophecy. Let's choose the first one. Another interesting exercise would be to write this example and the previous ones using one of these frameworks:

    class RateIdeaUseCaseTest extends \PHPUnit_Framework_TestCase
    {
        // ...
    
        /**
         * @test
         */
        public function whenRatingAnIdeaNewRatingShouldBeAdded()
        {
            $ideaRepository = new OneIdeaRepository();
            $useCase = new RateIdeaUseCase($ideaRepository);
            $response = $useCase->execute(
                new RateIdeaRequest(1, 5)
            );
    
            $this->assertSame(5, $response->idea->getRating());
            $this->assertTrue($ideaRepository->updateCalled);
        }
    }
    
    class OneIdeaRepository implements IdeaRepository
    {
        public $updateCalled = false;
    
        public function find($id)
        {
            $idea = new Idea();
            $idea->setId(1);
            $idea->setTitle('Subscribe to php[architect]');
            $idea->setDescription('Just buy it!');
            $idea->setRating(5);
            $idea->setVotes(10);
            $idea->setAuthor('john@example.com');
    
            return $idea;
        }
    
        public function update(Idea $idea)
        {
            $this->updateCalled = true;
        }
    }

Bam! 100% Coverage for the UseCase. Maybe, next time we can do it using TDD so the test will come first. However, testing this feature was really easy because of the way decoupling is promoted in this architecture. Maybe you are wondering about this:

    $this->updateCalled = true;

We need a way to guarantee that the update method has been called during the UseCase execution. This does the trick. This _test double_ object is called a _spy_, _mocks_ cousin.

When to use mocks? As a general rule, use mocks when crossing boundaries. In this case, we need mocks because we are crossing from the domain to the persistence boundary.

What about testing the infrastructure?



Testing Infrastructure
----------------------

* * *

If you want to achieve 100% coverage for your whole application you will also have to test your infrastructure. Before doing that, you need to know that those unit tests will be more coupled to your implementation than the business ones. That means that the probability to be broken with implementation details changes is higher. So it is a trade-off you will have to consider.

So, if you want to continue, we need to do some modifications. We need to decouple even more. Let's see the code in Listing 13:

    class IdeaController extends Zend_Controller_Action
    {
        public function rateAction()
        {
            $ideaId = $this->request->getParam('id');
            $rating = $this->request->getParam('rating');
    
            $useCase = new RateIdeaUseCase(
                new RedisIdeaRepository(
                    new Predis\Client()
                )
            );
    
            $response = $useCase->execute(
                new RateIdeaRequest($ideaId, $rating)
            );
    
            $this->redirect('/idea/' . $response->idea->getId());
        }
    }
    
    class RedisIdeaRepository implements IdeaRepository
    {
        private $client;
    
        public function __construct($client)
        {
            $this->client = $client;
        }
    
        // ...
    
        public function find($id)
        {
            $idea = $this->client->get($this->getKey($id));
            if (!$idea) {
                return null;
            }
    
           return $idea;
       }
    }

If we want to 100% unit test `RedisIdeaRepository` we need to be able to pass the `Predis\Client` as a parameter to the repository without specifying TypeHinting so we can pass a mock to force the code flow necessary to cover all the cases.

This forces us to update the Controller to build the Redis connection, pass it to the repository and pass the result to the UseCase.

Now, it is all about creating mocks, test cases and having fun doing asserts.



Arggg, So Many Dependencies!
----------------------------

* * *

Is it normal that I have to create so many dependencies by hand? No. It is common to use a Dependency Injection component or a Service Container with such capabilities. Again, Symfony comes to the rescue, however, you can also check [PHP-DI 4](http://php-di.org/).

Let's see the resulting code in Listing 14 after applying Symfony Service Container component to our application:

    class IdeaController extends ContainerAwareController
    {
        public function rateAction()
        {
            $ideaId = $this->request->getParam('id');
            $rating = $this->request->getParam('rating');
    
            $useCase = $this->get('rate_idea_use_case');
            $response = $useCase->execute(
                new RateIdeaRequest($ideaId, $rating)
            );
    
            $this->redirect('/idea/' . $response->idea->getId());
        }
    }

The controller has been modified to have access to the container, that's why it is inheriting from a new base controller `ContainerAwareController` that has a `get` method to retrieve each of the services contained:

    <?xml version="1.0" ?>
    <container xmlns="http://symfony.com/schema/dic/services"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="
            http://symfony.com/schema/dic/services
            http://symfony.com/schema/dic/services/services-1.0.xsd">
        <services>
            <service
                id="rate_idea_use_case"
                class="RateIdeaUseCase">
                <argument type="service" id="idea_repository" />
            </service>
    
            <service
                id="idea_repository"
                class="RedisIdeaRepository">
                <argument type="service">
                    <service class="Predis\Client" />
                </argument>
            </service>
        </services>
    </container>

In Listing 15, you can also find the XML file used to configure the Service Container. It is really easy to understand but if you need more information, take a look to the Symfony Service Container Component [site](http://symfony.com/doc/current/book/service_container.html) in.



Domain Services and Notification Hexagon Edge
---------------------------------------------

* * *

Are we forgetting something? _the author should be notified by email_, yeah! That's true. Let's see in Listing 16 how we have updated the UseCase for doing the job:

    class RateIdeaUseCase
    {
        private $ideaRepository;
        private $authorNotifier;
    
        public function __construct(
            IdeaRepository $ideaRepository,
            AuthorNotifier $authorNotifier
        ) {
            $this->ideaRepository = $ideaRepository;
            $this->authorNotifier = $authorNotifier;
        }
    
        public function execute(RateIdeaRequest $request)
        {
            $ideaId = $request->ideaId;
            $rating = $request->rating;
    
            try {
                $idea = $this->ideaRepository->find($ideaId);
            } catch(Exception $e) {
                throw new RepositoryNotAvailableException();
            }
    
            if (!$idea) {
                throw new IdeaDoesNotExistException();
            }
    
            try {
                $idea->addRating($rating);
                $this->ideaRepository->update($idea);
            } catch(Exception $e) {
                throw new RepositoryNotAvailableException();
            }
    
            try {
                $this->authorNotifier->notify(
                    $idea->getAuthor()
                );
            } catch(Exception $e) {
                throw new NotificationNotSentException();
            }
    
            return $idea;
        }
    }

As you realize, we have added a new parameter for passing `AuthorNotifier` Service that will send the email to the author. This is the _port_ in the _Ports and Adapters_ naming. We have also updated the business rules in the execute method.

Repositories are not the only objects that may access your infrastructure and should be decoupled using interfaces or abstract classes. Domain Services can too. When there is a behavior not clearly owned by just one Entity in your domain, you should create a Domain Service. A typical pattern is to write an abstract Domain Service that has some concrete implementation and some other abstract methods that the _adapter_ will implement.

As an exercise, define the implementation details for the `AuthorNotifier` abstract service. Options are SwiftMailer or just plain `mail` calls. It is up to you.



Let's Recap
-----------

* * *

In order to have a _clean architecture_ that helps you create easy to write and test applications, we can use Hexagonal Architecture. To achieve that, we encapsulate user story business rules inside a UseCase or Interactor object. We build the UseCase request from our framework request, instantiate the UseCase and all its dependencies and then execute it. We get the response and act accordingly based on it. If our framework has a Dependency Injection component you can use it to simplify the code.

The same UseCase objects can be used from different _delivery mechanisms_ in order to allow users access the features from different clients (web, API, console, and so on.)

For testing, play with mocks that behave like all the interfaces defined so special cases or error flows can also be covered. Enjoy the good job done.



Hexagonal Architecture
----------------------

* * *

In almost all the blogs and books you will find drawings about concentric circles representing different areas of software. As Robert C. Martin explains in his _Clean Architecture_ post, the outer circle is where your infrastructure resides. The inner circle is where your Entities live. The overriding rule that makes this architecture work is **The Dependency Rule**. This rule says that source code dependencies can only point inwards. Nothing in an inner circle can know anything at all about something in an outer circle.



Key Points
----------

* * *

Use this approach if 100% unit test code coverage is important to your application. Also, if you want to be able to switch your storage strategy, web framework or any other type of third-party code. The architecture is especially useful for long-lasting applications that need to keep up with changing requirements.



What's Next?
------------

* * *

If you are interested in learning more about Hexagonal Architecture and other near concepts you should review the related URLs provided at the beginning of the article, take a look at CQRS and Event Sourcing. Also, don't forget to subscribe to google groups and RSS about DDD such as [http://dddinphp.org](http://dddinphp.org) and follow on Twitter people like `@VaughnVernon`, and `@ericevans0`.
