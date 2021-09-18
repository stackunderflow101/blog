---
title: Covariance and Contravariance
author: Stack Underflow
date: '2021-09-16'
categories:
  - Tech
tags:
  - object-oriented-programming
slug: covariance-and-contravariance
image: images/bakcground.jpg
---

## Introduction

Generic classes can be made covariant or contravariant. An example of  covariant class is `List`, which could be declared as the following. (Taken from chapter 3 of [Functional Programming in Scala](https://learning.oreilly.com/library/view/functional-programming-in/9781617290657/))

```scala
sealed trait List[+A]

case object Nil extends List[Nothing]
case class Cons[+A](head: A, tail: List[A]) extends List[A]
```

Notice the `+` sign in `List[+A]`, which means `List` is covariant. For example, if we have `Cat` and `Dog` extending from `Animal`, `List[Dog]` and `List[Cat]` will be subtypes of `List[Animal]`. Also note that the object `Nil` extends from `List[Nothing]`, which makes it a subclass of `List[Dog]`, `List[Cat]`, or any other `List[A]` because `Nothing` is the subtype of all types. 

An example of contravariance in Scala is the following `Printer` examples. (Taken from [Tour of Scala](https://docs.scala-lang.org/tour/variances.html))

```scala
abstract class Printer[-A] {
    def print(value: A): Unit
}

class AnimalPrinter extends Printer[Animal] {
    def print(animal: Animal): Unit =
        println("The animal's name is: " + animal.name)
}

class CatPrinter extends Printer[Cat] {
    def print(cat: Cat): Unit =
        println("The cat's name is: " + cat.name)
}
```

It turns out that a `Printer[Animal]` can be used wherever a `Printer[Cat]` is required, as is shown below

```scala
def printMyCat(printer: Printer[Cat], cat: Cat): Unit =
    printer.print(cat)

val catPrinter: Printer[Cat] = new CatPrinter
val animalPrinter: Printer[Animal] = new AnimalPrinter

printMyCat(catPrinter, Cat("Boots"))      // The cat's name is: Boots
printMyCat(animalPrinter, Cat("Boots"))   // The animal's name is: Boots
```

As a result, `Printer[Animal]` is actually a subtype of `Printer[Cat]`, and that is why we 
we make `Printer` a contravariant generic class.

## The *get-put principle*

The covariance and contravariance of generic classes follow the *get-put principle* (from the book [Java Generics and Collections](https://learning.oreilly.com/library/view/java-generics-and/0596527756/)), which says we use covariance when we only *get* values from a structure, and use contravariance when we only *put* to it. An example in Java looks like the following.

```java
public class Collections { 
  public static <T> void copy(List<? super T> dest, List<? extends T> src) {
      for (int i = 0; i < src.size(); i++) 
        dest.set(i, src.get(i)); 
  } 
}
```

In the example above, we get values from `src` with `src.get()` and `put` values to `dest` with `dest.set()`, so we use `extends` for `src` (covariance) and `super` for `dest` (contravariance).

In Scala, the get-put principle also holds. `List` is immutable and we only `get` values from it, so it's made a covariant class. For `Printer` it's a little tricky. `Printer` classes doesn't store objects, but we can still think of the `print` method to be kind of a `put` operation since it takes a parameter of type `A`. As a result, the `Printer` classes is contravariant.

Notice the difference of Java and Scala generics: In Java we declare whether a generic class is covariant or contravariant when we declare variables of the class (using `super` or `extends`), but in Scala we do it when we define the class (using `+` or `-`). As a result, in Java a class can sometimes be covariant and sometimes be contravariant, like the `List` class in the Java `copy` example above, but in Scala, a class can only be either covariant or contravariant (or, of course, invariant) consistently. 

Notice that the get-put principle works slightly differently when we are using Scala-style class-level variation instead of Java-style usage-level variation. In Scala, it means:

- Covariant type parameters can only appear in return types and not in method parameters since we "get" return values from the objects. For example, we could define a method `head` for `List` which returns `A`, but we could not define an `append` method for `List` which takes `A`

- Contravariant type parameters can only appear in method parameters and cannot appear in method parameters since we "put" arguments to the objects. For example, we could define the method `print` for `Print` which takes `A`, but we could not define any methods which returns `A`

## A caveat in Java

Consider the Java code below

```java
Integer[] ints = new Integer[]{1,2,3};
Object[] objects = ints;
objects[2] = 3.14;
```

Think about the following questions:

- Does the code compile? Does the code run correctly?
- At which line will an error occur?
- What is the root cause of the error? If we could redesign Java, at which line should an error occur?

Here is the answer. The code compiles but it doesn't run correctly, a `java.lang.ArrayStoreException` will be thrown at line 3. The apparent problem is that we put a double into an array of integers, but the root problem happens at line 2: We are able to assign an `Integer[]` to an `Object[]` (i.e. arrays are considered covariant) even when we will "put" to the array subsequently at line 3! According to the get-put principle, in this case we should not consider arrays as covariant. It turns out Java (incorrectly) thinks arrays as covariant but they do not always behave covariantly. So if we could redesign Java we should made arrays invariant (it's hard to provide `super`, `extends` like syntax for arrays because arrays do not use `<>`), which means line 2 should get a compile time error (because `Object[]` is not a subtype of `Integer[]`). In Scala the class `Array` is invariant, which solves this issue in Java.

## Function types

Suppose we have a function which takes an animal and returns its name. like below

```scala
def getNameAnimal(animal: Animal): String = 
    s"animal: $animal.name"
```

We have a similar function taking a `Cat`.

```scala
def getNameCat(cat: Cat): String = 
    s"cat: $cat.name"
```

Is there a subtype relation between the two functions? The answer is yes. Whenever we want to use the function `getNameCat`, the function `getNameAnimal` could also suffice since is expects an `Animal` which is a supertype of `Cat`. For example, say if we have a higher order function which prints the name of a cat.

```scala
def printName(cat: Cat, getNameFn: Cat => String) = 
    println(getNameFn(cat))
```

We can definitely pass `getNameCat` to the `getNameFn` argument, but it's equally good to pass `getNameAnimal`, since it takes an `Animal`, and it should also work well if a `Cat` is passed.

```scala
val cat = Cat("kitty")
printName(cat, getNameCat)   // "cat: kitty"
printName(cat, getNameAnimal) // "animal: kitty"
```

In other words, the type of the function `getNameAnimal` (`Animal => String`) is a *subtype* of `Cat => String`. 

What if we have a function taking whatever arguments and returning a `Cat` or an `Animal`? It turns out `a => Cat` is a subtype of `a => Animal` for whatever type `a`, since whenever a function returning an `Animal` is expected, a function returning a `Cat` also works well.

Scala illustrates the subtyping relations of functions w.r.t. their parameters and return types with the definition of function types. Below is the definition of any functions taking one argument `T1 => R` ([reference](https://www.scala-lang.org/api/2.13.5/scala/Function1.html)).

```scala
trait Function1[-T1, +R]{
    def apply(v1: T1): R
}
```

The difference between `Function1` and `List` (or `Printer`) is that here we have more than one type parameters, `T1` and `R`, and their variation properties are different. A function is contravariant with respect to its argument (e.g. `Animal => String < Cat => String`), and covariant with respect to its return type (e.g. `String => Cat < String => Animal`). We can no longer say whether a generic class is covariant or contravariant when it has more than one type parameters. Instead, we call `T1` a contravariant type since `Function1` is contravariant w.r.t. it. Similarly we call `R` a covariant type.

Also note that the variation property of a function type is consistent with the get-put principle. We *get* the return value from a function by *putting* the argument into the function, so the return type is covariant and the argument type is contravariant.

For functions with more than one arguments, they are contravariant w.r.t. all of their arguments and covariant w.r.t. their return type. For example, below is the definition of functions taking two arguments.

```scala
trait Function2[-T1, -T2, +R]{
    def apply(v1: T1, v2: T2): R
}
```

## Variance and methods

Let's try to define a method `prepend` for the class `List`. We could define it in the companion object. 

```scala
sealed trait List[+A]

case object Nil extends List[Nothing]
case class Cons[+A](head: A, tail: List[A]) extends List[A]

object List {
    def prepend[A](v: A, l: List[A]): List[A] = 
        Cons(v, l)
}
```

It works well. However, how should we define `prepend` to be a method of `List`?

A naive definition would be the following:

```scala
sealed trait List[+A] {
    def prepend(v: A): List[A] = 
        Cons(v, this)
}

case object Nil extends List[Nothing]
case class Cons[+A](head: A, tail: List[A]) extends List[A]
```

However, the compiler will give you an error if you define `prepend` this way: 

> Covariant type A occurs in contravariant position in type A of value v

We already know what is "covariant type A": It means the generic class `List` is covariant w.r.t. type parameter `A`, but what is "contravariant position"? Let me explain.

`List` is covariant, which means `List[Cat]` is a subtype of `List[Animal]`. If the naive definition passes the compiler, what will happen to the following code snippet?

```scala
val cat1 = Cat("cat1")
val cat2 = Cat("cat2")
var cats: List[Cat] = Cons(cat1, Cons(cat2, Nil))
var animals: List[Animal] = cats
val dog = Dog("dog")
val animals2 = animals.prepend(dog) // oppppps!
```

If we somehow redesigned the Scala compiler so that it does not give us the "contravariant position" error, now questions: 
- Would the code compile? Would the code run correctly?
- At which line would an error occur?

The answer is, the code compiles correctly, but line 6 will fail. Line 6 passes the compiler since `animals` is a `List` of animals, whose `prepend` method takes an animal, and a dog is definitely suitable. However, we know that `animals` is actually a list of cats, whose `prepend` method takes a `Cat` instead of any arbitrary animals. In this way, line 6 should throw an exception that `Dog` is invalid here.

In fact, we are running into a similar situation as [the caveat in Java](#a-caveat-in-java). We also violates the get-put principle, since we can "put" a value of type `A` to a `List` with the `prepend` method, but `List` is covariant of `A`. 

### Contravariant position

Let's think about how to make the method `prepend` type safe. One approach is to make `List` an invariant class, just like what we do for `Array`. However, we have a better approach.

Let's assume we define the type of `v` in the `prepend` method to be a type derived from `A`, called `F[A]`, where `F` is called a *type constructor* that derives a type from another type. `List[A]` is an example of a type constructor. When `F` is the identity type constructor, `F[A] = A` so our naive definition is actually a special case. In this way, the signature of `prepend` looks like:

```scala
sealed trait List[+A] {
    def prepend(v: F[A])
}
```

Currently we do not care what `F` is, we just think of it in an abstract way. Later we will discuss how we should define `F`. Here for the new definition, we supply the `prepend()` method `F[Animal]` when it's called with `List[Animal]` and we supply it `F[Cat]` when called with `List[Dog]`. For type-safety, `List[Cat].prepend` has to be able to accept any possible values given to `List[Animal].prepend`. If we think of types as sets (for example, the type `String` corresponds to the set of all strings), it means the set corresponding to `F[Animal]` is a subset of the set corresponding to `F[Cat]`. In other words, `F[Cat]` has to be a supertype of `F[Animal]`. 

`List[Cat]` is a subtype of `List[Animal]`, while `F[Cat]` has to be a supertype of `F[Animal]`. You might notice that the variance of `F` goes in the contrary direction with respect to the variance of the generic class `List`. Here comes the definition of *contravariant position*:

We say something is in *contravariant position* if **the variance of its type has to go in the contrary direction with respect to the variance of the class**.

Method parameters are something that are in contravariant positions. With this definition, the meaning of the compiler error becomes clear to us. The value `v` is a parameter of method `prepend`, so it's in contravariant position, which means its variance has to go in the contrary direction w.r.t. the class `List`. `List` is covariant w.r.t. type `A`, so the type of `v` has to be contravariant w.r.t. type `A`. However, we defined `v`'s type to be `A`, which is actually covariant w.r.t. `A`. That is why we get the compiler error:

> Covariant type A occurs in contravariant position in type A of value v

### The solution

How to design the type constructor `F`? Here is a common approach.

```scala
sealed trait List[+A] {
    def prepend(v: B >: A): List[B] = 
        Cons(v, this)
}
```

Here `B >: A` is our `F[A]`, it means any supertypes of `A` and it's a type derived from `A`. It enables Scala compiler to infer a common super type of `A` and the type of the argument given to `prepend`. For example, when we call `prepend(dog)`, `B` will be inferred as `Animal`. When we call `prepend(123)`, `B` will be inferred to be `Any`.

```scala
val cat1 = Cat("cat1")
val cat2 = Cat("cat2")
var cats: List[Cat] = Cons(cat1, Cons(cat2, Nil))
var animals: List[Animal] = cats
val dog = Dog("dog")
// animals is List[Cat], but the argument is type Dog
// this does not pose any problems since Scala infer type B to be type Animal
// and animals2 is List[Animal]
val animals2 = animals.prepend(dog) 
```

Note that the definition of `prepend` illustrates the power of immutability. `prepend` does not modify the list. Instead, it returns a new list, which gives it a chance to return a list of different type. 

How does `B >: A` satisfies our contravariant position requirement? It turns out that the type `B >: A` is actually contravariant with respect to type `A`. For example, `B >: Animal` means any supertypes of `Animal` and `B >: Cat` means any supertypes of `Cat`. A supertype of `Animal` is automatically a supertype of `Cat`, so `B >: Animal` is a subtype of `B >: Cat`. Generally, `B >: A1` is a subtype of `B >: A2` if `A1` is a supertype of `A2`, or, in other words, `B >: A` is contravariant with respect to type parameter `A`.

Similar to contravariant position, we have something called *covariant position*. We say something is in *covariant position* if **the variance of its type has to go in the same direction with respect to the variance of the class**. The return value of a method is in the covariant position.

We introduced the fact that method parameters are in contravariant position using an example of covariant generic class, but actually it holds for all generic classes. Suppose we have a generic class `P` with a type parameter `T`, and `P[A] < P[B]`. Suppose `P[T].m(value: G[T])` is a method which takes a parameter of type `G[T]`. For type safety, `P[A].m(value: G[A])` should accept any arguments given to `P[B].m(value: G[B])`, which means the set corresponding to `G[B]` is a subset of the set corresponding to `G[A]`, or, in other words `G[B] < G[A]`. since `P[A]` is defined to be a subtype of `P[B]`, we can find that `value`, a method parameter, is in contravariant position. Notice that we do not specify the variance of `P` w.r.t. `T`. 

We already know that functions are contravariant w.r.t. their parameters, and now we also know that method parameters are in contravariant position. What is the relation between the two "contravariance"? It turns out that by ensuring contravariant position for `value`, we have `G[B] < G[A]`, making `P[A].m` become a subtype of `P[B].m` (because functions are contravariant w.r.t. their parameters), which ensures that `P[A].m` is always suitable when `P[B].m` is required, a.k.a. the "type safety" for `P[A]` and `P[B]`. Below is a illustration of the relations. By the same reason, the return value of a method is in the covariant position since functions are covariant w.r.t. their return value.

![contravariant position](images/contravaraiant_position.png)

Let's see a contravariant class: the `Printer` type we discusses above, which has a method `print`

```scala
abstract class Printer[-A] {
    def print(value: A): Unit
}
```

The parameter `value` of `print` is also in contravariant position. The variance of type `A` (the type of `value`) goes in contrary to the variance of the class, so it satisfies this requirement. 

Note that the `Printer` class also satisfies the get-put principle since it uses a contravariant type (`A`) in method parameters. Actually we can view variance position as an extension of the get-put principle. By using contravariant type parameters (like type `A` in `Printer`) for method parameters, the variance of the method parameter (like `value: A`) goes in contrary to the variance of the class, so it automatically satisfies the contravariant position requirement. Similarly, by using covariant type parameters for return values, it automatically satisfies the covariant position requirement.