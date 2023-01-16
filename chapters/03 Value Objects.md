Chapter 3. Value Objects
------------------------

By using the `self` keyword, we don't Value Objects are a fundamental building block of Domain-Driven Design, and they're used to model concepts of your Ubiquitous Language in code. A Value Object is not just a thing in your Domain — it measures, quantifies, or describes something. Value Objects can be seen as small, simple objects — such as money or a date range — whose equality is not based on identity, but instead on the content held.

For example, a product price could be modeled using a Value Object. In this case, it's not representing a thing, but rather a value that allows us to measure how much Money a product is worth. The memory footprint for these objects is trivial to determine (calculated by their constituent parts) and there's very little overhead. As a result, new instance creation is favored over reference reuse, even when being used to represent the same value. Equality is then checked based on the comparability of the fields of both instances.

Definition
----------

* * *

Ward Cunningham [defines](http://c2.com/cgi/wiki?ValueObject) a Value Object as:

A measure or description of something. Examples of Value Objects are things like numbers, dates, monies and strings. Usually, they are small Objects which are used quite widely. Their identity is based on their state rather than on their Object identity. This way, you can have multiple copies of the same conceptual Value Object. Every $5 note has its own identity (thanks to its serial number), but the cash economy relies on every $5 note having the same Value as every other $5 note.

Martin Fowler [defines](http://martinfowler.com/bliki/ValueObject.html) a Value Object as:

A small Object such as a Money or the date range object. Their key property is that they follow value semantics rather than reference semantics. You can usually tell them because their notion of equality isn't based on identity, instead two Value Objects are equal if all their fields are equal. Although all fields are equal, you don't need to compare all fields if a subset is unique — for example currency codes for currency objects are enough to test equality. A general heuristic is that Value Objects should be entirely immutable. If you want to change a Value Object you should replace the object with a new one and not be allowed to update the values of the value object itself — updatable value objects lead to aliasing problems.

Examples of Value Objects are numbers, text strings, dates, times, a person's full name (composed of first name, middle name, last name, and title), currencies, colors, phone numbers, and postal addresses.

> ### Note
> **`Exercise`** Try to locate more examples of potential Value Objects in your current Domain.


Value Object vs. Entity
-----------------------

* * *

Consider the following examples from [Wikipedia](http://en.wikipedia.org/wiki/Domain-driven_design#Building_blocks_of_DDD), in order to better understand the difference between Value Objects and Entities:

> *   **Value Object**: When people exchange dollar bills, they generally do not distinguish between each unique bill; they only are concerned about the face value of the dollar bill. In this context, dollar bills are Value Objects. However, the Federal Reserve may be concerned about each unique bill; in this context each bill would be an entity.
> *   **Entity**: Most airlines distinguish each seat uniquely on every flight. Each seat is an entity in this context. However, Southwest Airlines, EasyJet and Ryanair do not distinguish between every seat; all seats are the same. In this context, a seat is actually a Value Object.

> ### Note
> **`Exercise`** Think about the concept of an address (street, number, zip code, and so on). What is a possible context where an address could be modeled as an Entity and not as a Value Object? Discuss your findings with a peer.


Currency and Money Example
--------------------------

* * *

The `Currency` and `Money` Value Objects are probably the most used examples for explaining Value Objects, thanks to the [Money pattern](http://martinfowler.com/eaaCatalog/money.html). This design pattern provides a solution for modeling a problem that avoids a floating-point rounding issue, which in turn allows for deterministic calculations to be performed.

In the real world, a currency describes monetary units in the same way that meters and yards describe distance units. Each currency is represented with a three-letter uppercase ISO code:

```php
class Currency 
{ 
    private $isoCode;

    public function __construct($anIsoCode)
    {
        $this->setIsoCode($anIsoCode);
    }

    private function setIsoCode($anIsoCode)
    {
        if (!preg_match('/^[A-Z]{3}$/', $anIsoCode)) {
           throw new InvalidArgumentException();
        }

        $this->isoCode = $anIsoCode;
    }

    public function isoCode()
    {
        return $this->isoCode;
    }
}
```

One of the main goals of Value Objects is also the holy grail of Object-Oriented design: encapsulation. By following this pattern, you'll end up with a dedicated location to put all the validation, comparison logic, and behavior for a given concept.

> ### Note
> **`Extra Validations for Currency`** In the previous code example, we can build a Currency with an AAA Currency ISO code. That isn't valid at all. Write a more specific rule that will check if the ISO Code is valid. A full list of valid currency ISO codes can be found [here](http://www.xe.com/iso4217.php). If you need help, take a look at the [Money](https://github.com/moneyphp/money) packagist library.

Money is used to measure a specific amount of currency. It's modeled using an amount and a currency. Amount, in the case of the Money pattern, is implemented using an integer representation of the Currency's least-valuable fraction — For example   in the case of USD or EUR, cents.

As a bonus, you might also notice that we're using [self encapsulation](http://martinfowler.com/bliki/SelfEncapsulation.html) to set the ISO code, which centralizes changes in the Value Object itself:

```php
class Money 
{ 
    private $amount;
    private $currency;

    public function __construct($anAmount, Currency $aCurrency)
    {
        $this->setAmount($anAmount);
        $this->setCurrency($aCurrency);
    }

    private function setAmount($anAmount)
    {
        $this->amount = (int) $anAmount;
    }

    private function setCurrency(Currency $aCurrency)
    {
        $this->currency = $aCurrency;
    }

    public function amount()
    {
        return $this->amount;
    }

    public function currency()
    {
        return $this->currency;
    }
}
```

Now that you know the formal definition of Value Objects, let's dive deeper into some of the powerful features they offer.


Characteristics
---------------

* * *

While modeling a Ubiquitous Language concept in code, you should always favor Value Objects over Entities. Value Objects are easier to create, test, use, and maintain.

Keeping this in mind, you can determine whether the concept in question can be modeled as a Value Object if:

> *   It measures, quantifies, or describes a thing in the Domain
> *   It can be kept immutable
> *   It models a conceptual whole by composing related attributes as an integral unit
> *   It can be compared with others through value equality
> *   It is completely replaceable when the measurement or description changes
> *   It supplies its collaborators with side-effect-free behavior

### Measures, Quantifies, or Describes

As discussed before, a Value Object should not be considered just a _thing_ in your Domain. As a value, it measures, quantifies, or describes a concept in the Domain.

In our example, the `Currency` object describes what type of Money it is. The `Money` object measures or quantifies units of a given currency.

### Immutability

This is one of the most important aspects to grasp. Object values shouldn't be able to be altered over their lifetime. Because of this immutability, Value Objects are easy to reason and test and are free of undesired/unexpected side effects. As such, Value Objects should be created through their constructors. In order to build one, you usually pass the required primitive types or other Value Objects through this constructor.

Value Objects are always in a valid state; that's why we create them in a single atomic step. Empty constructors with multiple setters and getters move the creation responsibility to the client, resulting in the [Anemic Domain Model](http://www.martinfowler.com/bliki/AnemicDomainModel.html), which is considered an anti-pattern.

It's also good to point out that it's not recommended to hold references to Entities in your Value Objects. Entities are mutable, and holding references to them could lead to undesirable side effects occurring in the Value Object.

In languages with [method overloading](http://en.wikipedia.org/wiki/Function_overloading), such as Java, you can create multiple constructors with the same name. Each of these constructors are provided with different options to build the same type of resulting object. In PHP, we're able to provide a similar capability by way of [f](http://en.wikipedia.org/wiki/Factory_method_pattern)[actory methods](http://en.wikipedia.org/wiki/Factory_method_pattern). These specific factory methods are also known as semantic constructors. The main goal of `fromMoney` is to provide more contextual meaning than the plain constructor. More radical approaches propose to make the `__construct` method private and build every instance using a semantic constructor.

In our `Money` object, we could add some useful factory methods like the following:

```php
class Money 
{ 
    // ...
    public static function fromMoney(Money $aMoney)
    {
        return new self(
            $aMoney->amount(),
            $aMoney->currency()
        );
    }

    public static function ofCurrency(Currency $aCurrency)
    {
        return new self(0, $aCurrency);
    }
}
```

By using the `self` keyword, we don't couple the code with the class name. As such, a change to the class name or namespace won't affect these factory methods. This small implementation detail helps when refactoring the code at a later date.

> ### Note
> **`static vs. self`** Using static over self can result in undesirable issues when a Value Object inherits from another Value Object.

Due to this immutability, we must consider how to handle mutable actions that are common place in a stateful context. If we require a state change, we now have to return a brand new Value Object representation with this change. If we want to increase the amount of, for example, a `Money` Value Object, we're required to instead return a new `Money` instance with the desired modifications.

Fortunately, it's relatively simple to abide by this rule, as shown in the example below:

```php
class Money 
{ 
   // ...
    public function increaseAmountBy($anAmount)
    {
        return new self(
            $this->amount() + $anAmount,
            $this->currency()
        );
    }
}
```

The `Money` object returned by `increaseAmountBy` is different from the `Money` client object that received the method call. This can be observed in the example comparability checks below:

```php
$aMoney = new Money(100, new Currency('USD'));
$otherMoney = $aMoney->increaseAmountBy(100);

var_dump($aMoney === otherMoney); // bool(false)

$aMoney = $aMoney->increaseAmountBy(100); 
var_dump($aMoney === $otherMoney); // bool(false)
```

### Conceptual Whole

So why not just implement something similar to the following example, avoiding the need for a new Value Object class altogether?

```php
class Product 
{ 
    private id;
    private name;
    /**
     * @var int
     */
    private $amount;
    /**
     * @var string
     */
    private $currency;

    // ...
}
```

This approach has some noticeable flaws, if say, for example, you want to validate the ISO. It doesn't really make sense for the Product to be responsible for the Currency's ISO validation (thus violating the Single Responsibility Principle). This is highlighted even more so if you want to reuse the accompanying logic in other parts of your Domain (to abide by the DRY principle).

With these factors in mind, this use case is a perfect candidate for being abstracted out into a Value Object. Using this abstraction not only gives you the opportunity to group related properties together, but it also allows you to create higher-order concepts and a more concrete Ubiquitous Language.

> ### Note
> **`Exercise`**Discuss with a peer whether or not an email could be considered a Value Object. Does the context it's used in matter?

### Value Equality

As discussed at the beginning of the chapter, two Value Objects are equal if the content they measure, quantify, or describe is the same.

For example, imagine two `Money` objects representing 1 USD. Can we consider them equal? In the _real world_, are two bills of 1 USD valued the same? Of course they are. Directing our attention back to the code, the Value Objects in question refer to separate instances of `Money`. However, they both represent the same value, which makes them equal.

In regards to PHP, it's commonplace to compare two Value Objects using the `==` operator. Examining the [PHP Documentation](http://php.net/manual/en/language.oop5.object-comparison.php) definition of the operator highlights an interesting behavior:

When using the comparison operator `==`, object variables are compared in a simple manner, namely: Two object instances are equal if they have the same attributes and values, and are instances of the same class.

This behavior works in agreement with our formal definition of a Value Object. However, as an exact class match predicate is present, you should be wary when handling subtyped Value Objects.

Keeping this in mind, the even stricter `===` operator doesn't help us, unfortunately:

When using the identity operator `===`, object variables are identical if and only if they refer to the same instance of the same class.

The following example should help confirm these subtle differences:

```php
$a = new Currency('USD');
$b = new Currency('USD');

var_dump($a == $b); // bool(true) 
var_dump($a === $b); // bool(false)

$c = new Currency('EUR');

var_dump($a == $c); // bool(false)
var_dump($a === $c); // bool(false)
```

A solution is to implement a conventional equals method in each Value Object. This method is tasked with checking the type and equality of its composite attributes. Abstract data type comparability is easy to implement using the built-in type hinting in PHP. You can also use the `get_class()` function to aid in the comparability check if necessary.

The language, however, is unable to decipher what equality truly means in your Domain concept, meaning it's your responsibility to provide the answer. In order to compare the `Currency` objects, we just need to confirm that both their associated ISO codes are the same. The `===` operator does the job pretty well in this case:

```php
class Currency 
{ 
    // ...
    public function equals(Currency $currency)
    {
        return $currency->isoCode() === $this->isoCode();
    }
}
```

Because `Money` objects use `Currency` objects, the `equals` method needs to perform this comparability check, along with comparing the amounts:

```php
class Money 
{ 
    // ...
    public function equals(Money $money)
    {
        return
            $money->currency()->equals($this->currency()) &&
            $money->amount() === $this->amount();
    }
}
```

### Replaceability

Consider a `Product` Entity that contains a `Money` Value Object used to quantify its price. Additionally, consider two `Product` Entities with an identical price — for example 100 USD. This scenario could be modeled using  the two individual `Money` objects or two references pointing to a single Value Object.

Sharing the same Value Object can be risky; if one is altered, both will reflect the change. This behavior can be considered an unexpected side effect. For example, if Carlos was hired on February 20, and we know that Christian was also hired on the same day, we may set Christian's hire date to be the same instance as Carlos's. If Carlos then changes the month of his hire date to May, Christian's hire date changes too. Whether it's correct or not, it's not what people expect.

Due to the problems highlighted in this example, when holding a reference to a Value Object, it's recommended to replace the object as a whole rather than modifying its value:

```php
$this−>price = new Money(100, new Currency('USD'));
//...
$this->price = $this->price->increaseAmountBy(200);
```

This kind of behavior is similar to how basic types such as strings work in PHP. Consider the function `strtolower`. It returns a new string rather than modifying the original one. No reference is used; instead, a new value is returned.

### Side-Effect-Free Behavior

If we want to include some additional behavior — like an `add` method — in our `Money` class, it feels natural to check that the input fits any preconditions and maintains any invariance. In our case, we only wish to add monies with the same currency:

```php
class Money 
{ 
    // ...
    public function add(Money $money)
    {
        if ($money->currency() !== $this->currency()) {
            throw new InvalidArgumentException();
        }

        $this->amount += $money->amount();
    }
}
```

If the two currencies don't match, an exception is raised. Otherwise, the amounts are added. However, this code has some undesirable pitfalls. Now imagine we have a mysterious method call to `otherMethod` in our code:

```php
class Banking 
{
    public function doSomething() 
    { 
        $aMoney = new Money(100, new Currency('USD')); 

        $this->otherMethod($aMoney);//mysterious call
        // ...
    }
}
```

Everything is fine until, for some reason, we start seeing unexpected results when we're returning or finished with `otherMethod`. Suddenly, `$aMoney` no longer contains 100 USD. What happened? And what happens if `otherMethod` internally uses our previously defined `add` method? Maybe you're unaware that add mutates the state of the `Money` instance. This is what we call a side effect. You must avoid generating side effects. You must not mutate your arguments. If you do, the developer using your objects may experience strange behaviors. They'll complain, and they'll be correct.

So how can we fix this? Simple — by making sure that the Value Object remains immutable, we avoid this kind of unexpected problem. An easy solution could be returning a new instance for every potentially mutable operation, which the `add` method does:

```php
class Money 
{ 
    // ...
    public function add(Money $money)
    {
        if (!$money->currency()->equals($this->currency())) {
            throw new \InvalidArgumentException();
        }

        return new self(
            $money->amount() + $this->amount(),
            $this->currency()
        );
    }
}
```

With this simple change, immutability is guaranteed. Each time two instances of `Money` are added together, a new resulting instance is returned. Other classes can perform any number of changes without affecting the original copy. Code free of side effects is easy to understand, easy to test, and less error prone.


Basic Types
-----------

* * *

Consider the following code snippet:

```php
$a = 10;
$b = 10; 
var_dump($a == $b); 
// bool(true) 
var_dump($a === $b); 
// bool(true) 
$a = 20;
var_dump($a); 
// integer(20) 
$a = $a + 30; 
var_dump($a); 
// integer(50); 
```

Although `$a` and `$b` are different variables stored in different memory locations, when compared, they're the same. They hold the same value, so we consider them equal. You can change the value of `$a` from `10` to `20` at any time that you want, making the new value `20` and eliminating the `10`. You can replace integer values as much as you want without consideration of the previous value because you're not modifying it; you're just replacing it. If you apply any operation — such as addition (That is. `$a + $b`) — to these variables, you get another new value that can be assigned to another variable or a previously defined one. When you pass `$a` to another function, except when explicitly passed by reference, you're passing a value. It doesn't matter if `$a` gets modified within that function, because in your current code, you'll still have the original copy. Value Objects behave as basic types.


Testing Value Objects
---------------------

* * *

Value Objects are tested in the same way normal objects are. However, the immutability and side-effect-free behavior must be tested too. A solution is to create a copy of the Value Object you're testing before performing any modifications. Assert both are equal using the implemented equality check. Perform the actions you want to test and assert the results. Finally, assert that the original object and copy are still equal.

Let's put this into practice and test the side-effect-free implementation of our add method in the `Money` class:

```php
class MoneyTest extends FrameworkTestCase 
{ 
    /** 
     * @test
     */ 
    public function copiedMoneyShouldRepresentSameValue()
    { 
        $aMoney = new Money(100, new Currency('USD'));

        $copiedMoney = Money::fromMoney($aMoney);

        $this->assertTrue($aMoney->equals($copiedMoney));
    }

    /**
     * @test
     */
    public function originalMoneyShouldNotBeModifiedOnAddition()
    {
        $aMoney = new Money(100, new Currency('USD'));

        $aMoney->add(new Money(20, new Currency('USD')));

        $this->assertEquals(100, $aMoney->amount());
    }

    /**
     * @test
     */
    public function moniesShouldBeAdded()
    {
        $aMoney = new Money(100, new Currency('USD'));

        $newMoney = $aMoney->add(new Money(20, new Currency('USD')));

        $this->assertEquals(120, $newMoney->amount());
    }

    // ...
}
```


Persisting Value Objects
------------------------

* * *

Value Objects are not persisted on their own; they're typically persisted within an Aggregate. Value Objects shouldn't be persisted as complete records, though that's an option in some cases. Instead, it's best to use Embedded Value or Serialize LOB patterns. Both patterns can be used when persisting your objects with an open source ORM such as Doctrine, or with a bespoke ORM. As Value Objects are small, Embedded Value is usually the best choice because it provides an easy way to query Entities by any of the attributes the Value Object has. However, if querying by those fields isn't important to you, serialize strategies can be very easy to implement.

Consider the following `Product` Entity with string id, `name`, and `price` (`Money` Value Objects) attributes. We've intentionally decided to simplify this example, with the id being a string and not a Value Object:

```php
 class Product
 {
     private $productId;
     private $name;
     private $price;

     public function __construct(
         $aProductId,
         $aName,
         Money $aPrice
     ) {
         $this->setProductId($aProductId);
         $this->setName($aName);
         $this->setPrice($aPrice);
     }

     // ...
 }
```

Assuming you have a [Chapter 10](../chapters/10%20Repositories.md), _Repositories_ for persisting `Product` Entities, an implementation to create and persist a new `Product` could look like this:

    $product = new Product(
        $productRepository->nextIdentity(), 
        'Domain-Driven Design in PHP', 
        new Money(999, new Currency('USD')) 
    );
    $productRepository−>persist(product);

Now let's look at both the ad hoc ORM and the Doctrine implementations that could be used to persist a `Product` Entity containing Value Objects. We'll highlight the application of the Embedded Value and Serialized LOB patterns, along with the differences between persisting a single Value Object and a collection of them.

> ### Note
> **`Why Doctrine?`** The [Doctrine](http://www.doctrine-project.org/projects/orm.html) is a great ORM. It solves 80 percent of the requirements a PHP application faces. It has a great community. With a correctly tuned setup, it can perform the same or even better than a bespoke ORM (without losing maintainability). We recommend using Doctrine in most cases when dealing with Entities and business logic. It will save you a lot of time and headaches.

### Persisting Single Value Objects

Many different options are available for persisting a single Value Object. These range from using Serialize LOB or Embedded Value as mapping strategies, to using an Ad Hoc ORM or an open source alternative, such as Doctrine. We consider an Ad Hoc ORM to be a custom-built ORM that your company may have developed in order to persist Entities in a database. In our scenario, the Ad Hoc ORM code is going to be implemented using the [DBAL](http://docs.doctrine-project.org/projects/doctrine-dbal/en/latest/) library. According to the [official documentation](http://docs.doctrine-project.org/projects/doctrine-dbal/en/latest/reference/introduction.html), The **Doctrine Database Abstraction** & **Access Layer** (**DBAL**) offers a lightweight and thin runtime layer around a PDO-like API and a lot of additional, horizontal features like database schema introspection and manipulation through an OO API.

#### Embedded Value with an Ad Hoc ORM

If we're dealing with an Ad Hoc ORM using the Embedded Value pattern, we need to create a field in the Entity table for each attribute in the Value Object. In this case, two extra columns are needed when persisting a `Product` Entity — one for the amount of the Value Object, and one for its currency ISO code:

```mysql
CREATE TABLE `products`
(
    id             INT          NOT NULL,
    name           VARCHAR(255) NOT NULL,
    price_amount   INT          NOT NULL,
    price_currency VARCHAR(3)   NOT NULL
) ENGINE = InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

For persisting the Entity in the database, our [Chapter 10](../chapters/10%20Repositories.md), _Repositories_ has to map each of the fields of the Entity and the ones from the `Money` Value Object.

If you're using an `Ad hoc ORM` Repository based on DBAL—let's call it `DbalProductRepository`—you must take care of creating the `INSERT` statement, binding the parameters, and executing the statement:

```php
class DbalProductRepository
    extends DbalRepository
    implements ProductRepository
{
     public function add(Product $aProduct)
     {
         $sql = 'INSERT INTO products VALUES (?, ?, ?, ?)' ;
         $stmt = $this->connection()->prepare($sql);
         $stmt->bindValue(1, $aProduct->id());
         $stmt->bindValue(2, $aProduct->name());
         $stmt->bindValue(3, $aProduct->price()->amount());
         $stmt->bindValue(4, $aProduct->price()->currency()->isoCode());
         $stmt->execute();

       // ...
     }
 }
```

After executing this snippet of code to create a `Products` Entity and persist it into the database, each column is filled with the desired information:

```
mysql> select * from products \G
*************************** 1. row ***************************
id: 1
name: Domain-Driven Design in PHP
price_amount: 999
price_currency: USD
1 row in set (0.00 sec)
```

As you can see, you can map your Value Objects and query parameters in an `Ad hoc` manner in order to persist your Value Objects. However, everything is not as easy as it seems. Let's try to fetch the persisted Product with its associated `Money` Value Object. A common approach would be to execute a `SELECT` statement and return a new Entity:

```php
class DbalProductRepository extends DbalRepository implements ProductRepository
{
    public function productOfId($anId)
    {
        $sql = 'SELECT * FROM products WHERE id = ?';
        $stmt = $this->connection()->prepare($sql);
        $stmt->bindValue(1, $anId);
        $res = $stmt->execute();
        // ...

        return new Product(
            $row['id'],
            $row['name'],
            new Money(
                $row['price_amount'],
                new Currency($row['price_currency'])
            )
        );
    }
}
```

There are some benefits to this approach. First, you can easily read, step by step, how the persistence and subsequent creations occur. Second, you can perform queries based on any of the attributes of the Value Object. Finally, the space required to persist the Entity is just what is required — no more and no less.

However, using the ad hoc ORM approach has its drawbacks. As explained in the [Chapter 6](../chapters/06%20Domain-Events.md), _Domain-Events_, Entities (in Aggregate form) should fire an Event in the constructor if your Domain is interested in the Aggregate's creation. If you use the new operator, you'll be firing the Event as many times as the Aggregate is fetched from the database.

This is one of the reasons why Doctrine uses internal proxies and `serialize` and `unserialize` methods to reconstitute an object with its attributes in a specific state without using its constructor. An Entity should only be created with the new operator once in its lifetime:

> ### Note
> **`Constructors`**   Constructors don't need to include a parameter for each attribute in the object. Think about a blog post. A constructor may need an id and a title; however, internally it can also be setting its status attribute to draft. When publishing the post, a publish method should be called in order to alter its status accordingly and set a published date.

If your intention is still to roll out your own ORM, be ready to solve some fundamental problems such as Events, different constructors, Value Objects, lazy load relations, and so on. That's why we recommend giving Doctrine a try for Domain-Driven Design applications.

Besides, in this instance, you need to create a `DbalProduct` Entity that extends from the `Product` Entity and is able to reconstitute the Entity from the database without using the new operator, instead using a static factory method.

#### Embedded Value (Embeddables) with Doctrine >= 2.5.\*

The latest stable Doctrine release is currently _version 2.5_ and it comes with support for mapping Value Objects, thereby removing the need to do this yourself as in _Doctrine 2.4_. Since December 2015, Doctrine also has support for nested embeddables. The support is not 100 percent, but it's high enough to give it a try. In case it doesn't work for your scenario, take a look at the next section. For official documentation, check the Doctrine [Embeddables reference](http://doctrine-orm.readthedocs.org/en/latest/tutorials/embeddables.html). This option, if implemented correctly, is definitely the one we recommend most. It would be the simplest, most elegant solution, that also provides search support in your _DQL_ queries.

Because the `Product`, `Money`, and `Currency` classes have already been shown, the only thing remaining is to show the Doctrine mapping files:

```xml
<?xml version="1.0" encoding="utf-8"?>
<doctrine-mapping 
    xmlns="http://doctrine-project.org/schemas/orm/doctrine-mapping"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="
        http://doctrine-project.org/schemas/orm/doctrine-mapping
    https://raw.github.com/doctrine/doctrine2/master/doctrine-mapping.xsd">

    <entity
        name="Product"
        table="product">
        <id
            name="id"
            column="id"
            type="string"
            length="255">
            <generator strategy="NONE">
            </generator>
        </id>

        <field
            name="name"
            type="string"
            length="255"
        />

        <embedded
            name="price"
            class="Ddd\Domain\Model\Money"
        />
    </entity>
</doctrine-mapping>
```

In the product mapping, we're defining `price` as an instance variable that will hold a `Money` instance. At the same time, `Money` is designed with an amount and a `Currency` instance:

```xml
<?xml version="1.0" encoding="utf-8"?>
<doctrine-mapping
    xmlns="http://doctrine-project.org/schemas/orm/doctrine-mapping"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="
        http://doctrine-project.org/schemas/orm/doctrine-mapping
    https://raw.github.com/doctrine/doctrine2/master/doctrine-mapping.xsd">

    <embeddable
        name="Ddd\Domain\Model\Money">

        <field
            name="amount"
            type="integer"
        />
        <embedded
            name="currency"
            class="Ddd\Domain\Model\Currency"
        />
    </embeddable>
</doctrine-mapping>
```


Finally, it's time to show the Doctrine mapping for our `Currency` Value Object:

```xml
<?xml version="1.0" encoding="utf-8"?>
<doctrine-mapping
    xmlns="https://doctrine-project.org/schemas/orm/doctrine-mapping"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="
        httpS://doctrine-project.org/schemas/orm/doctrine-mapping
        https://raw.github.com/doctrine/doctrine2/master/doctrine-mapping.xsd">

    <embeddable
        name="Ddd\Domain\Model\Currency">

        <field
            name="iso"
            type="string"
            length="3"
        />
    </embeddable>
</doctrine-mapping>
```

As you can see, the above code has a standard embeddable definition with just one string field that holds the ISO code. This approach is the easiest way to use embeddables and is much more effective. By default, Doctrine names your columns by prefixing them using the Value Object name. You can change this behavior to meet your needs by changing the column-prefix attribute in the XML notation.

#### Embedded Value with Doctrine <= 2.4.\*

If you're still stuck in _Doctrine 2.4_, you may wonder what an acceptable solution for using Embedded Values with _Doctrine < 2.5_ is. We need to now surrogate all the Value Object attributes in the `Product` Entity, which means creating new artificial attributes that will hold the information of the Value Object. With this in place, we can map all those new attributes using Doctrine. Let's see what impact this has on the `Product` Entity:

```php
class Product 
{ 
    private $productId;
    private $name; 
    private $price;
    private $surrogateCurrencyIsoCode;
    private $surrogateAmount;

    public function __construct($aProductId, $aName, Money $aPrice)
    {
        $this->setProductId($aProductId);
        $this->setName($aName);
        $this->setPrice($aPrice);
    }

    private function setPrice(Money $aMoney)
    {
        $this->price = $aMoney;
        $this->surrogateAmount = $aMoney->amount();
        $this->surrogateCurrencyIsoCode =
            $aMoney->currency()->isoCode();
    }

    private function price()
    {
        if (null === $this->price) {
            $this->price = new Money(
                $this->surrogateAmount,
                new Currency($this->surrogateCurrency)
            );
        }
        return $this->price;
    }

    // ...
}
```

As you can see, there are two new attributes: one for the amount, and another for the ISO code of the currency. We've also updated the `setPrice` method in order to keep attribute consistency when setting it. On top of this, we updated the price getter in order to return the `Money` Value Object built from the new fields. Let's see how the corresponding XML Doctrine mapping should be changed:

```xml
 <?xml version="1.0" encoding="utf-8"?>
 <doctrine-mapping
    xmlns="http://doctrine-project.org/schemas/orm/doctrine-mapping"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="
        http://doctrine-project.org/schemas/orm/doctrine-mapping
    https://raw.github.com/doctrine/doctrine2/master/doctrine-mapping.xsd">

    <entity
        name="Product"
        table="product">

        <id
            name="id"
            column="id"
            type="string"
            length="255" >
            <generator strategy="NONE">
            </generator>
        </id>

       <field
           name="name"
           type="string"
           length="255"
       />

       <field
           name="surrogateAmount"
           type="integer"
           column="price_amount"
       />
       <field
           name="surrogateCurrencyIsoCode"
           type="string"
           column="price_currency"
       />
    </entity>
</doctrine-mapping>
```

> ### Note
> **`Surrogate Attributes`** These two new fields don't strictly belong to the Domain, as they don't refer to Infrastructure details. Rather, they're a necessity due to the lack of embeddable support in Doctrine. There are alternatives that can push these two attributes outside the pure Domain; however, this approach is simpler, easier, and, as a tradeoff, acceptable. There's another use of surrogate attributes in this book; you can find it in  the sub-section _Surrogate Identity_ of the section _Identity Operation_ of [Chapter 4](../chapters/04%20Entities.md)_, Entities_.

If we wanted to push these two attributes outside of the Domain, this could be achieved through the use of an [Abstract Factory](http://en.wikipedia.org/wiki/Abstract_factory_pattern). First, we need to create a new Entity, `DoctrineProduct`, in our Infrastructure folder. This Entity will extend from `Product` Entity. All surrogate fields will be placed in the new class, and methods such as price or `setPrice` should be reimplemented. We'll map Doctrine to use the new `DoctrineProduct` as opposed to the `Product` Entity.

Now we're able to fetch Entities from the database, but what about creating a new `Product`? At some point, we're required to call new `Product`, but because we need to deal with `DoctrineProduct` and we don't want our Application Services to know about Infrastructure details, we'll need to use Factories to create `Product` Entities. So, in every instance where Entity creation occurs with new, you'll instead call `createProduct` on `ProductFactory`.

There could be many additional classes required to avoid placing the surrogate attributes in the original Entity. As such, it's our recommendation to surrogate all the Value Objects to the same Entity, though this admittedly leads to a less pure solution.

### Serialized LOB and Ad Hoc ORM

If the addition of searching capabilities to the Value Objects attributes is not important, there's another pattern that can be considered: the Serialized LOB. This pattern works by serializing the whole Value Object into a string format that can easily be persisted and fetched. The most significant difference between this solution and the embedded alternative is that in the latter option, the persistence footprint requirements are reduced to a single column:

```mysql
CREATE TABLE ` products` (
    id INT NOT NULL,
    name VARCHAR( 255) NOT NULL,
    price TEXT NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

In order to persist the `Product` Entities using this approach, a change in the `DbalProductRepository` is required. The `Money` Value Object must be serialized into a string before persisting the `final` Entity:

```php
class DbalProductRepository extends DbalRepository implements ProductRepository
{ 
    public function add(Product $aProduct) 
    { 
        $sql = 'INSERT INTO products VALUES (?, ?, ?)'; 
        $stmt = this->connection()->prepare(sql);
        $stmt->bindValue(1, aProduct−>id());
        $stmt->bindValue(2, aProduct−>name());
        $stmt->bindValue(3, $this−>serialize($aProduct->price())); 

        // ...
    }

    private function serialize($object)
    {
        return serialize($object);
    }
}
```

Let's see how our Product is now represented in the database. The table column `price` is a `TEXT` type column that contains a serialization of a `Money` object representing 9.99 USD:

```
mysql > select * from products \G
*************************** 1.row***************************
id   : 1
name : Domain-Driven Design in PHP
price : O:22:"Ddd\Domain\Model\Money":2:{s:30:"Ddd\Domain\Model\\
Money amount";i :
999;s:32:"Ddd\Domain\Model\Money currency";O : 25:"Ddd\Domain\Model\\
Currency":1:{\
s:34:" Ddd\Domain\Model\Currency isoCode";s:3:"USD";}}1 row in set(\ 0.00 sec)
```

This approach does the job. However, it's not recommended due to problems occurring when refactoring classes in your code. Could you imagine the problems if we decided to rename our `Money` class? Could you imagine the changes that would be required in our database representation when moving the `Money` class from one namespace to another? Another tradeoff, as explained before, is the lack of querying capabilities. It doesn't matter whether you use Doctrine or not; writing a query to get the products cheaper than, say, 200 USD is almost impossible while using a serialization strategy.

The querying issue can only be solved by using Embedded Values. However, the serialization refactoring problems can be fixed using a specialized library for handling serialization processes.

#### Improved Serialization with JMS Serializer

The serialize/unserialize native PHP strategies have a problem when dealing with class and namespace refactoring. One alternative is to use your own serialization mechanism —  for example, concatenating the amount, a one character separator such as `|`, and the currency ISO code. However, there's another favored approach: using an open source serializer library such as [JMS Serializer](http://jmsyst.com/libs/serializer). Let's see an example of applying it when serializing a `Money` object:

```php
$myMoney = new Money(999, new Currency('USD'));

$serializer = JMS\Serializer\SerializerBuilder::create()->build(); 
$jsonData = $serializer−>serialize(myMoney, 'json');
```

In order to unserialize the object, the process is straightforward:

```php
$serializer = JMS\Serializer\SerializerBuilder::create()->build(); 
// ... 
$myMoney = $serializer−>deserialize(jsonData, 'Ddd', 'json');
```

With this example, you can refactor your `Money` class without having to update your database. JMS Serializer can be used in many different scenarios — for example, when working with REST APIs. An important feature is the ability to specify which attributes of an object should be omitted in the serialization process — for example, a password.

Check out the [Mapping Reference](http://jmsyst.com/libs/serializer/master/reference/xml_reference) and the [Cookbook](http://jmsyst.com/libs/serializer/master/cookbook) for more information. JMS Serializer is a must in any Domain-Driven Design project.

### Serialized LOB with Doctrine

In Doctrine, there are different ways of serializing objects in order to eventually persist them.

#### Doctrine Object Mapping Type

Doctrine has support for the Serialize LOB pattern. There are plenty of predefined mapping types you can use in order to match Entity attributes with database columns or even tables. One of those mappings is the object type, which maps an SQL CLOB to a PHP object using `serialize()` and `unserialize()`.

According to the [Doctrine DBAL 2 Documentation](http://doctrine-orm.readthedocs.io/projects/doctrine-dbal/en/latest/reference/types.html#object), `object` type:

Maps and converts object data based on PHP serialization. If you need to store an exact representation of your object data, you should consider using this type as it uses serialization to represent an exact copy of your object as string in the database. Values retrieved from the database are always converted to PHP's object type using unserialization or null if no data is present.

This type will always be mapped to the database vendor's text type internally as there is no way of storing a PHP object representation natively in the database. Furthermore this type requires a SQL column comment hint so that it can be reverse engineered from the database. Doctrine cannot correctly map back this type correctly using vendors that do not support column comments, and will instead fall back to the text type instead.

Because the built-in text type of `PostgreSQL` does not support NULL bytes, the object type will result in unserialization errors. A workaround to this problem is to `serialize()/unserialize()` and `base64_encode()/base64_decode()` PHP objects and store them into a text field manually.

Let's look at a possible XML mapping for the Product Entity by using the object type:

```xml
<?xml version="1.0" encoding="utf-8"?>
<doctrine-mapping
    xmlns="http://doctrine-project.org/schemas/orm/doctrine-mapping"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="
        http://doctrine-project.org/schemas/orm/doctrine-mapping
    https://raw.github.com/doctrine/doctrine2/master/doctrine-mapping.xsd">

    <entity
        name="Product"
        table="products">

        <id
            name="id"
            column="id"
            type="string"
            length="255">
            <generator strategy="NONE">
            </generator>
        </id>
        <field
            name="name"
            type="string"
            length="255"
        />
        <field
            name="price"
            type="object"
        />
    </entity>
</doctrine-mapping>
```


The key addition is `type="object"`, which tells Doctrine that we're going to be using an object mapping. Let's see how we could create and persist a `Product` Entity using Doctrine:

```php
// ... 
$em−>persist($product); 
$em−>flush($product);
```

Let's check that if we now fetch our `Product` Entity from the database, it's returned in an expected state:

```php
// ...
$repository = $em->getRepository('Ddd\\Domain\\Model\\Product');
$item = $repository->find(1);
var_dump($item);

/*
class Ddd\Domain\Model\Product#177 (3) {
    private $productId => int(1)
    private $name => string(41) "Domain-Driven Design in PHP"
    private $money => class Ddd\Domain\Model\Money#174 (2) {
        private $amount => string(3) "100"
        private $currency => class Ddd\Domain\Model\Currency#175 (1){
            private $isoCode => string(3) "USD"
        }
    }
}
* /
```


Last but not least, the [Doctrine DBAL 2 Documentation](http://docs.doctrine-project.org/projects/doctrine-orm/en/latest/reference/basic-mapping.html#doctrine-mapping-types) states that:

Object types are compared by reference, not by value. Doctrine updates this value if the reference changes and therefore behaves as if these objects are immutable value objects.

This approach suffers from the same refactoring issues as the Ad hoc ORM did. The object mapping type is internally using `serialize/unserialize`. What about instead using our own serialization?

#### Doctrine Custom Types

Another option is to handle the Value Object persistence using a Doctrine Custom Type. A Custom Type adds a new mapping type to Doctrine — one that describes a custom transformation between an Entity field and the database representation, in order to persist the former.

As the [Doctrine DBAL 2 Documentation](http://doctrine-orm.readthedocs.io/projects/doctrine-dbal/en/latest/reference/types.html#custom-mapping-types) explains:

Just redefining how database types are mapped to all the existing Doctrine types is not at all that useful. You can define your own Doctrine Mapping Types by extending `Doctrine\DBAL\Types\Type`. You are required to implement 4 different methods to get this working.

With the object type, the serialization step includes information, such as the class, that makes it quite difficult to safely refactor our code.

Let's try to improve on this solution. Think about a custom serialization process that could solve the problem.

One such way could be to persist the `Money` Value Object as a string in the database encoded in `amount|isoCode` format:

```php
use Ddd\Domain\Model\Currency;
use Ddd\Domain\Model\Money;
use Doctrine\DBAL\Types\TextType;
use Doctrine\DBAL\Platforms\AbstractPlatform;

class MoneyType extends TextType 
{ 
    const MONEY = 'money';

    public function convertToPHPValue(
        $value,
        AbstractPlatform $platform
    ) {
        $value = parent::convertToPHPValue($value, $platform);
        $value = explode('|', $value);
        return new Money(
            $value[0],
            new Currency($value[1])
        );
    }

    public function convertToDatabaseValue(
        $value,
        AbstractPlatform $platform
    ) {
        return implode(
           '|',
           [
               $value->amount(),
               $value->currency()->isoCode()
           ]
        );
    }

    public function getName()
    {
        return self::MONEY;
    }
}
```

Using Doctrine, you're required to register all Custom Types. It's common to use an `EntityManagerFactory` that centralizes this `EntityManager` creation.

Alternatively, you could perform this step by bootstrapping your application:

```php
use Doctrine\DBAL\Types\Type;
use Doctrine\ORM\EntityManager;
use Doctrine\ORM\Tools\Setup;

class EntityManagerFactory 
{ 
    public function build() 
    { 
        Type::addType( 
            'money',
            'Ddd\Infrastructure\Persistence\Doctrine\Type\MoneyType' 
        );

        return EntityManager::create(
            [
                'driver' => 'pdo_mysql',
                'user' => 'root',
                'password' => '',
                'dbname' => 'ddd',
            ],
            Setup::createXMLMetadataConfiguration(
                [__DIR__.'/config'],
                true
            )
        );
    }
}
```

Now we need to specify in the mapping that we want to use our Custom Type:

```xml
<?xml version = "1.0" encoding = "utf-8"?>
<doctrine-mapping>
    <entity
        name = "Product"
        table = "product">

        <!-- ... -->
        <field
            name = "price"
            type = "money"
        />
    </entity>
</doctrine-mapping>
```

> ### Note
> **`Why Use XML Mapping?`** Thanks to the XSD schema validation in the headers of the XML mapping file, many **Integrated Development Environment** (**IDEs**) setups provide auto-complete functionality for all the elements and attributes present in the mapping definition. However, in other parts of the book, we use YAML to show a different syntax.

Let's check the database to see how the price was persisted using this approach:

```
mysql> select * from products \G
*************************** 1. row***************************
id: 1
name: Domain-Driven Design in PHP
price: 999|USD
1 row in set (0.00 sec)
```

This approach is an improvement on the one before in terms of future refactoring. However, searching capabilities remain limited due to the format of the column. With the Doctrine Custom types, you can improve the situation a little, but it's still not the best option for building your DQL queries. See [Doctrine Custom Mapping Types](http://doctrine-orm.readthedocs.org/en/latest/cookbook/custom-mapping-types.html) for more information.

> ### Note
> **`Time to Discuss`** Think about and discuss with a peer how would you create a Doctrine Custom Type using JMS to `serialize` and `unserialize` a Value Object.

#### Persisting a Collection of Value Objects

Imagine that we'd now like to add a collection of prices to be persisted to our `Product` Entity. These prices could represent the different prices the product has borne throughout its lifetime or the product price in different currencies. This could be named `HistoricalPrice`, as shown below:

```php
class HistoricalProduct extends Product 
{ 
    /** 
     * @var Money[] 
     */ 
    protected $prices;

    public function __construct(
        $aProductId, 
        $aName, 
        Money $aPrice,
        array $somePrices
    ){
        parent::__construct($aProductId, $aName, $aPrice);
        $this->setPrices($somePrices);
    }

    private function setPrices(array $somePrices)
    {
        $this->prices = $somePrices;
    }

    public function prices()
    {
        return $this->prices;
    }
}
```

`HistoricalProduct` extends from `Product`, so it inherits the same behavior, plus the price collection functionality.

As in the previous sections, serialization is a plausible approach if you don't care about querying capabilities. However, Embedded Values are a possibility if we know exactly how many prices we want to persist. But what happens if we want to persist an undetermined collection of historical prices?

#### Collection Serialized into a Single Column

Serializing a collection of Value Objects into a single column is most likely the easiest solution. Everything that was previously explained in the section about persisting a single Value Object applies in this situation. With Doctrine, you can use an Object or Custom Type — with some additional considerations to bear in mind: Value Objects should be small in size, but if you wish to persist a large collection, be sure to consider the maximum column length and the maximum row width that your database engine can handle.

> ### Note
> **`Exercise`** Come up with both `Doctrine` Object Type and `Doctrine Custom` Type implementation strategies for persisting a Product with different prices.

#### Collection Backed by a Join Table

In case you want to persist and query an Entity by its related Value Objects, you have the choice to persist the Value Objects as Entities. In terms of the Domain, those objects would still be Value Objects, but we'll need to give them an id and set them up with a one-to-many/one-to-one relation with the owner, a real Entity. To summarize, your ORM handles the collection of Value Objects as Entities, but in your Domain, they're still treated as Value Objects.

The main idea behind the Join Table strategy is to create a table that connects the owner Entity and its Value Objects. Let's see a database representation:

```mysql
CREATE TABLE ` historical_products`
(
    `id`             char(36) COLLATE utf8mb4_unicode_ci     NOT NULL,
    `name`           varchar(255) COLLATE utf8mb4_unicode_ci NOT NULL,
    `price_amount`   int(11)                                 NOT NULL,
    `price_currency` char(3) COLLATE utf8mb4_unicode_ci      NOT NULL,
    PRIMARY KEY (`id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4
  COLLATE = utf8mb4_unicode_ci;
```

The `historical_products` table will look the same as products. Remember that `HistoricalProduct` extends `Product` Entity in order to easily show how to deal with persisting a collection. A new prices table is now required in order to persist all the different `Money` Value Objects that a `Product` Entity can handle:

```mysql
CREATE TABLE `prices`
(
    `id`       int(11)                            NOT NULL AUTO_INCREMENT,
    `amount`   int(11)                            NOT NULL,
    `currency` char(3) COLLATE utf8mb4_unicode_ci NOT NULL,
    PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```


Finally, a table that relates products and prices is needed:

```mysql
CREATE TABLE `products_prices`
(
    `product_id` char(36) COLLATE utf8mb4_unicode_ci NOT NULL,
    `price_id`   int(11)                             NOT NULL,
    PRIMARY KEY (`product_id`, `price_id`),
    UNIQUE KEY `UNIQ_62F8E673D614C7E7` (`price_id`),
    KEY `IDX_62F8E6734584665A` (`product_id`),
    CONSTRAINT `FK_62F8E6734584665A` 
        FOREIGN KEY (`product_id`) REFERENCES `historical_products` (`id`),
    CONSTRAINT `FK_62F8E673D614C7E7` 
        FOREIGN KEY (`price_id`) REFERENCES `prices` (`id`)
) ENGINE = InnoDB
  DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

##### Collection Backed by a Join Table with Doctrine

Doctrine requires that all database Entities have a unique identity. Because we want to persist `Money` Value Objects, we need to then add an artificial identity so Doctrine can handle its persistence. There are two options: including the surrogate identity in the `Money` Value Object, or placing it in an extended class.

The issue with the first option is that the new identity is only required due to the Database persistence layer. This identity is not part of the Domain.

An issue with the second option is the amount of alterations required in order to avoid this so-called boundary leak. With a class extension, creating new instances of the `Money` Value Object class from any Domain Object isn't recommended, as it would break the Inversion Principle. The solution is to again create a `Money` Factory that would need to be passed into Application Services and any other Domain Objects.

In this scenario, we recommend using the first option. Let's review the changes required to implement it in the `Money` Value Object:

```php
class Money 
{ 
    private $amount;
    private $currency;
    private $surrogateId;
    private $surrogateCurrencyIsoCode;

    public function __construct($amount, Currency $currency)
    {
        $this->setAmount($amount);
        $this->setCurrency($currency);
    }

    private function setAmount($amount)
    {
        $this->amount = $amount;
    }

    private function setCurrency(Currency $currency)
    {
        $this->currency = $currency;
        $this->surrogateCurrencyIsoCode =
            $currency->isoCode();  
    }

    public function currency()
    {
       if (null === $this->currency) {
           $this->currency = new Currency(
               $this->surrogateCurrencyIsoCode
           );
       }
       return $this->currency;
    } 

    public function amount()
    {
        return $this->amount;
    }

    public function equals(Money $aMoney)
    {
        return
            $this->amount() === $aMoney->amount() &&
            $this->currency()->equals($this->currency());
    }
}
```

As seen above, two new attributes have been added. The first one, `surrogateId`, is not used by our Domain, but it's required for the persistence Infrastructure to persist this Value Object as an Entity in our Database. The second one, `surrogateCurrencyIsoCode`, holds the ISO code for the currency. Using these new attributes, it's really easy to map our Value Object with Doctrine.

The `Money` mapping is quite straightforward:

```xml
<?xml version = "1.0" encoding = "utf-8"?>
<doctrine-mapping
    xmlns="http://doctrine-project.org/schemas/orm/doctrine-mapping"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="
        http://doctrine-project.org/schemas/orm/doctrine-mapping
    https://raw.github.com/doctrine/doctrine2/master/doctrine-mapping.xsd">

    <entity
        name="Ddd\Domain\Model\Money"
        table="prices">

        <id
            name="surrogateId"
            type="integer"
            column="id">
            <generator
                strategy="AUTO">
            </generator>

        </id>
        <field
            name="amount"
            type="integer"
            column="amount"
        />
        <field
            name="surrogateCurrencyIsoCode"
            type="string"
            column="currency"
        />
    </entity>
</doctrine-mapping>
```

Using Doctrine, the `HistoricalProduct` Entity would have following mapping:

```xml
<?xml version="1.0" encoding="utf-8"?>
<doctrine-mapping 
    xmlns="http://doctrine-project.org/schemas/orm/doctrine-mapping"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="
        http://doctrine-project.org/schemas/orm/doctrine-mapping
    https://raw.github.com/doctrine/doctrine2/master/doctrine-mapping.xsd">

    <entity
        name="Ddd\Domain\Model\HistoricalProduct" 
        table="historical_products"
        repository-class="
            Ddd\Infrastructure\Domain\Model\DoctrineHistoricalProductRepository
        ">
        <many-to-many
            field="prices"
            target-entity="Ddd\Domain\Model\Money">

            <cascade>
                <cascade-all/>
            </cascade>

            <join-table
                name="products_prices">
                <join-columns>
                    <join-column
                        name="product_id"
                        referenced-column-name="id"
                    />
                </join-columns>
                <inverse-join-columns>
                    <join-column
                        name="price_id"
                        referenced-column-name="id"
                        unique="true"
                    />
                </inverse-join-columns>
            </join-table>
        </many-to-many>
    </entity>
</doctrine-mapping>
```

##### Collection Backed by a Join Table with an Ad Hoc ORM

It's possible to do the same with an Ad hoc ORM, where Cascade `INSERTS` and `JOIN` queries are required. It's important to be careful about how the removal of Value Objects is handled, in order to not leave orphan  the `Money` Value Objects.

> ### Note
> **`Exercise`** Think up a solution for `DbalHistoricalRepository` that would handle the persist method.

#### Collection Backed by a Database Entity

Database Entity is the same solution as Join Table, with the addition of the Value Object that's only managed by the owner Entity. In the current scenario, consider that the `Money` Value Object is only used by the `HistoricalProduct` Entity; a Join Table would be overly complex. So the same result could be achieved using a one-to-many database relation.

> ### Note
> **`Exercise`** Think of the mapping required between `HistoricalProduct` and `Money` if a Database Entity approach is used.

### NoSQL

What about NoSQL mechanisms such as _Redis_, _MongoDB_, or _CouchDB_? Unfortunately, you can't escape these problems. In order to persist an Aggregate using _Redis_, you need to serialize it into a string before setting the value. If you use PHP `serialize`/`unserialize` methods, you'll face namespace or class name refactoring issues again. If you choose to serialize in a custom way (JSON, custom string, and so on.), you'll be required to again rebuild the Value Object during _Redis_ retrieval.

#### PostgreSQL JSONB and MySQL JSON Type

If our database engine would allow us to not only use the Serialized LOB strategy but also search based on its value, we would have the best of both approaches. Well, good news: now you _can_ do this. As of _PostgreSQL version 9.4_, support for [JSONB](http://www.postgresql.org/docs/9.4/static/functions-json.html) has been added. Value Objects can be persisted as JSON serializations and subsequently queried within this JSON serialization.

MySQL has done the same. As of _MySQL 5.7.8_, MySQL supports a native JSON data type that enables efficient access to data in **JSON** (**JavaScript Object Notation**) documents. According to the [MySQL 5.7 Reference Manual](https://dev.mysql.com/doc/refman/5.7/en/json.html), the JSON data type provides these advantages over storing JSON-format strings in a string column:

*   Automatic validation of JSON documents stored in JSON columns. Invalid documents produce an error.
*   Optimized storage format. JSON documents stored in JSON columns are converted to an internal format that permits quick read access to document elements. When the server later must read a JSON value stored in this binary format, the value need not be parsed from a text representation. The binary format is structured to enable the server to look up subobjects or nested values directly by key or array index without reading all values before or after them in the document.

If Relational Databases add support for document and nested document searches with high performance and with all the benefits of an **Atomicity**, **Consistency**, **Isolation**, **Durability**(**ACID**) philosophy, it could save a lot of complexity in many projects.


Security
--------

* * *

Another interesting detail of modeling your Domain concepts using Value Objects is regarding its security benefits. Consider an application within the context of selling flight tickets. If you deal with International Air Transport Association airport codes, also known as [IATA codes](https://en.wikipedia.org/wiki/International_Air_Transport_Association_airport_code), you can decide to use a string or model the concept using a Value Object. If you choose to go with the string, think about all the places where you'll be checking that the string is a valid IATA code. What's the chance you forget somewhere important? On the other hand, think about trying to instantiate new `IATA("BCN'; DROP TABLE users;--")`. If you centralize the _guards_ in the constructor and then pass an IATA Value Object into your model, avoiding SQL Injections or similar attacks gets easier.

If you want to know more about the security side of Domain-Driven Design, you can follow [Dan Bergh Johnsson](https://twitter.com/danbjson) or read his [blog](http://dearjunior.blogspot.com.es/search/label/domain%20driven%20security).
