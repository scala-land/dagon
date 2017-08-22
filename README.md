## Dagon

> Dagon [...] is an ancient Mesopotamian Assyro-Babylonian and Levantine
> (Canaanite) deity. He appears to have been worshipped as a fertility
> god in Ebla, Assyria, Ugarit and among the Amorites. The Hebrew Bible
> mentions him as the national god of the Philistines with temples at
> Ashdod and elsewhere in Gaza.
>
> -- [Dagon Wikipedia entry](https://en.wikipedia.org/wiki/Dagon)

### Overview

Dagon is a library for rewriting
[directed acyclic graphs](https://en.wikipedia.org/wiki/Directed_acyclic_graph)
(i.e. DAGs).

### Quick Start

### Details

Dagon is 

```scala
object Example {

  import com.stripe.dagon._

  sealed trait Eqn[T] {
    def unary_-(): Eqn[T] = Negate(this)
    def +(that: Eqn[T]): Eqn[T] = Add(this, that)
    def -(that: Eqn[T]): Eqn[T] = Add(this, Negate(that))
  }

  case class Const[T](value: Int) extends Eqn[T]
  case class Var[T](name: String) extends Eqn[T]
  case class Negate[T](eqn: Eqn[T]) extends Eqn[T]
  case class Add[T](lhs: Eqn[T], rhs: Eqn[T]) extends Eqn[T]
  
  object Eqn {
    def negate[T]: Eqn[T] => Eqn[T] = Negate(_)
    def add[T]: (Eqn[T], Eqn[T]) => Eqn[T] = Add(_, _)
  }

  val toLiteral: FunctionK[Eqn, Literal[Eqn, ?]] =
    Memoize.functionK[Eqn, Literal[Eqn, ?]](
      new Memoize.RecursiveK[Eqn, Literal[Eqn, ?]] {
        def toFunction[T] = {
          case (c @ Const(_), f) => Literal.Const(c)
          case (v @ Var(_), f) => Literal.Const(v)
          case (Negate(x), f) => Literal.Unary(f(x), Eqn.negate)
          case (Add(x, y), f) => Literal.Binary(f(x), f(y), Eqn.add)
        }
      })

  object SimplifyNegation extends PartialRule[Eqn] {
    def applyWhere[T](on: ExpressionDag[Eqn]) = {
      case Negate(Negate(e)) => e
      case Negate(Const(x)) => Const(-x)
    }
  }

  object SimplifyAddition extends PartialRule[Eqn] {
    def applyWhere[T](on: ExpressionDag[Eqn]) = {
      case Add(Const(x), Const(y)) => Const(x + y)
      case Add(Add(e, Const(x)), Const(y)) => Add(e, Const(x + y))
      case Add(Add(Const(x), e), Const(y)) => Add(e, Const(x + y))
      case Add(Const(x), Add(Const(y), e)) => Add(Const(x + y), e)
      case Add(Const(x), Add(e, Const(y))) => Add(Const(x + y), e)
    }
  }

  val rules = SimplifyNegation.orElse(SimplifyAddition)

  val a: Eqn[Unit] = Var("x") + Const(1)
  val b1 = a + Const(2)
  val b2 = a + Const(5) + Var("y")
  val c = b1 - b2

  val simplified: Eqn[Unit] =
    ExpressionDag.applyRule(c, toLiteral, rules)
}
```

### Future Work

### Copyright and License

Dagon is available to you under the [Apache License, version 2](https://www.apache.org/licenses/LICENSE-2.0).

Copyright 2017 Stripe.

Derived from [Summingbird](https://github.com/twitter/summingbird), which is copyright 2013-2017 Twitter.
