# Recursion Schemes in Scala

---

In this piece of work, I want to share some notions about recursion schemes. Recursion schemes are a paradigm based on the abstraction of the recursive steps and the base cases. I will try to provide some justifications about its utilities and a lot of examples where it can be useful. All the examples can be used as scala snippets and feel free to play with it in [Scastie](https://scastie.scala-lang.org/), an online scala editor. Enjoy!

## Introduction

---

I'm sure that all of you, in some way, have heard about recursion. The idea of recursion lies in the usage of base definition and some rules to build new structures based on the previous one. In this context, inductive and recursive constructions can be seen as points of view of the same idea. I prefer to talk about induction whenever I talk about data types and recursion referred to functions that traverse inductive types. One of the main examples are natural numbers, in mathematics, and lists in computer science. In both cases we have a base case that tell us an example of an element of our structure and rules to produce new elements. In the case of natural numbers, the base case is _0 is a natural number_ and for the case of Lists, Nil is a List. The next step is to provide a rule to build, inductively, new numbers and new lists. For the first one we have the basic rule _every natural number has a next element_ and, for lists, _every list is Nil or has a head of type A and a tail of type `List[A]`_. A simple implementation of these two types in Scala is:

```scala
sealed trait Nat
final object Zero extends Nat
final case class Suc(n: Nat) extends Nat

sealed trait List[+A]
final object Nil extends List[Nothing]
final case class Cons[A](head: A, tail: List[A]) extends List[A]
```

In both cases we can see the recursive strategy in the `Cons` and `Suc` definitions, with an argument defined by its own type. 

With every structure builded inductively, there are functions acting on them. Some of these functions are known as recursive functions, which are defined by specifying the image of the base cases and the image of a generic inductive step. For example, lets define the length of a list:

```scala
def length[A]: List[A] => Int = {
    case Nil => 0
    case Cons(h, t) => 1 + length(t)
} 
```
We must read this definition as: _if the given list is Nil, its length equals 0 and, suposed we know the length of the tail, this value plus one gives the full length of the list_.
Indeed, we can follow every step in the recursive calls to find the length of a given List. 

With these examples we can observe two behaviors:

   1) Every inductive structure uses its own type to define the rule to build the next iteration.
    
   2) Every recursive function with domain an inductive data type needs to traverse the structure until the base case and them iterates to produce the result.
   
These observations are pretty well defined over the previous examples. But, in general, the definition of recursive functions needs to use the recursive calls in some key points and, in so many cases, these points are not so clear. This may cause errors during the definition of recursive functions. For this reason, it would be interesting to have a clear separation between the recursion step and the base cases. In fact, what we will try to build is a blueprint for general recursion. The shape of this blueprint can be resumed as:

   1) Give me a functor `F[A]` and a way to evaluate it for a fixed type B (think in the evaluation of an expression to an integer). `F[A]` will represent a general inductive step, but without any reference to previous cases. The role of the evaluation can be seen as a single step of the recursive function.
   
   2) I have a magic type constructor to turn `F[A]` into its associated inductive type and a way to lift the base case evaluations to an evaluator that traverse the inductive type.
   
This blueprint, together with the inductive type constructor and the lifter operator is what is called recursion schemes. Lets develop these ideas.

## Functors, F-algebras and Fix

---

As we have told before, we want to build an abstraction of general recursion. I will try to build all these ideas over an example, to keep the focus on the intuition.
Lets start with a simple example. Imagine we want to implement the ring structure of the set of integers, i.e. , the set together with multiplication and addition. A simple implementation of this type would be:

```scala
sealed trait Ring 
case object Zero extends Ring
case object One extends Ring
case class Elem(x: Int) extends Ring
case class Add(x: Ring, y: Ring) extends Ring
case class Mult(x: Ring, y: Ring) extends Ring
```

Of course, the `Zero` and `One` can be used interchangeably with `Elem(0)` and `ELem(1)`, but we prefer to keep these two elements as special, because they are the neutral elements of addition and multiplication.

As we said before, we want to separate a general step from the inductive steps.  For this purpose, let's imagine that we have a base type `A` and we want to implement sum and product operations of just two elements of that type. It can be defined in Scala as:

```scala 
sealed trait RingF[+A]
case object Zero extends RingF[Nothing]
case object One extends RingF[Nothing]
case class Elem(x: Int) extends RingF[Nothing]
case class Add[A](x: A, y: A) extends RingF[A]
case class Mult[A](x:A, y: A) extends RingF[A]
```

Because of the inductive rules of `Ring`, we can build a value as deep as we want, for example:

```scala
val expresion1 = Mult(Elem(4), Add(Elem(3), One))
```
<p align="center" >
    <img src="examples/example.png" /> 
</p>  

But, for `RingF`, we are able to build just base cases like `Elem(7)`, `Add(3, 4)`, etc. 

Imagine that we want to evaluate the expressions produced by `Ring` and `RingF`, for example, the value resulting from solving the operations. For `Ring`, this can be made by the recursive function:

```scala
def toInt: Ring => Int = {
    case Zero => 0
    case One => 1
    case Elem(x) => x
    case Add(x, y) => toInt(x) + toInt(y)
    case Mult(x, y) => toInt(x) * toInt(y)
}
```
And, in Case we want to define it for `RingF` we can simply solve it without recursive calls:
 
```scala
def evalToInt: RingF[Int] => Int = {
    x => x match {
        case Zero => 0
        case One => 1
        case Elem(x) => x
        case Add(x, y) => x + y
        case Mult(x, y) => x * y
    }
}
```
We can observe that, obviously, is simpler to write `evalToInt` than `toInt`. Don't forget our main goal, an abstraction to transform the simpler fuction into the (a bit) complex one.

Here is where the magic comes, if we want to evaluate, for example, a value of the form `RingF[RingF[A]]`, i.e., an expression of deeph two,  we need first an evaluation of the form `RingF[RingF[A]] => RingF[A]` and then, another evaluatuion `RingF[A] => A`. In the previous example, `A` is `Int`. The way of doing this is based on a good property of `RingF`: it is a functor. A functor, for our porpouses, is a type constructor with the property that we can implement a way of translating functions between types in functions between the constructed types. So for this simple example, for every type `A` we can build a new type `RingF[A]`. Now, given a function `f: A => B`, we know how to build a function with the signature `map: RingF[A] => RingF[B]`. This can be easily define and implemented in Scala:

```scala 
trait Functor[F[_]] {
    def map[A,B](f: A => B): F[A] => F[B]
}

val ringFunctor = new Functor[RingF] {
    override def map[A, B](f: A => B): RingF[A] => RingF[B] = {
        case Zero => Zero
        case One => One
        case Elem(x) => Elem(x)
        case Add(x, y) => Add(f(x), f(y))
        case Mult(x, y) => Mult(f(x), f(y))
    }
}

```

We can summarize the functor behaviour in the following image:

<p align="center" >
    <img src="examples/example14.png" /> 
</p>  

For `RingF`, we simply encapsulate the image of `f` under the `RingF` constructor. Now that we know how to lift functions using the `map` property of the functor, we can lift the one step evaluation `evaltoInt` into an evaluation between `RingF[RingF[A]] => RingF[A]` by simply taking `map` with `A` as `RingF[A]` and `B` as `A` :

```scala
def liftInt: RingF[RingF[Int]] => RingF[Int] = {
    ringFunctor.map(evalToInt)
}
```

So, thanks to the map property, we know how to lift one floor of recursion our base cases. Using this, we can evaluate a two floor of recursion by simply reading the signature of our functions. The function able to evaluate this is `evalToInt(map(evalToInt)(_))` because of the following picture:

<p align="center" >
    <img src="examples/example15.png" /> 
</p>  

Our goals now are to define the full inductive expression of `RingF` and lift `evalToInt` to the infinite floor of recursion. 

Before digging into recursion, lets set some terminology. Given a functor `F`, any function with signature `F[A] => A` is called an F-algebra over `A`. A natural question can arise from this: **why on earth is this called algebra?**. In mathematics, very roughly speaking, an algebra is a set `A` and a set of operations acting on it. The only requirement over this operations is that it should be closed in `A`, i.e., if we apply a finite number of operations, the result is an element of `A`. For example, the integers forms an algebra (even more, it is a ring) with the operations of product and addition. Now, the concept of F-algebra is a generalization of this idea, because the functor is defining the set of operations (`Add`, `Mult` for our `RingF`) and the function `F[A] => A` is defining the laws of solving the operations. For the case of integers, this is encoded in the function `evalToInt`. So, we can conclude that there is the same information in the `evalToInt`function and in the algebra of integer numbers.

After this explanation, lets go back to recursion.
The argument to do this can be formally made in terms of initial objects of certain categories, but let me try to focus on the main ideas rather than the technical details.

If we want to find the full inductive behind `RingF` type that's mean that we don't care about the depth of the expression of our ring, so we can think about it as a type `H` related to `ring[A]` and with an infinite depth. It's obvious that an object with this property must verify, in some sense, that `ring[H] = H`, read as: _if we are in a full general inductive step, we don't get nothing if we repeat the application of `RingF`_. So, we can rephrase the equality as: _All the information inside `ring[H]` is inside `H` and the converse is also true_. The equality understood as this property is called an isomorphism between the types `H` and `RingF[H]`. This is why we prefer to call `H` as `Fix[RingF]` because it is a _fixpoint_ of the equation `RingF[X] = X`. So the final signature is `Fix[RingF] = RingF[Fix[RingF]]`. Avoiding the formal and technical proofs, we can just implement the previous definition. Even more, this argument works for every Functor `F[A]`, so we can simply write an abstraction like:

```scala
case class Fix[F[_]](value: F[Fix[F]] )
object Fix {
  def fix[F[_]](ff: F[Fix[F]]): Fix[F] = new Fix[F](ff)
  def unfix[F[_]]: Fix[F] => F[Fix[F]] = f => f.value
}
```
`fix` and `unfix` defines the way of translating information back and forth (these functions defines the isomorphism of the fixpoint).

Now, we can build our inductive type `Ring` by simply fixing the `RingF` functor:

```scala
type Ring = Fix[RingF]
val zero = Fix[RingF](Zero)
val one =  Fix[RingF](One)
def elem: Int  => Ring = x => Fix[RingF](Elem(x))
def add: (Ring, Ring) => Ring = (x, y) =>  Fix[RingF](Add(x, y))
def mult: (Ring, Ring) => Ring = (x, y) =>  Fix[RingF](Mult(x, y))
```
and we can build recursive values like:

```scala
val expresion2 = mult(elem(4), add(elem(3), one))
```
<p align="center" >
    <img src="examples/example1.png" /> 
</p>  

Yow can compare this with expression1. 

We actually have reached the first goal, i.e., to find an inductive abstraction for our starting functor `RingF`. What its left is to take our way to evaluate this data type starting with our `evalToInt` function.

### Catamorphisms

----

If we take a fast recap, what we have is a functor `RingF` and our `Fix` that can transform our functor into a recursive data Type. The `Fix[]` type constructor has two functions, called fix and unfix that, as we alrready said, defines the equality `Fix[F] = F[Fix[F]]` for any functor. Our goal of lifting `evalToInt: RingF[Int] => Int` can be translated to find a function `m: Fix[F] => Int` related to `evalToInt`. Lest make a simple diagram that represent all this ideas

<p align="center" >
    <img src="examples/example3.png" /> 
</p>      

Looking a this diagram, if we want to define `m` we only need to follow the diagram and composing the functions, i.e, taking `m` as `eval o map(m) o unfix`. But, to understand it better, lets see first the case of `RingF` and `evalToInt`:

<p align="center">
    <img src="examples/example4.png"/>
</p>      

In this case, the recursive call comes from the call `.map(m)`, because `m` is the function we are defining. In this case, we can implement such function as 

```scala
def cata: Fix[RingF] => Int = {
    x => evalToInt(ringFunctor.map(cata)(Fix.unfix(x)))
}
val exp1 = mult(elem(4), add(elem(3), one))
cata(exp1)

res: Int = 16
```

We can follow each step of recursion as:

![](examples/animation.gif)

So we are done! we achive our goal of lifting evalToInt to our recursive data type `Ring`. But, our implementation of `cata` looks pretty particular for this case. As we can see in our first diagram, we can do this for any functor and for any evaluation, i.e, an algebra over it. Lets do this in the same way, by reading the diagram:

Of course, the eval function can be parametriced as any function with signature `F[A] => A`. This kind of function is called an F-algebra over the fixed type A.



Lets start doing it simple. Lets apply it to our basic example `RingF` and `evalToInt`. In this case, our implementation of `m` should be:


```scala
def cata[F[_], A](alg: F[A] => A)(implicit F: Functor[F]): Fix[F] => A = {
    x => alg(F.map(cata(alg))(Fix.unfix(x)))
}
```
And, with this, we can call it by typing `cata(evalToInt)(ringFunctor)(exp1)`.

### A real example: Lists and folds

---

At the very begining, we talk about Lists as an essential example of recursive data type. There is as well one basic operation over Lists implemented in scala called foldLeft. What this function do is to traverse the List structure evaluating, a function with an starting value. Lest see an example of this:

```scala
val l1 = List("a", "b", "c")
```
<p align="center">
    <img src="examples/example7.png"/>
</p> 

```scala
l1.foldRight(0)((n, t) => n + 1)

res44: Int = 3
```

We need to read foldRight as: start with 0 and apply the function given while our next element is not Nil. This function compute the length of l1. If we look carefully, this function looks pretty similar to `cata`, where our algebra is the function acting over the base cases and then we lift it to traverse full list. Lets translate all of this using `Fix` and `cata`. So, lets implement our own foldLeft using cata. First, lets define the functor associated with `List`. To make it simple, we're gonna trait only `List[String]` to avoid type parameters.

```scala
sealed trait ListF[+A] 
case object NilF extends ListF[Nothing]
case class ConsF[A](head: String, tail: A) extends ListF[A]
```
And now, we can turn this `ListF` into a functor by implementing `map`:

```scala
val listOfStringFunctor = new Functor[ListF] {
 override def map[A, B](f: A => B): ListF[A] => ListF[B] = {
     case NilF => NilF
     case ConsF(h, t) => ConsF(h, f(t))
 }   
}
```

Using `Fix` we can build our own version of `l1`:

```scala
type ListString = Fix[ListF]
val nil = Fix[ListF](NilF)
def cons : (String, ListString) => ListString = (h, t) =>  Fix[ListF](ConsF(h, t))

val fixL1 = cons("a", cons("b", cons("c", nil)))
```

<p align="center">
    <img src="examples/example8.png"/>
</p>                            

Finally, lets try to implement foldLeft using cata:

```scala 
def myFoldLeft[A](baseValue: A)(evaluator: (String, A) => A)(l: ListString): A = {
    def alg: ListF[A] => A = {
        case NilF => baseValue
        case ConsF(h, t) => evaluator(h, t)
        }
    cata(alg)(listOfStringFunctor)(l)

}

myFoldLeft(0)((x, y) => y + 1)(fixL1)

res35: Int = 3
```

Of course, this is just a particular case, equivalent to `List[String]`.

We have build the foldRight method of `List[String]` in terms of `cata`, but we can go further. It turns out that `foldRight` is as useful as `cata` in the sense that `foldRight` recreates general recursion. The joke is: _every recursive function over a recursive data type can be defined as a foldRight_ and, this is the same as `cata`. The reason of this is that we can translate cata (in some sense) into foldRigth. Lets do this in our main example `Ring`:

```scala
def foldR[Z](e: Fix[RingF])(elem: Elem => Z)(add: (Z, Z) => Z)(mult: (Z, Z) => Z): Z = {
    e.value match {
        case Elem(x)  => elem(Elem(x))
        case Add(x, y) => add(foldR(x)(elem)(add)(mult), foldR(y)(elem)(add)(mult))
        case Mult(x, y) => mult(foldR(x)(elem)(add)(mult), foldR(y)(elem)(add)(mult))
    }
}
```
of course, we can recreate `evalToInt` using this by writing: 

```scala
a: Ring = Fix(
  Add(
    Fix(Add(Fix(Elem(3)), Fix(Elem(3)))),
    Fix(Add(Fix(Elem(3)), Fix(Elem(3))))
  )
)

foldR[Int](a)(x => x.x)((x, y) => x + y)((x,y) => x * y)

res61: Int = 12
```

To justify that this is equivalent to `cata(evalToInt)(a)`, lets do some arguments about the signature of this function. First of all, the signature of foldR is:

<p align="center" >
    <img src="examples/example10.png" /> 
</p>  

And, using som basic type equivalences, we can turn this last expression into 

<p align="center" >
    <img src="examples/example11.png" /> 
</p>  

And, finally, the signature of this Either (that in scala are case classes of the same trait) is basically the signature of our algebra: 

<p align="center" >
    <img src="examples/example12.png" /> 
</p>  

And, of course, reading the composition of arrows we get the shape of cata:

<p align="center" >
    <img src="examples/example13.png" /> 
</p>  

So, af we have already see, `foldR` is exactly the same idea as `cata`, but in a more explicit way. Both of this represents the essence of recursion and can be understood as an equivalence, this is known as ....


## F-coalgebras and Anamorphisms

---

If we think about the `evalToInt` function, what this function does is to reduce a tree of operations into an Integer. Imagine that we want to do some converse function, i.e, a function that takes an Int and produces a `Ring`. The signature for this function would be `Int => RingF[Int]`. In a more general case, we can take any fixed type `A` and any functor `F` and write `A => F[A]`. Functions with the previos signature are called F-coalgebras, because of duality with F-algebreas. 

To take an example of this kind of function, we can use a function that, given an integer, produces the esxpression of products of its factors. As we have done before, lets do this for a base case (just spliting our number in a product of two others) and then we will try to lift this function to produce the full factorization of the given number. 

For the base case, we can write:

```scala
def findDivisorsOf(r: Int): RingF[Int] = {
    def loop(n: Int): RingF[Int] = {
        n match {
            case 0 => Zero
            case 1 => One
            case x => 
                if (x >= r) Elem(x)
                else if (r%x == 0) Mult(r/x, x) 
                else loop(n+1)
        }
    }
    loop(2)
}

```

For example, `findDivisorsof(6)` produces `res55: RingF[Int] = Mult(3, 2)`, but for `findDivisorsof(12)` we only get `res55: RingF[Int] = Mult(6, 2)` and we may want to keep factorizing the 6 as `Mult(3, 2)`. To do this, we need to produce a new diagram that represent the idea of lifting this function to the `Fix` of `RingF`. This can be done by reading our first diagram, but reversing the arrows:

<p align="center" >
    <img src="examples/example6.png" /> 
</p>  

And, as we did before, we only need to read this diagram to get an implementation. The direct implemnetation, following the same argumens as with `cata`is 

```scala
def ana[F[_], A](coalg: A => F[A])(implicit F: Functor[F]): A => Fix[F] = {
    x => Fix((F.map(ana(coalg))(coalg(x))))
}
```

Asyou can see, this new lifter function is called ana. And now, all we need to do is to call `ana(findDivisors)(ringFunctor)(n)` to get a full expresion that represent the factorization of n. For example:

```scala
val expression3 = ana(findDivisorsOf)(ringFunctor)(12)

expression3: Fix[RingF] = Fix(
  Mult(Fix(Mult(Fix(Elem(3)), Fix(Elem(2)))), Fix(Elem(2)))
)
```
<p align="center" >
    <img src="examples/example5.png" /> 
</p>  

### A real example: Streams 

---

## Hylomorphisms

---

Now we can build expressions from integers using `ana`and know how to reduce it using `cata` so we can combine both reading the direct diagram:

<p align="center" >
    <img src="examples/example9.png" /> 
</p>  

The new function we get (`ana o cata`) is callled hylomorphism, and is pretty usefull. For example, we can use it to check that or `findDivisorsOf` and `evalToInt` rise the starting value if we did it one after another:

```scala
def hylo[A, B, F[_]](ev1: A => F[A])(ev2: F[B] => B)(implicit functor: Functor[F]): A => B = {
    x => cata(ev2)(functor)((ana(ev1)(functor)(x)))
}

hylo(findDivisorsOf)(evalToInt)(ringFunctor)(12)
---
res40: Int = 12
```

It works! But this is not so usefull, because we are getting the identity. Lets try to implemente a more fancy function. For example, lets write a simple fution to represent the factorization of an integer as a String. To do this, we need first a pretty print function and then simple apply hylo to it with findDivisorsOf.

```scala
def prettyPrint: RingF[String] => String = {
    case Zero => "0"
    case One => "1"
    case Elem(x: Int) => x.toString
    case Add(x, y) => x + " + " + y
    case Mult(x, y) => x + " * " + y
}

hylo(findDivisorsOf)(prettyPrint)(ringFunctor)(12)

---
res44: String = "3 * 2 * 2"
```

## Morphisms between recursive structures

---

We are gonne to end this talk with one last comment about recursive strctures. We have allready seen functions to evaluate (cata) and to generate (ana) recursive data types. But what if we want to use a function between recursive data Types, for example, a function `reduce: Ring => Ring` that simplifies expresions as `x + x => 2*x`. This can be made using our `Fix` and a special kind of algebra. The algebra we are gonne use has signature `RingF[Fix[RingF]] => Fix[RingF]`, because we need to work over the general recursive case. Take a look to the implementation of this:

```scala
def reduce[A]: RingF[Fix[RingF]] => Fix[RingF] = {
    case Add(x, y) => if (x == y) Fix[RingF](Mult(y, Fix[RingF](Elem(2)))) else Fix[RingF](Add(x,y))
    case other => Fix[RingF](other)
}
```

It looks pretty simple. You can compare it with its implementation using our first defined Ring without using `Fix`:

```scala
def reduce: Ring => Ring = {
    case Zero => Zero
    case One => One
    case Elem(x) => Elem(x)
    case Add(x, y) => if (x == y) Mult(reduce(x), Elem(2)) else Add(reduce(x), reduce(y))
    case Mult(x, y) => Mult(reduce(x), reduce(y))
}
```

Take a look to the shape of this function. It has a lot of recursive calls in so many places, what is error prompt and, second, its to verbose, because all is the same but for the case Add(x,x). This is one of the advantage of the recursion scheme pattern. 

## Further references

---

We have seen the recursion scheme pattern with lot of examples. The main functions has been presented but we have left under the hood some details and mathematica justifications of the results. If you want to get formal proofs of the equivalence of `Fix[F] = F[Fix[F]]` (called Lambeks lemma) and a presentation of .... 
