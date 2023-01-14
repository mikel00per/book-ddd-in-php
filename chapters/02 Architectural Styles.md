Chapter 2. Architectural Styles
-------------------------------

In order to be able to build complex applications, one of the key requirements is having an architectural design that fits the application's needs. One advantage of Domain-Driven Design is that it's not tied to any particular architecture style. Instead, we're free to choose the architecture that best fits the needs of every Bounded Context inside the Core Domain, which offers a diverse set of architectural choices for every specific Domain problem.

For example, an Order Processing System can use Event Sourcing to track all the different order operations; a Product Catalog can use CQRS to expose the product details to the different clients; and a Content Management System can use plain Hexagonal Architecture to expose requirements such as blogs, static pages, and so on.

This chapter presents an introduction to every relevant architecture style in the land of PHP, following the evolution from traditional old school PHP code to a more sophisticated architecture. Please note that although there are many other existing architecture styles, such as Data Fabric or SOA, we found some of them a bit too complex to introduce from the PHP perspective.



The Good Old Days
-----------------

* * *

Before the release of PHP 4, the language didn't embrace the Object-Oriented paradigm. Back then, the usual way of writing applications was by using procedures and global state. Concepts like **Separation of Concerns** (**SoC**) and **Model-View-Controller** (**MVC**) were alien among the PHP community. The example below is an application written in this traditional way, where applications were composed of many front controllers mixed with HTML code. During this time, Infrastructure-, Presentation-, UI-, and Domain-layer code were all tangled together:

    include __DIR__ . '/bootstrap.php';
    
    $link = mysql_connect('localhost', 'a_username', '4_p4ssw0rd');
    
    if (!$link) {
        die('Could not connect: ' . mysql_error());
    }
    
    mysql_set_charset('utf8', $link);
    mysql_select_db('my_database', $link);
    
    $errormsg = null ;
    if (isset($_POST['submit'] && isValid($_POST['post'])) {
        $post = getFrom($_POST['post']);
        mysql_query('START TRANSACTION', $link);
        $sql = sprintf(
            "INSERT INTO posts (title, content) VALUES ('%s','%s')",    
            mysql_real_escape_string($post['title']),
            mysql_real_escape_string($post['content']
        ));
    
        $result = mysql_query($sql, $link);
        if ($result) {
            mysql_query('COMMIT', $link);
        } else {
            mysql_query('ROLLBACK', $link);
            $errormsg = 'Post could not be created! :(';
        }
    }
    
    $result = mysql_query('SELECT id, title, content FROM posts', $link);
    ?>
    <html>
        <head></head>
        <body>
            <?php if (null !== $errormsg) : ?>
                <div class="alert error"><?php echo $errormsg; ?></div>
            <?php else: ?>
                <div class="alert success">
                    Bravo! Post was created successfully!
                </div>
            <?php endif; ?>
            <table>
                <thead><tr><th>ID</th><th>TITLE</th>
                <th>ACTIONS</th></tr></thead>
                <tbody>
                <?php while($post = mysql_fetch_assoc($result)) : ?>
                    <tr>
                        <td><?php echo $post['id']; ?></td>
                        <td><?php echo $post['title']; ?></td>
                        <td><?php editPostUrl($post['id']); ?></td>
                    </tr>
                <?php endwhile; ?>
                </tbody>
            </table>
       </body>
     </html>
     <?php mysql_close($link); ?>


This style of coding is often referred to as the _Big Ball of Mud_ we mentioned in the first chapter. An improvement seen in this style, however, was to encapsulate the header and the footer of the webpage in their own separate files, which were included in the header and footer files. This avoided duplication and favored reuse:

    include __DIR__ . '/bootstrap.php';
    
    $link = mysql_connect('localhost', 'a_username', '4_p4ssw0rd');
    
    if (!$link) {
        die('Could not connect: ' . mysql_error());
    }
    
    mysql_set_charset('utf8', $link);
    mysql_select_db('my_database', $link);
    
    $errormsg = null;
    
    if (isset($_POST['submit'] && isValid($_POST['post'])) {
        $post = getFrom($_POST['post']);
        mysql_query('START TRANSACTION', $link);
        $sql = sprintf(
            "INSERT INTO posts(title, content) VALUES('%s','%s')", 
            mysql_real_escape_string($post['title']),
            mysql_real_escape_string($post['content'])
        );
    
        $result = mysql_query($sql, $link);
        if ($result) {
            mysql_query('COMMIT', $link);
        } else {
            mysql_query('ROLLBACK', $link);
            $errormsg = 'Post could not be created! :(';
        }
    }
    
    $result = mysql_query('SELECT id, title, content FROM posts', $link);
    ?>
    <?php include __DIR__ . '/header.php'; ?>
    <?php if (null !== $errormsg) : ?>
        <div class="alert error"><?php echo $errormsg; ?></div>
    <?php else: ?>
        <div class="alert success">
            Bravo! Post was created successfully!
        </div>
    <?php endif; ?>
    <table>
        <thead>
            <tr>
                <th>ID</th>
                <th>TITLE</th>
                <th>ACTIONS</th>
            </tr>
        </thead>
        <tbody>
        <?php while($post = mysql_fetch_assoc($result)): ?>
            <tr>
                <td><?php echo $post['id']; ?></td>
                <td><?php echo $post['title']; ?></td>
                <td><?php editPostUrl($post['id']); ?></td>
            </tr>
        <?php endwhile; ?>
        </tbody>
    </table>
    <?php include __DIR__ . '/footer.php'; ?>

Nowadays, and although it is highly discouraged, there are still applications that use this procedural way of coding. The main disadvantage of this style of architecture is that there's no real Separation of Concerns — the maintenance and cost of evolving an application being developed this way increases drastically in relation to other well-known and proven architectures.



Layered Architecture
--------------------

* * *

From the code maintainability and reuse perspectives, the best way to make this code a bit easier to maintain would be by splitting up concepts, that is creating layers for each different concern. In our previous example, it's easy to shape different layers: one to encapsulate the data access and manipulation, another one to handle infrastructure concerns, and a final one for encapsulating the orchestration of the previous two. An essential rule of Layered Architecture — is that each layer must be tightly coupled with the layers beneath it, as shown in the following picture:

![](https://static.packt-cdn.com/products/9781787284944/graphics/layered-architecture.png)

Layered Architecture for SoC

What Layered Architecture really seeks is the separation of the different components of an application. For instance, in terms of the previous example, a blog post representation must be completely independent of a blog post as a conceptual entity. A blog post as a conceptual entity can instead be associated with one or more representations, as opposed to being tightly coupled to a specific representation. This is commonly referred to as Separation of Concerns.

Another architecture paradigm and pattern that seeks the same purpose is the Model-View-Controller pattern. It was initially thought of and widely used for building desktop GUI applications, and now it's mainly used in web applications, thanks to popular web frameworks like Symfony, Zend Framework, and CodeIgniter.

### Model-View-Controller

Model-View-Controller is an architectural pattern and paradigm that divides the application into three main layers, described in the following points:

*   **The Model**: Captures and centralizes all the Domain Model behavior. This layer manages all the data, logic, and business rules independently of the data representation. It has been said that **the Model layer is the heart and soul of every MVC application**.
*   **The Controller**: Orchestrates interactions between the other layers, triggers actions on the Model in order to update its state, and refreshes the representations associated with the Model. Additionally, the Controller can send messages to the View layer in order to change the specific Model representation.
*   **The View**: Exposes the differing representations of the Model layer and provides a way to trigger changes on the Model's state.

![](https://static.packt-cdn.com/products/9781787284944/graphics/mvc.png)

The MVC pattern

### Example of Layered Architecture

#### The Model

Continuing with the previous example, we mentioned that different concerns should be split up. In order to do so, all layers should be identified in our original tangled code. Throughout this process, we need to pay special attention to the code conforming to the Model layer, which will be the beating heart of the application:

    class Post
    {
        private $title;
        private $content;
    
        public static function writeNewFrom($title, $content)
        {
            return new static($title, $content);
        }
    
        private function __construct($title, $content)
        {
            $this->setTitle($title);
            $this->setContent($content);
        }
    
        private function setTitle($title)
        {
            if (empty($title)) {
                throw new RuntimeException('Title cannot be empty');
            }
    
            $this->title = $title;
        }
    
        private function setContent($content)
        {
            if (empty($content)) {
                throw new RuntimeException('Content cannot be empty');
            }
    
            $this->content = $content;
        }
    }
    
    class PostRepository
    {
        private $db;
    
        public function __construct()
        {
            $this->db = new PDO(
                'mysql:host=localhost;dbname=my_database',
                'a_username',
                '4_p4ssw0rd',
                [
                    PDO::MYSQL_ATTR_INIT_COMMAND => 'SET NAMES utf8mb4',
                ]
            );
        }
    
        public function add(Post $post)
        {
            $this->db->beginTransaction();
    
            try {
                $stm = $this->db->prepare(
                    'INSERT INTO posts (title, content) VALUES (?, ?)'
                );
    
                $stm->execute([
                    $post->title(),
                    $post->content(),
                ]);
    
                $this->db->commit();
            } catch (Exception $e) {
                $this->db->rollback();
                throw new UnableToCreatePostException($e);
            }
        }
    }


The Model layer is now defined by a `Post` class and a `PostRepository` class. The `Post` class represents a blog post, and the `PostRepository` class represents the whole collection of blog posts available. Additionally, another layer — one that coordinates and orchestrates the Domain Model behavior — is needed inside the Model. Enter the Application layer:

    class PostService
    {
        public function createPost($title, $content)
        {
            $post = Post::writeNewFrom($title, $content);
    
            (new PostRepository())->add($post);
    
            return $post;
        }
    }


The `PostService` class is what is known as an Application Service, and its purpose is to orchestrate and organize the Domain behavior. In other words, the Application services are the ones that make things happen, and they're the direct clients of a Domain Model. No other type of object should be able to directly talk to the internal layers of the Model layer.

#### The View

The View is a layer that can both send and receive messages from the Model layer and/or from the Controller layer. Its main purpose is to represent the Model to the user at the UI level, as well as to refresh the representation in the UI each time the Model is updated. Generally speaking, the View layer receives an object — often a **Data Transfer Object** (**DTO**) instead of instances of the Model layer — thereby gathering all the needed information to be successfully represented. For PHP, there are several template engines that can help a great deal in separating the Model representation from the Model itself and from the Controller. The most popular one by far is called [Twig](http://twig.sensiolabs.org/). Let's see how the View layer would look with Twig.

### Note

**`DTOs Instead of Model Instances?`** This is an old and active topic. Why create a DTO instead of giving an instance of the Model to the View layer? The main reason and the short answer is, again, Separation of Concerns. Letting the View inspect and use a Model instance leads to tight coupling between the View layer and the Model layer. In fact, a change in the Model layer can potentially break all the views that make use of the changed Model instances.

    {% extends "base.html.twig" %}
    
    {% block content %}
        {% if errormsg is defined %}
            <div class="alert error">{{ errormsg }}</div>
        {% else %}
            <div class="alert success">
                Bravo! Post was created successfully!
            </div>
        {% endif %}
        <table>
            <thead>
                <tr>
                    <th>ID</th>
                    <th>TITLE</th>
                    <th>ACTIONS</th>
                </tr>
            </thead>
            <tbody>
            {% for post in posts %}
                <tr>
                    <td>{{ post.id }}</td>
                    <td>{{ post.title }}</td>
                    <td><a href="{{ editPostUrl(post.id) }}">Edit Post</a></td>
                </tr>
            {% endfor %}
            </tbody>
        </table>
    {% endblock %}

Most of the time, when the Model triggers a state change, it also notifies the related Views so that the UI is refreshed. In a typical web scenario, the synchronization between the Model and its representations can be a bit tricky because of the client-server nature. In these kind of environments, some JavaScript-defined interactions are usually needed to maintain that synchronization. For this reason, JavaScript MVC frameworks like the ones below have become widely popular in recent years:

*   [AngularJS](https://angularjs.org/)
*   [Ember.js](http://emberjs.com/)
*   [Marionette.js](http://marionettejs.com/)
*   [React](https://facebook.github.io/react/)

#### The Controller

The Controller layer is responsible for organizing and orchestrating the View and the Model. It receives messages from the View layer and triggers Model behavior in order to perform the desired action. Furthermore, it sends messages to the View in order to display Model representations. Both operations are performed thanks to the Application layer, which is responsible for orchestrating, organizing, and encapsulating Domain behavior.

In terms of a web application in PHP, the Controller usually comprehends a set of classes, which, in order to fulfill their purpose, "speak HTTP." In other words, they receive an HTTP request and return an HTTP response:

    class PostsController
    {
        public function updateAction(Request $request)
        {
            if (
                $request->request->has('submit') &&
                Validator::validate($request->request->post)
            ) {
                $postService = new PostService();
    
                try {
                    $postService->createPost(
                        $request->request->get('title'),
                        $request->request->get('content')
                    );
    
                    $this->addFlash(
                        'notice',
                        'Post has been created successfully!'
                    );
                } catch (Exception $e) {
                    $this->addFlash(
                        'error',
                        'Unable to create the post!'
                    );
                }
            }
    
            return $this->render('posts/update-result.html.twig');
        }
    }

### Inverting Dependencies: Hexagonal Architecture

Following the essential rule of Layered Architecture, there's a risk when implementing Domain interfaces that contain infrastructural concerns.

As an example, with MVC, the `PostRepository` class from the previous example should be placed in the Domain Model. However, placing infrastructural details right in the middle of our Domain violates Separation of Concerns. This can be problematic; it's difficult to avoid violating the essential rules of Layered Architecture, which leads to a style of code that can become hard to test if the Domain layer is aware of technical implementations.

#### The Dependency Inversion Principle (DIP)

How can we fix this? As the Domain Model layer depends on concrete infrastructure implementations, the [Dependency Inversion Principle](https://en.wikipedia.org/wiki/Dependency_inversion_principle), or DIP, could be applied by relocating the Infrastructure layer on top of the other three layers.

### Note

**`The Dependency Inversion Principle`** High-level modules should not depend on low-level modules. Both should depend on abstractions. Abstractions should not depend on details. Details should depend on abstractions. _Robert C. Martin_

By using the Dependency Inversion Principle, the architecture schema changes, and the Infrastructure layer — which can be referred to as the low-level module — now depends on the UI, the Application layer, and the Domain layer, which are the high-level modules. The dependency has been inverted.

But what is Hexagonal Architecture, and how does it fit within all of this? Hexagonal Architecture (also known as Ports and Adapters) was defined by Alistair Cockburn in his book, [Hexagonal Architecture](http://alistair.cockburn.us/Hexagonal+architecture). It depicts the application as a hexagon, where each side represents a Port with one or more Adapters. A Port is a connector with a pluggable Adapter that transforms an outside input to something the inside application can understand. In terms of the DIP, a Port would be a high-level module, and an Adapter would be a low-level module. Furthermore, if the application needs to emit a message to the outside, it will also use a Port with an Adapter to send it and transform it into something that the outside can understand. For this reason, Hexagonal Architecture brings up the concept of symmetry in the application, and it's also the main reason why the schema of the architecture changes. It's often represented as a hexagon because it no longer makes sense to talk about a top layer or a bottom layer. Instead, Hexagonal Architecture talks mainly in terms of the outside and the inside.

### Note

There are great videos on YouTube by _Matthias Noback_ where he talks about Hexagonal Architecture. You may want to take a look at one of those for more [detailed information](https://www.youtube.com/watch?v=K1EJBmwg9EQ).

#### Applying Hexagonal Architecture

Continuing with the blog example application, the first concept we need is a Port where the outside world can talk to the application. For this case, we'll use an HTTP Port and its corresponding Adapter. The outside will use the Port to send messages to the application. The blog example was using a database to store the whole collection of blog posts, so in order to allow the application to retrieve blog posts from the database, a Port is needed:

    interface PostRepository
    {
        public function byId(PostId $id);
        public function add(Post $post);
    }

This interface exposes the Port that the application will retrieve information about blog posts through, and it'll be located in the Domain Layer. Now an Adapter for this Port is needed. The Adapter is in charge of defining the way in which the blog posts will be retrieved using a specific technology:

    class PDOPostRepository implements PostRepository
    {
        private $db;
    
        public function __construct(PDO $db)
        {
            $this->db = $db;
        }
    
        public function byId(PostId $id)
        {
            $stm = $this->db->prepare(
                'SELECT * FROM posts WHERE id = ?'
            );
    
            $stm->execute([$id->id()]);
    
            return recreateFrom($stm->fetch());
        }
    
        public function add(Post $post)
        {
            $stm = $this->db->prepare(
                'INSERT INTO posts (title, content) VALUES (?, ?)'
            );
    
            $stm->execute([
                $post->title(),
                $post->content(),
            ]);
        }
    }

Once we have the Port and its Adapter defined, the last step is to refactor the `PostService` class so that it uses them. This can be easily achieved by using [Dependency Injection](http://www.martinfowler.com/articles/injection.html):

    class PostService
    {
        private $postRepository;
    
        public function __construct(PostRepositor $postRepository)
        {
            $this->postRepository = $postRepository;
        }
    
        public function createPost($title, $content)
        {
            $post = Post::writeNewFrom($title, $content);
    
            $this->postRepository->add($post);
    
            return $post;
        }
    }

This is just a simple example of Hexagonal Architecture. It's a flexible architecture that promotes Separation of Concerns, like Layered Architecture. It also promotes symmetry, due to having an inside application that communicates with the outside via ports. From now on, this will be the foundational architecture used to build and explain CQRS and Event Sourcing.

For more examples about this architecture, you can check out the [Appendix](../chapters/13%20Hexagonal%20Architecture%20with%20PHP.md), _Hexagonal Architecture with PHP_. For a more detailed example, you should jump to the [Chapter 11](../chapters/11%20Application.md), _Application_, which explains advanced topics like transactionality and other cross-cutting concerns.

### Command Query Responsibility Segregation (CQRS)

Hexagonal Architecture is a good foundational architecture, but it has some limitations. For example, complex UIs can require Aggregate information displayed in diverse forms (**[Chapter 8](../chapters/08%20Aggregates.md)**, _Aggregates_), or they can require data obtained from multiple Aggregates. And in this scenario, we could end up with a lot of finder methods inside the Repositories (maybe as many as the UI views which exist within the application). Or, maybe we can decide to move this complexity to the Application Services, using complex structures to accumulate data from multiple Aggregates. Here's an example:

    interface PostRepository 
    { 
        public function save(Post $post);
        public function byId(PostId $id);
        public function all(); 
        public function byCategory(CategoryId $categoryId); 
        public function byTag(TagId $tagId); 
        public function withComments(PostId $id); 
        public function groupedByMonth(); 
        // ... 
    }

When these techniques are abused, the construction of the UI views can become really painful. We should evaluate the tradeoff between making Application Services return Domain Model instances and returning some kind of DTOs. With the latter option, we avoid tight coupling between the Domain Model and Infrastructure code (web controllers, CLI controllers, and so on).

Luckily, there's another approach. If the problem is having multiple and disparate views, we can exclude them from the Domain Model and start treating them as a purely infrastructural concern. This option is based on a design principle, the **Command Query Separation** (**CQS**). This principle was defined by Bertrand Meyer, and, in turn, it gave birth to a new architectural pattern named **Command Query Responsibility Segregation** (**CQRS**), as defined by Greg Young.

### Note

**`Command Query Separation (CQS)`**  _Asking a question should not change the answer_ - Bertrand Meyer This design principle states that every method should be either a command that performs an action, or a query that returns data to the caller, but not both, [Wikipedia](https://en.wikipedia.org/wiki/Command%E2%80%93query_separation)

CQRS seeks an even more aggressive Separation of Concerns, splitting the Model in two:

*   The **Write Model**: Also known as the **Command Model**, it performs the writes and takes responsibility for the true Domain behavior.
*   The **Read Model**: It takes responsibility of the reads within the application and treats them as something that should be out of the Domain Model.

Every time someone triggers a command to the Write Model, this performs the write to the desired data store. Additionally, it triggers the Read Model update, in order to display the latest changes on the Read Model.

This strict separation causes another problem: Eventual Consistency. The consistency of the Read Model is now subject to the commands performed by the Write Model. In other words, the Read Model is eventually consistent. That is, every time the Write Model performs a command, it will pull up a process that will be responsible for updating the Read Model according to the last updates on the Write Model. There's a window of time where the UI may present stale information to the user. In the web scenario, this happens often, as we're somewhat limited by the current technologies.

Think about a caching system in front of a web application. Every time the database is updated with new information, the data on the cache layer may potentially be stale, so every time it gets updated, there should be a process that updates the cache system. Cache systems are eventually consistent.

These kinds of processes, speaking in CQRS terminology, are called Write Model Projections, or just Projections. We project the Write Model onto the Read Model. This process can be synchronous or asynchronous, depending on your needs, and it can be done thanks to another useful tactical design pattern — Chapter Domain Events — which will be explained in detail later on in the book. The basis of the Write Model projections is to gather all the published Domain Events and update the Read Model with all the information coming from the events.

#### The Write Model

This is the true holder of Domain behavior. Continuing with our example, the Repository interface would be simplified to the following:

    interface PostRepository
    { 
        public function save(Post $post); 
        public function byId(PostId $id); 
    }

Now the `PostRepository` has been freed from all the read concerns except one: The `byId` function which is responsible for loading the Aggregate by its ID so that we can operate on it. And once this is done, all the query methods are also stripped down from the Post model, leaving it only with command methods. This means we'll effectively get rid of all the getter methods and any other methods exposing information about the Post Aggregate. Instead, Domain Events will be published in order to be able to trigger Write Model projections by subscribing to them:

    class AggregateRoot
    {
        private $recordedEvents = [];
    
        protected function recordApplyAndPublishThat(
            DomainEvent $domainEvent
        ) {
            $this->recordThat($domainEvent);
            $this->applyThat($domainEvent);
            $this->publishThat($domainEvent);
        }
    
        protected function recordThat(DomainEvent $domainEvent)
        {
            $this->recordedEvents[] = $domainEvent;
        }
    
        protected function applyThat(DomainEvent $domainEvent)
        {
            $modifier = 'apply' . get_class($domainEvent);
    
            $this->$modifier($domainEvent);
        }
    
        protected function publishThat(DomainEvent $domainEvent)
        {
            DomainEventPublisher::getInstance()->publish($domainEvent);
        }
    
        public function recordedEvents()
        {
            return $this->recordedEvents;
        }
    
        public function clearEvents()
        {
            $this->recordedEvents = [];
        }
    }
    
    class Post extends AggregateRoot
    {
        private $id;
        private $title;
        private $content;
        private $published = false;
        private $categories;
    
        private function __construct(PostId $id)
        {
            $this->id = $id;
            $this->categories = new Collection();
        }
    
        public static function writeNewFrom($title, $content)
        {
            $postId = PostId::create();
    
            $post = new static($postId);
    
            $post->recordApplyAndPublishThat(
                new PostWasCreated($postId, $title, $content)
            );
        }
    
        public function publish()
        {
            $this->recordApplyAndPublishThat(
                new PostWasPublished($this->id)
            );
        }
    
        public function categorizeIn(CategoryId $categoryId)
        {
            $this->recordApplyAndPublishThat(
                new PostWasCategorized($this->id, $categoryId)
            );
        }
    
        public function changeContentFor($newContent)
        {
            $this->recordApplyAndPublishThat(
                new PostContentWasChanged($this->id, $newContent)
            );
        }
    
        public function changeTitleFor($newTitle)
        {
            $this->recordApplyAndPublishThat(
                new PostTitleWasChanged($this->id, $newTitle)
            );
        }
    }


All actions that trigger a state change are implemented via Domain Events. For each Domain Event published, there's an apply method responsible for reflecting the state change:

    class Post extends AggregateRoot
    {
        // ...
    
        protected function applyPostWasCreated(
            PostWasCreated $event
        ) {
            $this->id = $event->id();
            $this->title = $event->title();
            $this->content = $event->content();
        }
    
        protected function applyPostWasPublished(
            PostWasPublished $event
        ) {
            $this->published = true;
        }
    
        protected function applyPostWasCategorized(
            PostWasCategorized $event
        ) {
            $this->categories->add($event->categoryId());
        }
    
        protected function applyPostContentWasChanged(
            PostContentWasChanged $event
        ) {
            $this->content = $event->content();
        }
    
        protected function applyPostTitleWasChanged(
            PostTitleWasChanged $event
        ) {
            $this->title = $event->title();
        }
    }



#### The Read Model

The Read Model, also known as the Query Model, is a pure denormalized data model lifted from Domain concerns. In fact, with CQRS, all the read concerns are treated as reporting processes, an infrastructure concern. In general, when using CQRS, the Read Model is subject to the needs of the UI and how complex the views compounding the UI are. In a situation where the Read Model is defined in terms of relational databases, the simplest approach would be to set one-to-one relationships between database tables and UI views. These database tables and UI views will be updated using Write Model projections triggered from the Domain Events published by the write side:

    -- Definition of a UI view of a single post with its comments
    CREATE TABLE single_post_with_comments (
        id INTEGER NOT NULL,
        post_id INTEGER NOT NULL,
        post_title VARCHAR(100) NOT NULL,
        post_content TEXT NOT NULL,
        post_created_at DATETIME NOT NULL,
        comment_content TEXT NOT NULL
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
    
    -- Set up some data
    INSERT INTO single_post_with_comments VALUES
        (1, 1, "Layered" , "Some content", NOW(), "A comment"),
        (2, 1, "Layered" , "Some content", NOW(), "The comment"),
        (3, 2, "Hexagonal" , "Some content", NOW(), "No comment"),
        (4, 2, "Hexagonal", "Some content", NOW(), "All comments"),
        (5, 3, "CQRS", "Some content", NOW(), "This comment"),
        (6, 3, "CQRS", "Some content", NOW(), "That comment");
    
    -- Query it
    SELECT * FROM single_post_with_comments WHERE post_id = 1;

An important feature of this architectural style is that the Read Model should be completely disposable, since the true state of the application is handled by the Write Model. This means the Read Model can be removed and recreated when needed, using Write Model projections.

Here we can see some examples of possible views within a blog application:

    SELECT * FROM
        posts_grouped_by_month_and_year 
    ORDER BY month DESC,year ASC;
    
    SELECT * FROM
        posts_by_tags 
    WHERE tag = "ddd";
    
    SELECT * FROM
        posts_by_author 
    WHERE author_id = 1;

It's important to point out that CQRS doesn't constrain the definition and implementation of the Read Model to a relational database. It depends exclusively on the needs of the application being built. It could be a relational database, a document-oriented database, a key-value store, or whatever best suits the needs of your application. Following the blog post application, we'll use [Elasticsearch](https://en.wikipedia.org/wiki/Elasticsearch) — a document-oriented database — to implement a Read Model:

    class PostsController
    {
        public function listAction()
        {
            $client = new ElasticsearchClientBuilder::create()->build();
    
            $response = $client-> search([
                'index' => 'blog-engine',
                'type' => 'posts',
                'body' => [
                    'sort' => [
                        'created_at' => ['order' => 'desc']
                    ]
                ]
            ]);
    
            return [
                'posts' => $response
            ];
        }
    }

The Read Model code has been drastically simplified to a single query against an Elasticsearch index.

This reveals that the Read Model doesn't really need an object-relational mapper, as this might be overkill. However, the Write Model might benefit from the use of an object-relational mapper, as this would allow you to organize and structure the Read Model according to the needs of the application.

#### Synchronizing the Write Model with the Read Model

Here comes the tricky part. How do we synchronize the Read Model with the Write Model? We already said we would do it by using Domain Events captured in a Write Model transaction. For each type of Domain Event captured, a specific projection will be executed. So a one-to-one relationship between Domain Events and projections will be set.

Let's have a look at an example of configuring projections so that we can get a better idea. First of all, we need to define a skeleton for the projections:

    interface Projection 
    { 
        public function listensTo(); 
        public function project($event); 
    }

So defining an `Elasticsearch` projection for a `PostWasCreated` event would be as simple as this:

    namespace Infrastructure\Projection\Elasticsearch;
    
    use Elasticsearch\Client;
    use PostWasCreated;
    
    class PostWasCreatedProjection implements Projection
    {
        private $client;
    
        public function __construct(Client $client)
        {
            $this->client = $client;
        }
    
        public function listensTo()
        {
            return PostWasCreated::class;
        }
    
        public function project($event)
        {
            $this->client->index([
                'index' => 'posts',
                'type' => 'post',
                'id' => $event->getPostId(),
                'body' => [
                    'content' => $event->getPostContent(),
                    // ...
                ]
            ]);
        }
    }

The Projector implementation is a kind of specialized Domain Event listener. The main difference between that and the default Domain Event listener is that the Projector reacts to a group of Domain Events instead of only one:

    namespace Infrastructure\Projection;
    
    class Projector
    {
        private $projections = [];
    
        public function register(array $projections)
        {
            foreach ($projections as $projection) {
                $this->projections[$projection->eventType()] = $projection;
            }
        }
    
        public function project( array $events)
        {
            foreach ($events as $event) {
                if (isset($this->projections[get_class($event)])) {
                    $this->projections[get_class($event)] 
                        ->project($event);
                }
            }
        }
    }

The following code shows how the flow between the projector and the events would appear:

    $client = new ElasticsearchClientBuilder::create()->build();
    
    $projector = new Projector();
    $projector->register([
        new Infrastructure\Projection\Elasticsearch\
            PostWasCreatedProjection($client), 
        new Infrastructure\Projection\Elasticsearch\
            PostWasPublishedProjection($client),
        new Infrastructure\Projection\Elasticsearch\
            PostWasCategorizedProjection($client),
        new Infrastructure\Projection\Elasticsearch\
            PostContentWasChangedProjection($client),
        new Infrastructure\Projection\Elasticsearch\
            PostTitleWasChangedProjection($client),
    ]);
    
    $events = [
        new PostWasCreated(/* ... */),
        new PostWasPublished(/* ... */),
        new PostWasCategorized(/* ... */),
        new PostContentWasChanged(/* ... */),
        new PostTitleWasChanged(/* ... */),
    ];
    
    $projector->project($event);

This code is kind of synchronous, but the process can be asynchronous if needed. And you could make your customers aware of this out-of-sync data by placing some alerts in the view layer.

For the next example, we'll use the `amqplib` PHP extension in combination with [ReactPHP](https://github.com/GeniusesOfSymfony/ReactAMQP):

    // Connect to an AMQP broker
    $cnn = new AMQPConnection();
    $cnn->connect();
    
    // Create a channel
    $ch = new AMQPChannel($cnn);
    
    // Declare a new exchange
    $ex = new AMQPExchange($ch);
    $ex->setName('events');
    
    $ex->declare();
    
    // Create an event loop
    $loop = ReactEventLoopFactory::create();
    
    // Create a producer that will send any waiting messages every half a second
    $producer = new Gos\Component\React\AMQPProducer($ex, $loop, 0.5);
    
    $serializer = JMS\Serializer\SerializerBuilder::create()->build();
    
    $projector = new AsyncProjector($producer, $serializer);
    
    $events = [
        new PostWasCreated(/* ... */),
        new PostWasPublished(/* ... */),
        new PostWasCategorized(/* ... */),
        new PostContentWasChanged(/* ... */),
        new PostTitleWasChanged(/* ... */),
    ];
    
    $projector->project($event);

For this to work, we need an asynchronous projector. Here's a naive implementation of that:

    namespace Infrastructure\Projection;
    
    use Gos\Component\React\AMQPProducer;
    use JMS\Serializer\Serializer;
    
    class AsyncProjector
    {
        private $producer;
        private $serializer;
    
        public function __construct(
            Producer $producer,
            Serializer $serializer
        ) {
            $this->producer = $producer;
            $this->serializer = $serializer;
        }
    
        public function project(array $events)
        {
            foreach ($events as $event) {
                $this->producer->publish(
                    $this->serializer->serialize(
                        $event, 'json'
                    )
                );
            }
        }
    }

And the event consumer on the RabbitMQ exchange would look something like this:

    // Connect to an AMQP broker
    $cnn = new AMQPConnection();
    $cnn-> connect();
    
    // Create a channel
    $ch = new AMQPChannel($cnn);
    
    // Create a new queue
    $queue = new AMQPQueue($ch);
    $queue->setName('events');
    $queue->declare();
    
    // Create an event loop
    $loop = React\EventLoop\Factory::create();
    
    $serializer = JMS\Serializer\SerializerBuilder::create()->build();
    
    $client = new Elasticsearch\ClientBuilder::create()->build();
    
    $projector = new Projector();
    $projector->register([
        new Infrastructure\Projection\Elasticsearch\
            PostWasCreatedProjection($client),
        new Infrastructure\Projection\Elasticsearch\
            PostWasPublishedProjection($client),
        new Infrastructure\Projection\Elasticsearch\
            PostWasCategorizedProjection($client),
        new Infrastructure\Projection\Elasticsearch\
            PostContentWasChangedProjection($client),
        new Infrastructure\Projection\Elasticsearch\
            PostTitleWasChangedProjection($client),              
    ]);
    
    // Create a consumer
    $consumer = new Gos\Component\ReactAMQP\Consumer($queue, $loop, 0.5, 10);
    
    // Check for messages every half a second and consume up to 10 at a time.
    $consumer->on(
        'consume',
        function ($envelope, $queue) use ($projector, $serializer) {
            $event = $serializer->unserialize($envelope->getBody(), 'json');
            $projector->project($event);
        }
    );
    
    $loop->run();


From now on, it could be as simple as making all the needed Repositories consume an instance of the projector and then making them invoke the projection process:

    class DoctrinePostRepository implements PostRepository
    {
        private $em;
        private $projector;
    
        public function __construct(EntityManager $em, Projector $projector)
        {
            $this->em = $em;
            $this->projector = $projector;
        }
    
        public function save(Post $post)
        {
            $this->em->transactional(
                function (EntityManager $em) use ($post)
                {
                    $em->persist($post);
    
                    foreach ($post->recordedEvents() as $event) {
                        $em->persist($event);
                    }
                }
            );
    
            $this->projector->project($post->recordedEvents());
        }
    
        public function byId(PostId $id)
        {
            return $this->em->find($id);
        }
    }

The `Post` instance and the recorded events are triggered and persisted in the same transaction. This ensures that no events are lost, as we'll project them to the Read Model if the transaction is successful. As a result, no inconsistencies will exist between the Write Model and the Read Model.

### Note

****`To ORM or Not To ORM`****   One of the most common questions when implementing CQRS is if an **Object-Relational Mapper** (**ORM**) is really needed. We strongly believe that using an ORM for the Write Model is perfectly fine and has all of the advantages of using a tool, which will help us save a lot of work in case we use a relational database. But we shouldn't forget that we still need to persist and retrieve the Write Model's state in a relational database.



Event Sourcing
--------------

* * *

CQRS is a powerful and flexible architecture. There's an added benefit to it in regard to gathering and saving the Domain Events (which occurred during an Aggregate operation), giving you a high-level degree of detail of what's going on within your Domain. Domain Events are one of the key tactical patterns because of their significance within the Domain, as they describe past occurrences.

### Note

****`Be careful with recording too many events`**** An ever-growing number of events is a smell. It might reveal an addiction to event recording at the Domain, most likely incentivized by the business. As a rule of thumb, remember to keep it simple.

By using CQRS, we've been able to record all the relevant events that occurred in the Domain Layer. The state of the Domain Model can be represented by reproducing the Domain Events we previously recorded. We just need a tool for storing all those events in a consistent way. We need an event store.

### Note

The fundamental idea behind Event Sourcing is to express the state of Aggregates as a linear sequence of events

With CQRS, we partially achieved the following: The `Post` entity alters its state by using Domain Events, but it's persisted, as explained already, thereby mapping the object to a database table.

Event Sourcing takes this a step further. If we were using a database table to store the state of all the blog posts, another to store the state of all the blog post comments, and so on, using Event Sourcing would allow us to use a single database table: A single append —  only database table that would store all the Domain Events published by all the Aggregates within the Domain Model. Yes, you read that correctly. A **single** database table.

With this model in mind, tools like object-relational mappers are no longer needed. The only tool needed would be a simple database abstraction layer by which events can be appended:

    interface EventSourcedAggregateRoot
    {
        public static function reconstitute(EventStream $events);
    }
    
    class Post extends AggregateRoot implements EventSourcedAggregateRoot
    {
        public static function reconstitute(EventStream $history)
        {
            $post = new static($history->getAggregateId());
    
            foreach ($events as $event) {
                $post->applyThat($event);
            }
    
            return $post;
        }
    }

Now the `Post` Aggregate has a method which, when given a set of events (or, in other words, an event stream), is able to replay the state step by step until it reaches the current state, all before saving. The next step would be building an adapter of the `PostRepository` port that will fetch all the published events from the `Post` Aggregate and append them to the data store where all the events are appended. This is what we call an event store:

    class EventStorePostRepository implements PostRepository
    {
        private $eventStore;
        private $projector;
    
        public function __construct($eventStore, $projector)
        {
            $this->eventStore = $eventStore;
            $this->projector = $projector;
        }
    
        public function save(Post $post)
        {
            $events = $post->recordedEvents();
    
            $this->eventStore->append(new EventStream(
                $post->id(),  
                $events)
            );
            $post->clearEvents();
    
            $this->projector->project($events);
        }
    }

This is how the implementation of the `PostRepository` looks when we use an event store to save all the events published by the `Post` Aggregate. Now we need a way to restore an Aggregate from its events history. A `reconstitute` method implemented by the `Post` Aggregate and used to rebuild a blog post state from triggered events comes in handy:

    class EventStorePostRepository implements PostRepository
    {
        public function byId(PostId $id)
        {
            return Post::reconstitute(
                $this->eventStore->getEventsFor($id)
            );
        }
    }

The event store is the workhorse that carries out all the responsibility in regard to saving and restoring event streams. Its public API is composed of two simple methods: They are `append` and `getEventsFrom`. The former appends an event stream to the event store, and the latter loads event streams to allow Aggregate rebuilding.

We could use a key-value implementation to store all events:

    class EventStore
    {
        private $redis;
        private $serializer;
    
        public function __construct($redis, $serializer)
        {
            $this->redis = $redis;
            $this->serializer = $serializer;
        }
    
        public function append(EventStream $eventstream)
        {
            foreach ($eventstream as $event) {
                $data = $this->serializer->serialize(
                    $event, 'json'
                );
    
                $date = (new DateTimeImmutable())->format('YmdHis');
    
                $this->redis->rpush(
                    'events:' . $event->getAggregateId(),
                    $this->serializer->serialize([
                        'type' => get_class($event),
                        'created_on' => $date,
                        'data' => $data
                    ],'json')
                );
            }
        }
    
        public function getEventsFor($id)
        {
            $serializedEvents = $this->redis->lrange('events:' . $id, 0, -1);
    
            $eventStream = [];
            foreach($serializedEvents as $serializedEvent){
                $eventData = $this->serializerdeserialize(
                    $serializedEvent, 
                    'array',
                    'json'
               );
    
                $eventStream[] = $this->serializer->deserialize(
                    $eventData['data'],
                    $eventData['type'],
                    'json'
                );
            }
    
            return new EventStream($id, $eventStream);
        }
    }

This event store implementation is built upon [Redis](http://redis.io), a widely used key-value store. The events are appended in a list using the prefix events: In addition, before persisting the events, we extract some metadata like the event class or the creation date, as it will come in handy later.

Obviously, in terms of performance, it's expensive for an Aggregate to go over its full event history to reach its final state all of the time. This is especially the case when an event stream has hundreds or even thousands of events. The best way to overcome this situation is to take a snapshot from the Aggregate and replay only the events in the event stream that occurred after the snapshot was taken. A snapshot is just a simple serialized version of the Aggregate state at any given moment. It can be based on the number of events of the Aggregate's event stream, or it can be time based. With the first approach, a snapshot will be taken every _n_ triggered events (every 50, 100, or 200 events, for example). With the second approach, a snapshot will be taken every _n_ seconds.

To follow the example, we'll use the first way of snapshotting. In the event's metadata, we store an additional field, the version, from which we'll start replaying the Aggregate history:

    class SnapshotRepository
    {
        public function byId($id)
        {
            $key = 'snapshots:' . $id;
            $metadata = $this->serializer->unserialize(
                $this->redis->get($key)
            );
    
            if (null === $metadata) {
                return;
            } 
    
            return new Snapshot(
                $metadata['version'],
                $this->serializer->unserialize(
                    $metadata['snapshot']['data'],
                    $metadata['snapshot']['type'],
                    'json'
                )
            );
        }
    
        public function save($id, Snapshot $snapshot)
        {
            $key = 'snapshots:' . $id;
            $aggregate = $snapshot->aggregate();
    
            $snapshot = [
                'version' => $snapshot->version(),
                'snapshot' => [
                    'type' => get_class($aggregate),
                    'data' => $this->serializer->serialize(
                        $aggregate, 'json'
                    )
                ]
            ];
    
            $this->redis->set($key, $snapshot);
        }
    }


And now we need to refactor the `EventStore` class so that it starts using the `SnapshotRepository` to load the Aggregate with acceptable performance times:

    class EventStorePostRepository implements PostRepository
    {
        public function byId(PostId $id)
        {
            $snapshot = $this->snapshotRepository->byId($id);
    
            if (null === $snapshot) {
                return Post::reconstitute(
                    $this->eventStore->getEventsFrom($id)
                );
            }
    
            $post = $snapshot->aggregate();
    
            $post->replay(
                $this->eventStore->fromVersion($id, $snapshot->version())
            );
    
            return $post;
        }
    }

We just need to take Aggregate snapshots periodically. We could do this synchronously or asynchronously by a process responsible for monitoring the event store. The following code is a simple example demonstrating the implementation of Aggregate snapshotting:

    class EventStorePostRepository implements PostRepository
    {
        public function save(Post $post)
        {
            $id = $post->id();
            $events = $post->recordedEvents();
            $post->clearEvents();
            $this->eventStore->append(new EventStream($id, $events));
            $countOfEvents =$this->eventStore->countEventsFor($id);
            $version = $countOfEvents / 100;
    
            if (!$this->snapshotRepository->has($post->id(), $version)) {
                $this->snapshotRepository->save(
                    $id,
                    new Snapshot(
                        $post, $version
                    )
                );
            }
    
            $this->projector->project($events);
        }
    }


### Note

**`To ORM or Not To ORM`** It's clear from the use case of this architectural style that using an ORM just to persist / fetch events would be overkill. Even if we use a relational database for storing them, we only need to persist / fetch events from the data store.



Wrap-Up
-------

* * *

As there are plenty of options for architectural styles, you may have gotten a bit confused in this chapter. You'll have to consider the tradeoffs for each one of them in order to choose wisely. One thing is clear: the Big Ball of Mud approach is not an option, as the code will rot pretty fast. Layered Architecture is a better option, but it presents some disadvantages, like tight coupling between layers. Arguably, the most balanced option would be Hexagonal Architecture, as it can be used as a foundational base architecture, and it promotes a high-level degree of decoupling and symmetry between the inside and outside of the application. This is what we recommend for most scenarios.

We've also seen CQRS and Event Sourcing as relatively flexible architectures that will help you in fighting serious complexity. CQRS and Event Sourcing both have their places, but don't let the _coolness factor_ distract you from the value they provide. As they both come with some overhead, you should have a technical reason for justifying their use. These architectural styles are indeed really useful, and the heuristics to start using them can be discovered in the number of finders on the Repositories for CQRS and the volume of triggered events for Event Sourcing. If the number of finder methods starts growing and Repositories become difficult to maintain, then it's time to consider the use of CQRS, in order to split read and write concerns. And after that, if the volume of events on each Aggregate operation tends to grow and the business is interested in more granular information, then an option to consider is whether a move toward Event Sourcing might pay off.

### Note

**`Extracted from a paper by Brian Foote and Joseph Yoder:`**_A BIG BALL OF MUD is haphazardly structured, sprawling, sloppy, duct-tape and bailing wire,_ [spaghetti code jungle](http://www.laputan.org/mud/mud.html#BigBallOfMud).
