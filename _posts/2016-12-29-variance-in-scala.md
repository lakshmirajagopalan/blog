---
layout: post
title:  Variance in Scala
categories: scala
tags: [scala, functional programming, variance]
---


There are different types of polymorphism in Scala. Inheritance, Parametric polymorphism (Generics in Java), etc.
We are concerned about Parametric Polymorphism in this post and a classic example would include the containers, `List[+T]`.

A List has a type parameter 'T' meaning you could create a list of Ints, Doubles, String, People, Universes.
Wait, but what is this little '+' that gets prepended to the type parameter? Enter Variance.

### Variance
Prepending with a '+' means it is covariant and with a '-' means contravariant.
Let's discuss both them with examples.
Imagine a Fruit Orchard.
{% highlight scala %}
object Orchard {
  trait Fruit {
    def name: String
  }
  case class Apple(name: String = "Apple") extends Fruit
  case class Orange(name: String = "Orange") extends Fruit
  case class Basket[+T](item: T)
}
{% endhighlight %}

For simplicity, let us assume that a basket contains only one item.
We will get one apple basket and an orange basket from the Orchard
{% highlight scala %}
object Community {
  import  Orchard._
  def main(args: Array[String]) = {
    val aBasket = Basket(Apple())
    val oBasket = Basket(Orange())
  }
}
{% endhighlight %}

Our community have lots of fruit lovers who would love to eat the fruits from their orchard.
{% highlight scala %}
object Community {
  case class FruitLover(name: String) {
    def take(fruitBasket: Basket[Fruit]): Unit = println(name + " ate " + fruitBasket.item.name + " from " + fruitBasket)
  }
  def main(args: Array[String]) = {
    ...
    FruitLover("Sam").take(aBasket)
    FruitLover("Frodo").take(oBasket)
    ...
  }
}
{% endhighlight %}

The FruitLover's "take" method expects a Basket of Fruit. Now since, we have marked Basket as covariant(`Basket[+T]`) it means, that
`Basket[Apple]` or `Basket[Orange]` can be passed for a `Basket[Fruit]`.
A extends B and a function asks for a Container[B]. If in the problem domain passing `Container[A]` for `Container[B]` makes sense, then the Container is covariant.

### Contravariance
We noticed that the apple in the "aBasket" had gone bad and has to be replaced with another apple. Could occur with any Basket, so let's add a replace method into Basket.
{% highlight scala %}
case class Basket[+T](item: T) {
  def replace(another: T): Basket[T] = this.copy(item=another) //compile fails
}
{% endhighlight %}
The compiler throws weird error saying that `covariant type T appears in a contravariant position`.

*Why does this occur?*  
Scala being a pure object oriented language stores functions as object as well. Single argument functions are represented as `trait Function1[-A, +B]`.
A refers to the argument type and B the return type. Why is A contravariant and B covariant?

### Function Subtyping
Just like how we defined subtyping of Basket, we also need to define subtyping for functions.
Functions being first-class in Scala, can be passed as arguments to functions. Thus a subtype of a function refers to those functions
that could be substituted instead of this. From the definition, subtype of single argument functions include those functions,
whose return value is a subtype however, "the argument is a supertype".

Let us extend our example of Basket to have a method "makeJam" that takes a recipe and applies it to the Basket's contents.
{% highlight scala %}
case class Basket[+T](item: T) {
  def makeJam(recipe: T => Jam) = recipe(item)
}
{% endhighlight %}
We have a RecipeStore that has a collection of recipes to make apple and orange jam.
{% highlight scala %}
object RecipeStore {
  val appleJamRecipe = (apple: Apple) => Jam(apple.name) //Just a dummy jam-making function.
  val orangeJamRecipe = (oranges: Orange) => Jam(oranges.name)
}
def main(args: Array[String]): Unit = {
  import RecipeStore._
  aBasket.makeJam(appleJamRecipe)
}
{% endhighlight %}

Now, our RecipeStore also has a magic recipe to make Jam out of any fruit.
{% highlight scala %}
val fruitJamRecipe = (fruit: Fruit) => Jam(fruit.name)
{% endhighlight %}

Meanwhile, we have added a Fuji apple variety into our Orchard and the RecipeStore has a specific recipe for preparing FujiApple Jam.
{% highlight scala %}
case class FujiApple(override val name: String = "FujiApple") extends Apple
val fujiAppleJamRecipe = (fuji: FujiApple) => Jam(fuji.name)
{% endhighlight %}
Now, it should be possible to apply the generic "fruitJamRecipe" to our apple basket. However, our "fujiAppleJamRecipe" is specific to Fuji apples and cannot be used
for preparing jam from any apple.
{% highlight scala %}
aBasket.makeJam(appleJamRecipe) //passes
aBasket.makeJam(fruitJamRecipe) //passes
aBasket.makeJam(fujiAppleJamRecipe) //compile fails
{% endhighlight %}

This is exactly why the argument in Function1 was contravariant[-A]. If someone expects, a function from `Apple => Jam`, it should be possible to pass `Fruit => Jam`, but not `FujiApple => Jam`. A and supertypes of A could be passed.
In Function1, the return value type should be covariant `+R`, because we expect to access some members/features of the returned value, which means that it should be (or extend) R.

Lets get back to the cryptic error of covariant in contravariant position.
{% highlight scala %}
case class Basket[+T](item: T) {
  def replace(another: T): Basket[T] = this.copy(item=another) //compile fails
}
{% endhighlight %}
So, in order to achieve function subtyping, we saw that arg was contravariant `[A-]`.
However, in the context of `Basket[+T]`, T is covariant. Thus, the error.

There is another type of variance called Invariance which means the container is neither covariant nor contravariant.
This is the default type of Variance and most of the classes that we create are generally Invariant.

So, we saw about the three types of variance in Scala and how to identify them in your problem space.
If `A extends B`, and  
1. `Container[A]` could be substituted for `Container[B]`, then the Container is covariant.  
2. `Container[B]` could be substituted for `Container[A]`,  then the Container is contravariant.  
3. Else, if neither relation is maintained, then it is invariant.  

Variance is an idea borrowed from the mathematics field of Category Theory.
We would talk more about that in the subsequent post.
