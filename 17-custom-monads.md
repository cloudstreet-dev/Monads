# Chapter 17: Building Your Own Monads

Sometimes the standard monads aren't enough. You need something domain-specific. Let's build custom monads from scratch.

## When to Build Your Own

**✅ Build a custom monad when:**
- You have a specific computational pattern that repeats
- Standard monads don't capture your domain well
- You want type-safe encapsulation of complexity
- The abstraction clarifies code significantly

**❌ Don't build a custom monad when:**
- A standard monad works fine
- You're abstracting for abstraction's sake
- It makes code harder to understand
- A simple function would suffice

## The Recipe

Every monad needs:

1. **A type constructor** - `M<T>`
2. **`of` (return)** - Wrap a pure value
3. **`map`** - Transform the value inside
4. **`flatMap` (bind)** - Chain operations
5. **Follow the laws** - Left identity, right identity, associativity

Let's build some.

## Example 1: Trace Monad

Track execution flow for debugging:

```typescript
// Trace<T> carries a value and its execution trace
class Trace<T> {
  constructor(
    private value: T,
    private trace: string[]
  ) {}

  // Wrap a value with empty trace
  static of<T>(value: T): Trace<T> {
    return new Trace(value, []);
  }

  // Add a trace message
  static traced<T>(value: T, message: string): Trace<T> {
    return new Trace(value, [message]);
  }

  // Transform the value
  map<U>(fn: (value: T) => U): Trace<U> {
    return new Trace(fn(this.value), this.trace);
  }

  // Chain operations (combines traces)
  flatMap<U>(fn: (value: T) => Trace<U>): Trace<U> {
    const next = fn(this.value);
    return new Trace(
      next.value,
      [...this.trace, ...next.trace]
    );
  }

  // Extract value and trace
  run(): [T, string[]] {
    return [this.value, this.trace];
  }

  getValue(): T {
    return this.value;
  }

  getTrace(): string[] {
    return this.trace;
  }
}

// Helper to create traced operations
function traced<T>(message: string, value: T): Trace<T> {
  return Trace.traced(value, message);
}

// Example usage
function add(a: number, b: number): Trace<number> {
  return traced(`Adding ${a} + ${b}`, a + b);
}

function multiply(a: number, b: number): Trace<number> {
  return traced(`Multiplying ${a} * ${b}`, a * b);
}

function divide(a: number, b: number): Trace<number> {
  return traced(`Dividing ${a} / ${b}`, a / b);
}

// Compose
const computation = Trace.of(10)
  .flatMap(x => add(x, 5))
  .flatMap(x => multiply(x, 2))
  .flatMap(x => divide(x, 3));

const [result, executionTrace] = computation.run();
console.log('Result:', result);  // 10
console.log('Trace:');
executionTrace.forEach(step => console.log(`  - ${step}`));
// - Adding 10 + 5
// - Multiplying 15 * 2
// - Dividing 30 / 3
```

### Verify the Laws

```typescript
// Left identity: Trace.of(x).flatMap(f) === f(x)
const x = 5;
const f = (n: number) => add(n, 10);

const left = Trace.of(x).flatMap(f);
const right = f(x);
// Both give Trace(15, ["Adding 5 + 10"]) ✓

// Right identity: m.flatMap(Trace.of) === m
const m = traced('Test', 42);
const leftId = m.flatMap(Trace.of);
// Both give Trace(42, ["Test"]) ✓

// Associativity: m.flatMap(f).flatMap(g) === m.flatMap(x => f(x).flatMap(g))
const g = (n: number) => multiply(n, 2);
const assocLeft = m.flatMap(f).flatMap(g);
const assocRight = m.flatMap(x => f(x).flatMap(g));
// Both give same result and trace ✓
```

## Example 2: Remote Data Monad

Represent data that's being fetched from a server:

```typescript
// RemoteData<E, T> represents data in various loading states
type RemoteData<E, T> =
  | { type: 'NotAsked' }
  | { type: 'Loading' }
  | { type: 'Failure'; error: E }
  | { type: 'Success'; data: T };

class Remote<E, T> {
  private constructor(private state: RemoteData<E, T>) {}

  // Create states
  static notAsked<E, T>(): Remote<E, T> {
    return new Remote({ type: 'NotAsked' });
  }

  static loading<E, T>(): Remote<E, T> {
    return new Remote({ type: 'Loading' });
  }

  static failure<E, T>(error: E): Remote<E, T> {
    return new Remote({ type: 'Failure', error });
  }

  static success<E, T>(data: T): Remote<E, T> {
    return new Remote({ type: 'Success', data });
  }

  // Monad operations
  static of<E, T>(data: T): Remote<E, T> {
    return Remote.success(data);
  }

  map<U>(fn: (data: T) => U): Remote<E, U> {
    if (this.state.type === 'Success') {
      return Remote.success(fn(this.state.data));
    }
    // TypeScript limitation: we need to cast here because the type system
    // doesn't track that non-Success states are valid for Remote<E, U>
    return new Remote(this.state as any);
  }

  flatMap<U>(fn: (data: T) => Remote<E, U>): Remote<E, U> {
    if (this.state.type === 'Success') {
      return fn(this.state.data);
    }
    // TypeScript limitation: we need to cast here because the type system
    // doesn't track that non-Success states are valid for Remote<E, U>
    return new Remote(this.state as any);
  }

  // Pattern matching
  fold<R>(handlers: {
    notAsked: () => R;
    loading: () => R;
    failure: (error: E) => R;
    success: (data: T) => R;
  }): R {
    switch (this.state.type) {
      case 'NotAsked': return handlers.notAsked();
      case 'Loading': return handlers.loading();
      case 'Failure': return handlers.failure(this.state.error);
      case 'Success': return handlers.success(this.state.data);
    }
  }

  // Helpers
  isSuccess(): boolean {
    return this.state.type === 'Success';
  }

  isLoading(): boolean {
    return this.state.type === 'Loading';
  }

  isFailure(): boolean {
    return this.state.type === 'Failure';
  }

  // Load from a promise (returns a function that updates state)
  static async fromPromise<E, T>(
    promise: Promise<T>,
    onError: (error: any) => E
  ): Promise<Remote<E, T>> {
    try {
      const data = await promise;
      return Remote.success(data);
    } catch (error) {
      return Remote.failure(onError(error));
    }
  }
}

// Usage in React
function UserProfile({ userId }: { userId: string }) {
  const [userData, setUserData] = useState(Remote.notAsked<Error, User>());

  useEffect(() => {
    setUserData(Remote.loading());

    fetchUser(userId)
      .then(user => setUserData(Remote.success(user)))
      .catch(error => setUserData(Remote.failure(error)));
  }, [userId]);

  return userData.fold({
    notAsked: () => <div>Click to load</div>,
    loading: () => <Spinner />,
    failure: (error) => <ErrorMessage error={error} />,
    success: (user) => <UserDetails user={user} />
  });
}

// Compose remote data operations
async function loadUserPosts(userId: string): Promise<Remote<Error, Post[]>> {
  return Remote.fromPromise(
    fetchUser(userId).then(user =>
      fetchPosts(user.id)
    ),
    error => new Error(error.message)
  );
}
```

## Example 3: Retry Monad

Automatically retry operations with backoff:

```typescript
interface RetryConfig {
  maxAttempts: number;
  delayMs: number;
  backoffMultiplier: number;
}

class Retry<T> {
  constructor(
    private operation: () => Promise<T>,
    private config: RetryConfig
  ) {}

  static of<T>(operation: () => Promise<T>): Retry<T> {
    return new Retry(operation, {
      maxAttempts: 1,
      delayMs: 0,
      backoffMultiplier: 1
    });
  }

  map<U>(fn: (value: T) => U): Retry<U> {
    return new Retry(
      () => this.operation().then(fn),
      this.config
    );
  }

  flatMap<U>(fn: (value: T) => Retry<U>): Retry<U> {
    return new Retry(
      async () => {
        const value = await this.run();
        return fn(value).run();
      },
      this.config
    );
  }

  // Configure retry behavior
  withRetries(maxAttempts: number): Retry<T> {
    return new Retry(this.operation, {
      ...this.config,
      maxAttempts
    });
  }

  withDelay(delayMs: number): Retry<T> {
    return new Retry(this.operation, {
      ...this.config,
      delayMs
    });
  }

  withBackoff(multiplier: number): Retry<T> {
    return new Retry(this.operation, {
      ...this.config,
      backoffMultiplier: multiplier
    });
  }

  // Execute with retries
  async run(): Promise<T> {
    let lastError: any;
    let delay = this.config.delayMs;

    for (let attempt = 1; attempt <= this.config.maxAttempts; attempt++) {
      try {
        return await this.operation();
      } catch (error) {
        lastError = error;

        if (attempt < this.config.maxAttempts) {
          await new Promise(resolve => setTimeout(resolve, delay));
          delay *= this.config.backoffMultiplier;
        }
      }
    }

    throw lastError;
  }
}

// Usage
const fetchUserWithRetry = Retry.of(() => fetch('/api/user'))
  .withRetries(3)
  .withDelay(1000)
  .withBackoff(2)
  .map(response => response.json());

// Compose operations
const program = fetchUserWithRetry
  .flatMap(user =>
    Retry.of(() => fetch(`/api/posts/${user.id}`))
      .withRetries(2)
      .map(response => response.json())
  )
  .map(posts => ({ user, posts }));

// Execute
program.run()
  .then(data => console.log('Success:', data))
  .catch(error => console.error('Failed after retries:', error));
```

## Example 4: Parser Monad

Build composable parsers:

```typescript
type ParseResult<T> =
  | { success: true; value: T; remaining: string }
  | { success: false; error: string };

class Parser<T> {
  constructor(private parse: (input: string) => ParseResult<T>) {}

  static of<T>(value: T): Parser<T> {
    return new Parser(input => ({
      success: true,
      value,
      remaining: input
    }));
  }

  map<U>(fn: (value: T) => U): Parser<U> {
    return new Parser(input => {
      const result = this.parse(input);
      if (result.success) {
        return {
          success: true,
          value: fn(result.value),
          remaining: result.remaining
        };
      }
      return result;
    });
  }

  flatMap<U>(fn: (value: T) => Parser<U>): Parser<U> {
    return new Parser(input => {
      const result = this.parse(input);
      if (result.success) {
        const nextParser = fn(result.value);
        return nextParser.parse(result.remaining);
      }
      return result;
    });
  }

  run(input: string): ParseResult<T> {
    return this.parse(input);
  }

  // Combinator: try this parser, or the other
  or(other: Parser<T>): Parser<T> {
    return new Parser(input => {
      const result = this.parse(input);
      if (result.success) return result;
      return other.parse(input);
    });
  }

  // Combinator: parse many times
  many(): Parser<T[]> {
    return new Parser(input => {
      const values: T[] = [];
      let remaining = input;

      while (true) {
        const result = this.parse(remaining);
        if (!result.success) break;

        values.push(result.value);
        remaining = result.remaining;
      }

      return {
        success: true,
        value: values,
        remaining
      };
    });
  }
}

// Basic parsers
function char(c: string): Parser<string> {
  return new Parser(input => {
    if (input[0] === c) {
      return {
        success: true,
        value: c,
        remaining: input.slice(1)
      };
    }
    return {
      success: false,
      error: `Expected '${c}', got '${input[0]}'`
    };
  });
}

function digit(): Parser<number> {
  return new Parser(input => {
    const c = input[0];
    if (/\d/.test(c)) {
      return {
        success: true,
        value: parseInt(c, 10),
        remaining: input.slice(1)
      };
    }
    return {
      success: false,
      error: `Expected digit, got '${c}'`
    };
  });
}

function letter(): Parser<string> {
  return new Parser(input => {
    const c = input[0];
    if (/[a-zA-Z]/.test(c)) {
      return {
        success: true,
        value: c,
        remaining: input.slice(1)
      };
    }
    return {
      success: false,
      error: `Expected letter, got '${c}'`
    };
  });
}

function whitespace(): Parser<string> {
  return new Parser(input => {
    const match = input.match(/^\s+/);
    if (match) {
      return {
        success: true,
        value: match[0],
        remaining: input.slice(match[0].length)
      };
    }
    return {
      success: false,
      error: 'Expected whitespace'
    };
  });
}

// Optional whitespace
function optionalWhitespace(): Parser<string> {
  return new Parser(input => {
    const match = input.match(/^\s*/);
    return {
      success: true,
      value: match ? match[0] : '',
      remaining: input.slice(match ? match[0].length : 0)
    };
  });
}

// Compose parsers
function number(): Parser<number> {
  return digit().many().map(digits =>
    digits.reduce((acc, d) => acc * 10 + d, 0)
  );
}

function word(): Parser<string> {
  return letter().many().map(letters => letters.join(''));
}

function identifier(): Parser<string> {
  return letter().flatMap(first =>
    letter().or(digit()).many().map(rest =>
      [first, ...rest].join('')
    )
  );
}

// Parse expressions with whitespace: "x + 5" or "x+5"
function expression(): Parser<{ left: string; op: string; right: number }> {
  return identifier()
    .flatMap(left => optionalWhitespace()
      .flatMap(() => char('+')
        .flatMap(() => optionalWhitespace()
          .flatMap(() => number()
            .map(right => ({ left, op: '+', right }))
          )
        )
      )
    );
}

// Usage
const result1 = expression().run('x+42');
if (result1.success) {
  console.log(result1.value);  // { left: 'x', op: '+', right: 42 }
}

const result2 = expression().run('x + 42');
if (result2.success) {
  console.log(result2.value);  // { left: 'x', op: '+', right: 42 }
}
```

## Example 5: Probability Monad

Represent probabilistic computations:

**Note:** This implementation works for discrete distributions with finite outcomes (like dice, coin flips, card draws). Continuous distributions (like normal, exponential) would require a different approach using probability density functions or sampling methods.

```typescript
type Probability = number;  // 0 to 1

class Dist<T> {
  constructor(private outcomes: Map<T, Probability>) {}

  static of<T>(value: T): Dist<T> {
    return new Dist(new Map([[value, 1.0]]));
  }

  // Create a distribution from outcomes
  static from<T>(outcomes: [T, Probability][]): Dist<T> {
    return new Dist(new Map(outcomes));
  }

  // Uniform distribution
  static uniform<T>(values: T[]): Dist<T> {
    const prob = 1.0 / values.length;
    return new Dist(new Map(values.map(v => [v, prob])));
  }

  map<U>(fn: (value: T) => U): Dist<U> {
    const newOutcomes = new Map<U, Probability>();

    for (const [value, prob] of this.outcomes) {
      const newValue = fn(value);
      const existingProb = newOutcomes.get(newValue) || 0;
      newOutcomes.set(newValue, existingProb + prob);
    }

    return new Dist(newOutcomes);
  }

  flatMap<U>(fn: (value: T) => Dist<U>): Dist<U> {
    const newOutcomes = new Map<U, Probability>();

    for (const [value, prob1] of this.outcomes) {
      const dist = fn(value);
      for (const [newValue, prob2] of dist.outcomes) {
        const combinedProb = prob1 * prob2;
        const existingProb = newOutcomes.get(newValue) || 0;
        newOutcomes.set(newValue, existingProb + combinedProb);
      }
    }

    return new Dist(newOutcomes);
  }

  // Get all outcomes
  getOutcomes(): [T, Probability][] {
    return Array.from(this.outcomes.entries());
  }

  // Most likely outcome
  mostLikely(): T | null {
    let maxProb = 0;
    let maxValue: T | null = null;

    for (const [value, prob] of this.outcomes) {
      if (prob > maxProb) {
        maxProb = prob;
        maxValue = value;
      }
    }

    return maxValue;
  }

  // Expected value (for numbers)
  expectation(this: Dist<number>): number {
    let sum = 0;
    for (const [value, prob] of this.outcomes) {
      sum += value * prob;
    }
    return sum;
  }
}

// Examples
const coinFlip = Dist.uniform(['Heads', 'Tails']);

const twoCoinFlips = coinFlip.flatMap(first =>
  coinFlip.map(second => [first, second])
);

console.log(twoCoinFlips.getOutcomes());
// [['Heads', 'Heads'], 0.25]
// [['Heads', 'Tails'], 0.25]
// [['Tails', 'Heads'], 0.25]
// [['Tails', 'Tails'], 0.25]

// Sum of two dice
const die = Dist.uniform([1, 2, 3, 4, 5, 6]);

const twoDice = die.flatMap(first =>
  die.map(second => first + second)
);

console.log('Most likely sum:', twoDice.mostLikely());  // 7
console.log('Expected value:', twoDice.expectation());  // 7
```

## Testing Custom Monads

Always test the monad laws:

```typescript
function testMonadLaws<T, U, V>(
  M: {
    of<A>(a: A): Monad<A>
  },
  x: T,
  m: Monad<T>,
  f: (a: T) => Monad<U>,
  g: (b: U) => Monad<V>,
  equals: (a: Monad<any>, b: Monad<any>) => boolean
) {
  // Left identity
  const leftId1 = M.of(x).flatMap(f);
  const leftId2 = f(x);
  if (!equals(leftId1, leftId2)) {
    throw new Error('Left identity failed');
  }

  // Right identity
  const rightId1 = m.flatMap(a => M.of(a));
  if (!equals(rightId1, m)) {
    throw new Error('Right identity failed');
  }

  // Associativity
  const assoc1 = m.flatMap(f).flatMap(g);
  const assoc2 = m.flatMap(a => f(a).flatMap(g));
  if (!equals(assoc1, assoc2)) {
    throw new Error('Associativity failed');
  }

  console.log('All monad laws verified ✓');
}
```

## Best Practices

1. **Always verify the laws** - Test left identity, right identity, associativity
2. **Document the context** - What does your monad represent?
3. **Provide helper functions** - Make it ergonomic to use
4. **Consider performance** - Monad chains can add overhead
5. **Keep it simple** - If a simpler abstraction works, use it
6. **Name it clearly** - The name should convey the computation type

## What We've Learned

- Custom monads encapsulate domain-specific patterns
- Follow the recipe: type constructor, of, map, flatMap
- Always test the laws
- Build when standard monads don't fit
- Examples: Trace, RemoteData, Retry, Parser, Probability
- Helper functions make monads ergonomic

## Coming Up

You've built your own monads. Now let's see monads in production code - how real-world libraries and frameworks use them to solve practical problems.

---

**Next: [Chapter 18 - Monads in the Wild](18-real-world.md)**
