---
layout: post
title:  Diving into Scala's Typeclasses
categories: scala
tags: [scala, functional programming, typeclasses]
---


Typeclasses are one of the most beautiful ways of extending existing classes and building new features and functionality over them. 
It is always considered a good practise to evolve programs/systems by extending and not altering them. 
There are also situations where you can't change the classes. Typeclasses come to our rescue in such cases.

Typeclasses are a feature in both Haskell and Scala. Haskell provides it as a language feature and its usecase is to achieve polymorphism(ad-hoc polymorphism). Many claim that typeclasses in scala are way more powerful because being an OO language with a very strong polymorphism through subtyping, it can combine typeclasses to add specialized functionality to this inheritance chain. Nope, you can't do that with typeclasses. In other words, typclasses are not like interfaces. And we will discuss why I believe so in this blog post. 

*Inheritance* is one of the key features of Object oriented programming to add new functionality. However, if we want to add new features to classes that we can't extend, we generally follow one of the following methods.  
Let us assume we need to a write a logger that prints a given variable along with its type.  

**1. Pattern match against type**    
{% highlight scala %}
object TypeLogger {
  def convert[T](a: T) = a match {
    case _: Int => "Int " + a.toString
    case _: Double => "Double " + a.toString
  }
  def log[T](a: T) = println(convert(a))
}

import TypeLogger._
log(1)
log(1.01)
{% endhighlight %}

**2. Writing wrapper classes**  
{% highlight scala %}
object TypLogger1 {
  trait Loggable {
    def convert: String
    def log = println(convert)
  }
  case class IntLogger(a: Int) extends Loggable {
    override def convert: String = "Int "+ a.toString
  }
  case class DoubleLogger(a: Double) extends Loggable {
    override def convert: String = "Double "+a.toString
  }
}

import TypLogger1._
IntLogger(1).log
DoubleLogger(1.01).log
{% endhighlight %}
But even here, you might have to pattern match to find the Logger for that type.

**3. Implicits**   
Scala provides an easier way of doing this by hiding such conversions behind the scenes using implicit conversions.
{% highlight scala %}
object TypeLogger2 {
  trait Loggable {
    def convert: String
    def log = println(convert)
  }
  implicit class Int2String(a: Int) extends Loggable {
    def convert = "Int " + a.toString
  }
  implicit class Double2String(a: Double) extends Loggable {
    def convert = "Double "+a.toString
  }
}

import TypeLogger2._
1.log
1.01.log
{% endhighlight %}
You could have seen something similar in scala.concurrent.duration 
{% highlight scala %}
import scala.concurrent.duration._
10 seconds
10.seconds
{% endhighlight %}
Yes, that is also achieved using implicit conversions.

Let's solve the same using typeclasses, 
{% highlight scala %}
object TypeLogger3 {
  trait Loggable[T] {
    def convert(a: T): String
    def log(a: T) = println(convert(a))
  }
  implicit object Int2String extends Loggable[Int] {
    def convert(a: Int) = "Int " + a.toString
  }
  implicit object Double2String extends Loggable[Double] {
    def convert(a: Double) = "Double " + a.toString
  }
  object Loggable{
    def apply[T: Loggable]: Loggable[T] = implicitly
  }
  implicit class LoggableOps[T: Loggable](a: T) {
    def log = Loggable[T].log(a)
  }
}

import TypeLogger3._
1.log
1.01.log
{% endhighlight %}
You can clearly see that you could achieve the same with implicits. What special is achieved by using typeclass then?

Another wrong assumption is that typeclasses can be used to add specialized functionality to inheritance chain which is partly true but not exactly the way one would think regular subtyping in inheritance works.
Let us take an example of the AnimalKingdom,
{% highlight scala %}
object AnimalKingdom {
  trait LivingBeing
  trait Animal extends LivingBeing
  case class Dog(name: String) extends Animal
  case class Cat(name: String) extends Animal
}

object SoundImplicits {
  trait CanMakeSound[T]{
    def makeNoise(t: T): String
  }
  implicit object CatCanMakeSound extends CanMakeSound[Cat] {
    override def makeNoise(t: Cat): String = "meow"
  }
  implicit object DogCanMakeSound extends CanMakeSound[Dog] {
    override def makeNoise(t: Dog): String = "woof"
  }
  object CanMakeSound {
    def apply[A: CanMakeSound]: CanMakeSound[A] = implicitly
  }
}

object God {
  import AnimalKingdom.Animal
  def assignCharacteristicSound(animals: List[Animal]) = {
    import SoundImplicits._
    //animals.map(animal => CanMakeSound[]) 
    //Can't do. It looks for an implementation CanMakeSound[Animal]
  }

  def main(args: Array[String]): Unit = {
    val dog = Dog("Snowie")
    val cat = Cat("Garfield")
    assignCharacteristicSound(List(dog, cat))
  }
}
{% endhighlight %}
Implicits are resolved at compile-time. However, subtyping relies on dynamic-dispatching. As a result, the new God here cannot give the animals their characteristic sound. It should have been well planned during design time(here, I mean during the creation of the animals in the animalkingdom).

In some examples I see something similar to this being provided.
{% highlight scala %}
def main(args: Array[String]): Unit = {
  val dog = Dog("Snowie")
  val cat = Cat("Garfield")

  import SoundImplicits._
  println(dog.makeNoise)
  println(cat.makeNoise)
}
{% endhighlight %}
This seems to work, cause we know the dog is of type Dog, cat is of type Cat, and the compiler can pick the `CanMakeSound[Dog]` and `CanMakeSound[Cat]`. However, we are actually explicitly providing the subclass Type, which defeats the entire purpose of inheritance of object-oriented programming.

From the above, I don't meant that typeclassess aren't useful in scala. Typeclasses are definitely useful but not in ways we generally think of polymorphism in OOPS. Taking the same AnimalKingdom example, we can add few powers to the new God.
{% highlight scala %}
object SleepImplicits {
  import AnimalKingdom.LivingBeing
  trait CanSleep[T]{
    def sleep(t:T): String
  }
  implicit object LivingBeingsCanSleep extends CanSleep[LivingBeing] {
    override def sleep(t: LivingBeing): String = "zzz"
  }
  object CanSleep {
    def apply[A: CanSleep]: CanSleep[A] = implicitly
  }
}

object God {
  import AnimalKingdom.Animal
  def putToSleep(animals: List[Animal]) = {
    import SleepImplicits._
    animals.map(animal: Animal => CanSleep[LivingBeing].sleep(animal)) //Can do, since Animals are LivingBeings
  }

  def main(args: Array[String]): Unit = {
    val dog = Dog("Snowie")
    val cat = Cat("Garfield")
    putToSleep(List(dog, cat)).foreach(println)
  }
}
{% endhighlight %}
This works, cause you have an sleep implementation available for all livingbeings. So, the new God can put all the animals to sleep.  
Another classic example from scala library is
{% highlight scala %}
def sorted[B >: A](implicit ord: Ordering[B]): Seq[A]
{% endhighlight %}

Ordering is used to resolve the relative ordering between two items of type A.

`Ordering[B]` is a typeclass here. Different implementations of compare function are available as implicits in the scope of the sorted method call.
`B >: A` means B is a supertype of A. Thus if you provide an Ordering of Numbers and Int extends Numbers, meant you could sort a List of Int(s).
