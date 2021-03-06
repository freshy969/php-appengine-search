[![Build Status](https://api.travis-ci.org/tomwalder/php-appengine-search.svg)](https://travis-ci.org/tomwalder/php-appengine-search)
[![Coverage Status](https://coveralls.io/repos/tomwalder/php-appengine-search/badge.svg)](https://coveralls.io/r/tomwalder/php-appengine-search)

# Full Text Search for PHP on Google App Engine #

This library provides native PHP access to the Google App Engine Search API.

At the time of writing there is no off-the-shelf way to access the Google App Engine full text search API from the PHP runtime.

Generally this means developers cannot access the service without using [Python/Java/Go proxy modules](https://github.com/tomwalder/phpne14-text-search) - which adds complexity, another language, additional potential points of failure and performance impact.

**ALPHA** This library is in the very early stages of development. Do not use it in production. It will change.

## Table of Contents ##

- [Examples](#examples)
- [Install with Composer](#install-with-composer)
- [Queries](#queries)
- [Geo Queries](#distance-from)
- [Autocomplete](#autocomplete)
- [Creating Documents](#creating-documents) - includes location (Geopoint) and Dates
- [Facets](#facets)
- [Deleting Documents](#deleting-documents)
- [Local Development](#local-development-environment)
- [Best Practice, Free Quotas, Costs](#best-practice-free-quotas-costs)
- [Google Software](#google-software)

## Examples ##

I find examples a great way to decide if I want to even try out a library, so here's a couple for you. 

```php
// Schema describing a book
$obj_schema = (new \Search\Schema())
    ->addText('title')
    ->addText('author')
    ->addAtom('isbn')
    ->addNumber('price');
        
// Create and populate a document
$obj_book = $obj_schema->createDocument([
    'title' => 'The Merchant of Venice',
    'author' => 'William Shakespeare',
    'isbn' => '1840224312',
    'price' => 11.99
]);
    
// Write it to the Index
$obj_index = new \Search\Index('library');
$obj_index->put($obj_book);
```

In this example, I've used the [Alternative Array Syntax](#alternative-array-syntax) for creating Documents - but you can also do it like this:

```php
$obj_book = $obj_schema->createDocument();
$obj_book->title = 'Romeo and Juliet';
$obj_book->author = 'William Shakespeare';
$obj_book->isbn = '1840224339';
$obj_book->price = 9.99;
```

Now let's do a simple search and display the output

```php
$obj_index = new \Search\Index('library');
$obj_response = $obj_index->search('romeo');
foreach($obj_response->results as $obj_result) {
    echo "Title: {$obj_result->doc->title}, ISBN: {$obj_result->doc->isbn} <br />", PHP_EOL;
}
```

### Demo Application ###

Search pubs!

Application: http://pub-search.appspot.com/

Code: https://github.com/tomwalder/pub-search

## Getting Started ##

### Install with Composer ###

To install using Composer, use this require line in your `composer.json` for bleeding-edge features, dev-master

`"tomwalder/php-appengine-search": "v0.0.4-alpha"`

Or, if you're using the command line:

`composer require tomwalder/php-appengine-search`

You may need `minimum-stability: dev`

# Queries #

You can supply a simple query string to `Index::search`

```php
$obj_index->search('romeo');
```

For more control and options, you can supply a `Query` object

```php
$obj_query = (new \Search\Query($str_query))
   ->fields(['isbn', 'price'])
   ->limit(10)
   ->sort('price');
$obj_response = $obj_index->search($obj_query);
```

## Query Strings ##

Some simple, valid query strings:
- `price:2.99`
- `romeo`
- `dob:2015-01-01`
- `dob < 2000-01-01`
- `tom AND age:36`

For *much* more information, see the Python reference docs: https://cloud.google.com/appengine/docs/python/search/query_strings 

## Sorting ##

```php
$obj_query->sort('price');
```

```php
$obj_query->sort('price', Query::ASC);
```

## Limits & Offsets ##

```php
$obj_query->limit(10);
```

```php
$obj_query->offset(5);
```

## Return Fields ##

```php
$obj_query->fields(['isbn', 'price']);
```

## Expressions ##

The library supports requesting arbitrary expressions in the results.

```php
$obj_query->expression('euros', 'gbp * 1.45']);
```

These can be accessed from the `Document::getExpression()` method on the resulting documents, like this:

```php
$obj_doc->getExpression('euros');
```

## Get Document by ID ##

You can fetch a single document from an index directly, by it's unique Doc ID:

```php
$obj_index->get('some-document-id-here');
```

## Scoring ##

You can enable the MatchScorer by calling the `Query::score` method.

If you do this, each document in the result set will be scored by the Search API "according to search term frequency" - Google.

Without it, documents will all have a score of 0.

```php
$obj_query->score();
```

And the results...

```php
foreach($obj_response->results as $obj_result) {
    echo $obj_result->score, '<br />'; // Score will be a float
}
```

### Multiple Sorts and Scoring ###

If you apply `score()` and `sort()` you may be wasting cycles and costing money. Only score documents when you intend to sort by score.

If you need to mix sorting of score and another field, you can use the magic field name `_score` like this - here we sort by price then score, so records with the same price are sorted by their score.

```php
$obj_query->score()->sort('price')->sort('_score');
```

# Helper Queries & Tools #

## Distance From ##

A common use case is searching for documents that have a Geopoint field, based on their distance from a known Geopoint. e.g. "Find pubs near me"

There is a helper method to do this for you, and it also returns the distance in meters in the response.

```php
$obj_query->sortByDistance('location', [53.4653381,-2.2483717]);
```

This will return results, nearest first to the supplied Lat/Lon, and there will be an expression returned for the distance itself - prefixed with `distance_from_`:

```php
$obj_result->doc->getExpression('distance_from_location');
```

## Autocomplete ##

Autocomplete is one of the most desired and useful features of a search solution.

This can be implemented fairly easily with the Google App Engine Search API, **with a little slight of hand!** 

The Search API does not natively support "edge n-gram" tokenisation (which is what we need for autocomplete!).

So, you can do this with the library - when creating documents, set a second text field with the output from the included `Tokenizer::edgeNGram` function

```php
$obj_tkzr = new \Search\Tokenizer();
$obj_schema->createDocument([
    'name' => $str_name,
    'name_ngram' => $obj_tkzr->edgeNGram($str_name),
]);
```

Then you can run autocomplete queries easily like this:

```php
$obj_response = $obj_index->search((new \Search\Query('name_ngram:' . $str_query)));
```

You can see a full demo application using this in my "pub search" demo app

# Creating Documents #

## Schemas & Field Types ##

As per the Python docs, the available field types are

- **Atom** - an indivisible character string
- **Text** - a plain text string that can be searched word by word
- **HTML** - a string that contains HTML markup tags, only the text outside the markup tags can be searched
- **Number** - a floating point number
- **Date** - a date with year/month/day and optional time
- **Geopoint** - latitude and longitude coordinates

### Dates ###

We support `DateTime` objects or date strings in the format `YYYY-MM-DD` (PHP `date('Y-m-d')`)

```php
$obj_person_schema = (new \Search\Schema())
    ->addText('name')
    ->addDate('dob');

$obj_person = $obj_person_schema->createDocument([
    'name' => 'Marty McFly',
    'dob' => new DateTime()
]);
```

### Geopoints - Location Data ###

Create an entry with a Geopoint field

```php
$obj_pub_schema = (new \Search\Schema())
    ->addText('name')
    ->addGeopoint('where')
    ->addNumber('rating');

$obj_pub = $obj_pub_schema->createDocument([
    'name' => 'Kim by the Sea',
    'where' => [53.4653381, -2.2483717],
    'rating' => 3
]);
```

## Batch Inserts ##

It's more efficient to insert in batches if you have multiple documents. Up to 200 documents can be inserted at once.

Just pass an array of Document objects into the `Index::put()` method, like this:

```php
$obj_index = new \Search\Index('library');
$obj_index->put([$obj_book1, $obj_book2, $obj_book3]);
```

## Alternative Array Syntax ##

There is an alternative to directly constructing a new `Search\Document` and setting it's member data, which is to use the `Search\Schema::createDocument` factory method as follows.

```php
$obj_book = $obj_schema->createDocument([
    'title' => 'The Merchant of Venice',
    'author' => 'William Shakespeare',
    'isbn' => '1840224312',
    'price' => 11.99
]);
```

## Namespaces ##

You can set a namespace when constructing an index. This will allow you to support multi-tenant applications.

```php
$obj_index = new \Search\Index('library', 'client1');
```

# Facets #

The Search API supports 2 types of document facets for categorisation, ATOM and NUMBER.

ATOM are probably the ones you are most familiar with, and result sets will include counts per unique facet, kind of like this:
 
For shirt sizes
* small (9)
* medium (37)

## Adding Facets to a Document ##

```php
$obj_doc->atomFacet('size', 'small');
$obj_doc->atomFacet('colour', 'blue');
```

## Getting Facets in Results ##

```php
$obj_query->facets();
```

# Deleting Documents #

You can delete documents by calling the `Index::delete()` method.

It support one or more `Document` objects - or one or more Document ID strings - or a mixture of objects and ID strings!

```php
$obj_index = new \Search\Index('library');
$obj_index->delete('some-document-id');
$obj_index->delete([$obj_doc1, $obj_doc2]);
$obj_index->delete([$obj_doc3, 'another-document-id']);
```

# Local Development Environment #

The Search API is supported locally, because it's included to support the Python, Java and Go App Engine runtimes.

# Best Practice, Free Quotas, Costs #

Like most App Engine services, search is free... up to a point!

- [Free quota information](https://cloud.google.com/appengine/docs/quotas?hl=en#search)
- [Search API pricing](https://cloud.google.com/appengine/pricing#search_pricing)

And some best practice that is most certainly worth a read

- [Best practice for the Google App Engine Search API](https://cloud.google.com/appengine/docs/python/search/best_practices)


# Google Software #

I've had to include 2 files from Google to make this work - they are the Protocol Buffer implementations for the Search API. You will find them in the `/libs` folder.

They are also available directly from the following repository: https://github.com/GoogleCloudPlatform/appengine-php-sdk

These 2 files are Copyright 2007 Google Inc.

As and when they make it into the actual live PHP runtime, I will remove them from here.

Thank you to @sjlangley for the assist.

# Other App Engine Software #

If you've enjoyed this, you might be interested in my [Google Cloud Datastore Library for PHP, PHP-GDS](https://github.com/tomwalder/php-gds)
