play-elastic4s
===========================

We've been using [elastic4s](https://github.com/sksamuel/elastic4s) with [Play framework](https://www.playframework.com/) for a while.
Without convenient way of configuration and injecting ElasticClient instance there was a lot boilerplate work to do.
This module enhances elastic4s with two main features:

1. Loading driver configuration from `application.conf`.
1. Automatic JSON conversions for Elasticsearch based on [Play JSON formatters](https://www.playframework.com/documentation/2.4.x/ScalaJson).

As a bonus, the Elasticsearch driver will automatically disconnect on Play application shutdown.


Quick Start
-----------

### 1. Installation


Add the dependency to your `build.sbt`.

	libraryDependencies += "com.evojam" %% "play-elastic4s" % "0.2.1-SNAPSHOT"

*Note:* the artifact is not published anywhere yet - you will have to download it and `sbt publish-local` yourself.

*Note 2:* Current version of the module works only with Elasticsearch 2.2.X.


### 2. Setup

Extend your application.conf to load the module and get all goodies injectable:

```hocon
elastic4s {
		clusters {
				myCluster {
					 uri: elasticsearch://host:port  // <-- pass something useful here
					 cluster.name: "mycluster"       // <-- and here
				}
		}
		indexAndTypes {                        // <-- this section is not required, but
				book {                             //     this index/type pair will be injectable later
						index: "library"
						type: "book"
				}
		}
}

play.modules.enabled += "com.evojam.play.elastic4s.Elastic4sModule"
```

### 3. Enjoy injectable configuration

Instead of instantiating `ElasticClient` manually, have the configuration
and our factory [injected](https://www.playframework.com/documentation/2.4.x/ScalaDependencyInjection):

```scala

class BookDao @Inject()(
	cs: ClusterSetup,
	elasticFactory: PlayElasticFactory,
	@Named("book") indexAndType: IndexAndType) extends
		ElasticDsl {

	private[this] lazy val client = elasticFactory(cs)

	def searchByAnything(q: String): Future[RichSearchResponse] = client execute {
		search in indexAndType query q
	}
}
```

Apart from the injection, nothing changes. The `client` generated by the factory is a regular `ElasticClient` from `elastic4s` library,
so all the original API stays.

### 4. Enjoy seamless JSON conversions

The original elastic4s API already provides automatic JSON conversions, but it requires you to provide
specific typeclasses ([Indexable](https://github.com/sksamuel/elastic4s#indexing-from-classes)
and [HitAs](https://github.com/sksamuel/elastic4s#search-conversion)). Play-elastic4s will derive them automatically
based on your Play JSON formatters - just mix in the `PlayElasticJsonSupport` trait:

```scala
case class Book(title: String, author: String, publishDate: DateTime)
object Book {
	implicit val format: Format[Book] = Json.format[Book]
}

class BookDao @Inject()(
	cs: ClusterSetup,
	elasticFactory: PlayElasticFactory,
	@Named("book") indexAndType: IndexAndType) extends
		ElasticDsl with
		PlayElasticJsonSupport {

	private[this] lazy val client = elasticFactory(cs)

	def searchByAnything(q: String): Future[Array[Book]] = client execute {
		search in indexAndType query q
	} map (_.as[Book])   // here the conversion happens. Will throw if documents are malformed.

	def add(bookId: String, book: Book) = client execute {
		index into indexAndType source book id bookId
	}

	// as an extra, you also get an extension method when getting by ID.
	// It returns None if there is no document with the given ID
	// and throws an exception if the document cannot be parsed:
	def getById(bookId: String): Future[Option[Book]] = client.execute {
		get id bookId from indexAndType
	} map (_.as[Book])
}
```

Advanced usage
--------------------------

### Detailed ES connection configuration
Simply pass more options to "elastic4s.clusters.<your-cluster>" node in the config file.
They will be passed to the ES java driver.

### Using mutliple ES clusters
Specify them all in the config file:

```hocon
elastic4s {
		clusters {
				firstCluster {
					 uri: elasticsearch://main.es.example.com:9300
					 cluster.name: "first-es-cluster"
				}
				loggingCluster {
					uri: elasticsearch://log.es.example.com:9300
					cluster.name: "log-es-cluster"
				}

		}
		indexAndTypes {
				book {
						index: "library"
						type: "book"
				}
		}
}

play.modules.enabled += "com.evojam.play.elastic4s.Elastic4sModule"
```

The unnamed binding for `ClusterSetup` is available only if there is exactly one cluster configured.
With multiple clusters, use `@Named` annotation:

```scala
class BookDao @Inject()(
	@Named("firstCluster") firstCs: ClusterSetup,
	@Named("loggingCluster") loggingCs: ClusterSetup,
	elasticFactory: PlayElasticFactory,
	@Named("book") indexAndType: IndexAndType) extends
		ElasticDsl {

	private[this] lazy val client = elasticFactory(firstCs)
	private[this] lazy val logClient = elasticFactory(loggingCs)
	
	...
}
```

















