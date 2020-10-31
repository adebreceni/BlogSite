# Refinement typo

There comes a time in every software engineer's life when their heart starts aching for every property they cannot force the compiler to check at compile time. When a 'int greater than 5' type seems so sensible, so close yet the compiler will laugh at you, your coworkers will laugh at you, even the 'int' type will laugh at you tooting, that they can assume whatever value they like. This property 'greater than 5', like sand starts slipping through your fingers, while trying to grab on to it is of no avail. You reach the next stage of grief: bargaining. You pull yourself together and create a 'IntGreaterThan5' class with a meticulously designed constructor along the lines of

```scala
constructor(int a) : value_{a} {
  if (value <= 5) throw std.logic_error("Must be greater than 5");
}
```

Nothing can stop you now, you create the PR and it (miraculously) gets accepted. 

An eon later an AI living in a planet sized computer has a strange wish, if only it could guarantee at compile time for an int to be 'greater than 6'. Combing through the archives and lo and behold it finds your code. Pondering on it for 1.2 microseconds, it synthesizes a 'InGreaterThan6' class with the following constructor:

```scala
constructor(int a) : value_{a} {
  if (value_ <= 6) throw std.logic_error("Must be greater than 6");
}
```

Beautiful.

While analyzing the codebase it finds a peculiar usage of these classes:

```scala
extern fn g(GreaterThan5 x);

fn f(GreaterThan6 x) {
  g(GreaterThan5(x.value_))
}
```

But... isn't every int that is greater than 6, also greater than 5? Indeed it seems like it. So we are doing an extra compare operation... what a waste. Also the generated code is very bloated, due to the (im)possibility of throwing an exception. 

(I know what you might be thinking: "oh just use reinterpret_cast". Upon hearing this thought I'm visibly hurt, after all, we are trying to offload the verification to the compiler not to the reviewers and our future selves.)

In 0.9 microseconds the AI generates the following extra constructor for GreaterThan5:

```scala
constructor(GreaterThan6 x): value_{x.value_} {}
```

Beautiful.

The AI is smelling a pattern here, only if it could generalize this to a "GreaterThan" template.

```scala
template<int N>
struct GreaterThan{
  template<int M, typename = typename std::enable_if<(M >= N)>::type>
  constructor(GreaterThan<M> x) : value_{x.value_} {}

  constructor(int x): value_{x} {
		if (value_ <= N)
			throw std.logic_error("Must be greater than $N");
	}
};
```

Do you feel it? I can already feel the cores cooling due to the immense amount of cycles we saved. (After all this was in the hot path)

Save. Compile. Error.... What?

The GreaterThan5 class is in an eon old third-party library. We cannot just go willy-nilly rewriting that, adding constructors or templatify it (I'm like 10% sure this is a word).

The AI is doomed. "If only it used a language with refinement types." - it ponders.

Cue upbeat music. Introducing:

### Refinement types

A "refined type" is a base type `T` with a predicate `P` that for every value `x` with type `T{P}` it is true that it is of type `T` and also that `P(x)` is true.

The previous problem translated to refinement types is as straightforward as this type alias

```scala
type GreaterThan[N] = int{_ > N}
```

This has an amazing added benefit of being truly a subtype of `int` instead of the implicit casting operators the class-based approach would require. All the `int` operations are immediately accessible on a `int{_ > 5}` object.

Moreover the actual runtime check that is required to create (or more specifically: cast to) `int{_ > 5}` from a `int` can happen anywhere, as long as the language supports flow typing or other utilities that allow you to access the "fact" that you are in a specific branch of a conditional.

```scala
extern fn g(x: int{_ > 5});

fn f(x: int) {
  if (x > 5) {
    // with flow typing the type of "x" is automatically refined
    // to int{_ > 5}
    g(x) // all good and well
  } else {
    g(x) // Error: oh no you don't
  }
}
```

You can also think of this as if the condition in the constructor `value_ > 5` is "lifted" into the type itself, allowing the compiler to use this fact whenever it sees fit.

So if there are no drawbacks why isn't every language using them?

It is really, really hard to implement it with imperative languages. Because why stop at bounded integers when we can encode arbitrary properties for arbitrary types.

```scala
type NonEmpty[T] = vector[T]{!_.empty()}
```

Being a `vector[int]` is itself an invariant, and the methods on this class try to ensure that when they return this invariant is restored. (An unfortunate fact worth noting is that nothing checks and verifies if they truly restored the invariants, mainly because they cannot be expressed in our language of choice here: C++). Now we have to go around and annotate each method, how they affect the truthfulness of the refinement predicate.

If we were to leave it to the compiler and we start pushing and popping items left and right, before we even know it, the compiler gets so confused it's not even sure what language it's compiling anymore.

Also: aliasing 😬

```scala
fn f(x: int&, y: int&){
  if (x > 5) {
    // all good "x" is now int{_ > 5}
    --y
    // what now?
  }
}
```

So for now it's usage is mostly restricted to functional languages, like F-star and the theorem proving languages like Coq, Idris and others.

Upon analyzing this post the AI spends 10.8 microseconds whipping up a new imperative language with refinement types and the accompanying verified compiler, transpiling all packages in existence (7 zettabytes in its `node_modules` directory) to this new language and marveling at the inadequacy of 21st century human beings.