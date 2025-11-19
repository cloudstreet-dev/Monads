# Chapter 4: Either - When Failure Has Details

Maybe solved the null problem, but it's frustratingly vague. When you get `None`, you have no idea *why* something failed.

```typescript
const result = divide(10, 0);  // None

// But why None?
// - Division by zero?
// - Invalid input?
// - Database error?
// - Network timeout?
// - Cosmic rays?
```

We need errors with explanations. Enter `Either`.

## The Idea

Either represents a value that can be one of two types:
- `Left` - conventionally holds an error
- `Right` - conventionally holds a success value

```typescript
type Either<L, R> = Left<L> | Right<R>;

class Left<L> {
  constructor(private value: L) {}

  getValue(): L {
    return this.value;
  }
}

class Right<R> {
  constructor(private value: R) {}

  getValue(): R {
    return this.value;
  }
}
```

Why "Left" and "Right"? Mnemonic: "Right" is right (correct), "Left" is what's left when things go wrong.

## Building Either

Let's add the monadic operations:

```typescript
class Left<L> {
  constructor(private value: L) {}

  map<U>(fn: (value: R) => U): Either<L, U> {
    return this;  // Errors pass through unchanged
  }

  flatMap<U>(fn: (value: R) => Either<L, U>): Either<L, U> {
    return this;  // Errors short-circuit
  }

  getValue(): L {
    return this.value;
  }

  isLeft(): boolean {
    return true;
  }

  isRight(): boolean {
    return false;
  }
}

class Right<R> {
  constructor(private value: R) {}

  map<U>(fn: (value: R) => U): Either<L, U> {
    return new Right(fn(this.value));
  }

  flatMap<U>(fn: (value: R) => Either<L, U>): Either<L, U> {
    return fn(this.value);
  }

  getValue(): R {
    return this.value;
  }

  isLeft(): boolean {
    return false;
  }

  isRight(): boolean {
    return true;
  }
}
```

Now we can chain operations that might fail *with details*:

```typescript
type MathError = "DivisionByZero" | "NegativeSquareRoot" | "InvalidInput";

function divide(a: number, b: number): Either<MathError, number> {
  return b === 0
    ? new Left("DivisionByZero")
    : new Right(a / b);
}

function squareRoot(n: number): Either<MathError, number> {
  return n < 0
    ? new Left("NegativeSquareRoot")
    : new Right(Math.sqrt(n));
}

// Chain operations
const result = new Right(16)
  .flatMap(x => divide(x, 2))     // Right(8)
  .flatMap(x => squareRoot(x))    // Right(2.828...)
  .map(x => x.toFixed(2));        // Right("2.83")

// When it fails, we know why
const failed = new Right(16)
  .flatMap(x => divide(x, 0))     // Left("DivisionByZero")
  .flatMap(x => squareRoot(x))    // Skipped
  .map(x => x.toFixed(2));        // Still Left("DivisionByZero")

// Extract the result
if (result.isRight()) {
  console.log(`Success: ${result.getValue()}`);
} else {
  console.log(`Error: ${result.getValue()}`);
}
```

## Railway-Oriented Programming

This pattern is called "railway-oriented programming" (coined by Scott Wlaschin):

```
    ──[op1]──[op2]──[op3]──> Success track (Right)
       ║      ║      ║
       ╚══════╩══════╝
          ║
          ▼
    ────────────────────────> Error track (Left)
```

Operations succeed on the "right" track. Any error switches you to the "left" track, where you stay until the end.

## Real-World Example: User Registration

Let's build a user registration system:

```typescript
type ValidationError =
  | { type: "InvalidEmail"; email: string }
  | { type: "WeakPassword"; reason: string }
  | { type: "UserExists"; email: string }
  | { type: "DatabaseError"; message: string };

interface User {
  email: string;
  passwordHash: string;
}

function validateEmail(email: string): Either<ValidationError, string> {
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  return emailRegex.test(email)
    ? new Right(email)
    : new Left({ type: "InvalidEmail", email });
}

function validatePassword(password: string): Either<ValidationError, string> {
  if (password.length < 8) {
    return new Left({
      type: "WeakPassword",
      reason: "Password must be at least 8 characters"
    });
  }
  if (!/[A-Z]/.test(password)) {
    return new Left({
      type: "WeakPassword",
      reason: "Password must contain uppercase letter"
    });
  }
  return new Right(password);
}

function checkUserExists(email: string): Either<ValidationError, string> {
  const exists = database.findByEmail(email);
  return exists
    ? new Left({ type: "UserExists", email })
    : new Right(email);
}

function hashPassword(password: string): Either<ValidationError, string> {
  try {
    return new Right(bcrypt.hashSync(password, 10));
  } catch (e) {
    return new Left({
      type: "DatabaseError",
      message: "Failed to hash password"
    });
  }
}

function createUser(email: string, hash: string): Either<ValidationError, User> {
  try {
    const user = database.insert({ email, passwordHash: hash });
    return new Right(user);
  } catch (e) {
    return new Left({
      type: "DatabaseError",
      message: "Failed to create user"
    });
  }
}

// The registration flow
function registerUser(
  email: string,
  password: string
): Either<ValidationError, User> {
  return validateEmail(email)
    .flatMap(checkUserExists)
    .flatMap(() => validatePassword(password))
    .flatMap(hashPassword)
    .flatMap(hash => createUser(email, hash));
}

// Usage
const result = registerUser("user@example.com", "weak");

if (result.isLeft()) {
  const error = result.getValue();
  switch (error.type) {
    case "InvalidEmail":
      console.error(`Invalid email: ${error.email}`);
      break;
    case "WeakPassword":
      console.error(`Weak password: ${error.reason}`);
      break;
    case "UserExists":
      console.error(`User already exists: ${error.email}`);
      break;
    case "DatabaseError":
      console.error(`Database error: ${error.message}`);
      break;
  }
} else {
  const user = result.getValue();
  console.log(`User created: ${user.email}`);
}
```

Compare this to nested try-catch blocks or if statements. The Either chain is:
- **Linear**: No nesting
- **Type-safe**: The compiler knows all error types
- **Explicit**: Functions declare they can fail
- **Composable**: Easy to add more validation steps

## Either Across Languages

### Rust: Result<T, E>

Rust's Result is the poster child for Either:

```rust
#[derive(Debug)]
enum MathError {
    DivisionByZero,
    NegativeSquareRoot,
}

fn divide(a: f64, b: f64) -> Result<f64, MathError> {
    if b == 0.0 {
        Err(MathError::DivisionByZero)
    } else {
        Ok(a / b)
    }
}

fn square_root(n: f64) -> Result<f64, MathError> {
    if n < 0.0 {
        Err(MathError::NegativeSquareRoot)
    } else {
        Ok(n.sqrt())
    }
}

// Chain with and_then
let result = Ok(16.0)
    .and_then(|x| divide(x, 2.0))
    .and_then(square_root)
    .map(|x| format!("{:.2}", x));

match result {
    Ok(s) => println!("Result: {}", s),
    Err(MathError::DivisionByZero) => println!("Cannot divide by zero"),
    Err(MathError::NegativeSquareRoot) => println!("Cannot take square root of negative"),
}

// Or use the ? operator (syntactic sugar for early return on error)
fn calculate(x: f64) -> Result<String, MathError> {
    let divided = divide(x, 2.0)?;  // Returns error if Left
    let root = square_root(divided)?;
    Ok(format!("{:.2}", root))
}
```

The `?` operator is brilliant - it unwraps Right values and early-returns Left values. It's like automatic error propagation.

### Scala: Either[L, R]

```scala
sealed trait MathError
case object DivisionByZero extends MathError
case object NegativeSquareRoot extends MathError

def divide(a: Double, b: Double): Either[MathError, Double] =
  if (b == 0) Left(DivisionByZero)
  else Right(a / b)

def squareRoot(n: Double): Either[MathError, Double] =
  if (n < 0) Left(NegativeSquareRoot)
  else Right(math.sqrt(n))

// Chain with flatMap
val result = Right(16.0)
  .flatMap(x => divide(x, 2))
  .flatMap(squareRoot)
  .map(x => f"$x%.2f")

result match {
  case Right(s) => println(s"Result: $s")
  case Left(DivisionByZero) => println("Cannot divide by zero")
  case Left(NegativeSquareRoot) => println("Cannot take square root of negative")
}

// For-comprehension
val result2 = for {
  x <- Right(16.0)
  y <- divide(x, 2)
  z <- squareRoot(y)
} yield f"$z%.2f"
```

### Haskell: Either e a

```haskell
data MathError = DivisionByZero | NegativeSquareRoot
  deriving (Show)

divide :: Double -> Double -> Either MathError Double
divide _ 0 = Left DivisionByZero
divide a b = Right (a / b)

squareRoot :: Double -> Either MathError Double
squareRoot n
  | n < 0     = Left NegativeSquareRoot
  | otherwise = Right (sqrt n)

-- Chain with >>=
result :: Either MathError String
result = Right 16.0
  >>= (\x -> divide x 2)
  >>= squareRoot
  >>= (\x -> Right (show x))

-- Do-notation
result' :: Either MathError String
result' = do
  x <- Right 16.0
  y <- divide x 2
  z <- squareRoot y
  return (show z)

-- Pattern matching
case result of
  Right s -> putStrLn $ "Result: " ++ s
  Left DivisionByZero -> putStrLn "Cannot divide by zero"
  Left NegativeSquareRoot -> putStrLn "Cannot take square root of negative"
```

### F#: Result<'T, 'TError>

```fsharp
type MathError =
    | DivisionByZero
    | NegativeSquareRoot

let divide a b =
    if b = 0.0 then Error DivisionByZero
    else Ok (a / b)

let squareRoot n =
    if n < 0.0 then Error NegativeSquareRoot
    else Ok (sqrt n)

// Chain with Result.bind
let result =
    Ok 16.0
    |> Result.bind (fun x -> divide x 2.0)
    |> Result.bind squareRoot
    |> Result.map (sprintf "%.2f")

match result with
| Ok s -> printfn "Result: %s" s
| Error DivisionByZero -> printfn "Cannot divide by zero"
| Error NegativeSquareRoot -> printfn "Cannot take square root of negative"

// Computation expression
let result' = result {
    let! x = Ok 16.0
    let! y = divide x 2.0
    let! z = squareRoot y
    return sprintf "%.2f" z
}
```

## Combining Multiple Errors

What if you want to collect *all* errors, not just the first one?

```typescript
// Standard Either - stops at first error
validateEmail("bad-email")
  .flatMap(() => validatePassword("weak"))
  .flatMap(() => validateAge(-5))
// Returns only the email error, password and age never checked

// For validation, we often want all errors
```

This is where **Applicative Functors** come in (we'll cover them in Chapter 13). But for now, here's a taste:

```typescript
type Validation<E, A> = Either<E[], A>;

function validateAll<E, A, B, C>(
  va: Validation<E, A>,
  vb: Validation<E, B>,
  fn: (a: A, b: B) => C
): Validation<E, C> {
  if (va.isLeft() && vb.isLeft()) {
    // Combine both error lists
    return new Left([...va.getValue(), ...vb.getValue()]);
  }
  if (va.isLeft()) {
    return va as any;
  }
  if (vb.isLeft()) {
    return vb as any;
  }
  return new Right(fn(va.getValue(), vb.getValue()));
}
```

## Helper Methods

Practical Either implementations include:

```typescript
class Either<L, R> {
  // Convert Either to Maybe (lose error details)
  toMaybe(): Maybe<R> {
    return this.isRight()
      ? new Some(this.getValue())
      : new None();
  }

  // Provide default value on Left
  getOrElse(defaultValue: R): R {
    return this.isRight() ? this.getValue() : defaultValue;
  }

  // Transform the Left value
  mapLeft<L2>(fn: (error: L) => L2): Either<L2, R> {
    return this.isLeft()
      ? new Left(fn(this.getValue()))
      : this;
  }

  // Swap Left and Right
  swap(): Either<R, L> {
    return this.isLeft()
      ? new Right(this.getValue())
      : new Left(this.getValue());
  }

  // Execute side effects without changing the Either
  tap(fn: (value: R) => void): Either<L, R> {
    if (this.isRight()) {
      fn(this.getValue());
    }
    return this;
  }
}
```

Usage:

```typescript
const result = registerUser(email, password)
  .tap(user => console.log(`Created user: ${user.email}`))
  .mapLeft(error => ({
    ...error,
    timestamp: new Date()
  }))
  .getOrElse(null);
```

## Either vs Exceptions

Why use Either instead of exceptions?

```typescript
// With exceptions (invisible failure modes)
function divide(a: number, b: number): number {
  if (b === 0) throw new Error("Division by zero");
  return a / b;
}

// Type signature says "returns number"
// But it can also throw!
// Caller has no warning

// With Either (explicit failure modes)
function divide(a: number, b: number): Either<MathError, number> {
  return b === 0
    ? new Left("DivisionByZero")
    : new Right(a / b);
}

// Type signature says "might fail"
// Caller is forced to handle both cases
```

Either makes failure **explicit and type-safe**:
- **Compile-time checking**: Forget to handle errors? Compiler complains
- **Self-documenting**: Function signature shows it can fail
- **Composable**: Errors propagate through chains automatically
- **Testable**: No try-catch needed in tests

## When to Use Either vs Maybe

- **Use Maybe** when: Absence of value is self-explanatory
  - `findUser(id)` - not found is obvious
  - `parseNumber(string)` - invalid is expected

- **Use Either** when: You need to know *why* it failed
  - `validatePassword(pw)` - which rule did it violate?
  - `connectDatabase()` - network error? auth error? timeout?
  - `processPayment()` - declined? insufficient funds? fraud?

## Pattern Emerging

Notice the similarity between Maybe and Either:

```typescript
// Maybe
Maybe<T> = Some<T> | None
  .map(fn: T => U): Maybe<U>
  .flatMap(fn: T => Maybe<U>): Maybe<U>

// Either
Either<L, R> = Left<L> | Right<R>
  .map(fn: R => U): Either<L, U>
  .flatMap(fn: R => Either<L, U>): Either<L, U>
```

They're the same pattern! Both are:
- Containers with multiple cases
- Support mapping over the success case
- Support chaining operations that return the container type
- Automatically handle the failure case

This is the monad pattern emerging. We'll see it again and again.

## Exercises

1. Implement `sequence` for Either - convert `Either<E, A>[]` to `Either<E, A[]>`, failing with the first error:

```typescript
function sequence<E, A>(eithers: Either<E, A>[]): Either<E, A[]> {
  // Your implementation
}
```

2. Implement `traverse` - map a function over an array, collecting results:

```typescript
function traverse<E, A, B>(
  array: A[],
  fn: (a: A) => Either<E, B>
): Either<E, B[]> {
  // Your implementation
}
```

3. Implement `fold` - handle both cases:

```typescript
class Either<L, R> {
  fold<U>(
    onLeft: (left: L) => U,
    onRight: (right: R) => U
  ): U {
    // Your implementation
  }
}
```

## What We've Learned

- Either represents success (Right) or failure (Left) with details
- Railway-oriented programming keeps error handling linear
- Errors are explicit in the type signature
- The pattern is the same as Maybe, just with error information
- Real languages use Either/Result extensively (Rust, Scala, Haskell, F#)

## Coming Up

We've seen Maybe and Either - two containers that hold zero or one value. But what about containers that hold *many* values? Surely arrays aren't monads... right?

Spoiler: They are. And that's where things get interesting.

---

## Exercise Answers

```typescript
function sequence<E, A>(eithers: Either<E, A>[]): Either<E, A[]> {
  const result: A[] = [];

  for (const either of eithers) {
    if (either.isLeft()) {
      return either as any;  // Return first error
    }
    result.push(either.getValue());
  }

  return new Right(result);
}

function traverse<E, A, B>(
  array: A[],
  fn: (a: A) => Either<E, B>
): Either<E, B[]> {
  return sequence(array.map(fn));
}

class Either<L, R> {
  fold<U>(
    onLeft: (left: L) => U,
    onRight: (right: R) => U
  ): U {
    return this.isLeft()
      ? onLeft(this.getValue())
      : onRight(this.getValue());
  }
}

// Usage
const numbers = [1, 2, 3];
const result = traverse(numbers, n =>
  n > 0 ? new Right(n * 2) : new Left("Negative number")
);
// Right([2, 4, 6])

result.fold(
  error => console.error(error),
  values => console.log(values)
);
```

---

**Next: [Chapter 5 - List: The Monad You Already Know](05-list-monad.md)**
