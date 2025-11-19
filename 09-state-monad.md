# Chapter 9: State - Hidden Plumbing

Functions with state are everywhere:

```typescript
let counter = 0;

function increment(): number {
  counter += 1;
  return counter;
}

increment(); // 1
increment(); // 2
increment(); // 3
```

But global state is evil:
- Hard to test (have to reset state between tests)
- Hard to reason about (any function could modify it)
- Not thread-safe (race conditions)
- Not composable (order of operations matters globally)

The State monad threads state through computations without globals.

## The Core Idea

Instead of modifying global state:

```typescript
let state = initialState;
state = step1(state);
state = step2(state);
state = step3(state);
return state;
```

We make state explicit:

```typescript
function step1(state: S): [A, S] { /* ... */ }
function step2(state: S): [B, S] { /* ... */ }
function step3(state: S): [C, S] { /* ... */ }

// Each function takes state and returns (value, newState)
```

The State monad automates threading state through a chain.

## Building State

```typescript
// State<S, A> represents:
// A function from state S to a pair (value A, new state S)
class State<S, A> {
  constructor(private runState: (state: S) => [A, S]) {}

  // Wrap a pure value (doesn't change state)
  static of<S, A>(value: A): State<S, A> {
    return new State(state => [value, state]);
  }

  // Get the current state
  static get<S>(): State<S, S> {
    return new State(state => [state, state]);
  }

  // Set the state
  static put<S>(newState: S): State<S, void> {
    return new State(() => [undefined, newState]);
  }

  // Modify the state
  static modify<S>(fn: (state: S) => S): State<S, void> {
    return new State(state => [undefined, fn(state)]);
  }

  // Transform the value
  map<B>(fn: (value: A) => B): State<S, B> {
    return new State(state => {
      const [value, newState] = this.runState(state);
      return [fn(value), newState];
    });
  }

  // Chain state computations
  flatMap<B>(fn: (value: A) => State<S, B>): State<S, B> {
    return new State(state => {
      const [value, newState] = this.runState(state);
      const nextState = fn(value);
      return nextState.runState(newState);
    });
  }

  // Run the computation with initial state
  run(initialState: S): [A, S] {
    return this.runState(initialState);
  }

  // Run and return only the value
  eval(initialState: S): A {
    return this.runState(initialState)[0];
  }

  // Run and return only the final state
  exec(initialState: S): S {
    return this.runState(initialState)[1];
  }
}
```

## Simple Example: Counter

```typescript
type CounterState = { count: number };

// Increment and return the new value
function increment(): State<CounterState, number> {
  return State.get<CounterState>().flatMap(state =>
    State.put({ count: state.count + 1 }).flatMap(() =>
      State.of(state.count + 1)
    )
  );
}

// Or more concisely:
function increment2(): State<CounterState, number> {
  return State.modify<CounterState>(state => ({
    count: state.count + 1
  })).flatMap(() =>
    State.get<CounterState>().map(state => state.count)
  );
}

// Decrement
function decrement(): State<CounterState, number> {
  return State.modify<CounterState>(state => ({
    count: state.count - 1
  })).flatMap(() =>
    State.get<CounterState>().map(state => state.count)
  );
}

// Compose operations
const program = increment()
  .flatMap(() => increment())
  .flatMap(() => decrement())
  .flatMap(finalCount => State.of(finalCount));

// Run with initial state
const [result, finalState] = program.run({ count: 0 });
console.log(result);      // 1
console.log(finalState);  // { count: 1 }
```

No global variables! State is threaded through implicitly.

## Real Example: Random Number Generator

Generating random numbers is stateful - you need to track the seed:

```typescript
type RNG = { seed: number };

// Generate a random number and update seed
function nextInt(): State<RNG, number> {
  return State.get<RNG>().flatMap(rng => {
    // Linear congruential generator
    const newSeed = (rng.seed * 1103515245 + 12345) & 0x7fffffff;
    const n = Math.floor(newSeed / 65536) % 32768;
    return State.put({ seed: newSeed }).map(() => n);
  });
}

// Generate a random number in range [0, max)
function nextIntBounded(max: number): State<RNG, number> {
  return nextInt().map(n => n % max);
}

// Generate a random boolean
function nextBoolean(): State<RNG, boolean> {
  return nextInt().map(n => n % 2 === 0);
}

// Generate a random element from array
function choose<T>(array: T[]): State<RNG, T> {
  return nextIntBounded(array.length).map(i => array[i]);
}

// Generate multiple random numbers
function randoms(count: number): State<RNG, number[]> {
  if (count === 0) return State.of([]);

  return nextInt().flatMap(n =>
    randoms(count - 1).map(rest => [n, ...rest])
  );
}

// Compose to generate test data
const program = nextInt()
  .flatMap(id =>
    nextBoolean().flatMap(isActive =>
      choose(['red', 'green', 'blue']).map(color => ({
        id,
        isActive,
        color
      }))
    )
  );

// Run with initial seed
const [randomUser, finalRNG] = program.run({ seed: 42 });
console.log(randomUser);
// { id: 16807, isActive: false, color: 'blue' }

// Same seed = same result (referentially transparent!)
const [sameUser, _] = program.run({ seed: 42 });
console.log(sameUser);
// { id: 16807, isActive: false, color: 'blue' }
```

This is huge: **random number generation becomes pure**! Same seed gives same results, making it testable and reproducible.

## Stack Machine Example

Building an interpreter for a simple stack-based language:

```typescript
type Stack = number[];

// Push a value onto the stack
function push(value: number): State<Stack, void> {
  return State.modify<Stack>(stack => [value, ...stack]);
}

// Pop a value from the stack
function pop(): State<Stack, number> {
  return State.get<Stack>().flatMap(stack => {
    if (stack.length === 0) {
      throw new Error("Stack underflow");
    }
    const [top, ...rest] = stack;
    return State.put(rest).map(() => top);
  });
}

// Peek at the top value
function peek(): State<Stack, number> {
  return State.get<Stack>().map(stack => {
    if (stack.length === 0) {
      throw new Error("Stack underflow");
    }
    return stack[0];
  });
}

// Binary operations
function binOp(op: (a: number, b: number) => number): State<Stack, void> {
  return pop().flatMap(b =>
    pop().flatMap(a =>
      push(op(a, b))
    )
  );
}

const add = binOp((a, b) => a + b);
const multiply = binOp((a, b) => a * b);
const subtract = binOp((a, b) => a - b);

// Evaluate a program
// Example: (3 + 5) * 2 = 16
const program = push(3)
  .flatMap(() => push(5))
  .flatMap(() => add)
  .flatMap(() => push(2))
  .flatMap(() => multiply)
  .flatMap(() => pop());

const [result, finalStack] = program.run([]);
console.log(result);      // 16
console.log(finalStack);  // []
```

## Practical Example: Parser State

Building a parser that tracks position in input:

```typescript
type ParserState = {
  input: string;
  position: number;
};

// Read a single character
function char(): State<ParserState, string | null> {
  return State.get<ParserState>().flatMap(state => {
    if (state.position >= state.input.length) {
      return State.of(null);
    }

    const c = state.input[state.position];
    return State.put({
      ...state,
      position: state.position + 1
    }).map(() => c);
  });
}

// Peek at next character without consuming
function peek(): State<ParserState, string | null> {
  return State.get<ParserState>().map(state =>
    state.position < state.input.length
      ? state.input[state.position]
      : null
  );
}

// Match a specific character
function matchChar(expected: string): State<ParserState, boolean> {
  return peek().flatMap(c => {
    if (c === expected) {
      return char().map(() => true);
    }
    return State.of(false);
  });
}

// Parse a number
function parseNumber(): State<ParserState, number | null> {
  return State.get<ParserState>().flatMap(initialState => {
    let digits = '';

    function parseDigits(): State<ParserState, void> {
      return peek().flatMap(c => {
        if (c && /\d/.test(c)) {
          return char().flatMap(digit => {
            digits += digit;
            return parseDigits();
          });
        }
        return State.of(undefined);
      });
    }

    return parseDigits().map(() =>
      digits.length > 0 ? parseInt(digits, 10) : null
    );
  });
}

// Parse whitespace
function skipWhitespace(): State<ParserState, void> {
  return peek().flatMap(c => {
    if (c && /\s/.test(c)) {
      return char().flatMap(() => skipWhitespace());
    }
    return State.of(undefined);
  });
}

// Parse a simple expression: number operator number
function parseExpression(): State<ParserState, number | null> {
  return parseNumber().flatMap(left => {
    if (left === null) return State.of(null);

    return skipWhitespace().flatMap(() =>
      char().flatMap(op => {
        if (!op || !['+', '-', '*'].includes(op)) {
          return State.of(null);
        }

        return skipWhitespace().flatMap(() =>
          parseNumber().map(right => {
            if (right === null) return null;

            switch (op) {
              case '+': return left + right;
              case '-': return left - right;
              case '*': return left * right;
              default: return null;
            }
          })
        );
      })
    );
  });
}

// Use the parser
const [result, finalState] = parseExpression().run({
  input: "42 + 17",
  position: 0
});

console.log(result);              // 59
console.log(finalState.position); // 7
```

## State in Other Languages

### Haskell

```haskell
-- State is built-in
import Control.Monad.State

-- Counter example
type Counter = Int

increment :: State Counter Int
increment = do
  count <- get
  put (count + 1)
  return (count + 1)

decrement :: State Counter Int
decrement = do
  count <- get
  put (count - 1)
  return (count - 1)

-- Run the computation
program :: State Counter Int
program = do
  increment
  increment
  decrement

-- Execute
result = evalState program 0  -- Returns 1
finalState = execState program 0  -- Returns 1
both = runState program 0  -- Returns (1, 1)
```

### Scala

```scala
case class State[S, A](run: S => (A, S)) {
  def map[B](f: A => B): State[S, B] =
    State(s => {
      val (a, s1) = run(s)
      (f(a), s1)
    })

  def flatMap[B](f: A => State[S, B]): State[S, B] =
    State(s => {
      val (a, s1) = run(s)
      f(a).run(s1)
    })
}

object State {
  def pure[S, A](a: A): State[S, A] =
    State(s => (a, s))

  def get[S]: State[S, S] =
    State(s => (s, s))

  def put[S](s: S): State[S, Unit] =
    State(_ => ((), s))

  def modify[S](f: S => S): State[S, Unit] =
    State(s => ((), f(s)))
}

// Usage
type Counter = Int

def increment: State[Counter, Int] = for {
  count <- State.get
  _ <- State.put(count + 1)
} yield count + 1

val program = for {
  _ <- increment
  _ <- increment
  result <- increment
} yield result

val (finalValue, finalState) = program.run(0)
```

### F#

```fsharp
type State<'S, 'A> = State of ('S -> 'A * 'S)

module State =
    let run (State f) s = f s

    let pure x = State (fun s -> (x, s))

    let get<'S> = State (fun s -> (s, s))

    let put s = State (fun _ -> ((), s))

    let modify f = State (fun s -> ((), f s))

    let bind f (State run1) =
        State (fun s ->
            let (a, s') = run1 s
            let (State run2) = f a
            run2 s'
        )

    let map f m = bind (f >> pure) m

// Usage
type Counter = int

let increment = state {
    let! count = State.get
    do! State.put (count + 1)
    return count + 1
}

let program = state {
    do! increment
    do! increment
    let! result = increment
    return result
}

let (finalValue, finalState) = State.run program 0
```

## Why State Matters

### 1. No Globals

```typescript
// Before (global state)
let idCounter = 0;

function generateId(): number {
  return ++idCounter;
}

function createUser(name: string): User {
  return { id: generateId(), name };
}

// Can't test without resetting globals
// Can't run in parallel
// Can't reason about state changes

// After (State monad)
type AppState = { nextId: number };

function generateId(): State<AppState, number> {
  return State.get<AppState>().flatMap(state =>
    State.put({ nextId: state.nextId + 1 })
      .map(() => state.nextId)
  );
}

function createUser(name: string): State<AppState, User> {
  return generateId().map(id => ({ id, name }));
}

// Testable, composable, explicit
```

### 2. Composability

```typescript
// Compose stateful computations easily
function createThreeUsers(): State<AppState, User[]> {
  return createUser("Alice").flatMap(u1 =>
    createUser("Bob").flatMap(u2 =>
      createUser("Charlie").map(u3 => [u1, u2, u3])
    )
  );
}

// Or with a helper
function sequence<S, A>(states: State<S, A>[]): State<S, A[]> {
  return states.reduce(
    (acc, state) =>
      acc.flatMap(list =>
        state.map(value => [...list, value])
      ),
    State.of<S, A[]>([])
  );
}

const users = ["Alice", "Bob", "Charlie"];
const program = sequence(users.map(createUser));

const [userList, finalState] = program.run({ nextId: 1 });
// userList: [{id:1, name:"Alice"}, {id:2, name:"Bob"}, {id:3, name:"Charlie"}]
// finalState: { nextId: 4 }
```

### 3. Time Travel

Because state is explicit, you can:

```typescript
// Save snapshots
const checkpoint = currentState;

// Run some operations
const [result, newState] = operation.run(checkpoint);

// Rollback if needed
const [retryResult, retryState] = operation.run(checkpoint);
```

## When to Use State

**Use State when:**
- You need to thread state through multiple operations
- You want to avoid global variables
- You need reproducible randomness
- You're building parsers, interpreters, or simulations
- You want explicit control over state transitions

**Don't use State when:**
- A simple parameter is clearer
- You actually need mutable state (e.g., performance-critical code)
- The state is truly global (e.g., database connection pool)

## Advanced: Stateful Random

Generate complex random data:

```typescript
// Random user generator
interface User {
  id: number;
  name: string;
  age: number;
  favoriteColor: string;
}

function randomUser(): State<RNG, User> {
  return nextInt().flatMap(id =>
    choose(['Alice', 'Bob', 'Charlie', 'Diana']).flatMap(name =>
      nextIntBounded(60).flatMap(age =>
        choose(['red', 'blue', 'green', 'yellow']).map(favoriteColor => ({
          id,
          name,
          age: age + 18,  // Age 18-77
          favoriteColor
        }))
      )
    )
  );
}

// Generate a list of random users
function randomUsers(count: number): State<RNG, User[]> {
  return sequence(
    Array.from({ length: count }, () => randomUser())
  );
}

// Generate test data
const [testUsers, _] = randomUsers(10).run({ seed: 12345 });
console.log(testUsers);
```

## What We've Learned

- State monad threads state through computations without globals
- Makes stateful code pure and testable
- Perfect for RNGs, parsers, interpreters, simulations
- Composes cleanly with other stateful operations
- Explicit state > hidden global state

## Coming Up

State is about internal state. But what about external configuration? Global settings that every function needs? That's the Reader monad - dependency injection, functionally.

---

**Next: [Chapter 10 - Reader: Dependency Injection, Functionally](10-reader-monad.md)**
