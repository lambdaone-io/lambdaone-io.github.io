---
title: Error Handling in Pure Functional Applications with ZIO
date: 2019-09-06 21:05:38
tags:
- IO
- ZIO
- Side Effects
- Error Handling 
---

# Error Handling with ZIO in Pure Functional Applications

In pure functional programming, functions are total - every input produces output. For example, `toString(i: Int) : String` 
produces a `String` form every `Int`. As not every `String` can be converted to an `Int`, we can use `Option[int]` to
return `None` when no conversion is possible, 
or `Either[Error, Int]` to provide error information on failure - every `String` produces either an `Error` or an `Int`. 

Typically, such computations are being chained - the result of one computation is used as input for the next one. 
For example, given a city name, we first look up its coordinates, then we look up the weather forecast for this location. 
Finally, imagine we do not need the whole forecast but only the temperature, which may or may not be present in the forecast.
Sequential computations like this are modelled by monads. Scala offers special syntax to make sequencing more readable.
In our example, given `lookupCity(city: String): Either[Error, Location]`,
 `weather(l: Location): Either[Error, Forecast]` and `getTemperature(f: Forecats): Option[Double]`,
we can do
```scala
def forecast(city: String) : Either[Error, Double] =
 for {
  location <- lookupCity(city)
  forecast <- weather(location)
  temp <- getTemperature(forecast).toEither("No temp in forecast")
 } yield (forecast)
```

Monads representing error situations shortcut on error - once there is `None`, or `Left`, other functions 
in the 
sequence to not get called and the error value gets passed 
directly to `yield`.

Real applications include 
interactions with the world, which may fail in their own way, on top of the errors modelled
by `Option` or `Either` types.
For example, if we call a remote service to look up the city, the call itself may fail, even 
with valid city name as input. Think of a network error, or a database error. In pure FP,
interactions with the world
are modelled by `IO`, which is a monad itself - chaining effects and short-cutting on failure
can be coded using the same `for` comprehension syntax. 

If 
we forget business errors for a moment and only think of `IO` errors, given 
`lookupCity(city: String]: IO[Location]` and `weather(l: Location]: IO[Forecast]`,
we can do: 
```scala
def forecast(city: String) : IO[Double] =
for {
  location <- lookupCity(city)
  forecast <- weather(location)
  temp <- getTemperature(forecast).map(IO.pure(_)).getOrElse(IO.fail(new Exception("No temp in forecast"))
} yield (temp)
```

`forecast` gives a value of type `IO` which describes the computation. To get the result ,we have to 
execute it. The execution may end up in either success or failure and in case of failure `flatMap` chains get short-circuited.

How does `IO[A]` shortcut, if its type contains no error? It does, because it is a `MonadError[A, Throwable]` next to being a monad. Its working is
 demonstrated by this short example:
 ```scala
(for { _ <- IO.raiseError(new Exception("boom")); b = 1} yield b).unsafeRunSync // throws exception
```
We could abstract `IO` to a type `F` with the necessary type classes to the sequencing and shortcutting: 
```scala
def forecast[F[_] : Monad : MonadError] (city: String) : F[Forecast] =
for {
  location <- lookupCity(city)
  forecast <- weather(location)
  temp <- getTemperature(forecast).map(IO.pure(_)).getOrElse(IO.fail(new Exception("No temp in forecast"))
} yield (forecast)
```

Most real applications will have to handle both types of errors - I/O errors and business logic errors. If
we stick to the generic `F` effect, the type will be `F[Either[Error, A]]` - we have to deal with nested effects.

If we try to handle both effects errors ourselves, things can get ugly fast.
Let's see what happens when we have `lookupCity(city: String]: F[Either[Error, Location]]` and 
`weather(l: Location]: F[Either[Error, Forecast]]`. The external monad errors are handled
by the `for` comprehension, but we have to take care of the internal one:

## Plain Scala

```scala
def forecast(city: String) : F[Either[Double]] =
for {
  location <- lookupCity(city)
  forecast <- location match {
    case Right(l) => weather(location)
    case err @ Left(_) => F.pure(err)
  }
  temp = forecast match {
    case Right(f) => getTemperature(forecast).toEither("No temp in forecast")
    case err => err
  }
} yield (temp)
```
And this is quite a simple case with 2 services and one effects-free function. Yes, things can get ugly fast this way.

## Monad Transformers

As we see in the last piece of code, nested monads do not combine automatically - we need to know details of the inner monad
in order to handle the combination of two monads. We can use monad transformers where this logic is encoded - every monadic type requires 
a particular transformer knowing how to flatMap over it.
For `F[Either[Error,A]]` we need the `EitherT` transformer - both `cats` and `scalaz` provide it. From the `cats` documentation:
 _`EitherT[F, A, B]` wraps a value of type `F[Either[A, B]]`. An `F[C]` can be lifted in to `EitherT[F, A, C]` via `EitherT.right`,
 and lifted in to a `EitherT[F, C, B]` via `EitherT.left`._ With a monad transformer, the combined type is a monad again.
 Let's rewrite our example:
 ```scala
 def forecast(city: String) : F[Either[Error, Double]] =
 for {
   location <- EitherT(lookupCity(city))
   forecast <- EitherT(weather(location))
   temp <- EitherT.fromEither(getTemperature(forecast).toEither("No temp in forecast"))
 } yield (temp)
```
 
Much better! This can be further improved by introducing some syntax, e.g.:

```scala
type ErrorOr[E,A] = EitherT[F, E, A]

  implicit class feitherToErrorOr[E, A](r: F[E Either A]) { def toErrorOr: ErrorOr[E, A] = EitherT(r) }
  implicit class optionToErrorOr[E, A](o: Option[A]) { def toErrorOr( e: Error): ErrorOr[E, A] = EitherT.fromEither(r.toEither(e) }
```
the code above becomes:

 ```scala
 def forecast(city: String) : ErrorOr[Double] =
 for {
   location <- lookupCity(city).toErrorOr
   forecast <-  weather(location).toErrorOr
   temp     <-  getTemperature(forecast).toErrorOr("No temp in forecast")
 } yield (temp)
```

The code above looks much nicer. In fact, I saw this on one of my first Scala projects and I was impressed by its power and purity. Later, 
I have created or contributed to many applications based on this approach. This approach is not perfect - reasoning about types 
withing the `for` comprehension 
can become difficult, and a mistake in that reasoning can call another `toErrorOr` than the one you had in mind. 

## ZIO

`ZIO` is an alternative IO implementation. Its approach to errors differs from cats `IO`. 
Cats has one type parameter - `IO[A]` returns a value of type `A`. It still shortcuts on error by inheriting `MonadError[A, Throwable]`. 
`ZIO` chooses to be explicit about the error. It is defined as
 `ZIO[R, E, A]`.  ZIO is the effectful version of `R => Either[E, A]` - given R, on execution ZIO returns either a value of type 
 `A` or an error of type `E`. Where `IO[A]` is functorial in `A`, `ZIO[R, E, A]` is functorial in both `E` and `A`. 
 
 If we decided we don't need `R` for now, ZIO provides `type IO[+E, +A] = ZIO[Any, E, A]`. 
 For our application, given a specific error type `type Error = String`, we would have 
 `type AppTask[A] = ZIO[Any, Error, A]`. In our ZIO-based application,
  the business functions would be `lookupCity(city: String]: AppTask[Location]` 
  and `weather(l: Location]: AppTask[Forecast]` and the code:
```scala
 def forecast(city: String) : AppTask[Double] =
 for {
   location <- lookupCity(city)
   forecast <-  weather(location)
   temp <-  ZIO.fromOption(getTemperature(forecast)).mapError(_ => "No temp in forecast")
 } yield (temp)

```

Similarly, syntax can be added to convert `Option` and other types to ZIO success/error.

# Conclusion

The developers of an application may choose to keep the effect type polymorphic
throughout the whole application. Today the cats typeclasses are de facto standard, so
the simplest polymorfic type would be:
   `F[_] : Sync`. Both `IO` and `ZIO` provide implementation of the `cats.еffect.Sync` type class, and
   of few more needed for asynchronous I/O.
   
The cost to pay for keeping compatibility with any implementation, is the extra steps needing to 
handle error handling flows - monad transformers, as shown above, or maybe MTL. If, on the other hand,
the application commits to ZIO, the bifunctor approach to errors results in code
that is much simpler to write and read. There are some performance gains too, but my guess is they would
not be noticeable in a typical business application.

## References

[1] [Cats Type Classes](https://typelevel.org/cats/typeclasses.html)
[2] [ZIO](https://zio.dev/docs/overview/overview_index)



