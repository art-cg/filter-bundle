Filter logic for API Platform
=============================
Combines existing API Platform ORM Filters with AND, OR and NOT according to client request.
- supports nested logic (parentheses)
- supports multiple criteria for the same property
- existing requests keep working unmodified if not using "and", "or" or "not" as query parameters

Usage
-----
Once the FilterLogic class and service configuration have been installed in you app,
just add it as the last ApiFilter annotation. Then make a request with filter parameters nested in and/or in the query string.

For example if you have:
```php8
#[ApiResource]
#[ApiFilter(SearchFilter::class, properties: ['id' => 'exact', 'price' => 'exact', 'description' => 'partial'])]
#[ApiFilter(FilterLogic::class)]
class Offer {
// ...
}
```
Callling the collection get operation with:
```uri
/offers/?or[price]=10&or[description]=shirt
```
will return all offers with a price being exactly 10 OR a description containing the word "shirt".

NOT expressions are combined like the other expressions trough the compound logic (and, or) they are nested in:  
```uri
/offers/?or[not][price]=10&or[not][description]=shirt 
```
will return (all offers with a price not being 10) OR (a description NOT containing the word "shirt").
If they are not nested in compound logic AND is used:

```uri
/offers/?price=10&not[description]=shirt
```
will return all offers with a price being exactly 10 AND a description NOT containing the word "shirt".
This is different from the other expressions which are combined by the filters themselves.

You can have nested logic and multiple criteria for the same property like this:
```uri
/offers/?and[price]=10&and[or][][description]=shirt&and[or][][description]=cotton
```
The api will return all offers with a price being exactly 10 AND (a description containing the word "shirt" OR the word "cotton").
Because of the nesting of or the criteria for the description are combined together through
AND with the criterium for price, which must allways be true while only one of the
criteria for the desciption needs to be true for an order to be returned.

You can in/exclude filters by class name by configuring classExp. For example:
```php docblock
* @ApiFilter(FilterLogic::class, arguments={"classExp"="/ApiPlatform\\Core\\Bridge\\Doctrine\\Orm\\Filter\\+/"})
```
will only apply API Platform ORM Filters in logic context.

Installation
------------
With composer:
```shell
composer require metaclass-nl/filter-bundle "dev-master"
```

Then add the bundle to your api config/bundles.php:
```php
    // (...)
    Metaclass\FilterBundle\MetaclassFilterBundle::class => ['all' => true],
];
```

Nested properties workaround
----------------------------
The built-in filters of Api Platform normally generate INNER JOINs. As a result
combining them with OR may not produce results as expected for properties
nested over nullable and to many associations, , see [this issue](https://github.com/metaclass-nl/filter-bundle/issues/2).

As a workaround FilterLogic can convert all inner joins into left joins:
```php8
#[ApiResource]
#[ApiFilter(SearchFilter::class, properties: ['title' => 'partial', 'keywords.word' => 'exact'])]
#[ApiFilter(FilterLogic::class, arguments: ['innerJoinsLeft' => true])]
class Article {
// ...
}
```

In case you do not like FilterLogic messing with the joins you can make
the built-in filters of Api Platform generate left joins themselves by first adding
a left join and removing it later:
```php8
#[ApiResource]
#[ApiFilter(AddFakeLeftJoin::class)]
#[ApiFilter(SearchFilter::class, properties: ['title' => 'partial', 'keywords.word' => 'exact'])]
#[ApiFilter(FilterLogic::class)]
#[ApiFilter(RemoveFakeLeftJoin::class)]
class Article {
// ...
}
```

With one fo these workarounds the following will find Articles whose title contains 'pro'
as well as those whose keywords contain one whose word is 'php'.
```uri
/articles/?or[title]=pro&or[keywords.word]=php
```
Without a workaround Articles without any keywords will not be found,
even if their titles contain 'pro'.

Both workarounds do change the behavior of ExistsFilter =false with nested properties.
Normally this filter only finds entities that reference at least one entity
whose nested property contains NULL, but with left joins it will also find entities
whose reference itself is empty or NULL. This does break backward compatibility.
This can be solved by extending ExistsFilter, but that is not included
in this Bundle because IMHO the old behavior is not like one would expect given
the semantics of "exists" and therefore should be considered a bug unless it is
documented explicitly to be intentional.

Limitations
-----------
Combining filters through OR and nested logic may be a harder task for your
database and require different indexes. Except for small tables performance
testing and analysis is advisable prior to deployment.  

Works with built in filters of Api Platform, except for DateFilter 
with EXCLUDE_NULL. A DateFilter subclass is provided to correct this. However,
the built in filters of Api Platform IMHO contain a bug with respect to the JOINs 
they generate. 
As a result, combining them with OR does not work as expected with properties
nested over to-many and nullable associations. Workarounds are provided, but they
do change the behavior of ExistsFilter =false.

Assumes that filters create semantically complete expressions in the sense that
expressions added to the QueryBundle through ::andWhere or ::orWhere do not depend
on one another so that the intended logic is not compromised if they are recombined
with the others by either Doctrine\ORM\Query\Expr\Andx or Doctrine\ORM\Query\Expr\Orx.

Only works with filters implementing ContextAwareFilterInterface, other filters
are ignoored.

May Fail if a filter uses QueryBuilder::where or ::add. You are advised to check the code of all custom and third party Filters and
not to combine those that use QueryBuilder::where or ::add with FilterLogic
or that produce complex logic that is not semantically complete. For an
example of semantically complete and incomplete expressions see [DateFilterTest](./tests/Filter/DateFilterTest.php).

Credits and License
-------------------
Copyright (c) [MetaClass](https://www.metaclass.nl/), Groningen, 2021. [MetaClass](https://www.metaclass.nl/) offers software development and support in the Netherlands for Symfony, API Platform, React.js and Next.js

[MIT License](./LICENSE).

[API Platform](https://api-platform.com/) is a product of [Les-Tilleuls.coop](https://les-tilleuls.coop)
created by [Kévin Dunglas](https://dunglas.fr).

