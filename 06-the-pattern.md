# Chapter 6: The Pattern Emerges

You've seen three monads. Let's zoom out and see what they have in common.

## The Common Structure

Every monad we've seen has three things:

### 1. A Type Constructor

A way to wrap a value in a context:

```typescript
Maybe<T>      // Context: might not exist
Either<E, T>  // Context: might be an error
Array<T>      // Context: multiple values
```

The type itself is generic - it works with any value type `T`.

### 2. A "Return" or "Unit" Function

A way to put a value into the minimal context:

```typescript
// Maybe
function of<T>(value: T): Maybe<T> {
  return new Some(value);
}

// Either
function of<T>(value: T): Either<never, T> {
  return new Right(value);
}

// Array
function of<T>(value: T): T[] {
  return [value];
}
```

This is sometimes called `return`, `pure`, `unit`, or `of`. It lifts a regular value into the monadic context.

### 3. A "Bind" or "FlatMap" Function

A way to chain operations that return a monad:

```typescript
// Maybe
class Some<T> {
  flatMap<U>(fn: (value: T) => Maybe<U>): Maybe<U> {
    return fn(this.value);
  }
}

// Either
class Right<R> {
  flatMap<U>(fn: (value: R) => Either<L, U>): Either<L, U> {
    return fn(this.value);
  }
}

// Array
Array.prototype.flatMap = function<T, U>(
  fn: (value: T) => U[]
): U[] {
  return this.map(fn).flat();
}
```

This is called `flatMap`, `bind`, `>>=`, `chain`, or `andThen` depending on the language.

## The Pattern in TypeScript

Let's write a generic monad interface:

```typescript
interface Monad<T> {
  // Transform the value inside
  map<U>(fn: (value: T) => U): Monad<U>;

  // Chain operations that return a monad
  flatMap<U>(fn: (value: T) => Monad<U>): Monad<U>;
}

// Plus a static method to wrap values
interface MonadStatic {
  of<T>(value: T): Monad<T>;
}
```

Now our monads fit this interface:

```typescript
class Some<T> implements Monad<T> {
  constructor(private value: T) {}

  static of<T>(value: T): Maybe<T> {
    return new Some(value);
  }

  map<U>(fn: (value: T) => U): Maybe<U> {
    return new Some(fn(this.value));
  }

  flatMap<U>(fn: (value: T) => Maybe<U>): Maybe<U> {
    return fn(this.value);
  }
}

class Right<R> implements Monad<R> {
  constructor(private value: R) {}

  static of<R>(value: R): Either<never, R> {
    return new Right(value);
  }

  map<U>(fn: (value: R) => U): Either<never, U> {
    return new Right(fn(this.value));
  }

  flatMap<U>(fn: (value: R) => Either<any, U>): Either<any, U> {
    return fn(this.value);
  }
}

// Arrays already have these methods!
```

## Why Map AND FlatMap?

You might wonder: why do we need both `map` and `flatMap`?

Here's the thing: **you can implement `map` using `flatMap`**:

```typescript
class Some<T> {
  flatMap<U>(fn: (value: T) => Maybe<U>): Maybe<U> {
    return fn(this.value);
  }

  // map in terms of flatMap
  map<U>(fn: (value: T) => U): Maybe<U> {
    return this.flatMap(value => Some.of(fn(value)));
    //                            ^^^^^^^ wrap result
  }
}
```

So technically, you only *need* `flatMap` and `of` to have a monad. But `map` is so common that it's always included for convenience.

## Seeing the Pattern Everywhere

Once you see it, you can't unsee it. Here are more monads:

### Promise (Async)

```typescript
// Promise is a monad!
Promise.resolve(5)                    // of
  .then(x => x * 2)                   // map
  .then(x => fetch(`/api/${x}`))      // flatMap (returns Promise)
  .then(response => response.json())  // flatMap
```

Promises represent "a value that will exist in the future." The context is asynchronous computation.

### Observable (RxJS)

```typescript
import { of } from 'rxjs';
import { map, flatMap } from 'rxjs/operators';

of(1, 2, 3)                           // of
  .pipe(
    map(x => x * 2),                  // map
    flatMap(x => fetchUser(x))        // flatMap
  )
```

Observables represent "a stream of values over time." The context is asynchronous, potentially infinite sequences.

### Function Composition

Even functions are monads! The context is "waiting for an argument."

```typescript
// Function as monad (also called Reader monad)
type Fn<A, B> = (a: A) => B;

// of: wrap a value in a function
function of<A, B>(value: B): Fn<A, B> {
  return (a: A) => value;
}

// flatMap: compose functions
function flatMap<A, B, C>(
  fn: Fn<A, B>,
  next: (b: B) => Fn<A, C>
): Fn<A, C> {
  return (a: A) => {
    const b = fn(a);
    const nextFn = next(b);
    return nextFn(a);
  };
}

// Usage: dependency injection without globals
interface Config {
  apiUrl: string;
  timeout: number;
}

type Reader<T> = (config: Config) => T;

function getApiUrl(): Reader<string> {
  return config => config.apiUrl;
}

function getTimeout(): Reader<number> {
  return config => config.timeout;
}

function fetchUser(id: string): Reader<Promise<User>> {
  return config =>
    fetch(`${config.apiUrl}/users/${id}`, {
      timeout: config.timeout
    }).then(r => r.json());
}

// Chain them
const getUserById = (id: string): Reader<Promise<User>> =>
  flatMap(getApiUrl(), url =>
    flatMap(getTimeout(), timeout =>
      fetchUser(id)
    )
  );
```

We'll explore this more in Chapter 10.

## The Three Operations

Let's be precise about what each operation does:

### Of (Return, Pure, Unit)

```typescript
of: <T>(value: T) => M<T>
```

Takes a regular value and puts it in the minimal monadic context.

- `Maybe.of(5)` → `Some(5)`
- `Either.of(5)` → `Right(5)`
- `Array.of(5)` → `[5]`
- `Promise.of(5)` → `Promise.resolve(5)`

### Map (Fmap)

```typescript
map: <T, U>(M<T>, fn: (T) => U) => M<U>
```

Transforms the value inside the monad, staying in the same context.

- `Some(5).map(x => x * 2)` → `Some(10)`
- `Right(5).map(x => x * 2)` → `Right(10)`
- `[5].map(x => x * 2)` → `[10]`

### FlatMap (Bind, Chain, >>=)

```typescript
flatMap: <T, U>(M<T>, fn: (T) => M<U>) => M<U>
```

Chains operations that produce monads, flattening the result.

- `Some(5).flatMap(x => Some(x * 2))` → `Some(10)`
- `Some(5).flatMap(x => None)` → `None`
- `[1, 2].flatMap(x => [x, x * 10])` → `[1, 10, 2, 20]`

## The Power of Composition

Why is this useful? Because it lets us compose operations without manually managing the context.

### Without Monads

```typescript
// Nested null checks
function getUserEmail(userId: string): string | null {
  const user = findUser(userId);
  if (user === null) return null;

  const profile = getProfile(user);
  if (profile === null) return null;

  const email = getEmail(profile);
  if (email === null) return null;

  return email.toLowerCase();
}

// Nested try-catch
function processOrder(orderId: string): Order | null {
  try {
    const order = fetchOrder(orderId);
    try {
      const payment = processPayment(order);
      try {
        return shipOrder(order, payment);
      } catch (e) {
        console.error("Shipping failed", e);
        return null;
      }
    } catch (e) {
      console.error("Payment failed", e);
      return null;
    }
  } catch (e) {
    console.error("Fetch failed", e);
    return null;
  }
}
```

### With Monads

```typescript
// Clean chain
function getUserEmail(userId: string): Maybe<string> {
  return findUser(userId)
    .flatMap(getProfile)
    .flatMap(getEmail)
    .map(email => email.toLowerCase());
}

// Clean chain
function processOrder(orderId: string): Either<Error, Order> {
  return fetchOrder(orderId)
    .flatMap(processPayment)
    .flatMap(shipOrder);
}
```

The monad handles the "plumbing" of checking for null, propagating errors, etc.

## Visualizing FlatMap

Think of `flatMap` as:

```
        Regular Function
        ──────────────
        T => M<U>

M<T> ──────────────▶ M<M<U>> ──flatten──▶ M<U>
     │              │
     │   apply fn   │  flatten
     └──────────────┘
```

Without flatten, you'd get nested monads (`Maybe<Maybe<T>>`, `Either<E, Either<E, T>>`, etc.).

`flatMap` applies the function AND flattens the result in one step.

## The Formal Definition

In category theory terms (you don't need to understand this, but some people like it):

A monad is a triple `(M, return, bind)` where:

- `M` is a type constructor (e.g., `Maybe<T>`)
- `return` (aka `of`) has type: `T → M<T>`
- `bind` (aka `flatMap`) has type: `M<T> → (T → M<U>) → M<U>`

And these satisfy three laws (which we'll cover in the next chapter).

## In Different Languages

### Haskell

```haskell
-- Monad typeclass
class Monad m where
  return :: a -> m a
  (>>=)  :: m a -> (a -> m b) -> m b

-- Maybe instance
instance Monad Maybe where
  return x = Just x

  (Just x) >>= f = f x
  Nothing  >>= f = Nothing

-- Usage
getUserEmail :: String -> Maybe String
getUserEmail userId =
  findUser userId
    >>= getProfile
    >>= getEmail
    >>= (\email -> return (map toLower email))

-- Or with do-notation
getUserEmail' :: String -> Maybe String
getUserEmail' userId = do
  user <- findUser userId
  profile <- getProfile user
  email <- getEmail profile
  return (map toLower email)
```

### Scala

```scala
// Monad trait (simplified)
trait Monad[M[_]] {
  def pure[A](a: A): M[A]
  def flatMap[A, B](ma: M[A])(f: A => M[B]): M[B]
}

// Option instance
implicit val optionMonad = new Monad[Option] {
  def pure[A](a: A): Option[A] = Some(a)

  def flatMap[A, B](ma: Option[A])(f: A => Option[B]): Option[B] =
    ma match {
      case Some(a) => f(a)
      case None => None
    }
}

// Usage
def getUserEmail(userId: String): Option[String] =
  findUser(userId)
    .flatMap(getProfile)
    .flatMap(getEmail)
    .map(_.toLowerCase)

// Or with for-comprehension
def getUserEmail(userId: String): Option[String] = for {
  user <- findUser(userId)
  profile <- getProfile(user)
  email <- getEmail(profile)
} yield email.toLowerCase
```

### Rust (Traits)

```rust
// Monad-like trait (Rust doesn't have HKT yet)
trait Monad<T> {
    fn pure(value: T) -> Self;
    fn flat_map<U, F>(self, f: F) -> impl Monad<U>
        where F: FnOnce(T) -> impl Monad<U>;
}

// Option implementation
impl<T> Monad<T> for Option<T> {
    fn pure(value: T) -> Self {
        Some(value)
    }

    fn flat_map<U, F>(self, f: F) -> Option<U>
        where F: FnOnce(T) -> Option<U>
    {
        match self {
            Some(value) => f(value),
            None => None,
        }
    }
}

// Usage (using and_then instead of flat_map)
fn get_user_email(user_id: String) -> Option<String> {
    find_user(user_id)
        .and_then(get_profile)
        .and_then(get_email)
        .map(|email| email.to_lowercase())
}

// Or with ? operator
fn get_user_email(user_id: String) -> Option<String> {
    let user = find_user(user_id)?;
    let profile = get_profile(user)?;
    let email = get_email(profile)?;
    Some(email.to_lowercase())
}
```

## Why "Monad"?

The name comes from category theory. A monad is a specific type of functor (something that can be mapped over) with additional structure.

But forget the name for a moment. The pattern is:

**A way to chain operations that produce contexts, automatically handling the context.**

That's it. Whether you call it a monad, a chainable context, or a programmable semicolon, the pattern is the same.

## What Makes Something Monadic

Not every type with `map` and `flatMap` is a useful monad. To be useful, it should:

1. **Represent a computational context**
   - Maybe: possible absence
   - Either: possible failure
   - List: nondeterminism
   - Promise: asynchrony

2. **Handle the context automatically**
   - Maybe propagates None
   - Either propagates Left
   - List explores all branches
   - Promise waits for resolution

3. **Allow composition**
   - Chain operations without nesting
   - Build complex pipelines from simple pieces

## The "Aha!" Moment

If you're still not sure you get it, try this mental model:

**A monad is a context-aware function composition operator.**

Regular function composition:
```typescript
f: A => B
g: B => C
h = g(f(x))  // A => C
```

Monadic composition:
```typescript
f: A => M<B>
g: B => M<C>
h = f(x).flatMap(g)  // M<C>
```

`flatMap` lets you compose functions that return contexts, handling the context automatically.

## Practical Checklist

Something is a monad if:

- ✅ It's a generic type `M<T>`
- ✅ You can put values in: `of: T => M<T>`
- ✅ You can transform values: `map: (M<T>, T => U) => M<U>`
- ✅ You can chain operations: `flatMap: (M<T>, T => M<U>) => M<U>`
- ✅ It follows three laws (next chapter)

## Exercise

Identify which of these are monads:

1. `Set<T>` - a collection with unique values
2. `Map<K, V>` - a key-value dictionary
3. `Tree<T>` - a binary tree
4. `Validation<E, T>` - like Either but accumulates errors
5. `Stream<T>` - an infinite lazy sequence

Try implementing `of` and `flatMap` for each!

## What We've Learned

- Monads are a pattern: wrap, map, flatMap
- The pattern appears across different contexts
- `flatMap` is the key - it chains operations and flattens results
- You've been using monadic patterns in many languages
- The "context" is what makes each monad unique

## Coming Up

We've seen what monads are. But what makes a *good* monad? There are three laws that every monad should follow. Break them, and composition breaks. Let's see why they matter.

---

**Next: [Chapter 7 - The Laws: Why They Matter](07-monad-laws.md)**
