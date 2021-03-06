Slick-pg
========
[![Build Status](https://travis-ci.org/tminglei/slick-pg.svg?branch=master)](https://travis-ci.org/tminglei/slick-pg)

[Slick](https://github.com/slick/slick "Slick") extensions for PostgreSQL, to support a series of pg data types and related operators/functions.

#### Currently supported pg types:
- ARRAY
- Date/Time
- Enum
- Range
- Hstore
- JSON
- `text` Search
- `postgis` Geometry

#### Currently supported pg features:
- inherits
- composite type (`basic`)


** _tested on `PostgreSQL` `v9.3` with `Slick` `v2.1.0`._

Install
-------
To use `slick-pg` in [sbt](http://www.scala-sbt.org/ "slick-sbt") project, add the following to your project file:
```scala
libraryDependencies += "com.github.tminglei" %% "slick-pg" % "0.6.0"
```

> If you need `joda-time` support, pls append dependency:
```scala
libraryDependencies += "com.github.tminglei" %% "slick-pg_joda-time" % "0.6.0"
```

> If you need `jts` geom support, pls append dependency:
```scala
libraryDependencies += "com.github.tminglei" %% "slick-pg_jts" % "0.6.0"
```

> If you need `jdk8 date` support, pls append dependency:
```scala
libraryDependencies += "com.github.tminglei" %% "slick-pg_date2" % "0.6.0"
```

> If you need `threeten-bp` support, pls append dependency:
```scala
libraryDependencies += "com.github.tminglei" %% "slick-pg_threeten" % "0.6.0"
```

> If you need `json4s` support, pls append dependency:
```scala
libraryDependencies += "com.github.tminglei" %% "slick-pg_json4s" % "0.6.0"
```

> If you need `play-json` support, pls append dependency:
```scala
libraryDependencies += "com.github.tminglei" %% "slick-pg_play-json" % "0.6.0"
```

> If you need `spray-json` support, pls append dependency:
```scala
libraryDependencies += "com.github.tminglei" %% "slick-pg_spray-json" % "0.6.0"
```

> If you need `argonaut json` support, pls append dependency:
```scala
libraryDependencies += "com.github.tminglei" %% "slick-pg_argonaut" % "0.6.0"
```


Or, in [maven](http://maven.apache.org/ "maven") project, you can add `slick-pg` to your `pom.xml` like this:
```xml
<dependency>
    <groupId>com.github.tminglei</groupId>
    <artifactId>slick-pg_2.10</artifactId>
    <version>0.6.0</version>
</dependency>

<!-- append play-json/json4s/joda-time/jts/threeten/spray-json dependencies if needed -->
```


Usage
------
Before using it, you need integrate it with PostgresDriver maybe like this:
```scala
import slick.driver.PostgresDriver
import com.github.tminglei.slickpg._

trait MyPostgresDriver extends PostgresDriver
                          with PgArraySupport
                          with PgDateSupport
                          with PgRangeSupport
                          with PgHStoreSupport
                          with PgPlayJsonSupport
                          with PgSearchSupport
                          with PgPostGISSupport {

  override val Implicit = new ImplicitsPlus {}
  override val simple = new SimpleQLPlus {}

  //////
  trait ImplicitsPlus extends Implicits
                        with ArrayImplicits
                        with DateTimeImplicits
                        with RangeImplicits
                        with HStoreImplicits
                        with JsonImplicits
                        with SearchImplicits
                        with PostGISImplicits

  trait SimpleQLPlus extends SimpleQL
                        with ImplicitsPlus
                        with SearchAssistants
                        with PostGISAssistants
}

object MyPostgresDriver extends MyPostgresDriver

```

then in your codes you can use it like this:
```scala
import MyPostgresDriver.simple._

class TestTable(tag: Tag) extends Table[Test](tag, Some("xxx"), "Test") {
  def id = column[Long]("id", O.AutoInc, O.PrimaryKey)
  def during = column[Range[Timestamp]]("during")
  def location = column[Point]("location")
  def text = column[String]("text", O.DBType("varchar(4000)"))
  def props = column[Map[String,String]]("props_hstore")
  def tags = column[List[String]]("tags_arr")

  def * = (id, during, location, text, props, tags) <> (Test.tupled, Test.unapply)
}

object tests extends TableQuery(new TestTable(_)) {
  ///
  def byId(ids: Long*) = tests.filter(_.id inSetBind ids).map(t => t)
  // will generate sql like: select * from test where tags && ?
  def byTag(tags: String*) = tests.filter(_.tags @& tags.toList.bind).map(t => t)
  // will generate sql like: select * from test where during && ?
  def byTsRange(tsRange: Range[Timestamp]) = tests.filter(_.during @& tsRange.bind).map(t => t)
  // will generate sql like: select * from test where case(props -> ? as [T]) == ?
  def byProperty[T](key: String, value: T) = tests.filter(_.props.>>[T](key.bind) === value.bind).map(t => t)
  // will generate sql like: select * from test where ST_DWithin(location, ?, ?)
  def byDistance(point: Point, distance: Int) = tests.filter(r => r.location.dWithin(point.bind, distance.bind)).map(t => t)
  // will generate sql like: select id, text, ts_rank(to_tsvector(text), to_tsquery(?)) from test where to_tsvector(text) @@ to_tsquery(?) order by ts_rank(to_tsvector(text), to_tsquery(?))
  def search(queryStr: String) = tests.filter(tsVector(_.text) @@ tsQuery(queryStr.bind)).map(r => (r.id, r.text, tsRank(tsVector(r.text), tsQuery(queryStr.bind)))).sortBy(_._3)
}

...
 
```

Configurable type/mappers
-------------------------
Since v0.2.0, `slick-pg` started to support configurable type/mappers.

Here's the related technical details:
> All pg type oper/functions related codes and some core type mapper logics were extracted to a new sub project "slick-pg_core", and the oper/functions and type/mappers binding related codes were retained in the main project "slick-pg".

**So, if you need bind different scala type/mappers to a pg type oper/functions, you can do it as "slick-pg" currently did.**


####Built in supported type/mappers:
|          scala Type                 |        pg Type        |
| ----------------------------------- | --------------------- |
| List[T]                             | ARRAY                 |
| `sql` Date<br> Time<br> Timestamp<br> slickpg Interval<br> Calendar | date<br> time<br> timestamp<br> interval<br> timestamptz |
| `jada` LocalDate<br> LocalTime<br> LocalDateTime<br> Period<br> DateTime  | date<br> time<br> timestamp<br> interval<br> timestamptz |
| `java.time` LocalDate<br> LocalTime<br> LocalDateTime<br> Duration<br> ZonedDateTime | date<br> time<br> timestamp<br> interval<br> timestamptz |
| `threeten.bp` LocalDate<br> LocalTime<br> LocalDateTime<br> Duration<br> ZonedDateTime | date<br> time<br> timestamp<br> interval<br> timestamptz |
| `scala` Enumeration                 | enum                  |
| `slickpg` Range[T]                  | range                 |
| Map[String,String]                  | hstore                |
| `json4s` JValue                     | json                  |
| `play-json` JsValue                 | json                  |
| `spray-json` JsValue                | json                  |
| `argonaut json` Json                | json                  |
| (TsQuery+TsVector)                  | `text` search         |
| `jts` Geometry                      | `postgis` geometry    |


Build instructions
------------------
`slick-pg` uses SBT for building and requires Java 8, since it provides support for `java.date` in addon `date2`. Assume you have already installed SBT, then you can simply clone the git repository and build `slick-pg` in the following way:
```
./sbt update
./sbt compile
```

_To run the test suite, you need:_
- create a user 'test' and db 'test' on your local postgres server, and 
- the user 'test' should be an super user and be the owner of db 'test'

Then you can run the tests like this:
```
./sbt test
```
_ps: in the code of unit tests, the `slick` database is setup like this:_
```scala
val db = Database.forURL(url = "jdbc:postgresql://localhost/test?user=postgres", driver = "org.postgresql.Driver")
```


Support details
------------------------------
- Array's [oper/functions](https://github.com/tminglei/slick-pg/tree/master/core/src/main/scala/com/github/tminglei/slickpg/array "Array's oper/functions"), usage  [cases](https://github.com/tminglei/slick-pg/blob/master/src/test/scala/com/github/tminglei/slickpg/PgArraySupportTest.scala "test cases")
- JSON's [oper/functions](https://github.com/tminglei/slick-pg/tree/master/core/src/main/scala/com/github/tminglei/slickpg/json "JSON's oper/functions"), usage cases for [json4s](https://github.com/tminglei/slick-pg/blob/master/addons/json4s/src/test/scala/com/github/tminglei/slickpg/PgJson4sSupportTest.scala "test cases"), [play-json](https://github.com/tminglei/slick-pg/blob/master/addons/play-json/src/test/scala/com/github/tminglei/slickpg/PgPlayJsonSupportTest.scala "test cases"), [spray-json](https://github.com/tminglei/slick-pg/blob/master/addons/spray-json/src/test/scala/com/github/tminglei/slickpg/PgSprayJsonSupportTest.scala "test cases") and [argonaut json](https://github.com/tminglei/slick-pg/blob/master/addons/argonaut/src/test/scala/com/github/tminglei/slickpg/PgArgonautSupportTest.scala)
- Date/Time's [oper/functions](https://github.com/tminglei/slick-pg/tree/master/core/src/main/scala/com/github/tminglei/slickpg/date "Date/Time's oper/functions"), usage cases for [java date](https://github.com/tminglei/slick-pg/blob/master/src/test/scala/com/github/tminglei/slickpg/PgDateSupportTest.scala "test cases"), [joda time](https://github.com/tminglei/slick-pg/blob/master/addons/joda-time/src/test/scala/com/github/tminglei/slickpg/PgDateSupportJodaTest.scala "test cases"), and [java 8 date](https://github.com/tminglei/slick-pg/blob/master/addons/date2/src/test/scala/com/github/tminglei/slickpg/PgDate2SupportTest.scala "test cases") and [threeten bp](https://github.com/tminglei/slick-pg/blob/master/addons/threeten/src/test/scala/com/github/tminglei/slickpg/PgDate2bpSupportTest.scala "test cases")
- Enum's [oper/functions](https://github.com/tminglei/slick-pg/tree/master/core/src/main/scala/com/github/tminglei/slickpg/enums "Enum's oper/functions"), usage [cases](https://github.com/tminglei/slick-pg/blob/master/src/test/scala/com/github/tminglei/slickpg/PgEnumSupportTest.scala "test cases")
- Range's [oper/functions](https://github.com/tminglei/slick-pg/tree/master/core/src/main/scala/com/github/tminglei/slickpg/range "Range's oper/functions"), usage [cases](https://github.com/tminglei/slick-pg/blob/master/src/test/scala/com/github/tminglei/slickpg/PgRangeSupportTest.scala "test cases")
- HStore's [oper/functions](https://github.com/tminglei/slick-pg/tree/master/core/src/main/scala/com/github/tminglei/slickpg/hstore "HStore's oper/functions"), usage [cases](https://github.com/tminglei/slick-pg/blob/master/src/test/scala/com/github/tminglei/slickpg/PgHStoreSupportTest.scala "test cases")
- Search's [oper/functions](https://github.com/tminglei/slick-pg/tree/master/core/src/main/scala/com/github/tminglei/slickpg/search "Search's oper/functions"), usage [cases](https://github.com/tminglei/slick-pg/blob/master/src/test/scala/com/github/tminglei/slickpg/PgSearchSupportTest.scala "test cases")
- Geometry's [oper/functions](https://github.com/tminglei/slick-pg/tree/master/core/src/main/scala/com/github/tminglei/slickpg/geom "Geometry's oper/functions"), usage [cases](https://github.com/tminglei/slick-pg/blob/master/addons/jts/src/test/scala/com/github/tminglei/slickpg/PgPostGISSupportTest.scala "test cases")
- `basic` Composite type [support](https://github.com/tminglei/slick-pg/tree/master/core/src/main/scala/com/github/tminglei/slickpg/composite "Composite type Support"), usage [cases](https://github.com/tminglei/slick-pg/blob/master/src/test/scala/com/github/tminglei/slickpg/PgCompositeSupportTest.scala "test cases")


Version history
------------------------------
v0.6.0 (4-Aug-2014):  
1) upgrade to slick v2.1.0  
2) added pg inherits support  
3) add argonaut json support  
4) re-implement composite support

v0.5.3 (13-Apr-2014):  
1) added jdk8 time support  
2) added pg enum support

v0.5.2 (13-Mar-2014):  
1) added spray-json support

v0.5.1 (22-Feb-2014):  
1) added more postgis/geom functions

v0.5.0 (7-Feb-2014):  
1) upgrade to slick v2.0.0  
2) add basic composite type support  
3) array support: allow nested composite type  
4) add play-json support  
5) add timestamp with zone support  
6) modularization for third party scala type (e.g. `play-json`/`jts`) support

v0.2.2 (04-Nov-2013):  
1) support Joda date/time, binding to Pg Date/Time  
2) support threetenbp date/time, binding to Pg Date/Time

v0.2.0 (01-Nov-2013):  
1) re-arch to support configurable type/mappers

v0.1.5 (29-Sep-2013):  
1) support pg json

v0.1.2 (31-Jul-2013):  
1) add pg datetime support  

v0.1.0 (20-May-2013):  
1) support pg array  
2) support pg range  
3) support pg hstore  
4) support pg search  
5) support pg geometry  


License
-------
Licensing conditions (BSD-style) can be found in LICENSE.txt.
