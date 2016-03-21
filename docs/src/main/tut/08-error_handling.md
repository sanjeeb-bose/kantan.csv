---
layout: default
title:  "Error handling"
section: tutorial
---
There are many ways of dealing with parse errors in kantan.csv. This tutorial shows the most common strategies, but
it essentially boils down to knowing how [`Result`] (the underlying type of [`CsvResult`]) works.

All the examples here are going to be using the following data:

```
1,Nicolas,true
2,Kazuma,28
3,John,false
```

Note how the second row's third column is not of the same type as that of the other rows.

Let's first declare the basic things we need to decode such a CSV file (see [this](03-rows_as_case_classes) if it does
not make sense to you):

```tut:silent
import kantan.csv.ops._
import kantan.csv.generic._

case class Person(id: Int, name: String, flag: Boolean)

val rawData: java.net.URL = getClass.getResource("/dodgy.csv")
```

## Throw on errors

The simplest, least desirable error handling mechanism is to ignore the possibility of failure and allow exceptions
to be thrown. This is achieved by using [`asUnsafeCsvReader`]:
 
```tut
scala.util.Try(rawData.asUnsafeCsvReader[Person](',', false).toList)
```

Note that this is hardly ever an acceptable solution. In idiomatic Scala, we pretend that exceptions don't exist and
rely on encoding errors in return types. Still, unsafe readers can be useful - when writing one-off scripts for which
reliability or maintainability are not an issue, for example.

## Drop errors
Another common, if not always viable strategy is to use [`collect`] to simply drop whatever rows failed to decode:

```tut
rawData.asCsvReader[Person](',', false).collect { case kantan.codecs.Result.Success(a) ⇒ a }.toList
```

[`collect`] is a bit like a [`filter`] and a [`map`] rolled into one, and allows us to:

* ignore all failures
* extract the value of all successes

This is achieved in an entirely safe way, validated at compile time.

Note that we're using types from a previously unseen package, `kantan.codecs`. This contains a lot of shared code
between various kantan libraries, and our [`CsvResult`] is simply a type alias for [`Result`] with hard-coded type
parameters. This can be ignored most of the time, but "advanced" error handling requires us to get closer to the metal.  


## Fail if at least one row fails to decode
When not streaming data, a good option is to fail if a single row fails to decode - turn a `List[CsvResult[A]]` into
a `CsvResult[List[A]]`. This is done through [`Result`]'s [`sequence`] method:

```tut
kantan.codecs.Result.sequence(rawData.readCsv[List, Person](',', false))
```

The only real downside to this approach is that it requires loading the entire data in memory.
 

## Turn errors into default values
Some data types have reasonable default values that can be used instead of errors - one could turn a `CsvError[Int]`
into `Success(0)`, for instance.

This is achieved through [`getOrElse`] (even if this example doesn't make much practical sense):

```tut
rawData.asCsvReader[Person](',', false).map(_.getOrElse(Person(0, "ERMAC", true))).toList
```

## Use more flexible types to prevent errors
Our problem here is that the `flag` field of our `Person` class is not always of the same type - some rows have it as a
`boolean`, others as an `Int`. This is something that the [`Either`] type is well suited for, so we could rewrite
`Person` as follows:

```tut:silent
case class Person(id: Int, name: String, flag: Either[Boolean, Int])
```

We can now load the whole data without an error:

```tut
rawData.readCsv[List, Person](',', false)
```

Following the same general idea, one could use [`Option`] for fields that are not always set.

This strategy is not always possible, but is good to keep in mind for these cases where it can be applied.

[`asUnsafeCsvReader`]:{{ site.baseurl }}/api/#kantan.csv.ops$$CsvInputOps@asUnsafeCsvReader[B](sep:Char,header:Boolean)(implicitevidence$4:kantan.csv.RowDecoder[B],implicitai:kantan.csv.CsvInput[A],implicite:kantan.csv.engine.ReaderEngine):kantan.csv.CsvReader[B]
[`CsvResult`]:{{ site.baseurl }}/api/#kantan.csv.package@CsvResult[A]=kantan.codecs.Result[kantan.csv.CsvError,A]
[`Result`]:http://nrinaudo.github.io/kantan.codecs/api/#kantan.codecs.Result
[`collect`]:{{ site.baseurl }}/api/#kantan.csv.CsvReader@collect[B](f:PartialFunction[A,B]):kantan.csv.CsvReader[B]
[`filter`]:{{ site.baseurl }}/api/#kantan.csv.CsvReader@filter(p:A=>Boolean):kantan.csv.CsvReader[A]
[`map`]:{{ site.baseurl }}/api/#kantan.csv.CsvReader@map[B](f:A=>B):kantan.csv.CsvReader[B]
[`Either`]:http://www.scala-lang.org/api/current/index.html#scala.util.Either
[`Option`]:http://www.scala-lang.org/api/current/index.html#scala.Option
[`sequence`]:http://nrinaudo.github.io/kantan.codecs/api/#kantan.codecs.Result$@sequence[F,S,M[X]<:TraversableOnce[X]](rs:M[kantan.codecs.Result[F,S]])(implicitcbf:scala.collection.generic.CanBuildFrom[M[kantan.codecs.Result[F,S]],S,M[S]]):kantan.codecs.Result[F,M[S]]
[`getOrElse`]:http://nrinaudo.github.io/kantan.codecs/api/#kantan.codecs.Result@getOrElse[SS>:S](default:=>SS):SS