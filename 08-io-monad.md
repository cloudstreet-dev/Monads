# Chapter 8: IO - Taming Side Effects

Here's a problem: functional programming values pure functions - functions that always return the same output for the same input, with no side effects.

But... how do you write useful programs without side effects? Reading files, printing to console, making HTTP requests, updating databases - these are all side effects. They're also *the entire point* of most programs.

The IO monad solves this with a clever trick: it doesn't perform side effects. It *describes* them.

## The Problem with Side Effects

Consider this function:

```typescript
function getUserName(): string {
  return prompt("What's your name?");  // Side effect!
}

// What's the type?
// string... but that's a lie
// It actually performs I/O, which isn't captured in the type
```

Problems:
1. **Not pure**: Calling it twice with no arguments gives different results
2. **Not testable**: Can't test without actual user input
3. **Not composable**: Can't reason about it without running it
4. **Hidden effects**: The type signature doesn't show it does I/O

Haskell's solution: make effects explicit in types.

```haskell
getUserName :: IO String
-- "This is a computation that, when run, will perform I/O and produce a String"
```

The key insight: **separate description from execution**.

## IO as a Description

Think of IO as a recipe, not a baked cake:

```typescript
// Not this (immediate execution):
const name = prompt("What's your name?");

// But this (description of execution):
const getNameAction: IO<string> = new IO(() => prompt("What's your name?"));

// Nothing happens yet! It's just a description.
// To execute:
const name = getNameAction.unsafeRun();
```

## Building the IO Monad

```typescript
class IO<T> {
  constructor(private effect: () => T) {}

  // Wrap a pure value
  static of<T>(value: T): IO<T> {
    return new IO(() => value);
  }

  // Transform the result
  map<U>(fn: (value: T) => U): IO<U> {
    return new IO(() => fn(this.effect()));
  }

  // Chain IO operations
  flatMap<U>(fn: (value: T) => IO<U>): IO<U> {
    return new IO(() => {
      const value = this.effect();
      const nextIO = fn(value);
      return nextIO.effect();
    });
  }

  // Execute the effects (at the end of the world)
  unsafeRun(): T {
    return this.effect();
  }
}
```

Notice: `map` and `flatMap` don't execute anything! They build up a description of effects to run later.

## Building Composable I/O

```typescript
// Primitive I/O operations
function readLine(): IO<string> {
  return new IO(() => {
    return prompt("Enter input:");
  });
}

function writeLine(message: string): IO<void> {
  return new IO(() => {
    console.log(message);
  });
}

function readFile(path: string): IO<string> {
  return new IO(() => {
    return fs.readFileSync(path, 'utf-8');
  });
}

function writeFile(path: string, content: string): IO<void> {
  return new IO(() => {
    fs.writeFileSync(path, content);
  });
}

// Compose them!
function greetUser(): IO<void> {
  return writeLine("What's your name?")
    .flatMap(() => readLine())
    .flatMap(name => writeLine(`Hello, ${name}!`));
}

// Still nothing has happened!
// Only when we run it:
greetUser().unsafeRun();
```

## Real Example: File Processing

```typescript
// Process a file: read, transform, write
function processFile(
  inputPath: string,
  outputPath: string,
  transform: (content: string) => string
): IO<void> {
  return readFile(inputPath)
    .map(transform)
    .flatMap(transformed => writeFile(outputPath, transformed));
}

// Specific transformations
const uppercase = (s: string) => s.toUpperCase();
const addLineNumbers = (s: string) =>
  s.split('\n')
   .map((line, i) => `${i + 1}: ${line}`)
   .join('\n');

// Compose operations
const program = processFile('input.txt', 'output.txt', uppercase)
  .flatMap(() => processFile('output.txt', 'final.txt', addLineNumbers))
  .flatMap(() => writeLine("Processing complete!"));

// Build the entire program as a value
// Execute once at the end
program.unsafeRun();
```

## Why This Matters

### 1. Effects Are Explicit

```typescript
// Before (hidden side effects)
function processUser(id: string): User {
  const user = database.findUser(id);  // Hidden I/O!
  sendEmail(user.email);               // Hidden I/O!
  return user;
}

// After (explicit side effects)
function processUser(id: string): IO<User> {
  return database.findUser(id)           // Returns IO<User>
    .flatMap(user =>
      sendEmail(user.email)              // Returns IO<void>
        .map(() => user)
    );
}
```

The type signature now tells you: "This function performs I/O."

### 2. Pure Core, Impure Shell

```typescript
// Pure transformation (testable!)
function formatUser(user: User): string {
  return `${user.name} (${user.email})`;
}

function validateUser(user: User): Either<string, User> {
  return user.email.includes('@')
    ? Either.of(user)
    : Either.left('Invalid email');
}

// Impure shell (only at boundaries)
function saveUser(user: User): IO<void> {
  return new IO(() => database.save(user));
}

// Compose: pure logic with impure I/O
function processAndSave(userId: string): IO<Either<string, void>> {
  return findUser(userId)
    .map(validateUser)          // Pure validation
    .flatMap(result =>
      result.fold(
        error => IO.of(Either.left(error)),
        user => saveUser(user).map(() => Either.right(undefined))
      )
    );
}
```

Most of your code is pure! Only the edges touch I/O.

### 3. Referential Transparency

```typescript
// These are the same value
const program1 = readLine().flatMap(x => writeLine(x));
const program2 = readLine().flatMap(x => writeLine(x));

// We can refactor freely
const readAndWrite = readLine().flatMap(x => writeLine(x));
const program3 = readAndWrite;
const program4 = readAndWrite;

// Each execution is independent
program3.unsafeRun();  // Performs I/O
program4.unsafeRun();  // Performs I/O again
```

IO values are just data. You can pass them around, store them, return them - they're not special.

## IO in Haskell

Haskell forces everything to be pure, so IO is mandatory for effects:

```haskell
-- IO actions are descriptions
getLine :: IO String
putStrLn :: String -> IO ()

-- Compose with >>= (bind/flatMap)
greetUser :: IO ()
greetUser = do
  putStrLn "What's your name?"
  name <- getLine
  putStrLn ("Hello, " ++ name ++ "!")

-- Or without do-notation:
greetUser' :: IO ()
greetUser' =
  putStrLn "What's your name?" >>=
  (\_ -> getLine) >>=
  (\name -> putStrLn ("Hello, " ++ name ++ "!"))

-- main is the one IO action that runs
main :: IO ()
main = greetUser
```

Key insight: **You can't escape IO in Haskell**. Once you're in IO, you stay in IO. This is by design - effects must stay at the boundaries.

```haskell
-- Can't do this:
badFunction :: String -> String
badFunction s = runIO (putStrLn s >> getLine)  -- No such function exists!

-- Must do this:
goodFunction :: String -> IO String
goodFunction s = putStrLn s >> getLine
```

## Lazy Evaluation

IO values are lazy - they don't execute until forced:

```typescript
class IO<T> {
  // ... previous methods ...

  // Delay execution
  static lazy<T>(thunk: () => IO<T>): IO<T> {
    return new IO(() => thunk().unsafeRun());
  }

  // Repeat an action
  forever(): IO<never> {
    return this.flatMap(() => this.forever());
  }

  // Retry on failure
  retry(times: number): IO<T> {
    return new IO(() => {
      for (let i = 0; i < times; i++) {
        try {
          return this.effect();
        } catch (e) {
          if (i === times - 1) throw e;
        }
      }
      throw new Error("Unreachable");
    });
  }
}

// Usage
const resilientFetch = fetchData()
  .retry(3)
  .flatMap(data => saveData(data));

// Doesn't execute until:
resilientFetch.unsafeRun();
```

## Combining IO with Other Monads

IO often needs to work with Maybe, Either, etc:

```typescript
// Find user, which might not exist (Maybe)
// But also performs I/O
function findUser(id: string): IO<Maybe<User>> {
  return new IO(() => {
    const user = database.findUser(id);
    return user ? Maybe.of(user) : Maybe.none();
  });
}

// Process only if found
function processUser(id: string): IO<Maybe<string>> {
  return findUser(id)
    .map(maybeUser =>
      maybeUser.map(user => user.name.toUpperCase())
    );
}

// This is clunky - we'll fix it with Monad Transformers in Chapter 14
```

## IO in Other Languages

### Scala (using cats-effect)

```scala
import cats.effect.IO

// Define IO actions
def readLine(): IO[String] =
  IO(scala.io.StdIn.readLine())

def writeLine(s: String): IO[Unit] =
  IO(println(s))

// Compose
def greetUser(): IO[Unit] = for {
  _ <- writeLine("What's your name?")
  name <- readLine()
  _ <- writeLine(s"Hello, $name!")
} yield ()

// Execute (usually at the end of main)
greetUser().unsafeRunSync()
```

### Haskell (built-in)

```haskell
-- File processing example
processFile :: FilePath -> FilePath -> (String -> String) -> IO ()
processFile input output transform = do
  content <- readFile input
  let transformed = transform content
  writeFile output transformed

-- HTTP request example
fetchUser :: Int -> IO (Maybe User)
fetchUser userId = do
  response <- httpGet ("https://api.example.com/users/" ++ show userId)
  return (parseUser response)

-- Database query
findUser :: Int -> IO (Maybe User)
findUser userId = do
  result <- query "SELECT * FROM users WHERE id = ?" [userId]
  return (listToMaybe result)
```

### F# (using FSharpPlus or custom)

```fsharp
type IO<'T> = IO of (unit -> 'T)

module IO =
    let run (IO effect) = effect()

    let pure x = IO (fun () -> x)

    let bind f (IO effect) =
        IO (fun () ->
            let result = effect()
            let (IO nextEffect) = f result
            nextEffect()
        )

    let map f io = bind (f >> pure) io

// Usage
let readLine() = IO (fun () -> System.Console.ReadLine())
let writeLine s = IO (fun () -> printfn "%s" s)

let greetUser() = io {
    do! writeLine "What's your name?"
    let! name = readLine()
    do! writeLine (sprintf "Hello, %s!" name)
}

// Execute
IO.run (greetUser())
```

## Practical Example: CLI Application

```typescript
// Command-line calculator
type Command = 'add' | 'multiply' | 'quit';

function parseCommand(input: string): Maybe<Command> {
  switch (input.toLowerCase()) {
    case 'add': return Maybe.of('add');
    case 'multiply': return Maybe.of('multiply');
    case 'quit': return Maybe.of('quit');
    default: return Maybe.none();
  }
}

function readNumber(): IO<Maybe<number>> {
  return readLine().map(input => {
    const num = parseFloat(input);
    return isNaN(num) ? Maybe.none() : Maybe.of(num);
  });
}

function calculate(cmd: Command, a: number, b: number): number {
  switch (cmd) {
    case 'add': return a + b;
    case 'multiply': return a * b;
    default: return 0;
  }
}

function calculatorLoop(): IO<void> {
  return writeLine("Enter command (add/multiply/quit):")
    .flatMap(() => readLine())
    .flatMap(input =>
      parseCommand(input).fold(
        () => writeLine("Invalid command").flatMap(() => calculatorLoop()),
        cmd => {
          if (cmd === 'quit') {
            return writeLine("Goodbye!");
          }

          return writeLine("Enter first number:")
            .flatMap(() => readNumber())
            .flatMap(maybeA =>
              maybeA.fold(
                () => writeLine("Invalid number").flatMap(() => calculatorLoop()),
                a => writeLine("Enter second number:")
                  .flatMap(() => readNumber())
                  .flatMap(maybeB =>
                    maybeB.fold(
                      () => writeLine("Invalid number").flatMap(() => calculatorLoop()),
                      b => {
                        const result = calculate(cmd, a, b);
                        return writeLine(`Result: ${result}`)
                          .flatMap(() => calculatorLoop());
                      }
                    )
                  )
              )
            );
        }
      )
    );
}

// The entire program is a pure value
const program: IO<void> = writeLine("Calculator v1.0")
  .flatMap(() => calculatorLoop());

// Execute once
program.unsafeRun();
```

## Testing with IO

The beauty of IO: you can test pure logic separately from effects.

```typescript
// Pure logic (easy to test)
function validateEmail(email: string): Either<string, string> {
  return email.includes('@')
    ? Either.right(email)
    : Either.left('Invalid email');
}

function formatWelcomeMessage(user: User): string {
  return `Welcome, ${user.name}! Your email is ${user.email}`;
}

// Impure effects (test with mocks or integration tests)
function sendEmail(to: string, message: string): IO<void> {
  return new IO(() => emailService.send(to, message));
}

// Composed (test the pure parts in unit tests)
function welcomeUser(user: User): IO<Either<string, void>> {
  return IO.of(validateEmail(user.email))
    .flatMap(result =>
      result.fold(
        error => IO.of(Either.left(error)),
        email => {
          const message = formatWelcomeMessage(user);
          return sendEmail(email, message)
            .map(() => Either.right(undefined));
        }
      )
    );
}

// Unit tests
describe('validateEmail', () => {
  it('accepts valid emails', () => {
    expect(validateEmail('user@example.com').isRight()).toBe(true);
  });

  it('rejects invalid emails', () => {
    expect(validateEmail('not-an-email').isLeft()).toBe(true);
  });
});

// Integration tests (actually run IO)
describe('welcomeUser integration', () => {
  it('sends welcome email', async () => {
    const user = { name: 'Alice', email: 'alice@example.com' };
    const result = welcomeUser(user).unsafeRun();
    expect(result.isRight()).toBe(true);
    expect(mockEmailService.sentEmails).toHaveLength(1);
  });
});
```

## The Philosophical Point

IO doesn't eliminate side effects - that's impossible. But it:

1. **Makes them explicit**: Functions that do I/O say so in their type
2. **Delays execution**: Build programs as values, run at the end
3. **Enables composition**: Chain effectful operations like pure ones
4. **Preserves purity**: The description of effects is pure

Your program becomes:
```
Pure Core (business logic) → IO Shell (effects at boundaries) → Runtime
```

## What We've Learned

- IO represents a description of effects, not their execution
- Separate building programs from running them
- Make side effects explicit in types
- Push effects to the boundaries
- Test pure logic separately from effects
- IO is a monad: composable effectful computations

## Coming Up

IO handles external effects, but what about internal state? Functions that need to maintain and update state without globals? That's where the State monad shines.

---

**Next: [Chapter 9 - State: Hidden Plumbing](09-state-monad.md)**
