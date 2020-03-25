---
layout: post
title: State Monads
categories: [Blogging, Scala]
tags: [scala, functional programming, monad, state monad]
seo:
  date_modified: 2020-03-25 06:51:27 -0300
---
Scala is a OOP language with full support for functional programming and beautifully mixes them together to bring the best from both the worlds.  

### Pure Functions
In functional paradigm, all your functions are pure, meaning you don't do any side effecting operations. 
But what about your memory/state? That definitely has to be mutable. If it is mutable, then the function that mutates it tends to be impure/side effecting. 
But you can solve this problem of state updates in Scala in a very pure way. We will see how.

### Program State
For the entirety of this discussion, we will consider a program/function evaluation in any general programming language. 
As variables are assigned values, they are available for reuse(in their scope/closure).
So, we could say that these variables form the state of the program and are updated as and when they are reassigned which we call state updates.

We will design a programmming language and for simplicity it has only the following expressions and statements.

{% highlight scala %}
sealed trait Expr
case class Value(value: Int) extends Expr
case class Variable(name: String) extends Expr
case class Add(lhs: Expr, rhs: Expr) extends Expr

trait Statement
case class Assignment(into: String, rhs: Expr) extends Statement
case class Return(ret: Expr) extends Statement
case class Print(expr: Expr) extends Statement
{% endhighlight %}

*Assignment* assigns a value into a variable and updates state. *Print* evaluates an expr and prints it. *Return* evaluates and returns an expression. The later two don't update state.   
(Do not confuse Return in our DSL with Scala's *return* statement)

Our program has a collection of these statements.
{% highlight scala %}
case class Program(statements: List[Statement])

def main(args: Array[String]): Unit = {
  val program = Program(List(
    Assignment("a", Add(Value(1), Value(1))),
    Print(Add(Variable("a"), Value(1))),
    Assignment("b", Variable("a")),
    Return(Add(Variable("b"), Value(2)))
  ))
}
{% endhighlight %}


### Program Evaluation
Let's start with building the program evaluator in object-oriented style with side-effects and iteratively progress towards making them pure.

The variables and their values are maintained in the program's state.
{% highlight scala %}
case class Program(statements: List[Statement]) {
  val state: Map[String, Any]
}
{% endhighlight %}
However, we need to update the state and thus we have to make the state a **var**. Since, we have state changes, let's make it a class rather than a case class.  
*Note: Case class are classes where the state is the identity. This is why it performs equality on its members(state) while java's default equality can check only on reference(identity).*

{% highlight scala %}
type Environment = Map[String, Expr]
class Program(statements: List[Statement]) {
  var state: Environment
}
{% endhighlight %}

Adding evaluation logic.
{% highlight scala %}
sealed trait Expr {
  def eval(env: Environment): Int
}
case class Value(value: Int) extends Expr {
  override def eval(env: Environment): Int = value
}
case class Variable(name: String) extends Expr {
  override def eval(env: Environment): Int = env(name).eval(env)
}
case class Add(lhs: Expr, rhs: Expr) extends Expr {
  override def eval(env: Environment): Int = lhs.eval(env) + rhs.eval(env)
}

sealed trait Result
case class Integer(value: Int) extends Result
case object Void extends Result // Similar to scala's unit. 
{% endhighlight %}
We use Void here cause we don't want to confuse with another unit function that we'll talk about.
Since, state is available in Program class, we need to add state updates in the Program class.

{% highlight scala %}
class Program(statements: List[Statement]) {
  var state: Environment = Map.empty
  def evaluate: List[Result] = {
    statements.map(st => st match {
     case assign: Assignment => {
      state = state + (assign.into -> Value(assign.rhs.eval(state)))
      Void
     }
     case returnSt: Return => Integer(returnSt.ret.eval(state))
     case print: Print => {
      println("Printing " + print.expr.eval(state))
      Void
     }
     case _ => Void
   })
  }
}

program.evaluate()
{% endhighlight %}

That gives us the results from each statement as well as the final state after the program has been executed.

Having the evaluation logic in program doesn't appear good. We would rather have it in their respective classes. 
Now, we have two ways to solve the state updates.  
1. Update/mutate the enviroment inside the evaluation  
2. Return the new environment   
Since mutation defeats the purpose of pure function, let's return a tuple of (Result, Environment)

{% highlight scala %}
trait Statement {
  def execute(env: Environment): (Result, Environment)
}
case class Assignment(into: String, rhs: Expr) extends Statement {
  override def execute(env: Environment): (Result, Environment) = (Unit, env + (into -> Value(rhs.eval(env))))
}
case class Return(ret: Expr) extends Statement {
  override def execute(env: Environment): (Result, Environment) = (Integer(ret.eval(env)), env)
}
case class Print(expr: Expr) extends Statement {
  override def execute(env: Environment): (Result, Environment) = {
    println("Printing " + expr.eval(env))
    (Unit, env)
  }
}

case class Program(statements: List[Statement]) {
  def evaluate = statements.foldLeft((List[Result](), Map[String, Expr]()))
  ((soFar, cur) => {
    val (res, newEnv) = cur.execute(soFar._2)
    (soFar._1++List(res), newEnv)
  })
}
{% endhighlight %}

### Composition 
In order to be give the ability for our execution environment to easily combine various state updates, we need to make our state update actions composable and define combinators to combine these actions. 

The execute function is of the form `Environment => (Result, Environment)` and let's pull it out into a type.
{% highlight scala %}
type StateAction = Environment => (Result, Environment)
{% endhighlight %}

This function could be applied for any **State** and **Result**, so let's parametrize.
{% highlight scala %}
type StateAction[S, A] = S => (A, S). 
{% endhighlight %}

We will also rename the execute function into something like build, cause we are actually returning functions and not evaluating at this stage.
{% highlight scala %}
trait Statement {
  def build: StateAction[Environment, Result]
}
{% endhighlight %}

There we go. Lets look into the foldLeft in our Program class.
{% highlight scala %}
(List[Result], Enviroment)(f: List[Result] => (List[Result], Environment))
{% endhighlight %}

This is same as 
{% highlight scala %}
(M[A])(f: A => M[B])
{% endhighlight %}
where `List[Result]` is `A` and `(List[Result], Enviroment)` is `M[A]`. Yes, that's your `flatMap` syntax and `M` is your Monad. 

### Monadic Combinators
Great, we have identified the most important combinator on our StateAction i.e. the *flatMap*. 
{% highlight scala %}
def flatMap[S, A, B](sa: StateAction[S, A])(f: A => StateAction[S, B]): StateAction[S, B] = state => {
 val (a, state2) = sa(state)
 f(a)(state2)
}
{% endhighlight %}

The unit combinator returns the given result without updating the state.
{% highlight scala %}
def unit[S, A](a: A): StateAction[S, A] = s => (a, s)
{% endhighlight %}

We now have been able to define the two primitive functions for a Monad and thus, our stateAction is a Monad or a "State Monad". 
Let us make these functions to be available as methods by pulling the StateAction into a class.
{% highlight scala %}
case class State[S, A](run: S => (A, S)) {
  def flatMap[B](f: A => State[S, B]): State[S, B] = State(s => {
    val (a, s2) = this.run(s)
    f(a).run(s2)
  })
}

def unit[A, S](a: A): State[S, A] = State(s => (a, s))
{% endhighlight %}

Implementing map in terms of the above two.
{% highlight scala %}
def map[B](f: A => B): State[S, B] = flatMap(a => unit(f(a))) 
{% endhighlight %}
Our program now is
{% highlight scala %}
val compiled = for {
   _ <- Assignment("a", Add(Value(1), Value(1))).build
   s <- Return(Add(Variable("a"), Value(1))).build
   _ <- Print(Add(Variable("a"), Value(1))).build
  } yield s
 
 compiled.run(Map[String, Expr]())
{% endhighlight %}
We have also seperated the build and the run phase. It is possible to add more statements to the compiled version, compose others and finally run it with the initial state.

Now we have built powerful combinators of *flatMap*, *map*, *unit* and used scala's *for-comprehension* to build bigger abstractions. However, with **for** we have to add each statements seperately. Since we are in the inheritance world and also our program provides us a list of statements(we don't know each specialized type), is there a generic way to compose this list of statements?

### Sequence 
Given a list of stateActions, we should be able to compose them and return a final State[**S**] and an accumulated Result[**A**] of each action.
{% highlight scala %}
def fn[A, S](stateActions: List[State[S, A]]): State[S, List[A]]
{% endhighlight %}
We call this operation *sequence*. 
{% highlight scala %}
def sequence[A, S](list: List[State[S, A]]): State[S, List[A]] = State(s => 
  list match {
    case Nil => (List.empty, s)
    case h::t => 
      h.flatMap(head => sequence(t).map(tail => head::tail)).run(s)
})
{% endhighlight %}
Our program reduces to
{% highlight scala %}
case class Program(statements: List[Statement]) {
  import StateMonad._
  private def build = for {
    responses <- sequence(statements.map(_.build))
  } yield responses.last //We generally return the last evaluated value.

  def evaluate = build.run(Map[String, Expr]())
}

val compiled = program.build
compiled.run(Map[String, Expr]())
{% endhighlight %}
      
And that's it. We built our Programming Enviroment as a **State Monad**. Any problem that actually involves state updates can be solved without side effects using our State Monad `State[S, A]` by parametrizing on `S` and `A` to suit the problem. Using the combinators available on this Monad, we can easily compose different functionalities to form newer ones.