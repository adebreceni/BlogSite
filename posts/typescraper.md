# Typescraper

> And ninety-nine years they built, day and night, growing ever higher, until they reached the Foundation itself.<br><br>
>    &mdash; <cite>nonsensical made-up quote</cite>

Take one cup of types, two cup of higher order types and a pinch of subtyping and you are in a hot bubbling mess. Now if the fake quote did not, the previous sentence surely weeded out 99% of the readers, so its just you and me now buddy. Let's take it piece by piece.

### Type system

If you don't have a static type system, there is really not much to talk about. Everything is runtime checked, we don't have to worry about promises the type system makes, because there is no type system. Good bye. (early return)

### Subtyping

(Did you realize that we are not going in the order I listed the components. Silly me.)

Take two types `A` and `B`, then it is just natural to wonder if given an object `x` of type `A`, could you treat this object as if it were of type `B`. If for all objects it is true that if they are of type `A` , they are of type `B`, then congratulation, you have subtyping. Or rather not so simple, because it is not enough that You know they are in a subtype relationship, the compiler must also be able to recognize this, moreover there is a level of "granularity", in how subtyping is treated in a given language.

In `C++` this granularity merely boils down to structural conformity. What the heck does this mean? It means that the only thing the compiler can promise you regarding the possible subtypes of `T` is that you will be able to call the set of methods one can call on `T` and that you may access its fields. Nothing about the *behavior* can be guaranteed and/or expressed in the type system (okay, maybe `noexcept` and `const`, but even those are lies (throwing from `noexcept` calls `std::terminate` and `mutable` circumvents `const` (how many layers of nested comments can you stand? (is this too much? (ok, I'll stop now)))))

Also subtyping only works in `C++` if you explicitly construct the classes that way, if you want to relate two unrelated classes you are looking for a system with structural subtyping. (Although in `C++` you could simulate this behavior by manually implementing v-tables.)

With structural subtyping you only have to ask for a set of methods or fields and if your (unrelated) class supports those, lo and behold you got yourself a cast. (Now if that cast is "free" or not, that is an other question.)

```scala
class A{
  f(): int
}

interface HasF{
  f(): int
}

let x: A
let y: HasF = x  // no compile error here
```

### Higher-order types

With type constructors or higher-order types you could really feel like Frankenstein, cooking up abominations in your lavatory<sup>1</sup>. 

```scala
class stack[T]{
  push(x: T): void
  pop(): T?
}
```

Look at this contraption. This is an interface of a stack, that holds some number of items of type `T` and you can push one onto the top of the stack and pop one from the top of the stack. You may then go on and dust off your trusty `W` type to instantiate a `stack[W]`. 

### The collapse

Now, this `W` of yours has seen some things, lived through the un-through-livable, dropping offspring here and there (obviously I'm talking about its many subtypes).

One such subtype is `Y`.

Now, your cute little stack would like nothing more, than getting a `W` pushed into it. Will a `Y` do? Oh yes it will, after all, that is what subtype is for. So you push a `Y` , then you pop a `W`. After all it may contain other `W`s after your push. So far so good, nothing heinous happened.

Your gears start turning, a `stack[Y]` would be something that contains `Y`s, you can surely pass it on as something containing `W`s right? It does contain `W`s, the math seems to add up. So then a `stack[Y]` is a subtype of `stack[W]` right? The problem is, that it is not only something that contains elements but also something that can have new elements added into. If you were to treat a `stack[Y]` as a `stack[W]`, soon enough you would start pushing `W`s into it left and right (and not only make-pretend `W`s that are secretly `Y`s, but also true pure blood `W`s). And then on a nice summer afternoon when you pop a fresh one off the `stack[Y]` you get a not-so-Y object, causing you to hemorrhage all over the place (not easy to look at, and even harder to clean up).

This is why `C++` goes and tells you to go someplace warmer, there is simply no subtyping relationship between a `vector[T]` and a `vector[U]` , whatever the relationship may be between `T` and `U` (unless of course they are the same, not even `const`-ness can differ)

How does `Java` handle this beauty? Not better than `C++`, and it also provides a secret door with a tiger behind, that will promptly eat your face off.

### Type erasure

`Java` says "fuck it, a list is a list, period". So you can do this:

```scala
import java.util.Stack;

class Main {  
  public static void main(String args[]) { 
    Stack<String> s = new Stack<String>();
    addOne_literally((Stack) s);
    String str = s.pop();
    
  } 
  public static void addOne_literally(Stack<Object> s) {
    s.push(new Integer(1));
  }
}
```

And we would have gotten away with it too, if it weren't for that meddling run-time exception that cuts our endeavor short.

(Okay, this was an example with a stack and I wrote "list", but what can I do? It's not like I can just willy-nilly go back and change "list" to "stack".)

### Solution

So the problem why we cannot establish a subtyping relationship between different stacks is that we want to have both the `push` and `pop` method. If we get rid of the `push` method, all of a sudden, we can safely establish a subtyping relationship. What if we rather discard the `pop` method? Then the arrow of "subtype-of" flips. Inspect and internalize the following setup.

```scala
type A, B;
B < A  // B is subtype of A
class GrowOnlyStack[T]{
  push(x: T): void
}

class ShrinkOnlyStack[T]{
  pop(): T
}

// then the following is true
GrowOnlyStack[A] < GrowOnlyStack[B]
// if we can push an "A", then we can push a "B" (since that is also an "A")
ShrinkOnlyStack[B] < ShrinkOnlyStack[A]
// if we can pop off only "B"s, then it's also true, that we are popping off "A"s
```

So we can conclude that the position of the type parameters in the method signature determine in which direction the subtyping can happen. The positions are called covariant and contravariant. If a given type parameter only occurs in covariant positions, then the type parameter itself is called covariant. A covariant parameter "preserves" the subtyping direction, e.g. in our example above `ShrinkOnlyStack`s first (and only) parameter is covariant, and sure enough `ShrinkOnlyStack[B]` is on the same side of the subtyping arrow as `B` itself. 

`Scala` has the means to annotate each type parameter of how we intended it to behave, and scolds us if we used the parameter in a wrong position.

```scala
// "-" means we want U to be contravariant
// "+" means we want T to be covariant
abstract class FineStack[-U, +T]{
  def push(x: U): Unit
  def pop(): T
  // def pop(): U => a nice compile error
}
```

Now we can take our `FineStack` out for a spin, and restrict what the callee may push into it, while simultaneously broadening what it can expect when popping from it. 

```scala
FineStack[Marshmallow, SoggyMarshmallow]
```

Then obviously you can push "special" toasted marshmallows, and expect "only" marshmallows.

```scala
FineStack[Marshmallow, SoggyMarshmallow] < FineStack[ToastedMarshmallow, Marshmallow]
```

Okay, maybe not the best example, but boy do I want to drink a hot cocoa now. 

<br>
<br>

---

[1]: Laboratory, made you look, didn't I.