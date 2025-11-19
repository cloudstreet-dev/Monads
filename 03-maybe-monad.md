# Chapter 3: Maybe - Your First Monad

Let's build a monad from scratch. Not to be academic, but because building it will make everything click.

## The Core Idea

We want to represent a value that might or might not exist, without using `null`. Why not `null`? Because `null` is a landmine - you forget to check it once and your program explodes.

Instead, let's make the "might not exist" part explicit in the type:

```typescript
type Maybe<T> = Some<T> | None;

class Some<T> {
  constructor(private value: T) {}

  getValue(): T {
    return this.value;
  }
}

class None {
  // Represents the absence of a value
}
```

Now `Maybe<string>` can be either:
- `Some("hello")` - there's a value
- `None` - there's no value

The type system forces you to handle both cases. No surprises.

## Building It Up

Let's start with just wrapping values:

```typescript
// Wrap a value in a Maybe
const someValue: Maybe<number> = new Some(42);
const noValue: Maybe<number> = new None();

// To use the value, we must handle both cases
function printMaybe(m: Maybe<number>): void {
  if (m instanceof Some) {
    console.log(`Got: ${m.getValue()}`);
  } else {
    console.log("Got nothing");
  }
}
```

This works, but we're still doing manual checks. Not much better than null checks.

## Adding Map

Here's where it gets interesting. Let's add a method to transform the value inside:

```typescript
class Some<T> {
  constructor(private value: T) {}

  map<U>(fn: (value: T) => U): Maybe<U> {
    return new Some(fn(this.value));
  }

  getValue(): T {
    return this.value;
  }
}

class None {
  map<U>(fn: (value: any) => U): Maybe<U> {
    return this; // Stay None
  }
}
```

Now we can transform values without unwrapping:

```typescript
const maybe = new Some(5);

// Transform the value inside
const doubled = maybe.map(x => x * 2);  // Some(10)
const asString = doubled.map(x => x.toString());  // Some("10")

// If it's None, transformations do nothing
const nothing = new None();
const stillNothing = nothing
  .map(x => x * 2)
  .map(x => x.toString());  // Still None
```

The `None` case automatically propagates through the chain. No manual checks needed!

## The FlatMap Problem

But what if your transformation returns a `Maybe`?

```typescript
function divide(a: number, b: number): Maybe<number> {
  return b === 0 ? new None() : new Some(a / b);
}

// Oops - this gives us Maybe<Maybe<number>>!
const result = new Some(10).map(x => divide(x, 2));
```

We end up with nested Maybes. We need a way to flatten them.

## Enter FlatMap

This is the secret sauce of monads:

```typescript
class Some<T> {
  constructor(private value: T) {}

  map<U>(fn: (value: T) => U): Maybe<U> {
    return new Some(fn(this.value));
  }

  flatMap<U>(fn: (value: T) => Maybe<U>): Maybe<U> {
    return fn(this.value);  // Apply and return directly
  }

  getValue(): T {
    return this.value;
  }
}

class None {
  map<U>(fn: (value: any) => U): Maybe<U> {
    return this;
  }

  flatMap<U>(fn: (value: any) => Maybe<U>): Maybe<U> {
    return this;  // Stay None
  }
}
```

Now we can chain operations that return Maybes:

```typescript
function divide(a: number, b: number): Maybe<number> {
  return b === 0 ? new None() : new Some(a / b);
}

function squareRoot(n: number): Maybe<number> {
  return n < 0 ? new None() : new Some(Math.sqrt(n));
}

// Chain operations that might fail
const result = new Some(16)
  .flatMap(x => divide(x, 2))    // Some(8)
  .flatMap(x => squareRoot(x))   // Some(2.828...)
  .map(x => x.toFixed(2));       // Some("2.83")

// If any step fails, the rest are skipped
const failed = new Some(16)
  .flatMap(x => divide(x, 0))    // None (division by zero)
  .flatMap(x => squareRoot(x))   // Skipped
  .map(x => x.toFixed(2));       // Still None
```

This is monadic chaining. Each operation can fail, but we don't check for failure manually - it's handled automatically.

## Real-World Example: User Lookup

Remember this nightmare from Chapter 2?

```typescript
const user = database.findUser(userId);
if (user !== null) {
  const profile = user.getProfile();
  if (profile !== null) {
    const email = profile.getPrimaryEmail();
    if (email !== null) {
      return email.toLowerCase();
    }
  }
}
return null;
```

With Maybe:

```typescript
function findUser(id: string): Maybe<User> {
  const user = database.findUser(id);
  return user ? new Some(user) : new None();
}

function getProfile(user: User): Maybe<Profile> {
  const profile = user.profile;
  return profile ? new Some(profile) : new None();
}

function getEmail(profile: Profile): Maybe<string> {
  const email = profile.email;
  return email ? new Some(email) : new None();
}

// Clean chain
const email = findUser(userId)
  .flatMap(getProfile)
  .flatMap(getEmail)
  .map(email => email.toLowerCase());
```

No nested ifs. No manual null checks. The chain automatically short-circuits on the first `None`.

## Maybe Across Languages

This pattern appears everywhere with different names:

### Rust: Option<T>

```rust
fn divide(a: f64, b: f64) -> Option<f64> {
    if b == 0.0 {
        None
    } else {
        Some(a / b)
    }
}

fn square_root(n: f64) -> Option<f64> {
    if n < 0.0 {
        None
    } else {
        Some(n.sqrt())
    }
}

// Chain with and_then (Rust's flatMap)
let result = Some(16.0)
    .and_then(|x| divide(x, 2.0))
    .and_then(|x| square_root(x))
    .map(|x| format!("{:.2}", x));

match result {
    Some(s) => println!("Result: {}", s),
    None => println!("Calculation failed"),
}
```

### Scala: Option[T]

```scala
def divide(a: Double, b: Double): Option[Double] =
  if (b == 0) None else Some(a / b)

def squareRoot(n: Double): Option[Double] =
  if (n < 0) None else Some(math.sqrt(n))

// Chain with flatMap
val result = Some(16.0)
  .flatMap(x => divide(x, 2))
  .flatMap(x => squareRoot(x))
  .map(x => f"$x%.2f")

result match {
  case Some(s) => println(s"Result: $s")
  case None => println("Calculation failed")
}

// Or use for-comprehension (syntactic sugar)
val result2 = for {
  x <- Some(16.0)
  y <- divide(x, 2)
  z <- squareRoot(y)
} yield f"$z%.2f"
```

### Haskell: Maybe a

```haskell
divide :: Double -> Double -> Maybe Double
divide _ 0 = Nothing
divide a b = Just (a / b)

squareRoot :: Double -> Maybe Double
squareRoot n
  | n < 0     = Nothing
  | otherwise = Just (sqrt n)

-- Chain with >>= (bind operator, same as flatMap)
result :: Maybe String
result = Just 16.0
  >>= (\x -> divide x 2)
  >>= squareRoot
  >>= (\x -> Just (show x))

-- Or use do-notation
result' :: Maybe String
result' = do
  x <- Just 16.0
  y <- divide x 2
  z <- squareRoot y
  return (show z)
```

### F#: Option<'T>

```fsharp
let divide a b =
    if b = 0.0 then None
    else Some(a / b)

let squareRoot n =
    if n < 0.0 then None
    else Some(sqrt n)

// Chain with Option.bind (flatMap)
let result =
    Some 16.0
    |> Option.bind (fun x -> divide x 2.0)
    |> Option.bind squareRoot
    |> Option.map (sprintf "%.2f")

// Or use computation expressions
let result' = option {
    let! x = Some 16.0
    let! y = divide x 2.0
    let! z = squareRoot y
    return sprintf "%.2f" z
}
```

### Java: Optional<T>

```java
Optional<Double> divide(double a, double b) {
    return b == 0 ? Optional.empty() : Optional.of(a / b);
}

Optional<Double> squareRoot(double n) {
    return n < 0 ? Optional.empty() : Optional.of(Math.sqrt(n));
}

// Chain with flatMap
Optional<String> result = Optional.of(16.0)
    .flatMap(x -> divide(x, 2))
    .flatMap(this::squareRoot)
    .map(x -> String.format("%.2f", x));

result.ifPresentOrElse(
    s -> System.out.println("Result: " + s),
    () -> System.out.println("Calculation failed")
);
```

## The Pattern

Notice what's the same across all languages:

1. **Wrap values**: `Some/Just/Some/Optional.of`
2. **Represent absence**: `None/Nothing/None/empty()`
3. **Transform values**: `map` (transform the value inside)
4. **Chain operations**: `flatMap/and_then/bind/>>=` (chain operations that return Maybe)

This is the monad pattern:
- A container type (`Maybe<T>`)
- A way to put values in (`Some`)
- A way to chain operations (`flatMap`)

## Why This Matters

Compare these two approaches:

```typescript
// Manual null handling
function processUser(id: string): string | null {
  const user = findUser(id);
  if (!user) return null;

  const profile = getProfile(user);
  if (!profile) return null;

  const email = getEmail(profile);
  if (!email) return null;

  return email.toLowerCase();
}

// Monadic handling
function processUser(id: string): Maybe<string> {
  return findUser(id)
    .flatMap(getProfile)
    .flatMap(getEmail)
    .map(email => email.toLowerCase());
}
```

The monadic version:
- **Shorter**: Less boilerplate
- **Clearer**: The happy path is obvious
- **Safer**: Can't forget null checks
- **Composable**: Easy to add more steps

## Helper Methods

Real implementations add useful helpers:

```typescript
class Some<T> {
  // ... map, flatMap as before ...

  getOrElse(defaultValue: T): T {
    return this.value;
  }

  filter(predicate: (value: T) => boolean): Maybe<T> {
    return predicate(this.value) ? this : new None();
  }

  toString(): string {
    return `Some(${this.value})`;
  }
}

class None {
  // ... map, flatMap as before ...

  getOrElse<T>(defaultValue: T): T {
    return defaultValue;
  }

  filter(predicate: (value: any) => boolean): Maybe<any> {
    return this;
  }

  toString(): string {
    return "None";
  }
}

// Usage
const result = findUser("123")
  .flatMap(getProfile)
  .flatMap(getEmail)
  .getOrElse("no-email@example.com");

const adults = findUser("123")
  .filter(user => user.age >= 18);
```

## The Aha Moment

Maybe is a monad because it:

1. **Wraps values in a context**: The context is "this might not exist"
2. **Lets you transform the value**: With `map`
3. **Lets you chain context-producing operations**: With `flatMap`
4. **Handles the context automatically**: None propagates through chains

That's it. That's what makes it monadic.

The `flatMap` function is the key. It's what lets you chain operations without nested structures. In formal terms, it's called "bind" and is often written as `>>=` in Haskell.

## What About Promises?

JavaScript developers: this should look familiar.

```javascript
// Promises
fetch('/user/123')
  .then(response => response.json())
  .then(user => fetch(`/profile/${user.id}`))  // flatMap!
  .then(profile => profile.email)
  .catch(error => console.error(error));

// Maybe
findUser('123')
  .flatMap(user => getProfile(user.id))
  .flatMap(profile => getEmail(profile))
  .getOrElse(null);
```

Promises are monads! They represent "a value that will exist in the future" and let you chain async operations. The `.then()` method is doing the same job as `flatMap`.

We'll explore this more in Chapter 12.

## Exercise Time

Try implementing these methods for Maybe:

```typescript
// Combine two Maybes with a function
function map2<A, B, C>(
  ma: Maybe<A>,
  mb: Maybe<B>,
  fn: (a: A, b: B) => C
): Maybe<C> {
  // Your implementation here
}

// Convert an array of Maybes to a Maybe of array
// (only if all are Some)
function sequence<T>(maybes: Maybe<T>[]): Maybe<T[]> {
  // Your implementation here
}

// Example usage:
const a = new Some(2);
const b = new Some(3);
map2(a, b, (x, y) => x + y);  // Some(5)

const list = [new Some(1), new Some(2), new Some(3)];
sequence(list);  // Some([1, 2, 3])

const listWithNone = [new Some(1), new None(), new Some(3)];
sequence(listWithNone);  // None
```

Answers at the end of this chapter.

## What We've Learned

- Maybe wraps values that might not exist
- `map` transforms the value inside
- `flatMap` chains operations that return Maybe
- The pattern appears in many languages with different names
- It's not magic - it's a design pattern for composition

## Coming Up

Maybe solves the null problem, but what about errors? When something fails, we often want to know *why*. That's where Either comes in.

---

## Exercise Answers

```typescript
function map2<A, B, C>(
  ma: Maybe<A>,
  mb: Maybe<B>,
  fn: (a: A, b: B) => C
): Maybe<C> {
  return ma.flatMap(a =>
    mb.map(b =>
      fn(a, b)
    )
  );
}

function sequence<T>(maybes: Maybe<T>[]): Maybe<T[]> {
  const result: T[] = [];

  for (const maybe of maybes) {
    if (maybe instanceof Some) {
      result.push(maybe.getValue());
    } else {
      return new None();  // Any None makes the whole thing None
    }
  }

  return new Some(result);
}

// More functional version of sequence
function sequence<T>(maybes: Maybe<T>[]): Maybe<T[]> {
  return maybes.reduce(
    (acc: Maybe<T[]>, maybe: Maybe<T>) =>
      map2(acc, maybe, (arr, value) => [...arr, value]),
    new Some([])
  );
}
```

---

**Next: [Chapter 4 - Either: When Failure Has Details](04-either-monad.md)**
