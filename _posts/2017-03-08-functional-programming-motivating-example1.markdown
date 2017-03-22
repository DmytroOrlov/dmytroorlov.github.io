---
layout: post
title:  "Functional Programming in Scala: A Motivating Example 1"
date:   2017-03-08 18:24:24 +0100
categories: fp
---

Scala is an object-oriented multi-paradigm programming language providing support for functional programming. Let's dig into it.

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
val someZero: Option[Int] = Some(0)
val invalid = someZero.traverse(isPositive)
// invalid: Option[Option[Int]] = None
val containsNegative = List(-1, 2).traverse(isPositive)
// containsNegative: Option[List[Int]] = None
~~~

## Simplified traverse implementation

For those who asking why `containsNegative` is `None`, not `Some(List(2))`. I can give short but probably demotivating answer. We need to start from the very begining and dive into implementation of `traverse` presented below in a simplified form:

~~~scala
import cats.Applicative

def traverse[F[_] : Applicative, A, B](as: List[A])(f: A => F[B]): F[List[B]] =
  as.foldRight(Applicative[F].pure(List.empty[B])) { (a: A, facc: F[List[B]]) =>
    val fb: F[B] = f(a)
    // fb and facc are independent effects so we can use Applicative to compose
    Applicative[F].map2(fb, facc) { (b: B, acc: List[B]) =>
      b :: acc
    }
  }
traverse(List(-1, 2))(isPositive)
// res3: Option[List[Int]] = None
~~~

We need *foldX* to traverse `as: List[A]` and for each `A` apply *effectful* function `f: A => F[B]`. Particular `foldRight` we need to preserve order.

We need `Applicative[F]` for two reasons:
- first for *foldRight* initial value taken by `Applicative.pure`;
- and second we can't *combine* our independent effects `fb: F[B]` and `facc: F[List[B]]` directly, thus `Applicative.map2` allows us to *compose* simply `b: B` and `acc: List[B]`.

~~~scala
type F[T] = Option[T]
type A = Int
type B = Int
val facc0 = Applicative[Option].pure(List.empty[Int])
// facc0: Option[List[Int]] = Some(List())
val fb1 = isPositive(2)
// fb1: Option[Int] = Some(2)
val facc1 = Applicative[Option].map2(fb1, facc0)(_ :: _)
// facc1: Option[List[Int]] = Some(List(2))
val fb2 = isPositive(-1)
// fb2: Option[Int] = None
val facc2 = Applicative[Option].map2(fb2, facc1)(_ :: _)
// facc2: Option[List[Int]] = None
~~~

And finally here is answer, why the last `Applicative[Option].map2(None, Some(List(2)))(_ :: _)` is not `Some(List(2))` but `None`. Just because for result `Option[Z]` we need to do both `flatMap` and than `map`:
~~~scala
cats.instances.option.catsStdInstancesForOption.map2(None, Some(List(2)))(_ :: _)
// res4: Option[List[Int]] = None

val applicativeInstancesForOption = new Applicative[Option] {
  override def map2[A, B, Z](fa: Option[A], fb: Option[B])(f: (A, B) => Z): Option[Z] =
    fa.flatMap(a => fb.map(b => f(a, b)))

  override def pure[A](x: A): Option[A] = ???

  override def ap[A, B](ff: Option[(A) => B])(fa: Option[A]): Option[B] = ???
}
applicativeInstancesForOption.map2(None, Some(List(2)))(_ :: _)
// res5: Option[List[Int]] = None
~~~


[cats-tl]: http://typelevel.org/cats/
[scalaz-gh]: https://github.com/scalaz/scalaz
