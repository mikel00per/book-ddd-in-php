Chapter 8. Aggregates
---------------------

Aggregates are probably the most difficult building blocks of Domain-Driven Design. They're hard to understand, and they're even harder to properly design. But don't worry; we're here to help you. However, before jumping into Aggregates, there are some key concepts we need to go through first: transactions and concurrency strategies.



Introduction
------------

* * *

If you've worked with e-commerce applications, it's likely you've faced bugs related to data inconsistencies in your database. For example, consider a shopping order with a total amount of $99.99, which doesn't match with the sum of the amounts of each line in the order, $89.99. Where did that extra $10 come from?

Or, consider a website that sells tickets for the cinema. There's a movie theater with 100 available seats, and after a successful movie promotion, everyone is on the website waiting for the tickets to become available for purchase. Once you open the sales, everything happens fast and you somehow end up selling 102 tickets. You may have specified that there are only 100 seats, but for some reason you exceeded that threshold.

You might even have experience with tracking systems such as JIRA or Redmine. Think about a team of Developers, QAs, and a Product Owner. What could happen if everyone sorts and moves around user stories during a planning meeting and then saves? The final backlog or sprint prioritization would be the one from the team member who saved last.

In general, data inconsistencies occur when we deal with our persistence mechanism in a non-atomic way. An example of this is when you send three queries to a database and some of them work and some don't. The final state of the database is inconsistent. Sometimes, you want these three queries to succeed or fail all together, and that can be fixed with transactions. However, be careful, because as you will see in this chapter, not all inconsistencies are fixed with transactions. In fact, sometimes other data inconsistencies need locking or concurrency strategies. These kinds of tools might come up against your application performance, so be aware that there's a tradeoff involved.

You may think that these kinds of data inconsistencies only occur in databases, but that's not true. For example, if we use a document-oriented database such as Elasticsearch, we can have data inconsistency between two documents. Furthermore, most of the NoSQL persistence storage systems don't support ACID transactions. This means you can't persist or update more than one document in a single operation. So, if we make different requests to Elasticsearch, one may fail, leaving the data persisted in Elasticsearch inconsistent.

Keeping data consistent is a challenge. Not leaking infrastructure issues into the Domain is a bigger challenge. Aggregates aim to help you with both of these things.



Key Concepts
------------

* * *

Persistence engines — and databases in particular — have some features for fighting data inconsistencies: ACID, constraints, referential integrity, locking, concurrency controls, and transactions. Let's review these concepts before working with Aggregates.

Most of these concepts are on the Internet and available to the public. We want to thank the people at Oracle, PostgreSQL, and Doctrine for doing amazing work with their documentation. They have carefully defined and explained these important terms, and rather than reinvent the wheel, we've compiled some of these official explanations to share with you.

### ACID

As discussed in a previous section, **ACID** stands for **atomicity**, **consistency**, **isolation**, and **durability**. According to the [MySQL Glossary](http://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_acid):

These properties are all desirable in a database system, and are all closely tied to the notion of a transaction. For example, the transactional features of MySQL InnoDB engine adhere to the ACID principles.

Transactions are _atomic_ units of work that can be committed or rolled back. When a transaction makes multiple changes to the database, either all the changes succeed when the transaction is committed, or all the changes are undone when the transaction is rolled back.

The database remains in a _consistent_ state at all times, after each commit or rollback, and while transactions are in progress. If related data is being updated across multiple tables, queries see either all old values or all new values, not a mix of old and new values.

Transactions are protected _isolated_ from each other while they are in progress. They cannot interfere with each other or see each other's uncommitted data. This isolation is achieved through the locking mechanism. Experienced users can adjust the isolation level, trading off less protection in favor of increased performance and concurrency, when they can be sure that the transactions really do not interfere with each other.

The results of transactions are _durable_: once a commit operation succeeds, the changes made by that transaction are safe from power failures, system crashes, race conditions, or other potential dangers that many non-database applications are vulnerable to. Durability typically involves writing to disk storage, with a certain amount of redundancy to protect against power failures or software crashes during write operations.

### Transactions

According to the [PostgreSQL 8.2.23 Documentation](https://www.postgresql.org/docs/8.2/static/tutorial-transactions.html):

Transactions are a fundamental concept of all database systems. The essential point of a transaction is that it bundles multiple steps into a single, all-or-nothing operation. The intermediate states between the steps are not visible to other concurrent transactions, and if some failure occurs that prevents the transaction from completing, then none of the steps affect the database at all.

For example, consider a bank database that contains balances for various customer accounts, as well as total deposit balances for branches. Suppose that we want to record a payment of $100.00 from Alice's account to Bob's account. Simplifying outrageously, the SQL commands for this might look like:

    UPDATE accounts
        SET balance = balance - 100.00
    WHERE name = 'Alice';
    
    UPDATE branches
        SET balance = balance - 100.00
    WHERE name = (SELECT branch_name FROM accounts WHERE name ='Alice');
    
    UPDATE accounts
        SET balance = balance + 100.00
    WHERE name = 'Bob';
    
    UPDATE branches
        SET balance = balance + 100.00
    WHERE name = (SELECT branch_name FROM accounts WHERE name ='Bob');

The details of these commands are not important here. The important point is that there are several separate updates involved to accomplish this rather simple operation. Our bank's officers will want to be assured that either all these updates happen, or none of them happen. It would certainly not do for a system failure to result in Bob receiving $100.00 that was not debited from Alice. Nor would Alice long remain a happy customer if she was debited without Bob being credited. We need a guarantee that if something goes wrong partway through the operation, none of the steps executed so far will take effect. Grouping the updates into a transaction gives us this guarantee. A transaction is said to be atomic: from the point of view of other transactions, it either happens completely or not at all.

We also want a guarantee that once a transaction is completed and acknowledged by the database system, it has indeed been permanently recorded and won't be lost even if a crash ensues shortly thereafter. For example, if we are recording a cash withdrawal by Bob, we do not want any chance that the debit to his account will disappear in a crash just after he walks out the bank door. A transactional database guarantees that all the updates made by a transaction are logged in permanent storage (That is: on disk) before the transaction is reported complete.

Another important property of transactional databases is closely related to the notion of atomic updates: when multiple transactions are running concurrently, each one should not be able to see the incomplete changes made by others. For example, if one transaction is busy totalling all the branch balances, it would not do for it to include the debit from Alice's branch but not the credit to Bob's branch, nor vice versa. So transactions must be all-or-nothing not only in terms of their permanent effect on the database, but also in terms of their visibility as they happen. The updates made so far by an open transaction are invisible to other transactions until the transaction completes, whereupon all the updates become visible simultaneously.

In PostgreSQL, for example, a transaction is set up by surrounding the SQL commands of the transaction with `BEGIN` and `COMMIT` commands. So our banking transaction would actually look like:

    BEGIN;
    UPDATE accounts 
        SET balance = balance - 100.00 
    WHERE name = 'Alice';
    -- etc etc
    COMMIT;

If, partway through the transaction, we decide we do not want to commit (perhaps we just noticed that Alice's balance went negative), we can issue the command `ROLLBACK` instead of `COMMIT`, and all our updates so far will be canceled.

PostgreSQL actually treats every SQL statement as being executed within a transaction. If you do not issue a `BEGIN` command, then each individual statement has an implicit `BEGIN` and (if successful) `COMMIT` wrapped around it. A group of statements surrounded by `BEGIN` and `COMMIT` is sometimes called a transaction block.

All this is happening within the transaction block, so none of it is visible to other database sessions. When and if you commit the transaction block, the committed actions become visible as a unit to other sessions, while the rolled-back actions never become visible at all.

### Isolation Levels

According to the [MySQL Glossary](http://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_isolation_level), transaction isolation is:

One of the foundations of database processing. Isolation is the "I" in the acronym ACID. The isolation level is the setting that fine-tunes the balance between performance and reliability, consistency, and reproducibility of results when multiple transactions are making changes and performing queries at the same time.

From highest amount of consistency and protection to the least, the isolation levels supported by InnoDB, for example, are: `SERIALIZABLE`, `REPEATABLE READ`, `READ COMMITTED`, and `READ UNCOMMITTED`.

With InnoDB tables, many users can keep the default isolation level `REPEATABLE READ` for all operations. Expert users might choose the read committed level as they push the boundaries of scalability with OLTP processing, or during data warehousing operations where minor inconsistencies do not affect the aggregate results of large amounts of data. The levels on the edges (`SERIALIZABLE` and `READ UNCOMMITTED`) change the processing behavior to such an extent that they are rarely used.

### Referential Integrity

According to the [MySQL Glossary](http://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_referential_integrity), referential integrity is:

The technique of maintaining data always in a consistent format, part of the ACID philosophy. In particular, data in different tables is kept consistent through the use of foreign key constraints, which can prevent changes from happening or automatically propagate those changes to all related tables. Related mechanisms include the unique constraint, which prevents duplicate values from being inserted by mistake, and the `NOT NULL` constraint, which prevents blank values from being inserted by mistake.

### Locking

According to the [MySQL Glossary](http://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_locking), locking is:

The system of protecting a transaction from seeing or changing data that is being queried or changed by other transactions. The locking strategy must balance reliability and consistency of database operations (the principles of the ACID philosophy) against the performance needed for good concurrency. Fine-tuning the locking strategy often involves choosing an isolation level and ensuring all your database operations are safe and reliable for that isolation level.

### Concurrency

According to the [MySQL Glossary](http://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_concurrency), concurrency is:

The ability of multiple operations (in database terminology, transactions) to run simultaneously, without interfering with each other. Concurrency is also involved with performance, because ideally the protection for multiple simultaneous transactions works with a minimum of performance overhead, using efficient mechanisms for locking.

#### Pessimistic Concurrency Control (PCC)

The book [Elasticsearch: The Definitive Guide](https://github.com/elastic/elasticsearch-definitive-guide/blob/master/030_Data/40_Version_control.asciidoc) by Clinton Gormley and Zachary Tong discusses PCC, saying that:

Widely used by relational databases, this approach assumes that conflicting changes are likely to happen and so blocks access to a resource in order to prevent conflicts. A typical example is locking a row before reading its data, ensuring that only the thread that placed the lock is able to make changes to the data in that row.

##### With Doctrine

According to the [Doctrine 2 ORM Documentation](http://doctrine-orm.readthedocs.io/projects/doctrine-orm/en/latest/reference/transactions-and-concurrency.html#locking-support) on locking support:

Doctrine 2 offers support for Pessimistic- and Optimistic-locking strategies natively. This allows to take very fine-grained control over what kind of locking is required for your Entities in your application.

According to the [Doctrine 2 ORM Documentation](http://doctrine-orm.readthedocs.io/projects/doctrine-orm/en/latest/reference/transactions-and-concurrency.html#pessimistic-locking) on pessimistic locking:

Doctrine 2 supports Pessimistic Locking at the database level. No attempt is being made to implement pessimistic locking inside Doctrine, rather vendor-specific and ANSI-SQL commands are used to acquire row-level locks. Every Doctrine Entity can be part of a pessimistic lock, there is no special metadata required to use this feature.

However for Pessimistic Locking to work you have to disable the Auto-Commit Mode of your Database and start a transaction around your pessimistic lock use-case using the _Explicit Transaction Demarcation_. Doctrine 2 will throw an Exception if you attempt to acquire an pessimistic lock and no transaction is running.

Doctrine 2 currently supports two pessimistic lock modes:

*   Pessimistic Write `Doctrine\DBAL\LockMode::PESSIMISTIC_WRITE`, locks the underlying database rows for concurrent Read and Write Operations.
*   Pessimistic Read `Doctrine\DBAL\LockMode::PESSIMISTIC_READ`, locks other concurrent requests that attempt to update or lock rows in write mode.

You can use pessimistic locks in three different scenarios:

*   Using `EntityManager#find($className, $id, \Doctrine\DBAL\LockMode::PESSIMISTIC_WRITE)` or `EntityManager#find($className, $id, \Doctrine\DBAL\LockMode::PESSIMISTIC_READ)`
*   Using `EntityManager#lock($entity, \Doctrine\DBAL\LockMode::PESSIMISTIC_WRITE)` or `EntityManager#lock($entity, \Doctrine\DBAL\LockMode::PESSIMISTIC_READ)`
*   Using `Query#setLockMode(\Doctrine\DBAL\LockMode::PESSIMISTIC_WRITE)` or `Query#setLockMode(\Doctrine\DBAL\LockMode::PESSIMISTIC_READ)`

#### Optimistic Concurrency Control

According to [Wikipedia](https://en.wikipedia.org/wiki/Optimistic_concurrency_control):

**Optimistic concurrency control** (**OCC**) is a concurrency control method applied to transactional systems such as relational database management systems and software transactional memory. OCC assumes that multiple transactions can frequently complete without interfering with each other. While running, transactions use data resources without acquiring locks on those resources. Before committing, each transaction verifies that no other transaction has modified the data it has read. If the check reveals conflicting modifications, the committing transaction rolls back and can be restarted. Optimistic concurrency control was first proposed by H.T. Kung.

OCC is generally used in environments with low data contention. When conflicts are rare, transactions can complete without the expense of managing locks and without having transactions wait for other transactions' locks to clear, leading to higher throughput than other concurrency control methods. However, if contention for data resources is frequent, the cost of repeatedly restarting transactions hurts performance significantly; it is commonly thought that other concurrency control methods have better performance under these conditions. However, locking-based "pessimistic" methods also can deliver poor performance because locking can drastically limit effective concurrency even when deadlocks are avoided.

##### With Elasticsearch

According to [Elasticsearch: The Definitive Guide](https://github.com/elastic/elasticsearch-definitive-guide/blob/master/030_Data/40_Version_control.asciidoc), when OCC is used by Elasticsearch:

This approach assumes that conflicts are unlikely to happen and doesn't block operations from being attempted. However, if the underlying data has been modified between reading and writing, the update will fail. It is then up to the application to decide how it should resolve the conflict. For instance, it could reattempt the update, using the fresh data, or it could report the situation to the user.

Elasticsearch is distributed. When documents are created, updated, or deleted, the new version of the document has to be replicated to other nodes in the cluster. Elasticsearch is also asynchronous and concurrent, meaning that these replication requests are sent in parallel, and may arrive at their destination out of sequence. Elasticsearch needs a way of ensuring that an older version of a document never overwrites a newer version.

Every document has a \_version number that is incremented whenever a document is changed. Elasticsearch uses this \_version number to ensure that changes are applied in the correct order. If an older version of a document arrives after a new version, it can simply be ignored.

We can take advantage of the \_version number to ensure that conflicting changes made by our application do not result in data loss. We do this by specifying the version number of the document that we wish to change. If that version is no longer current, our request fails.

Let's create a new blog post:

        PUT /website/blog/1/_create
        { 
           "title": "My first blog entry",
           "text": "Just trying this out..." 
        } 

The response body tells us that this newly created document has \_version number 1. Now imagine that we want to edit the document: we load its data into a web form, make our changes, and then save the new version.

First we retrieve the document:

        GET /website/blog/1

The response body includes the same \_version number of 1:

    { 
        "index": "website", 
        "type": "blog", 
        "id": "1", 
        "version": 1, 
        "found": true, 
        "_source": { 
            "title": "My first blog entry", 
            "text": "Just trying this out..." 
        } 
    }

Now, when we try to save our changes by reindexing the document, we specify the version to which our changes should be applied. We want this update to succeed only if the current \_version of this document in our index is version 1:

       PUT /website/blog/1?version=1
       { 
          "title": "My first blog entry", 
          "text": "Starting to get the hang of this..."  
       }

This request succeeds, and the response body tells us that the \_version has been incremented to 2:

       { 
          "index": "website", 
          "type": "blog", 
          "id": "1", 
          "version": 2,
          "created": false 
       }

However, if we were to rerun the same index request, still specifying `version=1`, Elasticsearch would respond with a `409` Conflict HTTP response code, and a body like the following:

    {
        "error": {
            "root_cause": [{ 
                 "type": "version_conflict_engine_exception",
                 "reason":
                     "[blog][1]: version conflict,current[2],provided [1]",   
                 "index": "website",
                 "shard": "3"
            }],
            "type": "version_conflict_engine_exception" ,
            "reason":
                "[blog][1]:version conflict,current [2],provided[1]",
            "index": "website",
            "shard": "3"
        },
        "status": 409
    }

This tells us that the current \_version number of the document in Elasticsearch is 2, but that we specified that we were updating version 1.

What we do now depends on our application requirements. We could tell the user that somebody else has already made changes to the document, and to review the changes before trying to save them again. Alternatively, as in the case of the widget `stock_count` previously, we could retrieve the latest document and try to reapply the change.

All APIs that update or delete a document accept a version parameter, which allows you to apply optimistic concurrency control to just the parts of your code where it makes sense.

##### With Doctrine

According to the [Doctrine 2 ORM Documentation](http://doctrine-orm.readthedocs.io/projects/doctrine-orm/en/latest/reference/transactions-and-concurrency.html#optimistic-locking) on optimistic locking:

Database transactions are fine for concurrency control during a single request. However, a database transaction should not span across requests, the so-called "user think time". Therefore a long-running "business transaction" that spans multiple requests needs to involve several database transactions. Thus, database transactions alone can no longer control concurrency during such a long-running business transaction. Concurrency control becomes the partial responsibility of the application itself.

Doctrine has integrated support for automatic optimistic locking via a version field. In this approach any entity that should be protected against concurrent modifications during long-running business transactions gets a version field that is either a simple number (mapping type: `integer`) or a timestamp (mapping type: `datetime`). When changes to such an entity are persisted at the end of a long-running conversation the version of the entity is compared to the version in the database and if they don't match, an `OptimisticLockException` is thrown, indicating that the entity has been modified by someone else already.

You designate a version field in an entity as follows. In this example we'll use an integer:

       class User
       { 
           // ... 
           /** @Version @Column(type="integer") */ 
           private $version; 
           // ... 
       }

When a version conflict is encountered during `EntityManager#flush()`, an `OptimisticLockException` is thrown and the active transaction rolled back (or marked for rollback). This exception can be caught and handled. Potential responses to an `OptimisticLockException` are to present the conflict to the user or to refresh or reload objects in a new transaction and then retrying the transaction.

With PHP promoting a share-nothing architecture, the time between showing an update form and actually modifying the entity can in the worst scenario be as long as your applications session timeout. If changes happen to the entity in that time frame you want to know directly when retrieving the entity that you will hit an optimistic locking exception:

You can always verify the version of an entity during a request either when calling `EntityManager#find()`:

    use Doctrine\DBAL\LockMode; 
    use Doctrine\ORM\OptimisticLockException;
    
    $theEntityId = 1; 
    $expectedVersion = 184;
    try{
       $entity = $em->find(
           'User',
           $theEntityId,
           LockMode::OPTIMISTIC,
           $expectedVersion
       );
       // do the work
       $em->flush();
    } catch (OptimisticLockException $e){
        echo
            'Sorry, someone has already changed this entity.' .
            'Please apply the changes again!';
    }

Or you can use `EntityManager#lock()` to find out:

    use DoctrineDBALLockMode; 
    use DoctrineORMOptimisticLockException;
    
    $theEntityId = 1;
    $expectedVersion = 184;
    $entity = $em->find('User', $theEntityId);
    try {
        // assert version em−>lock(entity, LockMode::OPTIMISTIC,
        $expectedVersion);
    } catch (OptimisticLockException $e){
        echo
            'Sorry, someone has already changed this entity.' .
            'Please apply the changes again!';
    }

According to [Doctrine 2 ORM](http://doctrine-orm.readthedocs.io/projects/doctrine-orm/en/latest/reference/transactions-and-concurrency.html#important-implementation-notes) [Document](http://doctrine-orm.readthedocs.io/projects/doctrine-orm/en/latest/reference/transactions-and-concurrency.html#important-implementation-notes)[ation's](http://doctrine-orm.readthedocs.io/projects/doctrine-orm/en/latest/reference/transactions-and-concurrency.html#important-implementation-notes)  important implementation notes:

You can easily get the optimistic locking workflow wrong if you compare the wrong versions. Say you have Alice and Bob editing a hypothetical blog post:

*   Alice reads the headline of the blog post being "Foo", at optimistic lock version 1 (`GET` Request)
*   Bob reads the headline of the blog post being "Foo", at optimistic lock version 1 (`GET` Request)
*   Bob updates the headline to "Bar", upgrading the optimistic lock version to 2 (`POST` Request of a Form)
*   Alice updates the headline to "Baz", ... (`POST` Request of a Form)

Now at the last stage of this scenario the blog post has to be read again from the database before Alice's headline can be applied. At this point you will want to check if the blog post is still at version 1 (which it is not in this scenario).

Using optimistic locking correctly, you have to add the version as an additional hidden field (or into the `SESSION` for more safety). Otherwise you cannot verify the version is still the one being originally read from the database when Alice performed her `GET` request for the blog post. If this happens you might see lost updates you wanted to prevent with Optimistic Locking.

See the example code, The form (`GET` Request):

    $post = $em->find('BlogPost', 123456);
    echo '<input type="hidden" name="id" value="' .
             $post->getId() . '"/>';
    echo '<input type="hidden" name="version" value="' .
             $post->getCurrentVersion() . '" />';

And the change headline action (`POST` Request):

    $postId = (int) $_GET['id']; 
    $postVersion = (int) $_GET['version'];                
    $post = $em->find(
        'BlogPost',
        $postId,
        DoctrineDBALLockMode::OPTIMISTIC,
        $postVersion
    );

Wow — that was a lot of information to take in. However, don't worry if you don't completely understand everything. The more you work with Aggregates and Domain-Driven Design, the more you'll encounter moments when transactionality has to be considered in designing your Application.

To summarize, if you want to keep your data consistent, use transactions. However, be careful about overusing transactions or locking strategies because these can slow your Application down or make it unusable. If you want to have a really fast Application, optimistic concurrency can help you. Last but not least, some data can eventually be consistent. This means that we allow our data to not be consistent for a particular window of time. During that time, some inconsistencies are acceptable. Eventually, an asynchronous process will perform the final task to remove such inconsistencies.



What Is an Aggregate?
---------------------

* * *

Aggregates are Entities that hold other Entities and Value Objects that help keep data consistent. From Vaughn Vernon's [Implementing Domain-Driven Design](https://www.amazon.com/Implementing-Domain-Driven-Design-Vaughn-Vernon/dp/0321834577):

Aggregates are carefully crafted consistency boundaries that cluster Entities and Value Objects.

Another amazing book that you should definitely buy and read is [NoSQL Distilled: A Brief Guide to the Emerging World of Polyglot Persistence](https://www.amazon.com/NoSQL-Distilled-Emerging-Polyglot-Persistence/dp/0321826620?) by Pramod J. Sadalage and Martin Fowler. This book says that:

In Domain-Driven Design, an aggregate is a collection of related objects that we wish to treat as a unit. In particular, it is a unit for data manipulation and management of consistency. Typically, we like to update aggregates with atomic operations and communicate with our data storage in terms of aggregates.

### What Martin Fowler Says...

From [http://martinfowler.com/bliki/DDD\_Aggregate.html](http://martinfowler.com/bliki/DDD_Aggregate.html):

Aggregate is a pattern in Domain-Driven Design. A DDD aggregate is a cluster of domain objects that can be treated as a single unit. An example may be an order and its line-items, these will be separate objects, but it is useful to treat the order (together with its line items) as a single aggregate.

An aggregate will have one of its component objects be the aggregate root. Any references from outside the aggregate should only go to the aggregate root. The root can thus ensure the integrity of the aggregate as a whole.

Aggregates are the basic element of transfer of data storage you request to load or save whole aggregates. Transactions should not cross aggregate boundaries.

DDD Aggregates are sometimes confused with collection classes (lists, maps, and so on). DDD aggregates are domain concepts (order, clinic visit, playlist), while collections are generic. An aggregate will often contain multiple collections, together with simple fields. The term _aggregate_ is a common one, and is used in various different contexts (example: UML), in which case it does not refer to the same concept as a DDD aggregate.

### What Wikipedia Says...

From [https://en.wikipedia.org/wiki/Domain-driven\_design#Building\_blocks\_of\_DDD](https://en.wikipedia.org/wiki/Domain-driven_design#Building_blocks_of_DDD):

Aggregate: A collection of objects that are bound together by a root entity, otherwise known as an aggregate root. The aggregate root guarantees the consistency of changes being made within the aggregate by forbidding external objects from holding references to its members.

Example: When you drive a car, you do not have to worry about moving the wheels forward, making the engine combust with spark and fuel, etc.; you are simply driving the car. In this context, the car is an aggregate of several other objects and serves as the aggregate root to all of the other systems.



Why Aggregates?
---------------

* * *

The avid reader will probably be wondering what all of this has to do with Aggregates and Aggregate Design. And actually, that's a pretty good question. There's a direct relation, so let's explore it. The Relational Model uses tables to store data. Those tables are composed of rows, where each row usually represents an instance of a concept of the application's interest. Additionally, each row can point to other rows on other tables of the same database, and the consistency between this relationship can be kept by the use of referential integrity. This model is fine; however, it lacks a very basic word: the _object_ word.

Indeed, when we talk about the Relational Model, we're namely talking about tables, rows, and relationships between rows. And when we talk about the Object-Oriented Model, we're talking mainly about compositions of objects. So every time we fetch data — a set of rows — from a relational database, we run a translation process responsible for building an in-memory representation we can operate with. The same applies to the opposite direction. Every time we need to store an object in the database, we should run the other translation process to translate that object to a given set of rows or tables. This translation, from object to rows or tables, means that you may run different queries against your database. As such, without using any specific tool, such as transactions, it's impossible to guarantee the data will be persisted consistently. This problem is the so-called **impedance mismatch**.

### Note

**`Impedance Mismatch`** The object-relational impedance mismatch is a set of conceptual and technical difficulties that are often encountered when a **relational database management system** (**RDBMS**) is being used by a program written in an object-oriented programming language or style, particularly when objects or class definitions are mapped in a straightforward way to database tables or relational schemata. Extracted from [Wikipedia](https://en.wikipedia.org/wiki/Object-relational_impedance_mismatch)

The impedance mismatch [is not an easy problem to solve](http://martinfowler.com/bliki/OrmHate.html), so we highly discourage trying to solve it on your own. It would be a huge undertaking, and it's simply not worth the effort. Luckily, there are some libraries out there that take care of this translation process. They're commonly known as Object-Relational Mappers (which we've discussed in earlier chapters) and their primary concern is to ease the process of translating from the Relational Model to the Object-Oriented Model, and vice versa.

This is an issue that also affects NoSQL persistence engines and not just databases. Most NoSQL engines use documents. An Entity is translated into a document representation such as JSON, XML, binary, and so on. and then persisted. The main difference with RDBMS databases is that if a main Entity (such as `Order`) has other related Entities (such as `OrderLines`), you can more easily design a single JSON document that will contain all the information. With this approach, with a single request to your NoSQL engine, you don't need transactions.

Nevertheless, if you're using NoSQL or RDBMS for fetching and persisting your Entities, you'll need one or more queries. In order to ensure data consistency, those queries or requests need to be executed as a single operation. Running as a single operation can guarantee that data will be consistent.

What does consistent mean? It means that all data persisted into our database must be compliant with all business rules, also known as invariants. An example of a business invariant could be how on GitHub, a user is able to have unlimited public repositories but no private repositories. However, if this user pays $12 per month, then they're able to have up to 10 private repositories.

Relational databases provide three main tools for helping us with data consistency: \* **Referential** integrity: Foreign keys, nullable checks, and so on. \* **Transactions**: Run multiple queries as a single operation. The problem with transactions is the same as that of branches and merges in your code repository. Keeping a branch has a performance cost (memory, CPU, storage, indexing, and so on.). If too many people (concurrency) are touching the same data, conflicts will occur and transaction commits will fail. \* **Locking**: Block rows or tables. Other queries around the same tables and rows must wait for the block to be removed. Locking has a negative impact on the performance of your application.

Suppose we have an e-commerce application we want to expand to other countries and regions, and suppose the release goes fairly well and sales increase. A pretty evident side effect of the release is that the database should be able to handle the additional load increase. As seen earlier, there are two scaling methods: up or out.

Scaling up means we improve the hardware infrastructure we have (For example: better CPU, more memory, better hard disks). Scaling out means adding more machines that will organize in a cluster for doing specific work. In this case, we could have a cluster of databases.

But relational databases aren't designed to scale horizontally, since we can't configure them to save one set of rows to a given machine and another set of rows to a different one. Relational databases are easy to scale up, but **the Relational Model doesn't scale horizontally**.

In the NoSQL world, data consistency is a bit more difficult: transactions and referential integrity aren't generally supported, while locking is supported but generally not encouraged.

NoSQL databases aren't affected as drastically by the impedance mismatch. They match perfectly with Aggregate Design because they enable us to easily store and retrieve single units atomically. For example, when using a key-value store such as Redis, an Aggregate could be serialized and stored on a specific key. On a document-oriented store such as Elasticsearch, an Aggregate would be serialized into a JSON and persisted as a document. As mentioned before, the problem comes when multiple documents must be updated at once.

For that reason, when persisting any object with a single representation (one document, so no multiple queries needed), it's easy to distribute those single units across several machines, called nodes, which make up a cluster of NoSQL databases. It's common knowledge that these databases are easy to distribute, which means that the style of databases is easy to scale horizontally.



A Bit of History
----------------

* * *

Around the beginning of the 21st century, companies such as Amazon and Google grew massively. In order to consolidate their growth, they used clustering techniques: not only did they have better servers, but they also relied on many more of them working together.

In a scenario such as this, deciding how to store your data is key. If you take an Entity and spread its information throughout multiple servers, in multiple nodes of a cluster, the effort needed to control transactions is high. The same thing applies if you want to fetch an Entity. So if you can design your Entity in a way that is persisted in the node of a cluster, it makes things much easier. That's one of the reasons why Aggregate Design is so important.

If you want to know more about the history of Aggregate Design outside of Domain-Driven Design, take a look at [NoSQL Distilled: A Brief Guide to the Emerging World of Polyglot Persistence](https://www.amazon.com/NoSQL-Distilled-Emerging-Polyglot-Persistence/dp/0321826620?).



Anatomy of an Aggregate
-----------------------

* * *

An Aggregate is an Entity that may hold other Entities and Value Objects. The parent Entity is known as the root Entity.

A single Entity without any child Entities or Value Objects is also an Aggregate by itself. That's why in some books, the term Aggregates is used instead of the term Entity. When we use them here, Entity and Aggregate mean the same thing.

The main goal of an Aggregate is to keep your Domain Model consistent. Aggregates centralize most of the business rules. Aggregates are persisted atomically in your persistence mechanism. No matter how many child Entities and Value Objects live inside the root Entity, all of them will be persisted atomically, as a single unit. Let's see an example.

Consider an e-commerce application, website, and so on. Users can place orders, which have multiple lines that define what product was bought, the price, the quantity, and the line total amount. An order has a total amount too, which is the sum of all line amounts.

What could happen if you update a line amount but not the order amount? Data inconsistency. To fix this, any modification to any Entity or Value Object within the Aggregate is performed through the root Entity. Most PHP developers we've worked with are more comfortable building objects and then handling their relationships from the client code, rather than pushing the business logic inside the Entities:

    $order = ...
    $orderLine = new OrderLine(
        'Domain-Driven Design in PHP', 24.99
    );
    $order->addOrderLine($orderLine);

As seen in the previous code example, newbie or even average developers generally build child objects first and then relate them to the parent object using a setter. Consider the following approach:

    $order = ...
    $orderLine = $order->addOrderLine(
        'Domain-Driven Design in PHP', 24.99
    );

Or, consider this approach:

    $order = ...
    $order->addOrderLine(
        'Domain-Driven Design in PHP', 24.99
    );

These approaches are very interesting because they follow two Software Design principles: Tell-Don't-Ask and Law of Demeter.

According to [Martin Fowler](http://martinfowler.com/bliki/TellDontAsk.html):

Tell-Don't-Ask is a principle that helps people remember that object-orientation is about bundling data with the functions that operate on that data. It reminds us that rather than asking an object for data and acting on that data, we should instead tell an object what to do. This encourages to move behavior into an object to go with the data.

According to [Wikipedia](https://en.wikipedia.org/wiki/Law_of_Demeter):

The **Law of Demeter** (**LoD**) or principle of least knowledge is a design guideline for developing software, particularly object-oriented programs. In its general form, the LoD is a specific case of loose coupling...and can be succinctly summarized in each of the following ways:

*   Each unit should have only limited knowledge about other units: only units "closely" related to the current unit.
*   Each unit should only talk to its friends; don't talk to strangers.
*   Only talk to your immediate friends.

The fundamental notion is that a given object should assume as little as possible about the structure or properties of anything else (including its subcomponents), in accordance with the principle of "information hiding".

Let's continue with the order example. You've already learned how to run operations through the root Entity. Now let's update a product quantity of a line in an order. This increases the quantity, the line total amount, and the order amount. Great! Now it's time to persist the order with the changes.

If you're using MySQL, you can imagine that we'll need two `UPDATE` statements: one for the orders table, and another one for the `order_lines` table. What could happen if these two queries aren't performed inside a transaction?

Let's assume that the `UPDATE` statement that updates the line order works properly. However, the `UPDATE` on the order total amount fails due to network connectivity issues. In such a scenario, you would end up with a data inconsistency in your Domain Model. Transactions help you keep this consistency.

If you're using Elasticsearch, the situation is a bit different. You can map the order with a JSON document that holds order lines internally, so just a single request is needed. However, if you decide to map the order with one JSON document and each of its order lines with another JSON document, you're in trouble, as Elasticsearch doesn't support transactions. Ouch!

An Aggregate is fetched and persisted using its own [Chapter 10](/chapters/10%20Repositories.md), _Repositories_. If two Entities don't belong to the same Aggregate, both will have their own Repository. If a true business invariant exists and two Entities belong to the same Aggregate, you'll only have one Repository. This Repository will be the one for the root Entity.

What are the cons of Aggregates? The problem when dealing with transactions is the possibility of performance issues and operation errors. We'll explore this in depth soon.



Aggregate Design Rules
----------------------

* * *

When designing an Aggregate, there are some rules and considerations to follow in order to get all the benefits and minimize the negative effects. Don't worry too much if you don't understand everything now; as an example, we'll show you a small application where we'll be referencing the rules we introduce you to.

### Design Aggregates Based in Business True Invariants

First of all, what's an invariant? An invariant is a rule that must be true and consistent during code execution. For example, a [stack](https://en.wikipedia.org/wiki/Stack_(abstract_data_type)) is a **LIFO** (**Last In, First Out**) data structure that we can _push_ items into and _pop_ items out of. We can also ask how many items are inside of the stack; this is what's called the size of the stack. Consider a pure PHP implementation without using any specific PHP array functions such as `array_pop`:

    class Stack 
    {   
        private $data;
    
        public function __construct()
        {
            $this->data = [];
        }
    
        public function push($value)
        {
            $this->data[] = $value;
        }
    
        public function size()
        {
            $size = 0;
            for ($i = 0; $i < count($this->data); $i++) {
                $size++;
            }
    
            return $size;
        }
    
        /**
         * @return mixed
         */
        public function pop()
        {
            $topIndex = $this->size() - 1;
            $top = $this->data[$topIndex];
            unset($this->data[$topIndex]);
            return $top;
        }
    }

Consider the previous `size` method implementation. It's far from perfect, but it works. However, as it's implemented in the code above, it's a CPU-intensive and high-cost call. Luckily, there's an option to improve this method, by introducing a private attribute to keep track of the number of elements in the internal array:

    class Stack 
    { 
        private $data; 
        private $size;
    
        public function __construct()
        {
            $this->data = [];
            $this->size = 0;
        }
    
        public function push($value)
        {
            $this->data[] = $value;
            $this->size++;
        }
    
        public function size()
        {
           return $this->size;
        }
    
        /**
         * @return mixed
         */
        public function pop()
        {
            $topIndex = $this->size--;
            $top = $this->data[$topIndex];
            unset($this->data[$topIndex]);
    
            return $top;
        }
    }

With these changes, the `size` method is now a fast operation, as it just returns the value of the `size` field. To accomplish this, we introduced a new integer attribute called `size`. When a new `Stack` is created, the value of `size` is `0`, and there's no element in the Stack. When we add a new element into the Stack using the `push` method, we also increase the value of the `size` field. Similarly, we reduce the value of `size` when we remove a value from the Stack using the `pop` method.

By incrementing and decreasing the value of size, we keep it consistent with the real number of elements that are inside the `Stack`. The size value is consistent right before and right after calling any public method in the `Stack` class. As a result, the size value is always equal to the number of elements in the `Stack`. That's an invariant! We could write it down as `$this->size === count($this->data)`.

A true business invariant is a business rule that must always be true and transactionally consistent within an Aggregate. By transactionally consistent, we mean that updating an aggregate must be an atomic operation. All the data contained inside an Aggregate must be persisted atomically. If we don't follow this rule, we could persist data representing a non-valid Aggregate.

According to [Vaughn Vernon](https://www.amazon.com/Implementing-Domain-Driven-Design-Vaughn-Vernon/dp/0321834577):

A properly designed Aggregate is one that can be modified in any way required by the business with its invariants completely consistent within a single transaction. And a properly designed Bounded Context modifies only one Aggregate instance per transaction in all cases. What is more, we cannot correctly reason on Aggregate design without applying transactional analysis.

As discussed in the introduction, in an e-commerce application, the amount of an order must match the sum of the amounts of every line in that order. That's an invariant, or business rule. We have to persist the `Order` and its `OrderLines` into the database in the same transaction. This forces us to make `Order` and `OrderLine` be part of the same Aggregate, where `Order` would be the Aggregate Root. Because `Order` is the root, all operations related to `OrderLines` must go through the Order. So no more instantiating `OrderLine` objects outside of an `Order` and then using a setter method to add `OrderLines` to the `Order`. Instead, we must use Factory Methods on the `Order`.

With this approach, we have a single entry point to perform operations on the Aggregate: the `Order`. It means there's no chance of invoking a method to break such a rule. Each time you add or update an `OrderLine` through the `Order`, the `Order` amount gets recalculated internally. Making all operations go through the root help us keep the Aggregate consistent. In this way, it's more difficult to break any invariant.

### Small Aggregates Vs. Big Aggregates

For most of the websites and projects where we've worked, almost 95 percent of Aggregates were formed by one single root Entity and some Value Objects. No other Entities were required to be in the same Aggregate. So in most cases, there was no real true business invariant to keep consistent.

Be careful with the has-a/has-many relations that don't necessarily make two Entities become one Aggregate, with one of those being the root. Relations, as we will see, can be handled by referencing Entity Identities.

As explained in the introduction, an Aggregate is a transactional boundary. The smaller the boundary is, the fewer chances there are for conflicts when committing multiple concurrent transactions. When designing Aggregates, you should strive to create them small. If there's no true invariant to protect, that means all single Entities form an Aggregate by themselves. That's great, because it's the best scenario for achieving the best performance. Why? Because locking issues and failed transaction issues are minimized.

If you decide to go for big Aggregates, keeping data consistent can be easier but is probably impractical. When applications with big Aggregates run in production, they start to experience issues when multiple users perform operations. When using optimistic concurrency, the main problem is transactional failures. When using locking, the problem is slowness and timeouts.

Let's consider some radical examples. When using optimistic concurrency, imagine that the whole Domain is versioned, and each operation on any Entity creates a new version for the whole Domain. With this scenario, if two users were performing different operations on different Entities that couldn't be related at all, the second request would experience a transaction failure because of a different version. On the other hand, when using pessimistic concurrency, imagine a scenario where we lock the database on each operation. That would block all the users until the lock is released. This means many requests would be waiting, and at some point, probably timed out. Both of these examples keep data consistent, but the application can't be used by more than one user.

Last but not least, when designing big Aggregates, because they may hold collections of Entities, it's important to consider the performance implications of loading such collections in memory. Even using an ORM such as Doctrine, which can lazy load collections (load collections only when they are needed), if a collection is too big, it can't fit into memory.

### Reference Other Entities by Identity

When two Entities don't form an Aggregate but are related, the best option to have Entities reference one another is by using _Identities_. Identities were already explained in the [Chapter 4](/chapters/04%20Entities.md), _Entities_.

Consider a `User` and their `Orders`, and assume we haven't found any true invariant. `User` and `Order` wouldn't be part of the same Aggregate. If you need to know which `User` owns a specific `Order`, you can probably ask the `Order` what its `UserId` is. `UserId` is a Value Object that holds the `User` Identity. We get the whole `User` by using its Repository, the `UserRepository`. This code generally lives in the Application Service.

As a general explanation, each Aggregate has its own Repository. If you've fetched a specific Aggregate and you need to fetch another related Aggregate, you'll do it in your Application Services or Domain Services. The Application Service will depend on Repositories to fetch the Aggregates needed.

Jumping from one Aggregate to another is what's generally called traversing or navigating your Domain. With ORMs, it's easy to do it by mapping all the relations between your Entities. However, it's also really dangerous, as you can easily end up running countless queries in a specific feature. As a rule, you shouldn't do this. Don't map all the relations between your Entities because you can. Instead, only map the relations between Entities inside an Aggregate in your ORM if two Entities form an Aggregate. If this isn't the case, you'll use Repositories to get referenced Aggregates.

### Modify One Aggregate Per Transaction and Request

Consider the following scenario: you make a request, it gets into your controller, and it intends to update two different Aggregates. Each Aggregate keeps the data consistent within that Aggregate. However, what would happen if the request goes well over the first Aggregate update but suddenly stops (server restarted, reloaded, out of memory, and so on.) and the second Aggregate isn't updated? Is that a data consistency issue? It may be. Let's consider some solutions.

From Vaughn Vernon's [Implementing Domain-Driven Design](https://www.amazon.com/Implementing-Domain-Driven-Design-Vaughn-Vernon/dp/0321834577):

In a properly designed Bounded Context modifies only one Aggregate instance per transaction in all cases. What is more, we cannot correctly reason on Aggregate design without applying transactional analysis. Limiting modification to one Aggregate instance per transaction may sound overly strict. However, it is a rule of thumb and should be the goal in most cases. It addresses the very reason to use Aggregates.

If, in a single request, you need to update two Aggregates, it may just be that those two Aggregates are a single one and they need to both be updated in the same transaction. If not, you can wrap the entire request in a transaction, but we wouldn't recommend this as the main option because of the performance issues and the transaction errors involved.

If both updates on different Aggregates don't need to be wrapped into a transaction, this means we can assume some delay between one update and the other. In such a scenario, a more Domain-Driven Design approach is to use Domain Events. When doing so, the first Aggregate update will fire a Domain Event. That event will be persisted in the same transaction as the Aggregate update and then published into our message queue. Later, a worker will take the event from the queue and perform the second Aggregate update. Such an approach pushes for Eventual Consistency, reduces the size of the transaction boundaries, improves performance, and reduces transaction errors.



Sample Application Service: User and Wishes
-------------------------------------------

* * *

Now you know the basic rules for Aggregate Design.

The best way to learn about Aggregates is by seeing code. So let's consider the scenario of a web application where users can make wishes to be granted if something happens to them, similar to a will. For example, I would like to send an email to my wife explaining what to do with my GitHub account if I die in a horrible accident, or maybe I want to send an email telling her how much I loved her. The way to check that I'm still alive is to answer emails the platform sends to me. (If you want to know more about [this application](https://github.com/dddinphp/last-wishes), you can visit our [GitHub](https://github.com/dddinphp) account. So we have users and their wishes. Let's consider only one use case: "As a `User`, I want to make a Wish." How could we model this? Using good practices when designing Aggregates, let's try to push for small Aggregates. In this case, that means using two different Aggregates of one Entity each, `User` and `Wish`. For the relationship between them, we should use an identifier, such as `UserId`.

### No Invariant, Two Aggregates

We'll discuss Application Services in the following chapters, but for now, let's check different approaches for making a Wish. The first approach, particularly for a novice, would likely be something similar to this:

    class MakeWishService
    {
        private $wishRepository;
    
        public function __construct(WishRepository $wishRepository)
        {
            $this->wishRepository = $wishRepository;
        }
    
        public function execute(MakeWishRequest $request)
        {
            $userId = $request->userId();
            $address = $request->address();
            $content = $request->content();
    
            $wish = new Wish(
                $this->wishRepository->nextIdentity(),
                new UserId($userId),
                $address,
                $content
            );
    
            $this->wishRepository->add($wish);
        }
    }

This code probably allows for the best performance possible. You can almost see the `INSERT` statement behind the scenes; the minimum number of operations for such a use case is one, which is good. With the current implementation, we can create as many wishes as we want, according to the business requirements, which is also good.

However, there may be a potential issue: we can create wishes for a user who may not exist in the Domain. This is a problem, regardless of what technology we're using for persisting Aggregates. Even if we're using an in-memory implementation, we can create a Wish without its corresponding User.

This is broken business logic. Of course, this can be fixed using a foreign key in the database, from `wish (user_id)` to `user`(`id`), but what happens if we're not using a database with foreign keys? And what happens if it's a NoSQL database, such as Redis or Elasticsearch?

If we want to fix this issue so that the same code can work properly in different infrastructures, we need to check if the user exists. It's likely that the easiest approach is in the same Application Service:

    class MakeWishService
    {
        // ...
        public function execute(MakeWishRequest $request)
        {
            $userId = $request->userId();
            $address = $request->address();
            $content = $request->content();
    
            $user = $this->userRepository->ofId(new UserId($userId));
            if (null === $user) {
                throw new UserDoesNotExistException();
            }
    
            $wish = new Wish(
                $this->wishRepository->nextIdentity(),
                $user->id(),
                $address,
                $content
            );
    
            $this->wishRepository->add($wish);
        }
    }

That could work, but there's a problem performing the check in the Application Service: this check is high in the delegation chain. If any other code snippet that isn't this Application Service — such as a Domain Service or another Entity — wants to create a `Wish` for a non-existing User, it can do it. Take a look at the following code:

    // Somewhere in a Domain Service or Entity
    $nonExistingUserId = new UserId('non-existing-user-id');
        $wish = new Wish(
            $this->wishRepository->nextIdentity(),
            $nonExistingUserId,
            $address,
            $content
    );

If you've already read [Chapter 9](/chapters/09%20Factories.md), _Factories_, then you have the solution. Factories help us keep the business invariants, and that's exactly what we need here.

There's an implicit invariant saying that we're not allowed to make a wish without a valid user. Let's see how a factory can help us:

    abstract class WishService
    {
        protected $userRepository;
        protected $wishRepository;
    
        public function __construct(
            UserRepository $userRepository,
            WishRepository $wishRepository
        ) {
            $this->userRepository = $userRepository;
            $this->wishRepository = $wishRepository;
        }
    
        protected function findUserOrFail($userId)
        {
            $user = $this->userRepository->ofId(new UserId($userId));
            if (null === $user) {
                throw new UserDoesNotExistException();
            }
    
            return $user;
        }
    
        protected function findWishOrFail($wishId)
        {
            $wish = $this->wishRepository->ofId(new WishId($wishId));
            if (!$wish) {
                throw new WishDoesNotExistException();
            }
    
            return $wish;
        }
    
        protected function checkIfUserOwnsWish(User $user, Wish $wish)
        {
            if (!$wish->userId()->equals($user->id())) {
                throw new \InvalidArgumentException(
                    'User is not authorized to update this wish'
                );
            }
        }
    }
    
    class MakeWishService extends WishService
    {
        public function execute(MakeWishRequest $request)
        {
            $userId = $request->userId();
            $address = $request->address();
            $content = $request->content();
    
            $user = $this->findUserOrFail($userId);
    
            $wish = $user->makeWish(
                $this->wishRepository->nextIdentity(),
                $address,
                $content
            );
    
            $this->wishRepository->add($wish);
        }
    }

As you can see, `Users` make `Wishes`, and so does our code. `makeWish` is a Factory Method for building Wishes. The method returns a new `Wish` built with the `UserId` from the owner:

    class User
    {
        // ...
    
        /**
         * @return Wish
         */
        public function makeWish(WishId $wishId, $address, $content)
        {
            return new Wish(
                $wishId,
                $this->id(),
                $address,
                $content
            );
        }
    
        // ...
    }

Why are we returning a `Wish` and not just adding the new `Wish` to an internal collection as we would do with Doctrine? To summarize, in this scenario, `User` and `Wish` don't conform to an Aggregate because there's no true business invariant to protect. A `User` can add and remove as many `Wishes` as they want. `Wishes` and their `User` can be updated independently in the database in different transactions, if needed.

Following the rules about Aggregate Design explained before, we should aim for small Aggregates, and that's the result here. Each Entity has its own Repository. Wish references its owning `User` using Identities — `UserId` in this case. Getting all the wishes of a user can be performed by a finder in the `WishRepository` and paginated easily without any performance issues:

    interface WishRepository
    {
        /**
         * @param WishId $wishId
         *
         * @return Wish
         */
        public function ofId(WishId $wishId);
    
        /**
         * @param UserId $userId
         *
         * @return Wish[]
         */
        public function ofUserId(UserId $userId);
    
        /**
         * @param Wish $wish
         */
        public function add(Wish $wish);
    
        /**
         * @param Wish $wish
         */
        public function remove(Wish $wish);
    
        /**
         * @return WishId
         */
        public function nextIdentity();
    }

An interesting aspect of this approach is that we don't have to map the relation between `User` and `Wish` in our favorite ORM. Because we reference the `User` from the `Wish` using the `UserId`, we just need the Repositories. Let's consider how the mapping of such Entities using Doctrine might appear:

    Lw\Domain\Model\User\User:
        type: entity
        id:
            userId:
                column: id
                type: UserId
        table: user
        repositoryClass:
            Lw\Infrastructure\Domain\Model\User\DoctrineUser\Repository
        fields:
            email:
                type: string
            password:
                type: string

    Lw\Domain\Model\Wish\Wish:
        type: entity
        table: wish
        repositoryClass:
            Lw\Infrastructure\Domain\Model\Wish\DoctrineWish\Repository
        id:
            wishId:
                column: id
                type: WishId
        fields:
            address:
                type: string
            content:
                type: text
            userId:
                type: UserId
            column: user_id

No relation is defined. After making a new wish, let's write some code for updating an existing one:

    class UpdateWishService extends WishService
    {
        public function execute(UpdateWishRequest $request)
        {
            $userId = $request->userId();
            $wishId = $request->wishId();
            $email = $request->email();
            $content = $request->content();
    
            $user = $this->findUserOrFail($userId);
            $wish = $this->findWishOrFail($wishId);
            $this->checkIfUserOwnsWish($user, $wish);
    
            $wish->changeContent($content);
            $wish->changeAddress($email);
        }
    }

Because `User` and `Wish` don't form an Aggregate, in order to update the `Wish`, we need first to retrieve it using the `WishRepository`. Some extra checks ensure that only the owner can update the wish. As you may have seen, `$wish` is already an existing Entity in our Domain, so there's no need to add it back again using the Repository. However, in order to make changes durable, our ORM must be aware of the information updated and flush any remaining changes to the database after the work is done. Don't worry; we'll take a look closer at this in [Chapter 11](/chapters/11%20Application.md), _Application_. In order to complete the example, let's take a look at how to remove a wish:

    class RemoveWishService extends WishService
    {
        public function execute(RemoveWishRequest $request)
        {
            $userId = $request->userId();
            $wishId = $request->wishId();
    
            $user = $this->findUserOrFail($userId);
            $wish = $this->findWishOrFail($wishId);
            $this->checkIfUserOwnsWish($user, $wish);
    
            $this->wishRepository->remove($wish);
        }
    }

As you may have seen, you could refactor some parts of the code, such as the constructor and the ownership checks, to reuse them in both Application Services. Feel free to consider how you would do that. Last but not least, how could we get all the wishes of a specific user:

    class ViewWishesService extends WishService
    {
        /**
         * @return Wish[]
         */
        public function execute(ViewWishesRequest $request)
        { 
            $userId = $request->userId();
            $wishId = $request->wishId();
    
            $user = $this->findUserOrFail($userId);
            $wish = $this->findWishOrFail($wishId);
            $this->checkIfUserOwnsWish($user, $wish);
    
            return $this->wishRepository->ofUserId($user->id());
         }
    }

This is quite straightforward. However, we'll go deeper into how to render and return information from Application Services in the corresponding chapter. For now, returning a collection of Wishes will do the job. Let's sum up this non-Aggregate approach. We couldn't find any true business invariant to consider `User` and `Wish` as an Aggregate, which is why each of them is an Aggregate. `User` has its own `Repository`, `UserRepository`. `Wish` has its own Repository too, `WishRepository`. Each `Wish` holds a `UserId` reference to owner, `User`. Even so, we didn't require a transaction. That's the best scenario in terms of performance and scalability issues. However, life is not always so wonderful. Consider what could happen with a true business invariant.

### No More Than Three Wishes Per User

Our application is a huge success and now it's time to get some money from it. We want new users to have a maximum of three wishes available. As a user, if you want to have more wishes, you'll probably have to pay for a premium account in the future. Let's see how we could change our code to follow the new business rule about the maximum number of wishes (in this instance, don't consider the premium user).

Consider the following code for a moment. Apart from what was explained in the previous section about pushing logic into our Entities, could the following code work:

    class MakeWishService
    {
       // ...
    
        public function execute(MakeWishRequest $request)
        {
            $userId = $request->userId();
            $address = $request->email();
            $content = $request->content();
    
            $count = $this->wishRepository->numberOfWishesByUserId(
                new UserId($userId)
            );
            if ($count >= 3) {
                throw new MaxNumberOfWishesExceededException();
            }
    
            $wish = new Wish(
                $this->wishRepository->nextIdentity(),
                new UserId($userId),
                $address,
                $content
            );
    
            $this->wishRepository->add($wish);
        }
    }

It looks like it could. That was easy — probably too easy. And here we come across different problems. The first is that Application Services must coordinate but shouldn't contain business logic. Instead, a better place is to put the check for the maximum three wishes into the `User`, where we could have more control of the relationship between `User` and `Wish`. However, for the approach shown here, the code seems to work.

The second problem is that it **doesn't work under race conditions**. Forget about Domain-Driven Design for a moment. What's the problem with this code under heavy traffic? Think for a minute. Is it possible to break the rule of a `User` to have more than three wishes? Why will your QA be so happy after running some stress tests?

Your QA tries making a wish feature two times and ends up with a user with two wishes. That's correct. Your QA carries on testing the feature. Imagine for a second that they open two tabs in their browser, fill out each of the forms in each tab, and manage to submit the two buttons at the same time. Suddenly, after two requests, the user ends up with four wishes in the database. That's wrong! What happened?

Think as a debugger and consider two different requests getting the `if ($count > 3) {` line at the same time. Both of the requests will return false because the user has just two wishes. So both requests will create the `Wish` and both of the requests will add it into the database. The result is four wishes for one `User`. That's an inconsistency!

We know what you're thinking. It's because we missed putting everything into a transaction. Well, imagine that a user with id 1 already has two wishes, so there's one remaining. Two HTTP requests to create two different wishes arrive at the same time. We start one database transaction per request (we'll review how to deal with transactions and requests in [Chapter 11](/chapters/11%20Application.md), _Application_). Consider all the queries that the previous PHP code is going to run against our database. Remember that you need to disable any auto-commit flag if you're using any Visual Database Tool:

![](https://static.packt-cdn.com/products/9781787284944/graphics/Code1P.png)

![](https://static.packt-cdn.com/products/9781787284944/graphics/code2P-1.png)

How many wishes does the user with id `1` have? That's right, four. How did this happen? If you take this SQL block and execute it line by line in two different connections, you'll see how the wishes table is going to have four rows at the end of both executions. So it looks like it's not about protecting with a transaction. How could we fix this issue? As explained in the introduction, a concurrency control could help.

For those developers more advanced in database techniques, tweaking the isolation level could work. However, we consider that option too complex, as the problem could be solved with other approaches, and we're not always dealing with databases.

#### Pessimistic Concurrency Control

There's an important consideration when placing locks: any other connection trying to update or query the same data is going to hang until the lock is released. Locks can easily generate most of the performance problems. In MySQL, for example, there are different options for placing locks: explicit locking tables `UN/LOCK tables`, and locking reads `SELECT ... FOR UPDATE and SELECT ... LOCK IN SHARE MODE`.

As we already shared above in the beginning, according to the book _Elasticsearch:_ [TheDefinitive Guide](https://github.com/elastic/elasticsearch-definitive-guide/blob/master/030_Data/40_Version_control.asciidoc) by Clinton Gormley and Zachary Tong:

Widely used by relational databases, this approach assumes that conflicting changes are likely to happen and so blocks access to a resource in order to prevent conflicts. A typical example is locking a row before reading its data, ensuring that only the thread that placed the lock is able to make changes to the data in that row.

We could [`LOCK`](http://dev.mysql.com/doc/refman/5.7/en/lock-tables.html) the table, but we consider such an approach complex and risky. When using locks, you have to be careful because you can end up with situations where two threads or requests are waiting for the other one to release the lock. This is what's called a deadlock.

Based on our experience, some developers use `SELECT ... FOR UPDATE` approaches. Let's see the same two request scenarios with this option:

![](https://static.packt-cdn.com/products/9781787284944/graphics/Code3P-1.png)

As you can see, after the `COMMIT` of the first request, the count of the number of wishes of the second request is three. That's consistent, but the second request was waiting while the lock wasn't released. That means that in an environment with a lot of requests, it may generate performance issues. If the first request takes too much time to release the lock, the second request may fail due to a timeout:

    ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction

The above looks like it's a valid option, but we need to be aware of the possible performance issues. Is there any other alternative?

#### Optimistic Concurrency Control

There's another alternative: not using locks at all. Consider adding a version attribute to our Aggregates. When we persist them, the persistence engine sets 1 as the version of the persisted Aggregate. Later, we retrieve the same Aggregate and perform some changes to it. We persist the Aggregate. The persistence engine checks that the version we have is the same as the one that's currently persisted, version 1. The persistence engine persists the Aggregate with the new state and updates its version to 2. If multiple requests retrieve the same Aggregate, make some changes to it, and then try to persist it, the first request will work, and the second will experiment and error. The last request just changed an outdated version, so the persistence engine throws an error. However, the second request can try to retrieve the Aggregate again, merge the new status, attempt to perform the changes, and then persist the Aggregate.

According to [Elasticsearch: The Definitive Guide](https://github.com/elastic/elasticsearch-definitive-guide/blob/master/030_Data/40_Version_control.asciidoc):

This approach assumes that conflicts are unlikely to happen and does not block operations from being attempted. However, if the underlying data has been modified between reading and writing, the update will fail. It is then up to the application to decide how it should resolve the conflict. For instance, it could reattempt the update, using the fresh data, or it could report the situation to the user.

This idea was covered before, but it bears repeating. If you try to apply Optimistic Concurrency to this scenario where we're checking maximum wishes in the Application Service, it's not going to work. Why? We're making a new wish, so two requests would create two different wishes. How can we make it work? Well, we need an object to centralize adding the wishes. We could apply the Optimistic Concurrency trick on that object, so it looks like we need a parent object that will hold wishes. Any ideas?

To summarize, after reviewing concurrency controls, there's a pessimistic option working, but there are some concerns about performance impact. There's an optimistic option, but we need to find a parent object. Let's consider the final `MakeWishService`, but with some modifications:

    class WishAggregateService
    {
        protected $userRepository;
    
        public function __construct(UserRepository $userRepository)
        {
            $this->userRepository = $userRepository;
        }
    
        protected function findUserOrFail($userId)
        {
            $user = $this->userRepository->ofId(new UserId($userId));
            if (null === $user) {
                throw new UserDoesNotExistException();
            }
    
            return $user;
        }
    }
    
    class MakeWishService extends WishAggregateService
    {
        public function execute(MakeWishRequest $request)
        {
            $userId = $request->userId();
            $address = $request->address();
            $content = $request->content();
    
            $user = $this->findUserOrFail($userId);
    
            $user->makeWish($address, $content);
    
            // Uncomment if your ORM can not flush
            // the changes at the end of the request
            // $this->userRepository->add($user);
        }
    }

We don't pass the `WishId` because it should be something internal to the User. `makeWish` doesn't return a `Wish` either; it stores the new wish internally. After the execution of the Application Service, our ORM will flush the changes performed on the `$user` to the database. Depending on how good our ORM is, we may need to explicitly add our `User` Entity again using the Repository. What changes to the `User` class are needed? First of all, there should be a collection that could hold all the wishes inside a user:

    class User
    {
        // ...
    
        /**
         * @var ArrayCollection
         */
        protected $wishes;
    
        public function __construct(UserId $userId, $email, $password)
        {
            // ...
            $this->wishes = new ArrayCollection();
            // ...
        }
    
        // ...
    }

The `wishes` property must be initialized in the `User` constructor. We could use a plain PHP array, but we've chosen to use an `ArrayCollection`. `ArrayCollection` is a PHP array with some extra features provided by the Doctrine Common Library, and it can be used separate from the ORM. We know that some of you may think that this could be a boundary leaking and that no references to any infrastructure should be here, but we really believe that's not the case. In fact, the same code works using plain PHP arrays. Let's see how the `makeWish` implementation is affected:

    class User
    {
        // ...
    
        /**
         * @return void
         */
        public function makeWish($address, $content)
        {
            if (count($this->wishes) >= 3) {
                throw new MaxNumberOfWishesExceededException();
            }
    
            $this->wishes[] = new Wish(
                new WishId,
                $this->id(),
                $address,
                $content
            );
        }
    
        // ...
    }

So far, so good. Now, it's time to review how the rest of the operations are implemented.

### Note

****`Pushing for Eventual Consistency`**** It looks like the business doesn't want a user to have more than three wishes. That's going to force us to consider `User` as the root Aggregate with `Wish` inside. This will affect our design, performance, scalability issues, and so on. Consider what would happen if we could just let users add as many wishes as they wanted, beyond the limit. We could check who is exceeding that limit and let them know they need to purchase a premium account. Allowing a user to go over the limit and warning them by telephone afterward would be a really nice commercial strategy. That might even allow the developers on your team to avoid designing `User` and `Wish` as part of the same Aggregate, with User as its root. You've already seen the benefits of not designing a single Aggregate: maximum performance.

    class UpdateWishService extends WishAggregateService
    {
        public function execute(UpdateWishRequest $request)
        {
            $userId = $request->userId();
            $wishId = $request->wishId();
            $email = $request->email();
            $content = $request->content();
    
            $user = $this->findUserOrFail($userId);
    
            $user->updateWish(new WishId($wishId), $email, $content);
        }
    }

Because `User` and `Wish` now form an Aggregate, the `Wish` to be updated is no longer retrieved using the `WishRepository`. We fetch the user using the `UserRepository`. The operation of updating a `Wish` is performed via the root Entity, which is the `User` in this case. The `WishId` is necessary in order to identify which `Wish` we want to update:

    class User
    {
        // ...
    
        public function updateWish(WishId $wishId, $email, $content)
        {
            foreach ($this->wishes as $wish) {
                if ($wish->id()->equals($wishId)) {
                    $wish->changeContent($content);
                    $wish->changeAddress($address);
                    break;
                }
            }
        }
    }

Depending on the features of your framework, this task may or may not be cheaper to perform. Iterating through all the wishes could mean making too many queries, or even worse, fetching too many rows, which will create a huge impact on memory. In fact, that's one of the main problems of having big Aggregates. So let's consider how to remove a Wish:

    class RemoveWishService extends WishAggregateService
    {
        public function execute(RemoveWishRequest $request)
        {
            $userId = $request->userId();
            $wishId = $request->wishId();
    
            $user = $this->findUserOrFail($userId);
    
            $user->removeWish($wishId):
        }
    }

As seen before, `WishRepository` is no longer necessary. We fetch the `User` using its Repository and perform the action of removing a `Wish`. In order to remove a `Wish`, we need to remove it from the inner collection. An option would be iterating through all the elements and matching the one with the same `WishId`:

    class User
    {
        // ...
    
        public function removeWish(WishId $wishId)
        {
            foreach ($this->wishes as $k => $wish) {
                if ($wish->id()->equals($wishId)) {
                    unset($this->wishes[$k]);
                    break;
                }
            }
        }
    
        // ...
    }

That's probably the most ORM-agnostic code possible. However, behind the scenes, Doctrine is fetching all the wishes and iterating through all of them. A more specific approach to fetch only the Entity needed that isn't so ORM agnostic would be the following: Doctrine mapping must also be updated in order to make all the magic work as expected. While the Wish mapping remains the same, the `User` mapping has the new `oneToMany` unidirectional relationship:

    Lw\Domain\Model\Wish\Wish:
        type: entity
        table: lw_wish
        repositoryClass:
            Lw\Infrastructure\Domain\Model\Wish\DoctrineWish\Repository
        id:
            wishId:
                column: id
                type: WishId
         fields:
             address: 
                 type: string
             content:
                 type: text
             userId:
                 type: UserId
                 column: user_id

    Lw\Domain\Model\User\User:
        type: entity
        id:
        userId:
            column: id
            type: UserId
        table: user
        repositoryClass:
            Lw\Infrastructure\Domain\Model\User\DoctrineUser\Repository
        fields:
            email:
                type: string
            password:
                type: string
        manyToMany:
            wishes:
                orphanRemoval: true
                cascade: ["all"]
                targetEntity: Lw\Domain\Model\Wish\Wish
                joinTable:
                    name: user_wishes
                    joinColumns:
                        user_id:
                            referencedColumnName: id
                    inverseJoinColumns:
                        wish_id:
                            referencedColumnName: id
                            unique: true

In the code above, there are two important configurations: `orphanRemoval` and cascade. According to the Doctrine 2 ORM Documentation on [orphan removal](http://doctrine-orm.readthedocs.io/projects/doctrine-orm/en/latest/reference/working-with-associations.html#orphan-removal) and [transitive persistence / cascade operations](http://doctrine-orm.readthedocs.io/projects/doctrine-orm/en/latest/reference/working-with-associations.html#transitive-persistence-cascade-operations):

If an Entity of type A contains references to privately owned Entities B then if the reference from A to B is removed the entity B should also be removed, because it is not used anymore. `OrphanRemoval` works with one-to-one, one-to-many and many-to-many associations. When using the `orphanRemoval=true` option Doctrine makes the assumption that the entities are privately owned and will `NOT` be reused by other entities. If you neglect this assumption your entities will get deleted by Doctrine even if you assigned the orphaned entity to another one.

Persisting, removing, detaching, refreshing and merging individual entities can become pretty cumbersome, especially when a highly interweaved object graph is involved. Therefore Doctrine 2 provides a mechanism for transitive persistence through cascading of these operations. Each association to another entity or a collection of entities can be configured to automatically cascade certain operations. By default, no operations are cascaded.

For more information, please take a closer look at the Doctrine 2 ORM 2 Documentation on [working with associations](http://doctrine-orm.readthedocs.io/projects/doctrine-orm/en/latest/reference/working-with-associations.html).

Finally, let's see how we can get the wishes from a user:

    class ViewWishesService extends WishService
    {
        /**
         * @return Wish[]
         */
        public function execute(ViewWishesRequest $request)
        {
            return $this
                ->findUserOrFail($request->userId())
                ->wishes();
        }
    }

As mentioned before, especially in this scenario using Aggregates, returning a collection of Wishes is not the best solution. You should never return Domain Entities, as this will prevent code outside of your Application Services — such as Controllers or your UI — from unexpectedly modifying them. With Aggregates, it makes even more sense. Entities that aren't root — the ones that belong to the Aggregate but aren't root  — should appear private to others outside.

We'll go deeper into this in the [Chapter 11](/chapters/11%20Application.md), _Application_. For now, to summarize, you have different options:

*   The Application Service returns a DTO build accessing Aggregates information.
*   The Application Service returns a DTO returned by the Aggregate.
*   The Application Service uses an Output dependency where it writes the Aggregate. Such an Output dependency will handle the transformation to a DTO or other format.

### Note

Render the Number of Wishes  As an exercise, consider that we want to render the number of wishes a user has made on their account page. How would you implement this, considering User and Wish don't form an Aggregate? How would you implement it if User and Wish did form an Aggregate? Consider how Eventual Consistency could help in your solutions.



Transactions
------------

* * *

We haven't shown `beginTransaction`, `commit`, or `rollback` in any of the examples. This is because transactions are handled at Application Service level. Don't worry for now; you'll find more details about this in [Chapter 11]( /chapters/11%20Application.md), _Application_.



Wrap Up
-------

* * *

Aggregates are all about persistence and transactions. In fact, you can't design Aggregates without thinking about how they're going to be persisted. The basic rules to design proper Aggregates are: make them small, find true business invariants, push for eventual consistency using Domain Events, reference other Aggregates by Identity, and modify one Aggregate per request. Review how the code changes if two Entities form a single Aggregate or not. Use factories to enrich your Entities. Finally, relax. In most of the PHP applications we've seen, only five percent of the Entities were Aggregates formed by two Entities or more. Discuss with your workmates when designing and implementing Aggregates.
