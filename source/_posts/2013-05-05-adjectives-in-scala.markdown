---
layout: post
title: "Adjectives in Scala"
date: 2013-05-05 05:59
comments: true
categories: scala
---

Last year I had the pleasure of working a period of time for Yoav Hollander (creator of the [e verification language](http://en.wikipedia.org/wiki/E_%28verification_language%29)). During that time, I came across a very nice piece of syntactic sugar in *e*. I don't recall how he used to call it, but I understood it as being all about extracting adjectives from a class, for unambiguous use during instance construction.

I've had it in my mind to try to achieve something similar in Scala. Here is a contrived example in invalid Scala pseudo code:

``` scala

object Length extends Enumeration {
  type Length = Value
  val short, long = Value
}
import Length._

class Book(interesting: Boolean = false, length: Length = short)

val myBook: Book = a long interesting Book // <-- this

```

This syntax could allow some very expressive code when modeling a domain using a few case classes. It could also be made to work well with default parameter values and monoidal zeros, for some additional conciseness.

Notice that this has several requirements/limitations for the class being constructed:

1. parameter-less instantiation, i.e.: zero-arg constructor, default parameter values, or a provided builder function
2. no ambiguity -- adjectives can only be extracted if the can be uniquely associated with a single property of the class. (e.g., having two `Length` would prevent us from using `long` as an adjective)

The pseudo code shown above is impossible to mimic directly in Scala, but I've been getting pretty close so far. 

I started out by trying to build the necessary boilerplate. In the future I hope to add macros for extracting adjectives from classes, which is what's needed to make this truly useful.

At first I only considered case classes with default parameter values as candidates, but I later realized there's nothing to stop this from being applied to mutable classes (by extracting adjectives from e.g., vars, setters, parameterless mutators).

An adjective can be described as a function from Noun to Noun: `type Adj[Noun] = Noun => Noun`.  
A sequence of adjectives (disregarding English terminology...) can itself be regarded as an adjective by folding the adjectives over the noun. See the definition of `a` below:

    object Adjectives {
      type Adj[Noun] = Noun => Noun
      def a[Noun](description: Adj[Noun]*): Adj[Noun] = noun0 =>
        (noun0 /: description) { (noun, adjective) => adjective(noun) }
    }
    import Adjectives._

Let's now define the case class `Book` again, and add some adjectives:

    object Length extends Enumeration {
      type Length = Value
      val short, long = Value
    }
    import Length._

    case class Book(interesting: Boolean = false, length: Length = Length.short)
    
    // in the future I will want to derive these automatically using macros:
    object BookAdjectives {
      def interesting: Adj[Book] = _.copy(interesting = true)
      def long: Adj[Book] = _.copy(length = Length.long)
      def short: Adj[Book] = _.copy(length = Length.short)
    }
    import BookAdjectives._
    
    val book = a (long, interesting) (Book())
    
The `(Book())` at the end looks particularly awkward. This can be fixed by some additional boilerplate:

    import language.postfixOps
    
    object BookAdjectives {
      // ...skipping previous definitions
      implicit class BookBuilder(description: Adj[Book]) {
        def Book = description(Book())
      }
    }
    import BookAdjectives._
    
Which allows us the nicer `val book = a (long, interesting) Book`.

As I mentioned before, this could just as easily be generalized to, e.g., mutable classes with vars:

    class Book {
      var interesting: Boolean = _
      var length: Length = _
    }
    
    import Adjectives._ // same as above
    
    // I hope to also be able to generate these in the future
    object BookAdjectives {
      def interesting: Adj[Book] = book => { book.interesting = true; book }
      def long: Adj[Book] = book => { book.length = Length.long; book }
      def short: Adj[Book] = book => { book.length = Length.short; book }
      implicit class BookBuilder(description: Adj[Book]) {
        def Book = description(new Book)
      }
    }
    import BookAdjectives._
    
    val book = a (long, interesting) Book
    
We can also add default reflection-based instantiation, making `BookBuilder` even more brain-dead:

    import scala.reflect._
    
    trait Adjectives {
      protected implicit class NounBuilder[Noun : ClassTag](description: Adj[Noun]) {
        // TODO: should be made to also support case classes with default parameters
        private def newInstance = implicitly[ClassTag[Noun]].runtimeClass.newInstance.asInstanceOf[Noun]
        def build = description(newInstance)
      }
    }

    object BookAdjectives extends Adjectives {
      implicit class BookBuilder(description: Adj[Book]) {
        def Book = description.build // available from NounBuilder
      }
    }

Though I suspect I can just reify `def Book = new Book` and it will work for both cases (class Book, as well as case class Book with default parameters).

Next post on this subject will include some of the missing macro work.