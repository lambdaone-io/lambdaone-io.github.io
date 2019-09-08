---
title: io-errors-handle
date: 2019-09-06 21:05:38
tags:
- IO
- Side Effects
- Error Handling
- Pure Functional Programming
---

# Error Handling with ZIO in Pure Functional Applications

In pure functional programming, functions are total - every input produces output. For example, `toString[Int] : String` 
produces a `String` form every `Int`. As not every `String` can be converted to an `Int`, we can use `Option[int]` to
return `None` when no conversion is possible, 
or `Either[Error, Int]` to provide error information on failure. - every `String` produces either an `Error` or an `Int`. 

Typically, such computations are being chained - the result from one computation is used as input for the next one. 
For example, given a city name, we first look up its coordinates, then we look up the weather forecast for this location. 
As last step, we do not need need the whole forecast but only the temperature, which may be present or not.
Sequential computations are represented by monads. Scala offers special syntax to make them more readable.
For example, given `lookupCity(city: String]: Either[Error, Location]`,
 ``weather(l: Location): Either[Error, Forecast]`` and ``getTemperature(f: Forecats): Option[Double]``,
we can sequence them to `forecast(city: String) : Either[Error, Forecast]` with
```
def forecast(city: String) : Either[Error, Forecast] =
 for {
  location <- lookupCity(city)
  forecast <- weather(location)
  temp <- getTemperature(forecast).toEither("No temp in forecast")
 } yield (forecast)
```

Monads representing error situations shortcut on error - once there is `None`, or `Left`, this value gets passed 
directly to `yield`.

Real applications rarely can be modeled only with the `Option` or `Either` types - they include 
interactions with the world, which mail fail.
For example, if we call a remote service to look up the city, the call itself may fail, even 
with valid city name as input. Think of a network error, or a database error. Intereactions with the worlds
are modelled by `IO`, which is a monad itself - chaining effects and short-cutting on failure
can be coded using the same for comprehension syntax. 

If 
we forget business errors and only think of `IO` errors, given 
`lookupCity(city: String]: IO[Location` and ``weather(l: Location]: IO[Forecast]``,
we can do: 
```
def forecast(city: String) : IO[Forecast] =
for {
  location <- lookupCity(city)
  forecast <- weather(location)
  temp <- getTemperature(forecast).map(IO.pure(_)).getOrElse(IO.fail(new Exception("No temp in forecast"))
} yield (forecast)
```
`forecast` gives a value of type `IO` which describes the computation. To get the result,we have to 
execute it. `IO` can end in either success or failure and in case of failure `flatMap` chains get short-circuited.

`IO` shortcuts because it is a `MonadError` besides being a monad. We could abstract this to:
```
def forecast[F[_] : Monad : MonadError] (city: String) : F[Forecast] =
for {
  location <- lookupCity(city)
  forecast <- weather(location)
  temp <- getTemperature(forecast).map(IO.pure(_)).getOrElse(IO.fail(new Exception("No temp in forecast"))
} yield (forecast)
```

Most real applications will have to handle both types of errors - IO errors and business logic errors. If
we replace `IO` with a generic `F` effect, the type becomes `F[Either[Error, A]]` - nested effects.

If we try to handle both effects errors ourselves, things can get ugly fast.
Let's see what happens when we have `lookupCity(city: String]: F[Either[Error, Location]]` and 
`weather(l: Location]: F[Either[Error, Forecast]]`. The external monad errors are handled
by the for comprehension, but we have to take care of the internal one:

## Plain Scala

```
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
Yes, things can get ugly fast this way.

## Monad Transformers

As we see in the last piece of code, monads do not combine automatically - we need to handle the inner monad
ourselves. Monad transformers can do the inner monad handling for us. Each monad requires a transformer knowing how to
flatMap this particular type.
 For `F[Either[Error,A]]` we need the `EitherT` transformer - both `cats` and `scalaz` have them. For `cats` documentation:
 _`EitherT[F, A, B]` wraps a value of type `F[Either[A, B]]`. An `F[C]` can be lifted in to `EitherT[F, A, C]` via `EitherT.right`,
 and lifted in to a `EitherT[F, C, B]` via `EitherT.left`._
 Let's see what our example looks like:
 ```
 def forecast(city: String) : F[Either[Error, Double]] =
 for {
   location <- EitherT(lookupCity(city))
   forecast <- EitherT(weather(location))
   temp <- EitherT.fromEither(getTemperature(forecast).toEither("No temp in forecast"))
 } yield (temp)
```
 
Much better! This can be further improved by introducing some syntax, e.g.:

```
type ErrorOr[E,A] = EitherT[F, E, A]

  implicit class feitherToErrorOr[E, A](r: F[E Either A]) { def toErrorOr: ErrorOr[E, A] = EitherT(r) }
  implicit class optionToErrorOr[E, A](o: Option[A], e: Error) { def toErrorOr: ErrorOr[E, A] = EitherT.fromEither(r.toEither(e) }
```
the code above becomes:

 ```
 def forecast(city: String) : ErrorOr[Double] =
 for {
   location <- lookupCity(city).toErrorOr
   forecast <-  weather(location).toErrorOr
   temp <-  getTemperature(forecast).toErrorOr("No temp in forecast"))
 } yield (temp)
```

The code above looks much nicer. In fact, I saw this on one of my first Scala projects and I was impressed by its power and purity. 

## ZIO

`ZIO` is an alternative IO implementation. Its approach to errors differs from cats `IO`. 
Cats has one type parameter - `IO[A]` returns a value of type `A`. It still shortcuts on error by inheriting `MonadError[E]`. 
`ZIO` chooses to be explicit about the error. It is defined as
 `ZIO[R,E,A]`.  ZIO is the effectful version of `R => Either[E, A]` - given R, on execution ZIO returns either a value of type 
 `A` or an error of type `E`. Where `IO[A]` is functorial in `A`, `ZIO[R, E, A]` is functorial in both `E` and `A`. 
 
 If we decided we don't need `R` for now, ZIO provides `type IO[+E, +A] = ZIO[Any, E, A]`. 
 For our application, given a specific error type `type Error = String`, we would have 
 `type AppTask[A] = ZIO[Any, Error, A]`. In our ZIO-based application,
  the business functions would be `lookupCity(city: String]: AppTask[Location]` 
  and `weather(l: Location]: AppTask[Forecast]` and the code:
```
 def forecast(city: String) : AppTask[Double] =
 for {
   location <- lookupCity(city)
   forecast <-  weather(location)
   temp <-  ZIO.fromEither(getTemperature(forecast).toEither("No temp in forecast"))
 } yield (temp)

```

# Conclusion

If you are implementing a library and do not want to tie to a specific IO implementation, it probably makes sense to leep 
the code abstract to allow user to select between cats IO, ZIO or other alternatives. In an application code, once you have chosen ZIO 
, the bifunctor approach to error values results in greatly improved readability.



