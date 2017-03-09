---
layout: post
title:  "Functional Programming in Scala: A Motivating Example"
date:   2017-03-08 18:24:24 +0100
categories: fp
---

Scala is a multi-paradigm general-purpose programming language providing support for functional programming. Let's dig into it.

## An example from the standard library

~~~scala
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.duration._
import scala.concurrent.{Await, Future}

val eventualInts = Future.traverse(List(1, 2))(fn =
  n => Future.successful(n + 1)
)
// eventualInts: scala.concurrent.Future[List[Int]] = Future(<not completed>)
Await.result(eventualInts, 1.second)
// res0: List[Int] = List(2, 3)
~~~

`Future.traverse` takes a `List[A]` and for each `A` applies *effectful* function `fn`.

But what if the effect we wanted wasn’t `Future`? What if instead of concurrency for our effect we wanted validation (`Option`, `Either`). This way we can abstract out type of effect and even abstract over data types that can be traversed over such as `List` and `Option`.

As you can see standard library is useful but not generic. You need a library intended to provide abstractions for functional programming in Scala like [Cats][cats-tl] or [Scalaz][scalaz-gh].

## Same example rewritten with Cats

[Cats][cats-tl] contans all functional programming machinery needed to generalize the standard library’s `Future.traverse` into its fully abstract and most reusable form:

~~~scala
import cats.instances.all._
import cats.syntax.traverse._

import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.duration._
import scala.concurrent.{Await, Future}

val eventualInts = List(1, 2).traverse(n => Future.successful(n + 1))
// eventualInts: scala.concurrent.Future[List[Int]] = Future(<not completed>)
Await.result(eventualInts, 1.second)
// res0: List[Int] = List(2, 3)
~~~

## Abstract over traversed data types

Instead of `List` we can use `Option`:

~~~scala
val someInt: Option[Int] = Some(1)
val eventualSomeInt = someInt.traverse(n => Future.successful(n + 1))
// eventualSomeInt: scala.concurrent.Future[Option[Int]] = Future(<not completed>)
Await.result(eventualSomeInt, 1.second)
// res1: Option[Int] = Some(2)

val noneInt: Option[Int] = None
val eventualNoneInt = noneInt.traverse(n => Future.successful(n + 1))
// eventualNoneInt: scala.concurrent.Future[Option[Int]] = Future(<not completed>)
Await.result(eventualNoneInt, 1.second)
// res2: Option[Int] = None
~~~

## Abstract out type of effect

Effect can be other than concurrent execution in `Future`, let's do validation for all positive integers:

~~~scala
def isPositive: Int => Option[Int] = {
  case n if n > 0 => Some(n)
  case _ => None
}

val forallPositive = List(1, 2).traverse(isPositive)
// forallPositive: Option[List[Int]] = Some(List(1, 2))
val containsNegative = List(-1, 2).traverse(isPositive)
// containsNegative: Option[List[Int]] = None
val someZero: Option[Int] = Some(0)
val invalid = someZero.traverse(isPositive)
// invalid: Option[Option[Int]] = None
~~~

[cats-tl]: http://typelevel.org/cats/
[scalaz-gh]: https://github.com/scalaz/scalaz
