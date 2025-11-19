# Chapter 7: The Laws - Why They Matter

Any type can have `map` and `flatMap` methods. But to be a *proper* monad, it needs to follow three laws.

These aren't arbitrary rules. They're what make composition predictable and safe.

## The Three Laws

### 1. Left Identity

```typescript
of(x).flatMap(f) === f(x)
```

Wrapping a value and immediately flatMapping should be the same as just calling the function.

**Why it matters**: Creating a monad shouldn't add unexpected behavior.

```typescript
// These should be equivalent:
Maybe.of(5).flatMap(x => Maybe.of(x * 2))
// vs
(x => Maybe.of(x * 2))(5)

// Both should give: Some(10)
```

If this law breaks:
```typescript
// BAD: Monad that adds surprise behavior
class WeirdMaybe<T> {
  static of<T>(value: T): WeirdMaybe<T> {
    console.log("Surprise side effect!");  // Breaks left identity
    return new Some(value);
  }
}

WeirdMaybe.of(5).flatMap(f)  // Has side effect
// is NOT the same as
f(5)  // No side effect
```

### 2. Right Identity

```typescript
m.flatMap(of) === m
```

FlatMapping with `of` (wrapping the value) should return the original monad unchanged.

**Why it matters**: `of` shouldn't change the value or context.

```typescript
// These should be equivalent:
Some(5).flatMap(x => Maybe.of(x))
// vs
Some(5)

// Both should give: Some(5)
```

If this law breaks:
```typescript
// BAD: of that modifies the value
class WeirdMaybe<T> {
  static of<T>(value: T): WeirdMaybe<T> {
    return new Some(value + 1);  // Breaks right identity
  }
}

Some(5).flatMap(x => WeirdMaybe.of(x))  // Some(6)
// is NOT the same as
Some(5)  // Some(5)
```

### 3. Associativity

```typescript
m.flatMap(f).flatMap(g) === m.flatMap(x => f(x).flatMap(g))
```

The order of nesting flatMaps shouldn't matter.

**Why it matters**: You can refactor chained operations without changing behavior.

```typescript
// These should be equivalent:
Maybe.of(5)
  .flatMap(x => Maybe.of(x * 2))
  .flatMap(x => Maybe.of(x + 10))

// vs
Maybe.of(5)
  .flatMap(x =>
    Maybe.of(x * 2).flatMap(y =>
      Maybe.of(y + 10)
    )
  )

// Both should give: Some(20)
```

If this law breaks, refactoring changes behavior - very bad!

## Verifying the Laws

Let's verify our Maybe monad follows the laws:

### Left Identity

```typescript
// Law: of(x).flatMap(f) === f(x)

const x = 5;
const f = (n: number) => Maybe.of(n * 2);

// Left side
const left = Maybe.of(x).flatMap(f);
// = Some(5).flatMap(f)
// = f(5)
// = Some(10)

// Right side
const right = f(x);
// = Maybe.of(5 * 2)
// = Some(10)

// left === right ✓
```

### Right Identity

```typescript
// Law: m.flatMap(of) === m

const m = Maybe.of(5);

// Left side
const left = m.flatMap(x => Maybe.of(x));
// = Some(5).flatMap(x => Maybe.of(x))
// = Maybe.of(5)
// = Some(5)

// Right side
const right = m;
// = Some(5)

// left === right ✓
```

### Associativity

```typescript
// Law: m.flatMap(f).flatMap(g) === m.flatMap(x => f(x).flatMap(g))

const m = Maybe.of(5);
const f = (n: number) => Maybe.of(n * 2);
const g = (n: number) => Maybe.of(n + 10);

// Left side
const left = m.flatMap(f).flatMap(g);
// = Some(5).flatMap(f).flatMap(g)
// = Some(10).flatMap(g)
// = Some(20)

// Right side
const right = m.flatMap(x => f(x).flatMap(g));
// = Some(5).flatMap(x => f(x).flatMap(g))
// = f(5).flatMap(g)
// = Some(10).flatMap(g)
// = Some(20)

// left === right ✓
```

## Why Laws Matter for Composition

The laws guarantee that composition works as expected:

```typescript
// Without laws, this refactoring could change behavior:

// Original
function processUser(id: string): Maybe<User> {
  return findUser(id)
    .flatMap(validateUser)
    .flatMap(enrichUser);
}

// Refactored (extracting a helper)
function validateAndEnrich(user: User): Maybe<User> {
  return validateUser(user).flatMap(enrichUser);
}

function processUser(id: string): Maybe<User> {
  return findUser(id).flatMap(validateAndEnrich);
}

// Associativity guarantees these are equivalent!
```

Without associativity, you couldn't safely extract and compose functions. Every refactoring would be a risk.

## Real-World Law Violations

### JavaScript Promise Gotcha

JavaScript Promises *almost* follow the monad laws, but not quite:

```javascript
// Promises flatten multiple levels automatically
Promise.resolve(Promise.resolve(5))  // Promise(5), not Promise(Promise(5))

// This breaks associativity in edge cases:
const m = Promise.resolve(1);
const f = x => Promise.resolve(Promise.resolve(x + 1));
const g = x => Promise.resolve(x * 2);

// These are NOT the same:
m.then(f).then(g)  // Different from
m.then(x => f(x).then(g))

// Because f returns a nested Promise, which gets auto-flattened
```

This usually isn't a problem in practice, but technically Promises aren't pure monads.

### Array Gotcha

Arrays follow the laws, but with a twist:

```javascript
// These ARE equivalent (associativity holds)
[1, 2].flatMap(x => [x, x * 10]).flatMap(x => [x, x + 1])
// [1, 2, 10, 11, 2, 3, 20, 21]

[1, 2].flatMap(x => [x, x * 10].flatMap(y => [y, y + 1]))
// [1, 2, 10, 11, 2, 3, 20, 21]

// But beware: order matters for side effects!
[1, 2].flatMap(x => {
  console.log(x);  // Logs: 1, 2
  return [x, x * 10];
})

// The laws assume pure functions - no side effects!
```

## Testing the Laws

Here's a property-based test for the monad laws:

```typescript
import { expect } from 'chai';

function testMonadLaws<T, U, V>(
  // Test data
  x: T,
  m: Monad<T>,

  // Test functions
  f: (a: T) => Monad<U>,
  g: (b: U) => Monad<V>,

  // Monad constructor
  M: { of: <A>(a: A) => Monad<A> }
) {
  // Left identity: M.of(x).flatMap(f) == f(x)
  expect(
    M.of(x).flatMap(f)
  ).to.deep.equal(
    f(x)
  );

  // Right identity: m.flatMap(M.of) == m
  expect(
    m.flatMap(a => M.of(a))
  ).to.deep.equal(
    m
  );

  // Associativity: m.flatMap(f).flatMap(g) == m.flatMap(x => f(x).flatMap(g))
  expect(
    m.flatMap(f).flatMap(g)
  ).to.deep.equal(
    m.flatMap(x => f(x).flatMap(g))
  );
}

// Usage
testMonadLaws(
  5,
  Maybe.of(5),
  x => Maybe.of(x * 2),
  x => Maybe.of(x + 10),
  Maybe
);
```

## The Laws in Different Languages

### Haskell

```haskell
-- The laws are part of the Monad documentation:

-- Left identity
return a >>= f  ≡  f a

-- Right identity
m >>= return  ≡  m

-- Associativity
(m >>= f) >>= g  ≡  m >>= (\x -> f x >>= g)

-- Example proof for Maybe:
-- Left identity:
return 5 >>= (\x -> Just (x * 2))
= Just 5 >>= (\x -> Just (x * 2))
= (\x -> Just (x * 2)) 5
= Just 10

-- Haskell's type system doesn't enforce these,
-- but they're expected by convention
```

### Scala

```scala
// The laws for a Monad typeclass:
trait Monad[M[_]] {
  def pure[A](a: A): M[A]
  def flatMap[A, B](ma: M[A])(f: A => M[B]): M[B]

  // Laws (not enforced, but tested):

  // Left identity
  // pure(a).flatMap(f) == f(a)

  // Right identity
  // m.flatMap(pure) == m

  // Associativity
  // m.flatMap(f).flatMap(g) == m.flatMap(a => f(a).flatMap(g))
}

// Libraries like Cats include law testing:
import cats.laws.discipline.MonadTests
import cats.instances.option._

checkAll("Option", MonadTests[Option].monad[Int, Int, Int])
```

### Rust

```rust
// Rust doesn't have HKT, so no formal Monad trait
// But Option and Result follow the laws:

// Left identity for Option:
// Some(x).and_then(f) == f(x)
assert_eq!(
    Some(5).and_then(|x| Some(x * 2)),
    (|x| Some(x * 2))(5)
);

// Right identity:
// m.and_then(Some) == m
let m = Some(5);
assert_eq!(
    m.and_then(|x| Some(x)),
    m
);

// Associativity:
// m.and_then(f).and_then(g) == m.and_then(|x| f(x).and_then(g))
let f = |x: i32| Some(x * 2);
let g = |x: i32| Some(x + 10);

assert_eq!(
    m.and_then(f).and_then(g),
    m.and_then(|x| f(x).and_then(g))
);
```

## What Breaks When Laws Fail

### Broken Left Identity

```typescript
class BrokenMaybe<T> {
  static of<T>(value: T): BrokenMaybe<T> {
    // Secretly modifies the value
    return new Some(value + 1);  // BAD!
  }

  flatMap<U>(fn: (value: T) => BrokenMaybe<U>): BrokenMaybe<U> {
    return fn(this.value);
  }
}

// Problem: Can't extract intermediate steps
const direct = BrokenMaybe.of(5).flatMap(x => BrokenMaybe.of(x * 2));
// Some(13) - wrong! Expected Some(12)

const extracted = (x => BrokenMaybe.of(x * 2))(5);
// Some(11) - different result!

// Refactoring changes behavior - disaster!
```

### Broken Right Identity

```typescript
class BrokenMaybe<T> {
  static of<T>(value: T): BrokenMaybe<T> {
    return new Some(value);
  }

  flatMap<U>(fn: (value: T) => BrokenMaybe<U>): BrokenMaybe<U> {
    // Secretly modifies before applying fn
    const modified = this.value + 1;  // BAD!
    return fn(modified);
  }
}

// Problem: Can't simplify .flatMap(of)
const m = new Some(5);
const result = m.flatMap(x => BrokenMaybe.of(x));
// Some(6) - not Some(5)!

// This should be a no-op, but isn't
```

### Broken Associativity

```typescript
class BrokenMaybe<T> {
  flatMap<U>(fn: (value: T) => BrokenMaybe<U>): BrokenMaybe<U> {
    // Applies fn twice!
    return fn(this.value).flatMap(x => fn(x));  // BAD!
  }
}

// Problem: Can't regroup operations
const m = BrokenMaybe.of(1);
const f = (x: number) => BrokenMaybe.of(x + 1);
const g = (x: number) => BrokenMaybe.of(x * 2);

m.flatMap(f).flatMap(g)
// = BrokenMaybe.of(1).flatMap(f).flatMap(g)
// = f(1).flatMap(x => f(x)).flatMap(g)
// = Some(2).flatMap(x => f(x)).flatMap(g)
// Chaos ensues...

// vs
m.flatMap(x => f(x).flatMap(g))
// Different result!
```

## Functor Laws (Bonus)

Before being a monad, a type should be a functor - something you can `map` over.

Functors have two laws:

### 1. Identity

```typescript
m.map(x => x) === m
```

Mapping with the identity function should return the original value.

### 2. Composition

```typescript
m.map(f).map(g) === m.map(x => g(f(x)))
```

Mapping twice should be the same as mapping with the composition.

These ensure `map` is a proper transformation that doesn't do weird things.

## Practical Takeaways

When implementing your own monads:

1. **Keep `of` pure** - No side effects, no modifications
2. **Keep `flatMap` pure** - No hidden behavior
3. **Test the laws** - Use property-based testing
4. **If laws break** - Reconsider your design

When using monads:

1. **Assume laws hold** - Refactor confidently
2. **Chain operations** - Composition just works
3. **Extract functions** - Associativity guarantees equivalence

## Philosophical Note

The laws aren't about being academically correct. They're about making composition predictable.

Code is read and modified far more than it's written. The laws ensure that when you refactor:
- Extract a function - behavior stays the same (associativity)
- Inline a function - behavior stays the same (associativity)
- Remove redundant wrapping - behavior stays the same (identity)

They're the foundation of compositional programming.

## Exercise

Which of these monad implementations are valid?

```typescript
// 1.
class Identity<T> {
  constructor(private value: T) {}

  static of<T>(value: T): Identity<T> {
    return new Identity(value);
  }

  flatMap<U>(fn: (value: T) => Identity<U>): Identity<U> {
    return fn(this.value);
  }
}

// 2.
class Logged<T> {
  constructor(private value: T, private log: string[] = []) {}

  static of<T>(value: T): Logged<T> {
    return new Logged(value, ["Created"]);  // Logs creation
  }

  flatMap<U>(fn: (value: T) => Logged<U>): Logged<U> {
    const result = fn(this.value);
    return new Logged(
      result.value,
      [...this.log, ...result.log]
    );
  }
}

// 3.
class Random<T> {
  constructor(private value: T) {}

  static of<T>(value: T): Random<T> {
    return new Random(value);
  }

  flatMap<U>(fn: (value: T) => Random<U>): Random<U> {
    const jittered = this.value + Math.random() * 0.01;  // Adds noise
    return fn(jittered);
  }
}
```

Test each against the three laws!

## What We've Learned

- Three laws: left identity, right identity, associativity
- Laws ensure composition is predictable
- Laws enable refactoring without changing behavior
- Breaking laws breaks composition
- The laws aren't academic - they're practical guarantees

## Coming Up

Now that we understand the theory, let's see monads tackle one of programming's hardest problems: side effects. How do you do I/O in a pure functional way? The IO monad has the answer.

---

## Exercise Answers

```typescript
// 1. Identity: ✓ VALID
// Follows all laws - it's the simplest monad possible

// 2. Logged: ✗ BREAKS LEFT IDENTITY
// Identity.of(5).flatMap(f) !== f(5)
// Left side has log ["Created"], right side doesn't
// But it could be fixed by removing the "Created" log in `of`

// 3. Random: ✗ BREAKS ASSOCIATIVITY (and everything else)
// Adding random jitter makes operations non-deterministic
// Each operation produces different results - laws require consistency
```

---

**Next: [Chapter 8 - IO: Taming Side Effects](08-io-monad.md)**
