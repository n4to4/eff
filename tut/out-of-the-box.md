# Out of the box

This library comes with a few available effects (in `org.specs2.control.eff._`):

 Name                | Description
 ------------------- | ---------- 
 `EvalEffect`        | an effect for delayed computations
 `OptionEffect`      | an effect for optional computations, stopping when there's no available value
 `DisjunctionEffect` | an effect for computations with failures, stopping when there is a failure
 `ErrorEffect`       | a mix of Eval and Disjunction, catching exceptions and returning them as failures
 `ReaderEffect`      | an effect for depending on a configuration or an environment
 `WriterEffect`      | an effect to log messages
 `StateEffect`       | an effect to pass state around
 `ListEffect`        | an effect for computations returning several values (for non-determinism)

Each object provides methods to create effects and to interpret them.

## Eval

This effect is a very simple one. It allows the delayed execution of computations and it can serve as some sort of overall `IO` effect.

Two methods are available to execute this effect: 

 - `runEval[R <: Effects, A](r: Eff[Eval[?] |: R, A]): Eff[R, A]` to just execute the computations
 - `attemptEval[R <: Effects, A](r: Eff[Eval[?] |: R, A]): Eff[R, Throwable \/ A]` to execute the computations but also catch any `Throwable` that would be thrown
  
## Option

Adding an `Option` effect in your stack allows to stop computations when necessary. 
If you create a value with `some(a)` this value will be used downstream but if you use `none` all computations will stop
```scala
import org.specs2.control.eff._
import Eff._
import Effects._
import OptionEffect._
import cats.syntax.all._

/**
 * Stack declaration
 */
type S = Option |: NoEffect

implicit def OptionMember: Member[Option, S] =
  Member.MemberNatIsMember

// compute with this stack
val map: Map[String, Int] = 
  Map("key1" -> 10, "key2" -> 20) 

val addKeys: Eff[S, Int] = for {
  a <- fromOption(map.get("key1"))
  b <- fromOption(map.get("key2"))
} yield a + b

val addKeysWithMissingKey: Eff[S, Int] = for {
  a <- fromOption(map.get("key1"))
  b <- fromOption(map.get("missing"))
} yield a + b
```
```scala
scala> run(runOption(addKeys))
res7: Option[Int] = Some(30)

scala> run(runOption(addKeysWithMissingKey))
res8: Option[Int] = None
```

## Disjunction

The `Disjunction` effect is similar to the `Option` effect but adds the possibility to specify why a computation stopped.
```scala
import org.specs2.control.eff._
import Eff._
import Effects._
import DisjunctionEffect._
import cats.syntax.all._
import cats.data.Xor

/**
 * Stack declaration
 */
type XorString[A] = String Xor A 
type S = XorString |: NoEffect

implicit def XorStringMember: Member[XorString, S] =
  Member.MemberNatIsMember

// compute with this stack
val map: Map[String, Int] = 
  Map("key1" -> 10, "key2" -> 20) 

val addKeys: Eff[S, Int] = for {
  a <- fromOption(map.get("key1"), "'key1' not found")
  b <- fromOption(map.get("key2"), "'key2' not found")
} yield a + b

val addKeysWithMissingKey: Eff[S, Int] = for {
  a <- fromOption(map.get("key1"),    "'key1' not found")
  b <- fromOption(map.get("missing"), "'missing' not found")
} yield a + b
```
```scala
scala> run(runDisjunction(addKeys))
res16: cats.data.Xor[String,Int] = Right(30)

scala> run(runDisjunction(addKeysWithMissingKey))
res17: cats.data.Xor[String,Int] = Left('missing' not found)
```

## Error

The `Error` effect is both an `Eval` effect and a `Disjunction` one with `Throwable Xor F` on the "left" side.
 The idea is to represent computation which can fail, either with an exception or a failure. You can:

 - create delayed computations with `ok`
 - fail with `fail(f: F)` where `F` is the failure type
 - throw an exception with `exception`

Other useful combinators are available:

 - `andFinally(last)` register an action to be executed even in case of an exception or a failure
 - `orElse` run an action and then run another one if the first is not successful
 - `whenFailed` same thing than `orElse` but use the error for `action1` to decide which action to run next

When you run an `Error` effect you get back an `Error Xor A` where `Error` is a type alias for `Throwable Xor Failure`.

The `Error` object implements this effect with `String` as the `Failure` type but you are encouraged to create our own 
failure datatype and extends the `Error[MyFailureDatatype]` trait. 

## Reader

The `Reader` effect is used to request values from the "environment". The main method is `ask` to get the current environment (or "configuration" if you prefer to see it that way)
and you can run an effect stack containing a `Reader` effect by providing a value for the environment with the `runReader` method.

It is also possible to stack several independent environments in the same effect stack by "tagging" them:
```scala
import ReaderEffect._
import Tag._
import cats.data._

trait Port1
trait Port2

type R1[A] = Reader[Int, A] @@ Port1
type R2[A] = Reader[Int, A] @@ Port2

type S = R1 |: R2 |: NoEffect

implicit def R1Member: Member[R1, S] = Member.MemberNatIsMember
implicit def R2Member: Member[R2, S] = Member.MemberNatIsMember

val getPorts: Eff[S, String] = for {
  p1 <- askTagged[S, Port1, Int]
  p2 <- askTagged[S, Port2, Int]
} yield "port1 is "+p1+", port2 is "+p2
```
```scala
scala> run(runTaggedReader(50)(runTaggedReader(80)(getPorts)))
res23: String = port1 is 80, port2 is 50
```

## Writer

The `Writer` effect is classically used to log values alongside computations. However it generally suffers from the following drawbacks:

 - values have to be accumulated in some sort of `Monoid`
 - it can not really be used to log values to a file because all the values are being kept in memory until the 
  computation ends
  
The `Writer` effect has none of these issues. When you want to log a value you simply `tell` it and when you run the effect you can select exactly the strategy you want:
  
  - `runWriter` simply accumulates the values in a `List` which is ok if you don't have too many of them
  - `runWriterFold` uses a `Fold` to act on each value, keeping some internal state between each invocation
  
You can then define your own custom `Fold` to log the values to a file:

```scala
import java.io.PrintWriter
import WriterEffect._

type W[A] = Writer[String, A]
type S = W |: NoEffect

implicit def WMember: Member[W, S] = Member.MemberNatIsMember

val action: Eff[S, Int] = for {
 a <- EffMonad[S].pure(1) 
 _ <- tell("first value "+a) 
 b <- EffMonad[S].pure(2) 
 _ <- tell("second value "+b) 

} yield a + b

// define a fold to output values
def fileFold(path: String) = new Fold[String, Unit] {
  type S = PrintWriter
  val init: S = new PrintWriter(path)
  
  def fold(a: String, s: S): S = 
    { s.println(a); s }
    
  def finalize(s: S): Unit = 
    s.close
}
```
```scala
scala> run(runWriterFold(action)(fileFold("target/log")))
res29: (Int, Unit) = (3,())

scala> io.Source.fromFile("target/log").getLines.toList
res30: List[String] = List(first value 1, second value 2)
```

## State

A `State` effect can be seen as the combination of both a `Reader` and a `Writer` with these operations:

 - `get` get the current state
 - `put` set a new state
 
Let's see an example showing that we can also use tags to track different states at the same time:
```scala
import StateEffect._
import cats.state._

trait Var1
trait Var2

type S1[A] = State[Int, A] @@ Var1
type S2[A] = State[Int, A] @@ Var2

type S = S1 |: S2 |: NoEffect

implicit def S1Member: Member[S1, S] = Member.MemberNatIsMember
implicit def S2Member: Member[S2, S] = Member.MemberNatIsMember

val swapVariables: Eff[S, String] = for {
  v1 <- getTagged[S, Var1, Int]
  v2 <- getTagged[S, Var2, Int]
  _  <- putTagged[S, Var1, Int](v2)
  _  <- putTagged[S, Var2, Int](v1)
  w1 <- getTagged[S, Var1, Int]
  w2 <- getTagged[S, Var2, Int]
} yield "initial: "+(v1, v2).toString+", final: "+(w1, w2).toString
```
```scala
scala> run(evalTagged(50)(evalTagged(10)(swapVariables)))
res36: String = initial: (10,50), final: (50,10)
```

In the example above we have used `eval` methods to get the `A` in `Eff[R, A]` but it is also possible to get both the
 value and the state with `run` or only the state with `exec`.
 
## List
 
The `List` effect is used for non-deterministic computations, that is computations which may return several values.
 A simple example using this effect would be:
```scala
import ListEffect._

type S = List |: NoEffect

implicit def ListMember: Member[List, S] = Member.MemberNatIsMember

// create all the possible pairs for a given list
def pairsBiggerThan(list: List[Int], n: Int): Eff[S, (Int, Int)] = for {
  a <- values(list:_*)
  b <- values(list:_*)
  found <- 
    if (a + b > n) singleton((a, b))
    else empty
} yield found

```
```scala
     | 
     | run(runList(pairsBiggerThan(List(1, 2, 3, 4), 5)))
res43: List[(Int, Int)] = List((2,4), (3,3), (3,4), (4,2), (4,3), (4,4))
```