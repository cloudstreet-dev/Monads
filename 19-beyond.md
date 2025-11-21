# Chapter 19: Beyond Monads

Monads are powerful, but they're not the only abstraction in functional programming. Let's see the bigger picture.

## The Hierarchy

```
Functor
  ↓
Applicative
  ↓
Monad
```

Each level adds capability and constraints.

## Functor: Things You Can Map Over

A Functor is anything with a `map` operation:

```typescript
interface Functor<T> {
  map<U>(fn: (value: T) => U): Functor<U>;
}
```

**Examples:**
- `Array<T>` - map over a list
- `Maybe<T>` - map over an optional value
- `Promise<T>` - map over a future value
- `Function<A, T>` - map over a function's output

**The Law:**
```typescript
// Identity: x.map(id) === x
array.map(x => x) === array

// Composition: x.map(f).map(g) === x.map(x => g(f(x)))
array.map(f).map(g) === array.map(x => g(f(x)))
```

### What Makes a Good Functor

```typescript
// Array is a functor
[1, 2, 3].map(x => x * 2)  // [2, 4, 6]

// Maybe is a functor
Some(5).map(x => x * 2)  // Some(10)
None.map(x => x * 2)     // None

// Promise is a functor
Promise.resolve(5).then(x => x * 2)  // Promise(10)

// Functions are functors (compose with output)
const f = (x: number) => x + 1;
const mapped = (x: number) => (x + 1) * 2;  // f mapped with (* 2)
```

### Building a Functor

```typescript
class Box<T> {
  constructor(private value: T) {}

  // Only need map to be a functor
  map<U>(fn: (value: T) => U): Box<U> {
    return new Box(fn(this.value));
  }

  getValue(): T {
    return this.value;
  }
}

// Usage
const result = new Box(5)
  .map(x => x * 2)
  .map(x => x + 10)
  .map(x => x.toString());

console.log(result.getValue());  // "20"
```

## Applicative: Independent Computations

Applicative sits between Functor and Monad. It lets you combine *independent* computations.

```typescript
interface Applicative<T> extends Functor<T> {
  // Lift a value
  static of<U>(value: U): Applicative<U>;

  // Apply a wrapped function to a wrapped value
  ap<U>(fn: Applicative<(value: T) => U>): Applicative<U>;
}
```

**Note on `ap` conventions:** Different libraries use different argument orders for `ap`:
- This book uses: `value.ap(wrappedFunction)` - the value calls `ap` with the function
- Some libraries use: `wrappedFunction.ap(value)` - the function calls `ap` with the value
- Haskell uses: `wrappedFunction <*> value` (infix operator)

Both are equivalent, just different styles. Check your library's documentation to see which convention it follows.

### Why Applicative?

```typescript
// With Monad, the second operation can depend on the first
findUser(id).flatMap(user =>
  // Can use 'user' here to decide what to do next
  user.isAdmin ? fetchAdminData() : fetchUserData()
);

// With Applicative, operations are independent
// Can potentially run in parallel!
const result = Applicative.of((a: A) => (b: B) => (c: C) => combine(a, b, c))
  .ap(computeA)  // Independent
  .ap(computeB)  // Independent
  .ap(computeC);  // Independent
```

### Applicative Examples

```typescript
// Validation - collect all errors
Validation.success((name: string) => (email: string) => (age: number) =>
  ({ name, email, age })
)
  .ap(validateName(data.name))
  .ap(validateEmail(data.email))
  .ap(validateAge(data.age));

// Maybe - combine optional values
Maybe.of((a: number) => (b: number) => a + b)
  .ap(Maybe.of(5))
  .ap(Maybe.of(10));
// Some(15)

// If any is None, result is None
Maybe.of((a: number) => (b: number) => a + b)
  .ap(Maybe.of(5))
  .ap(Maybe.none());
// None
```

### Applicative vs Monad

```typescript
// Monad: sequential, dependent
const program = fetchUser(id).flatMap(user =>
  // This depends on 'user'
  fetchPosts(user.id)
);

// Applicative: parallel, independent
const program = liftA2(
  (user, settings) => ({ user, settings }),
  fetchUser(id),
  fetchSettings(id)  // Can run in parallel!
);
```

**Key difference:** Monad's `flatMap` lets the second operation depend on the first. Applicative's `ap` requires independence.

### Building Applicative

```typescript
class MaybeApplicative<T> {
  constructor(private data: T | null) {}

  static of<T>(value: T): MaybeApplicative<T> {
    return new MaybeApplicative(value);
  }

  map<U>(fn: (value: T) => U): MaybeApplicative<U> {
    return this.data !== null
      ? MaybeApplicative.of(fn(this.data))
      : new MaybeApplicative(null);
  }

  ap<U>(
    mfn: MaybeApplicative<(value: T) => U>
  ): MaybeApplicative<U> {
    if (mfn.data === null || this.data === null) {
      return new MaybeApplicative(null);
    }
    return MaybeApplicative.of(mfn.data(this.data));
  }

  // Convenience: combine two values
  static lift2<A, B, C>(
    fn: (a: A, b: B) => C,
    ma: MaybeApplicative<A>,
    mb: MaybeApplicative<B>
  ): MaybeApplicative<C> {
    return MaybeApplicative.of<(a: A) => (b: B) => C>(
      a => b => fn(a, b)
    )
      .ap(ma)
      .ap(mb);
  }
}

// Usage
const result = MaybeApplicative.lift2(
  (a, b) => a + b,
  MaybeApplicative.of(5),
  MaybeApplicative.of(10)
);
// MaybeApplicative(15)
```

## Other Useful Abstractions

### Semigroup: Things You Can Combine

```typescript
interface Semigroup<T> {
  concat(other: T): T;
}

// Numbers with addition
class Sum {
  constructor(private value: number) {}

  concat(other: Sum): Sum {
    return new Sum(this.value + other.value);
  }
}

// Strings with concatenation
const s1 = "Hello";
const s2 = " World";
s1.concat(s2);  // "Hello World"

// Arrays
[1, 2].concat([3, 4]);  // [1, 2, 3, 4]

// Max/Min
class Max {
  constructor(private value: number) {}

  concat(other: Max): Max {
    return new Max(Math.max(this.value, other.value));
  }
}
```

### Monoid: Semigroup with Identity

A Monoid is a Semigroup with an empty/identity element:

```typescript
interface Monoid<T> extends Semigroup<T> {
  empty(): T;
}

// Numbers with addition
class Sum implements Monoid<Sum> {
  constructor(private value: number) {}

  concat(other: Sum): Sum {
    return new Sum(this.value + other.value);
  }

  static empty(): Sum {
    return new Sum(0);  // Identity for addition
  }
}

// Numbers with multiplication
class Product implements Monoid<Product> {
  constructor(private value: number) {}

  concat(other: Product): Product {
    return new Product(this.value * other.value);
  }

  static empty(): Product {
    return new Product(1);  // Identity for multiplication
  }
}

// Arrays
class List<T> implements Monoid<List<T>> {
  constructor(private items: T[]) {}

  concat(other: List<T>): List<T> {
    return new List([...this.items, ...other.items]);
  }

  static empty<T>(): List<T> {
    return new List([]);  // Identity for concatenation
  }
}
```

**Laws:**
```typescript
// Identity
x.concat(Monoid.empty()) === x
Monoid.empty().concat(x) === x

// Associativity
a.concat(b).concat(c) === a.concat(b.concat(c))
```

**Why useful?**
```typescript
// Reduce without an initial value
function fold<T>(monoid: Monoid<T>, values: T[]): T {
  return values.reduce(
    (acc, value) => acc.concat(value),
    monoid.empty()
  );
}

// Works for any monoid!
fold(new Sum(0), [new Sum(1), new Sum(2), new Sum(3)]);  // Sum(6)
fold(new List([]), [new List([1]), new List([2, 3])]);   // List([1, 2, 3])
```

### Foldable: Things You Can Reduce

```typescript
interface Foldable<T> {
  reduce<U>(fn: (acc: U, value: T) => U, initial: U): U;
}

// Array is foldable
[1, 2, 3, 4].reduce((sum, x) => sum + x, 0);  // 10

// Tree is foldable
class Tree<T> implements Foldable<T> {
  constructor(
    private value: T,
    private left?: Tree<T>,
    private right?: Tree<T>
  ) {}

  reduce<U>(fn: (acc: U, value: T) => U, initial: U): U {
    // Process left subtree first
    let acc = this.left ? this.left.reduce(fn, initial) : initial;
    // Process current node
    acc = fn(acc, this.value);
    // Process right subtree
    if (this.right) acc = this.right.reduce(fn, acc);
    return acc;
  }
}
```

### Traversable: Foldable + Functor

Traversable combines mapping with effects. The key insight: **flip the nesting of types**.

**The problem:** You have `List<Maybe<T>>` but want `Maybe<List<T>>`. Traversable does this flip.

**Two operations:**
- `sequence`: Flip the nesting - `F<G<T>>` becomes `G<F<T>>`
- `traverse`: Map then flip - combines `map` and `sequence`

```typescript
// sequence: Turn List<Maybe<T>> into Maybe<List<T>>
// If any Maybe is None, result is None
// If all are Some, result is Some with the list
function sequence<T>(list: Maybe<T>[]): Maybe<T[]> {
  const result: T[] = [];

  for (const maybe of list) {
    if (maybe.isNone()) return Maybe.none();
    result.push(maybe.getValue());
  }

  return Maybe.of(result);
}

// Examples of sequence
sequence([Maybe.of(1), Maybe.of(2), Maybe.of(3)]);
// Maybe.of([1, 2, 3]) - all present, so Some

sequence([Maybe.of(1), Maybe.none(), Maybe.of(3)]);
// Maybe.none() - one missing, so None

// traverse: Map and sequence in one step
// More efficient than mapping then sequencing separately
function traverse<T, U>(
  list: T[],
  fn: (value: T) => Maybe<U>
): Maybe<U[]> {
  const result: U[] = [];

  for (const item of list) {
    const maybe = fn(item);
    if (maybe.isNone()) return Maybe.none();
    result.push(maybe.getValue());
  }

  return Maybe.of(result);
}

// Usage: fetch multiple users
const userIds = ['1', '2', '3'];
const maybeUsers: Maybe<User[]> = traverse(userIds, findUser);
// Maybe<User[]> - Some only if all users found

// Real-world example: validate a list
function validateAll<T>(
  items: T[],
  validate: (item: T) => Either<Error, T>
): Either<Error, T[]> {
  const result: T[] = [];

  for (const item of items) {
    const validated = validate(item);
    if (validated.isLeft()) return validated as any;
    result.push(validated.getValue());
  }

  return Either.right(result);
}

// Usage
const emails = ['a@example.com', 'invalid', 'b@example.com'];
const validated = validateAll(emails, validateEmail);
// Left("Invalid email") - stops at first error
```

**Why useful:**
- Fetch multiple items, fail if any fails
- Validate a list, collect all valid or stop at first error
- Perform effects on a collection, propagate failure
- "Turn inside-out" nested types

## The Family Tree

```
        Semigroup
            ↓
         Monoid
            ↓
        Functor ──→ Foldable
            ↓           ↓
      Applicative   Traversable
         ↓    ↓
    Alternative  Monad
                  ↕
               Comonad (dual)
```

Each adds capabilities:
- **Semigroup**: `concat`
- **Monoid**: `concat` + `empty`
- **Functor**: `map`
- **Applicative**: `map` + `ap` (parallel composition)
- **Alternative**: `ap` + `orElse`/`empty` (choice and failure)
- **Monad**: `map` + `flatMap` (sequential composition)
- **Foldable**: `reduce`
- **Traversable**: `map` + `reduce` + effects
- **Comonad**: `extract` + `extend` (dual of Monad, for context-dependent computation)

### Alternative: Choice and Failure

Alternative adds the ability to choose between options or handle failure:

```typescript
interface Alternative<T> extends Applicative<T> {
  // Empty value (represents failure/no choice)
  static empty<U>(): Alternative<U>;

  // Choose between two alternatives
  orElse(other: Alternative<T>): Alternative<T>;
}

// Maybe with Alternative
Maybe.none()
  .orElse(Maybe.of(5))
  .orElse(Maybe.of(10));
// Some(5) - first success wins

// Try multiple parsers
parseNumber(input)
  .orElse(parseString(input))
  .orElse(parseBoolean(input));
// First successful parse wins

// List as Alternative (concatenation)
[1, 2].orElse([3, 4]);  // [1, 2, 3, 4]
```

**In Haskell:** The `<|>` operator represents "choice" - try the left side, if it fails, try the right.

```haskell
-- Try to find a user, fallback to default
findUser id <|> pure defaultUser

-- Parse alternatives
parseJson input <|> parseXml input <|> parseYaml input
```

### Comonad: The Dual of Monad

Comonad is the categorical dual of Monad. While monads let you put values *into* a context, comonads let you *extract* values from a context and extend computations that depend on context.

```typescript
interface Comonad<T> {
  // Extract the value (dual of 'of')
  extract(): T;

  // Extend a context-dependent computation (dual of 'flatMap')
  extend<U>(fn: (wa: Comonad<T>) => U): Comonad<U>;
}
```

**Example: Store Comonad** (value with context)
```typescript
// Store represents a value with a "cursor" position
class Store<T> {
  constructor(
    private position: number,
    private lookup: (pos: number) => T
  ) {}

  // Extract current value
  extract(): T {
    return this.lookup(this.position);
  }

  // Extend: compute new values based on whole context
  extend<U>(fn: (store: Store<T>) => U): Store<U> {
    return new Store(
      this.position,
      (pos) => fn(new Store(pos, this.lookup))
    );
  }
}

// Example: Game of Life
// Each cell's next state depends on its neighborhood (context)
const grid = new Store(0, cellAtPosition);
const nextGen = grid.extend(cell =>
  countNeighbors(cell) === 3 ? alive : dead
);
```

**Where comonads appear:**
- Image processing (pixel + neighborhood)
- Cellular automata (cell + surrounding cells)
- Streaming data with history
- Zippers (data structure with focus)

**Note:** Comonads are less common than monads in everyday programming. Most developers never need them. They're mentioned here for completeness and for the curious reader exploring the deeper category theory.

## When to Use What

### Use Functor when:
- You just need to transform values
- No need to combine or sequence

```typescript
array.map(x => x * 2)
maybe.map(x => x.toUpperCase())
```

### Use Applicative when:
- Combining independent computations
- Validation (collect all errors)
- Parallel execution

```typescript
Validation.combine3(
  validateName(data.name),
  validateEmail(data.email),
  validateAge(data.age),
  (name, email, age) => ({ name, email, age })
)
```

### Use Monad when:
- Operations depend on previous results
- Need to chain computations
- Sequential operations

```typescript
findUser(id)
  .flatMap(user => fetchPosts(user.id))
  .flatMap(posts => fetchComments(posts[0].id))
```

### Use Semigroup/Monoid when:
- Combining values of the same type
- Folding/reducing collections
- Parallel aggregation

```typescript
strings.reduce((a, b) => a.concat(b), "")
numbers.reduce((a, b) => Sum.concat(a, b), Sum.empty())
```

## Alternative Notations

Different languages express these differently:

### Haskell
```haskell
-- Functor
fmap :: (a -> b) -> f a -> f b

-- Applicative
pure :: a -> f a
(<*>) :: f (a -> b) -> f a -> f b

-- Monad
return :: a -> m a
(>>=) :: m a -> (a -> m b) -> m b

-- Semigroup
(<>) :: a -> a -> a

-- Monoid
mempty :: a
mappend :: a -> a -> a
```

### Scala (Cats)
```scala
// Functor
trait Functor[F[_]] {
  def map[A, B](fa: F[A])(f: A => B): F[B]
}

// Applicative
trait Applicative[F[_]] extends Functor[F] {
  def pure[A](a: A): F[A]
  def ap[A, B](ff: F[A => B])(fa: F[A]): F[B]
}

// Monad
trait Monad[F[_]] extends Applicative[F] {
  def flatMap[A, B](fa: F[A])(f: A => F[B]): F[B]
}

// Semigroup
trait Semigroup[A] {
  def combine(x: A, y: A): A
}

// Monoid
trait Monoid[A] extends Semigroup[A] {
  def empty: A
}
```

## Practical Implications

### Code Reuse

Generic functions work with any appropriate abstraction:

```typescript
// Works with any Functor
function double<F>(functor: Functor<number>): Functor<number> {
  return functor.map(x => x * 2);
}

// Works with any Monad
function flattenTwice<M>(monad: Monad<Monad<Monad<T>>>): Monad<T> {
  return monad.flatMap(m => m.flatMap(x => x));
}

// Works with any Monoid
function concat<T>(monoid: Monoid<T>, values: T[]): T {
  return values.reduce(
    (acc, val) => acc.concat(val),
    monoid.empty()
  );
}
```

### Composition

Abstractions compose:

```typescript
// Functor composition
const f = (x: number) => x * 2;
const g = (x: number) => x + 10;

array.map(f).map(g)
// vs
array.map(x => g(f(x)))

// Monad composition
const h = (x: number) => Maybe.of(x * 2);
const i = (x: number) => Maybe.of(x + 10);

Maybe.of(5).flatMap(h).flatMap(i)
// vs
Maybe.of(5).flatMap(x => h(x).flatMap(i))
```

## What We've Learned

- Monad is part of a larger family
- Functor: map over values
- Applicative: combine independent computations
- Monad: chain dependent computations
- Semigroup: combine values
- Monoid: Semigroup + identity
- Each abstraction has specific use cases
- They compose and build on each other
- Understanding the hierarchy helps choose the right tool

## Final Thoughts

Monads are powerful, but they're not always the answer. Sometimes:
- A Functor is enough (just mapping)
- An Applicative is better (parallel operations)
- A Monoid suffices (combining values)
- A plain function works fine

Understanding the hierarchy helps you choose appropriately. Start simple, reach for more powerful abstractions only when needed.

---

This completes your journey through monads and beyond. You now understand:
- What monads are
- When to use them
- How to build them
- Where they fit in the bigger picture

You get it now.

---

**Return to: [Chapter 20 - Conclusion: You Get It Now](20-conclusion.md)**
