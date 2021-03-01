---
title: "Church Numerals in Scala 3"
date: 2021-02-28T19:00:00-05:00
draft: false
---

Modeling church numerals as functions in most languages is straightforward. 
Scala's type system is powerful enough that we can represent them _and_ express 
computation with them directly in the type system!

The definitions in Scala follow immediately from the definitions in the untyped
lambda calculus; functions in the latter correspond to typelevel functions in
the former.

```scala
object ChurchNumerals extends App {

  type zero[s[_], z] = z
  type succ[m[s[_], z], s[_], z] = s[m[s, z]]

  type one[s[_], z] = succ[zero, s, z]
  type two[s[_], z] = succ[one, s, z]

  type plus = [m[s[_], z]] =>> [n[s[_], z]] =>> [s[_], z] =>> m[s, n[s, z]]
  type +[m[s[_], z], n[s[_], z]] = [s[_], z] =>> plus[m][n][s, z]
  
  type times = [m[s[_], z]] =>> [n[s[_], z]] =>> [s[_], z] =>> m[[z0] =>> n[s, z0], z]
  type *[m[s[_], z], n[s[_], z]] = [s[_], z] =>> times[m][n][s, z]

  trait Succ[T]
  trait Zero

  trait Nat[T]:
    def real: Int

  given natForSucc[T](using N: Nat[T]): Nat[Succ[T]] with
    override def real: Int = N.real + 1

  given natForZero: Nat[Zero] with
    override def real: Int = 0

  def real[T](using N: Nat[T]): Int =
    N.real
    
  type expr[s[_], z] = (one + two)[s, z]
  type expr2[s[_], z] = (expr * expr)[s, z]

  println(real[expr2[Succ, Zero]]) // 9
  
}
```

It's been possible to write this out in Scala 2 for a long time
(see [https://michid.wordpress.com/2008/04/18/meta-programming-with-scala-part-i-addition/](this post)), 
but the presence of type lambdas in Scala 3 lets us defer type application
until we need to "perform evaluation".
