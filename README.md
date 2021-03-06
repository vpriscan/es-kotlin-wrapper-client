[![](https://jitpack.io/v/jillesvangurp/es-kotlin-wrapper-client.svg)](https://jitpack.io/#jillesvangurp/es-kotlin-wrapper-client)

# Introduction

ES Kotlin Wrapper client for the Elasticsearch Highlevel REST client is a client library written in Kotlin that adapts the official Highlevel Elasticsearch HTTP client for Java (introduced with Elasticsearch 6.x) with some Kotlin specific goodness. 

The highlevel elasticsearch client is written in Java and provides access to essentially everything in the REST API that Elasticsearch exposes. The kotlin wrapper takes away none of that and adds some power and convenience to it.

It adds extension methods, cuts down on boilerplate through use of several kotlin features for creating DSLs, default arguments, sequences. etc. Some of this is also  usable by Java developers (with some restrictions). Android is not supported as the minimum requirements for the highlevel client are Java 8. 

# Get it

I'm using jitpack for releases currently; the nice thing is all I need to do is tag the release in Git and they do the rest. They have nice instructions for setting up your gradle or pom file:

[![](https://jitpack.io/v/jillesvangurp/es-kotlin-wrapper-client.svg)](https://jitpack.io/#jillesvangurp/es-kotlin-wrapper-client)

This may change when this stuff becomes more stable. I'm planning to push this to maven central via Sonatype's OSS repository eventually.

See [release notes](https://github.com/jillesvangurp/es-kotlin-wrapper-client/releases) with each github release tag for an overview what changed.

# Examples/highlights

The examples below are not the whole story. Please look at the tests for more details on how to use this and for working examples. I try to add tests for all the features along with lots of inline documentation. Also keeping markdown in sync with code is a bit of a PITA, so beware minor differences.

## Initialization

```kotlin
// we need the official client but we can create it with a new extension function
val esClient = RestHighLevelClient()

```

Or specify some optional parameters

```kotlin
# great if you need to interact with ES Cloud ...
val esClient = RestHighLevelClient(host="domain.com",port=9999,https=true,user="thedude",password="lebowski")
```

Or pass in the builder and rest client as you would normally.

**Everything** you do normally with the Java client works exactly as it does normally. But, we've added lots of extra methods to make things easier. For example searching is supported with a Kotlin DSL that adapts the existing `SearchRequest` and adds an alternative `source` method that takes a String, which in Kotlin you can template as well:


```kotlin
# simple example that we use as part of 
# a health check against elastic cloud
# we are alive if we are not logging errors ...
val baseUrl="https://api.inbot.io"
val minutes=60
val results = esClient.doSearch {
            indices("inbot-api*")

            source(
                """
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        {
          "range": {
            "@timestamp": {
              "gte": "now-${minutes}m"
            }
          }
        },
        {
          "term":{
            "inbot_base_url":"$baseUrl"
          }
        },
        {
          "term":{
            "level":"ERROR"
          }
        }
      ]
    }
  },
  "aggs": {
    "loggers": {
      "terms": {
        "field": "logger_name",
        "size": 1000
      }
    }
  }
}""".trimIndent()
            )
        }

```

## DAOs and serialization/deserialization

A key feature in this library is using Data Access Objects or DAOs to abstract the business of using indices (and the soon to be removed types). Indices store JSON that conforms to your mapping. Presumably, you want to serialize and deserialize that JSON. DAOs take care of this and more for you.

You create a DAO per index:

```kotlin
// Lets use a simple Model class as an example
data class TestModel(var message: String)

// we use the refresh API in tests; you have to opt in to that being allowed explicitly 
// since you should not use that in production code.
val dao = esClient.crudDao<TestModel>("myindex", refreshAllowed = true,
modelReaderAndWriter = JacksonModelReaderAndWriter(
                TestModel::class,
                ObjectMapper().findAndRegisterModules()
))

// OR
// type is optional (default is the index name) as this is to be deprecated 
// in Elasticsearch and in any case you can have only 1 type per index these days.
val dao = esClient.crudDao<TestModel>("myindex", refreshAllowed = true, type: "mytype",
modelReaderAndWriter = JacksonModelReaderAndWriter(
                TestModel::class,
                ObjectMapper().findAndRegisterModules()
))
```

The modelReaderAndWriter parameter takes care of serializing/deserializing. In this case we are using an implementation that uses Jackson and we are using the kotlin extension for that, which registers itself via `findAndRegisterModules`. 

Typically, most applications that use jackson, would want to inject their own custom object mappers. It's of course straightforward to use alternatives like GSon or whatever else you prefer. In the code below, we assume Jackson is used.

## CRUD with Entities

Managing documents in Elasticsearch basically means doing CRUD operations. The DAO supports this and makes it painless to manipulate documents.

```kotlin
# creates a document or fails if it already exists
# note you should probably apply a mapping to your index before you index something ...
dao.index("xxx", TestModel("Hello World"))

// if you want to do an upsert, you can 
// do that with a index and setting create=false
dao.index("xxx", TestModel("Hello World"), create=false)
val testModel = dao.get("xxx")

// updates with conflict handling and optimistic locking
// you apply a lambda against an original 
// that is fetched from the index using get()
dao.update("xxx") { original -> 
  original.message = "Bye World"
}
// in case of a version conflict, update 
// retries. The default is 2 times 
// but you can override this.
// version conflicts happen when you
// have concurrent updates to the same document
dao.update("xxx", maxUpdateTries=10) { original -> 
  original.message = "Bye World"
}

// delete by id
dao.delete("xxx")
```
See [Crud Tests](https://github.com/jillesvangurp/es-kotlin-wrapper-client/blob/master/src/test/kotlin/io/inbot/eskotlinwrapper/IndexDAOTest.kt) for more.

## Bulk Indexing using the Bulk DSL

The Bulk indexing API in Elasticsearch is complicated and it requires a bit of boiler plate to use and more boiler plate to use responsibly. This is made trivially easy with a `BulkIndexingSession` that abstracts all the hard stuff and a DSL that drives that:

```kotlin
// creates a BulkIndexingSession and uses it 
dao.bulk {
  // lets fill our index
  for (i in 0..1000000) {
    // inside the block, this refers to 
    // the BulkIndexingSession instance
    // that bulk manages for you
    // The BulkIndexingSession creates 
    // BulkRequests on the fly and sends 
    // them off 100 operations (default) 
    // at the time.
    // this example indexes a document. 
    // But you can also update, updateAndGet, 
    // and delete
    index("doc_$i", TestModel("Hi again for the $i'th time"))
  }
}
// when the block exits, the last 
// bulkRequest is send. 
// The BulkIndexingSession is AutoClosable.
```
See [Bulk Indexing Tests](https://github.com/jillesvangurp/es-kotlin-wrapper-client/blob/master/src/test/kotlin/io/inbot/eskotlinwrapper/BulkIndexingSessionTest.kt) for more. 

There are many features covered there including per item callbacks to deal with success/failure per bulk item (rather than per page), optimistic locking for updates (using a callback), setting the bulk request page size, etc. You can tune this a lot but the defaults should be fine for simple usecases.

## Search

For search, the wrapper offers several useful features that make make constructing search queries and working with the response easier.

Given an index with some documents:

```kotlin
dao.bulk {
    index(randomId(), TestModel("the quick brown emu"))
    index(randomId(), TestModel("the quick brown fox"))
    index(randomId(), TestModel("the quick brown horse"))
    index(randomId(), TestModel("lorem ipsum"))
    // throw in some more documents
    for(i in 0..100) {
        index(randomId(), TestModel("another document $i"))
    }
}
dao.refresh()
```

... lets do some searches. We want to find matching TestModel instances:

```kotlin
// our dao has a search short cut that does all the right things
val results = dao.search {
  // inside the block this is now the searchRequest, the index is already set correctly
  // you can use it as you would normally. Here we simply use the query DSL in the high level client.
  val query = SearchSourceBuilder.searchSource()
      .size(20)
      .query(BoolQueryBuilder().must(MatchQueryBuilder("message", "quick")))
  // set the query as the source on the search request
  source(query)
}

// SeachResults wraps the original SearchResponse and adds some features
// e.g. we put totalHits at the top level for convenience. 
assert(results.totalHits).isEqualTo(3L)

// results.searchHits is a Kotlin Sequence
results.searchHits.forEach { searchHit ->
    // the SearchHits from the elasticsearch client
}

// mappedHits maps the source using jackson, done lazily because we use a Kotlin sequence
results.mappedHits.forEach {
    // and we deserialized the results
    assert(it.message).contains("quick")
}
// or if you need both the original and mapped result, you can
results.hits.forEach {(searchHit,mapped) ->
    assert(mapped.message).contains("quick")
}
```

## Raw Json queries and multi line strings

The wrapper adds a few strategic extension methods. One of those takes care of the boilerplate you need to turn strings or streams into a valid query source.

So, the same search also works with multiline json strings in Kotlin and you don't have to jump through hoops to use raw json. Multiline strings are awesome in Kotlin and you can even inject variables and expressions.

```kotlin
val keyword="quick"
val results = dao.search {
    source("""
    {
      "size": 20,
      "query": {
        "match": {
          "message": "$keyword"
        }
      }
    }
    """)
}
```

In the same way you can also use `InputStream` and `Reader`. Together with some templating, this may be easier to deal with than programatically constructing complex queries via  builders. Up to you. I'm aware of some projects attempting a Kotlin query DSL and may end up adding support for that or something similar as well.

See [Search Tests](https://github.com/jillesvangurp/es-kotlin-wrapper-client/blob/master/src/test/kotlin/io/inbot/eskotlinwrapper/SearchTest.kt) for more.

## Scrolling searches made easy

Scrolling is kind of tedious in Elastisearch. You have to keep track of scroll ids and get pages of results one by one. This requires boilerplate. We use Kotlin sequences to solve this. Sequences are lazy, so this won't run out of memory and you can safely stream process huge indiceses. Very nice in combination with bulk indexing.  

The Kotlin API is exactly the same as a normal paged search (see above). But please note that things like ranking don't work with scrolling searches and there are other subtle differences on the Elasticsearch side. 

```kotlin
// We can scroll through everything, all you need to do is set scrolling to true
// you can optionally override scrollTtlInMinutes, default is 1 minute
val results = dao.search(scrolling=true) {
  val query = SearchSourceBuilder.searchSource()
      .size(7) // lets use weird page size
      .query(matchAllQuery())
  source(query)
}

// Note: using the sequence causes the client to request for pages. 
// If you want to use the sequence again, you have to do another search.
results.mappedHits.forEach {
    println(it.message)
}
```

The `ScrollingSearchResults` implementation that is returned takes care of fetching all the pages, clearing the scrollId at the end, and of course mapping the hits to TestModel. You can only do this once of course since we don't keep the whole result set in memory and the scrollids are invalidated as you use them.

See [Search Tests](https://github.com/jillesvangurp/es-kotlin-wrapper-client/blob/master/src/test/kotlin/io/inbot/eskotlinwrapper/SearchTest.kt) for more.

# Building

You need java >= 8 and docker + docker compose (to run elasticsearch and the tests).

Simply use the gradle wrapper to build things:

```
./gradlew build
```

Look inside the build file for comments and documentation.

Gradle will spin up Elasticsearch using docker compose and then run the tests against that. If you want to run the tests from your IDE, just use `docker-compose up -d` to start ES. The tests expect to find that on a non standard port of `9999`. This is to avoid accidentally running tests against a real cluster and making a mess there (I learned that lesson a long time ago).

# Development status

**This is a pre-1.0 version**. The main reason is that I'm still adding features and there may be some minor refactoring and changes. However, we are using this in our own product and it should be perfectly fine for general use at this point. Also note, that you can always access the underlying Java client, which is stable. 

While using Kotlin is recommended, you can technically also use this from Java. Checkout some of the Java specific tests for examples for this. 

## Compatibility

The general goal is to keep this client compatible with the current stable version of Elasticsearch. 

Currently we update this libary regularly for the current stable version of Elasticsearch. With the upcoming 7.x versions, we may start having to do release branches. There have been minor Java API changes in the 6.x series in the client. Currently, we rely on the most recent 6.x version. Presumably, this works fine against any 6.x node (the REST protocol should be more stable); and possibly some older versions. If you experience issues, please file a ticket or pull request.

## Features (done)

- CRUD: index, update, get and delete of any jackson serializable object
- Reliable update with retries and optimistic locking that uses a `T -> T` lambda to transform what is in the index to what it needs to be. Retry kicks in if there's a version conflict and it simply re-fetches the latest version and applies the lambda.
- Bulk indexing with support for index, update, and delete. Supports callbacks for items and takes care of sending and creating bulk requests. The default callback can take care of retrying updates if they fail with a conflict if you set retryConflictingUpdates to a non zero value.
- Easy search with jackson based result mapping
- Easy scrolling search, also with jackson based result mapping. You can do this with the same method that does the normal paged search simply by setting `scrolling = true` (defaults to false)

## Future feature ideas/TODO

- ES 7.x branch - should be straightforward. Will probably do this when the betas come out.
- Cut down on the builder cruft and boilerplate in the query DSL and use extension methods with parameter defaults.
- Make creating and using aggregations less painful and port over some work I've done for that in the past. 
- Index and alias management with read and write alias support built in.
- Schema versioning and migration support that uses aliases and the reindexing API.
- API documentation.
- Set up CI, travis? Docker might be tricky.

# License

This project is licensed under the [MIT license](LICENSE). This maximizes everybody's freedom to do what needs doing. Please exercise your rights under this license in any way you feel is right. Forking is allowed and encouraged. I do appreciate attribution and pull requests...
