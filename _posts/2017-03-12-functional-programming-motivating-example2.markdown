---
layout: post
title:  "Functional Programming in Scala: A Motivating Example #2"
date:   2017-03-12 18:12:12 +0100
categories: fp
---

Now we will try to write and test asynchronous code with Futures.

Let's imagine that we need to write some asynchronous processor.

~~~scala
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.Future

class FutureProcessor {
  def apply(value: Future[Int]): Future[Int] = value.map(_ + 1)
}
~~~

It is simple to write and read this code. But testing requires to use `org.scalatest.concurrent.ScalaFutures.whenReady` and actually to test both *your code* and `Future`.

~~~scala
import org.scalatest.concurrent.ScalaFutures
import org.scalatest.{FunSuite, MustMatchers}

import scala.concurrent.Future

class FutureProcessorSuite extends FunSuite with MustMatchers with ScalaFutures {
  test("FutureProcessor") {
    whenReady(new FutureProcessor().apply(Future.successful(1))) {
      _ mustBe 2
    }
  }
}
~~~

If we start to think in terms of functional programming. It's obvious that `Future[Int]` is meaningful `value: Int` inside some *context* that could be abstract.

~~~scala
import cats.Functor

class ContextProcessor {
  def apply[F[_]: Functor](value: F[Int]): F[Int] = Functor[F].map(value)(_ + 1)
}
~~~

We can use it same conventional way.

~~~scala
import cats.instances.all._

import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.duration._
import scala.concurrent.{Await, Future}

val f = new ContextProcessor().apply(Future.successful(1))
// f: scala.concurrent.Future[Int] = Future(<not completed>)
Await.result(f, 1.second)
// res0: Int = 2
~~~

Now we can test only our code, not `Future`, and also we will not need context specific `org.scalatest.concurrent.ScalaFutures.whenReady`.

~~~scala
import org.scalatest.{FunSuite, MustMatchers}

import cats.Id

class ContextProcessorSuite extends FunSuite with MustMatchers {
  test("ContextProcessor") {
    val one: Id[Int] = 1
    new ContextProcessor().apply(one) mustBe 2
  }
}
~~~
