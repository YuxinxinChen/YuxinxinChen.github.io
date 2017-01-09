---
layout: post
title: "Scala Reading Notes on Jan 2"
data: 2017-01-02
tags: [reading notes, scala]
comments: true
share: false
---
### Scala Study on Jan 2

```scala
object rationals extends App{
        val x = new Rational(1, 3)
//      println(x.toString)
        x.RationaltoString
        val y = new Rational(5,7)
        val z = new Rational(3,2)
        (x-y-z).RationaltoString
        (z+x*y).RationaltoString //don't get surprice by it can find the right sequence to execute. The precedence of an operator is determined by its first character. Scala has a table to define the order of priority precedence. But the use fo prefiex operation means to make clearer and easy read program, don't get sidetracked. 
        (x+y+z).RationaltoString
//      y.add(y).RationaltoString
//      val a = new Rational(1,0)
        val a = new Rational(2)
        a.RationaltoString
}


class Rational(x: Int, y: Int)
{
        require(y!=0, "denominator is zero! What is wrong with you!") // require vs assert: require:illegalArgumentException, assert: AssertionError, require: used to enforce a precondition on the caller of the function, assert: to check the code of the function itself. 
            
        def this(x: Int) = this(x, 1)
        private def gcd (x: Int, y: Int): Int = if (y == 0) x else gcd(y, x % y) // recursive method needs result type
        val numer = x/gcd(x, y) // var or val: passing by value, def: passing by name. Also val numer=x/gcd(x,y) is better than def number=x/gcd(x,y) if numer or denom are frequently called, because val avoid repeating re-computation, and number, denom can be called directly
        val denom = y/gcd(x, y)
            
        def RationaltoString: Unit = println(numer + "/" + denom) // use this because I didnt' install compile on runtime, to I still need to println. 
//      override def toString  = numer + "/" + denom // succeed. 
//      override with def toString: Unit = println (numer + "/" + denom) // unsucceed, because return value type is changed. But it is not a new function and a used nanme with the same signiture, compiler can't tell which function (the toString in string or the toString you define) to use. So you can't use this in this way
//      def toString(f: (Int, Int) => Int) = println(numer/f(numer, denom) + "/" + denom/f(numer, denom)) // succeed
            
//      def add ( r: Rational) = // return value type can be inferred from context, so it is optional in this curcumstance 
//              new Rational(numer*r.denom + denom*r.numer, denom*r.denom) 
        def + ( r: Rational) = // return value type can be inferred from context, so it is optional in this curcumstance 
                new Rational(numer*r.denom + denom*r.numer, denom*r.denom)  // infix operation! most characters are valied for identifiers, so use it! 
                    
//      def neg = new Rational(-numer, denom) //self reference: this.numer = numer, same for denom
        def unary_- = new Rational(-numer, denom) // so to use -x and differetiate from x - y, use unary_- instead. Mark that there is a space after unary_- .

//      def sub(r: Rational) = this.+ (r.neg)
        def - (r: Rational) = this + -r

        def * (r: Rational) = new Rational( numer*r.numer, denom*r.denom )
}
```
* Marked Points:
	* def x = 1 : passing by name; val x = 1 : passing by value. Optimization: How freqenct they are used and any potiential error requiring to use passing by name.
	* Operator charactors and those weird charactors can be used as identifier, so use infix operation is possible. Also precedence of operators are predefined in Scala. 
	* Require or assert function are predefined functions in scala. require: used to enforce a precondition on the caller of the function; assert: to check the code of the function itself. 
	* Recursive method need result type.
	* Read above toStrings to see override.
	* Self reference
	* Use of unary_ to define infix operator.


# H1
## H2
### H3
#### H4
##### H5
###### H6

Alternatively, for H1 and H2, an underline-ish style:

Alt-H1
======

Alt-H2
------



### Scala Notes on Jan 3

```scala
object Exercise1 extends App{
//      val n1 = new IntSet(3, new Empty, new Empty) //  class IntSet is abstract; cannot be instantiated
        val n1 = new NonEmpty(3, Empty, Empty)
        n1.toString_IntSet
        val n2 = n1 incl 4
        n2.toString_IntSet
}

abstract class IntSet {
        def incl(x: Int): IntSet
        def contains(x: Int): Boolean
        def toString_IntSet
        def union(other: IntSet): IntSet
}

/**
class Empty extends IntSet{
        def contains(x: Int) = false
        def incl(x: Int): IntSet = new NonEmpty(x, new Empty, new Empty)
        def toString_IntSet = println(".")
        override def toString = "."
}
*/

/** First optimization: we only need one single empty IntSet, so use: */
object Empty extends IntSet{ 
        def contains(x: Int) = false
        def incl(x: Int): IntSet = new NonEmpty(x, Empty, Empty)
        def toString_IntSet = println(".")
        override def toString = "."
        def union(other: IntSet) = other
}
/** This defines a singleton object called Empty, no other Emyty instances can be created, singleton object are values, so Empty evaluate itself*/
class NonEmpty(elem: Int, left: IntSet, right: IntSet) extends IntSet{
        def contains(x: Int): Boolean = {
                if (x>elem) right contains x
                else if (x<elem) left contains x
                else true }

        def incl(x: Int): IntSet = {
                if (x>elem) new NonEmpty(elem, left, right incl x)
                else if (x<elem) new NonEmpty(elem, left incl x, right)
                else this }

        def toString_IntSet = println("{" + left + elem + right + "}")
        override def toString = "{"+ left + elem + right + "}"
        def union(other: IntSet): IntSet = ((left union right) union other) incl elem
}
```

* dynamic method dispatch model in object-oriented languages: dynamic binding is analogous to higher-order functions in functional languages.
* abstract class
* extends: an object of type Empty and NonEmpty can be used wherever an object of type IntSet is required
* override
* singleton object
* In standalone application in Scala, each application contains an object with a main metho, e.g.

```
     object Hello{
                def main(args: Array[String)] = println("Hello World")
        }
```
* To place a class or object inside a package, use: **package progfun.example** at the top of your source code
* To use a class or object inside a package, use import: **import week3.Rational, import week3.{Rational, Hello}, import week3._**
* Standard Scala Library: <a href="http://www.scala-lang.org/api/current/" > http://www.scala-lang.org/api/current/ </a>
* If a class wants to have several supertypes, use trait keyword: classes, objects and traits can only inherit from one class, but many traits. E.g. **class Square extends Shape with Plannar with Movable**. Traits resemble interfaces in Java, but are more powerful because they can contain fields and concrete methods. But traits cannot have parameters, only class can.
* Nothing is at the bottom of Scala's type hierarchy. It is used to signal abnormal termination and as an element type of empty collections
* Scala exception handling: throw Exc
* Null type: every reference class type also has Null as a value. Null is a subcctype of every class that inherits from object; it is incompatible with subtype of AnyVal
     * val x = null //ok
     * val x: String = null //ok
     * val x: Int = null //not ok

![Class Hierarchy](https://github.com/YuxinxinChen/YuxinxinChen.github.io/tree/master/images/classhierarchy.png "Class Hierarchy")

---
### Scala Study on Jan 4 and my Lyft coupon expired

* You cann't do:

```scala
 def foo(f: Int=>Int)(var a: Double). 
```
Mutating the input parameters is often seen as bad style and makes it harder to reason about code. The reason could be simple: alias problem. so: 

```scala
def foo(f: Int=>Int)(val a: Double)
```
* Type parameters do not effect evaluation in Scala, this is called type erasure

```scala
package lesson4

trait List[T] { // make it scalable and apply to kinds of types lists. Use type parameter: [T]. Not only class, functions also can have type parameters: def singleton[T](elem: T) = new Cons[T](elem, new Nil[T])
        def isEmpty: Boolean
        def head: T
        def tail: List[T]
}

class Cons[T](val head: T, val tail: List[T]) extends List[T]{
        def isEmpty = false
        // head and tail is defined in cons parameters
}

class Nil[T] extends List[T]{
        def isEmpty = true
        def head = throw new NoSuchElementException("Nil.head")
        def tail = throw new NoSuchElementException("Nil.tail")
}
``` 
In other file:

```scala
import lesson4._

object nth extends App{
        def nth[T]( n: Int, xs: List[T]) :T = {
                if (xs.isEmpty) throw new IndexOutOfBoundsException
                else if (n==0) xs.head
                else nth(n-1, xs.tail)
        }
        val list1 = new Cons(1, new Cons(2, new Cons(3, new Nil)))
        println(nth(2, list1))
}
```
---
### Scala Study on Jan 8 and decide to study some relative high level stuff? 

* <a href="https://blog.codecentric.de/en/2015/03/scala-type-system-parameterized-types-variances-part-1/" > Scala type system: parameterized type and variance </a>
* <a href="https://blog.codecentric.de/en/2015/04/the-scala-type-system-parameterized-types-and-variances-part-2/" > Subtype polymorphism and allowed variance annotations </a>
