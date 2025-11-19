# Chapter 16: Common Patterns and Antipatterns

You know what monads are. You've seen the theory. Now let's talk about *using* them effectively in real code.

## When to Use Monads

### ✅ Use Maybe When

**Absence is expected and self-explanatory**

```typescript
// Good: lookup operations
function findUser(id: string): Maybe<User>
function getConfig(key: string): Maybe<ConfigValue>
function parseNumber(s: string): Maybe<number>

// The absence speaks for itself - not found, invalid, etc.
```

**Not Good: When you need to know WHY it's absent**

```typescript
// Bad: what went wrong?
function login(username: string, password: string): Maybe<Session>
// Did the user not exist? Wrong password? Account locked?

// Better:
function login(username: string, password: string): Either<LoginError, Session>
```

### ✅ Use Either When

**You need error details**

```typescript
type PaymentError =
  | { type: 'InsufficientFunds'; available: number; required: number }
  | { type: 'CardDeclined'; reason: string }
  | { type: 'NetworkError'; message: string };

function processPayment(amount: number): Either<PaymentError, Receipt>
```

**Operations that fail in multiple ways**

```typescript
function parseConfig(json: string): Either<ConfigError, Config>
function validateForm(data: FormData): Either<ValidationError[], ValidatedData>
function connectDatabase(url: string): Either<ConnectionError, Connection>
```

### ✅ Use IO When

**You need to track side effects**

```typescript
function readFile(path: string): IO<string>
function writeFile(path: string, content: string): IO<void>
function fetchUrl(url: string): IO<Response>

// Makes effects explicit in types
```

**You want to separate description from execution**

```typescript
// Build a program
const program = readFile('input.txt')
  .map(transform)
  .flatMap(data => writeFile('output.txt', data));

// Execute later
program.unsafeRun();
```

### ✅ Use State When

**Threading state through operations**

```typescript
// Good: parsers
function parseToken(): State<ParserState, Token>
function parseExpression(): State<ParserState, Expression>

// Good: random number generation
function nextRandom(): State<RNG, number>
function shuffle<T>(array: T[]): State<RNG, T[]>

// Good: stateful transformations
function allocateId(): State<IdGenerator, string>
```

**Not Good: When mutation is simpler**

```typescript
// Bad: unnecessary complexity
function increment(): State<Counter, void>

// Better: just use a variable
let counter = 0;
counter++;
```

### ✅ Use Reader When

**Multiple functions need the same configuration**

```typescript
function getApiUrl(): Reader<Config, string>
function getTimeout(): Reader<Config, number>
function fetchUser(id: string): Reader<Config, Promise<User>>

// Config threaded automatically
```

**Dependency injection without a framework**

```typescript
interface Services {
  database: Database;
  logger: Logger;
  cache: Cache;
}

function createUser(data: UserData): Reader<Services, User>
function sendEmail(user: User): Reader<Services, void>
```

## Common Patterns

### Pattern 1: Early Return with Maybe/Either

```typescript
// Instead of nested checks
function process(id: string): Maybe<Result> {
  const user = findUser(id);
  if (!user) return Maybe.none();

  const profile = getProfile(user);
  if (!profile) return Maybe.none();

  return transform(profile);
}

// Use flatMap for clean flow
function process(id: string): Maybe<Result> {
  return findUser(id)
    .flatMap(getProfile)
    .map(transform);
}
```

### Pattern 2: Validation with Accumulation

```typescript
// Collect all validation errors
function validateUser(data: UserData): Validation<Error[], ValidUser> {
  return Validation.combine3(
    validateEmail(data.email),
    validatePassword(data.password),
    validateAge(data.age),
    (email, password, age) => ({ email, password, age })
  );
}

// All errors reported at once
```

### Pattern 3: Optional Chaining

```typescript
// Deep optional access
function getStreetName(userId: string): Maybe<string> {
  return findUser(userId)
    .flatMap(user => Maybe.fromNullable(user.address))
    .flatMap(address => Maybe.fromNullable(address.street))
    .map(street => street.name);
}
```

### Pattern 4: Fallback Values

```typescript
// Provide defaults
const email = findUser(id)
  .flatMap(user => Maybe.fromNullable(user.email))
  .getOrElse('no-email@example.com');

// Or fallback chains
const result = primarySource()
  .orElse(() => secondarySource())
  .orElse(() => fallbackSource())
  .getOrElse(defaultValue);
```

### Pattern 5: Async Chains

```typescript
// Sequential async operations
async function loadUserData(id: string): Promise<UserData> {
  const user = await fetchUser(id);
  const posts = await fetchPosts(user.id);
  const comments = await fetchComments(posts[0].id);
  return { user, posts, comments };
}

// Or with Task for better composition
function loadUserData(id: string): Task<UserData> {
  return fetchUser(id)
    .flatMap(user => fetchPosts(user.id).map(posts => ({ user, posts })))
    .flatMap(({ user, posts }) =>
      fetchComments(posts[0].id).map(comments => ({ user, posts, comments }))
    );
}
```

### Pattern 6: Error Recovery

```typescript
// Retry with exponential backoff
function fetchWithRetry(url: string): Task<Response> {
  return fetch(url)
    .retry(3, 1000)
    .catch(error => {
      logger.error('Fetch failed after retries', error);
      return Task.of(fallbackResponse);
    });
}

// Transform errors
function processData(id: string): Either<AppError, Data> {
  return fetchData(id)
    .mapLeft(networkError => ({
      type: 'FetchError',
      originalError: networkError,
      timestamp: new Date()
    }));
}
```

### Pattern 7: Conditional Operations

```typescript
// Execute based on conditions
function processIfActive(user: User): Maybe<Result> {
  return Maybe.of(user)
    .filter(u => u.isActive)
    .flatMap(processUser);
}

// Guard pattern
function withdraw(account: Account, amount: number): Either<Error, Account> {
  return Either.of(account)
    .flatMap(acc =>
      acc.balance >= amount
        ? Either.right(acc)
        : Either.left({ type: 'InsufficientFunds' })
    )
    .map(acc => ({ ...acc, balance: acc.balance - amount }));
}
```

### Pattern 8: Lift and Transform

```typescript
// Lift functions into monadic context
const add = (a: number) => (b: number) => a + b;

Maybe.of(5)
  .ap(Maybe.of(add))  // Lift function into Maybe
  .ap(Maybe.of(3));   // Apply to another Maybe
// Some(8)

// Or use liftA2 helper
const liftedAdd = liftA2(Maybe)(add);
liftedAdd(Maybe.of(5), Maybe.of(3));  // Some(8)
```

## Antipatterns

### ❌ Antipattern 1: Unnecessary Nesting

```typescript
// Bad: nested monads
function getEmail(userId: string): Maybe<Maybe<string>> {
  return findUser(userId).map(user =>
    Maybe.fromNullable(user.email)
  );
}

// Good: flatten with flatMap
function getEmail(userId: string): Maybe<string> {
  return findUser(userId)
    .flatMap(user => Maybe.fromNullable(user.email));
}
```

### ❌ Antipattern 2: Breaking Out Too Early

```typescript
// Bad: extracting value prematurely
function processUser(id: string): User {
  const maybeUser = findUser(id);
  if (maybeUser.isNone()) throw new Error('Not found');

  const user = maybeUser.getValue();  // Break out of Maybe
  // Now back to null checking...
  return user;
}

// Good: stay in the monad
function processUser(id: string): Maybe<User> {
  return findUser(id)
    .map(enrichUser)
    .map(validateUser);
}
```

### ❌ Antipattern 3: Using Monads for Everything

```typescript
// Bad: unnecessary abstraction
function add(a: number, b: number): Maybe<number> {
  return Maybe.of(a + b);
}

// Good: simple functions stay simple
function add(a: number, b: number): number {
  return a + b;
}

// Only wrap when needed
function safeAdd(a: Maybe<number>, b: Maybe<number>): Maybe<number> {
  return a.flatMap(x => b.map(y => x + y));
}
```

### ❌ Antipattern 4: Ignoring Errors

```typescript
// Bad: swallowing errors
function getUser(id: string): Maybe<User> {
  return fetchUser(id)
    .catch(() => Maybe.none());  // Lost error information!
}

// Good: preserve error details
function getUser(id: string): Either<FetchError, User> {
  return fetchUser(id)
    .catch(error => Either.left({
      type: 'FetchFailed',
      reason: error.message,
      userId: id
    }));
}
```

### ❌ Antipattern 5: Mixing Paradigms Carelessly

```typescript
// Bad: mixing monadic and imperative
function processUsers(ids: string[]): User[] {
  const users: User[] = [];

  for (const id of ids) {
    const maybeUser = findUser(id);
    if (maybeUser.isSome()) {
      users.push(maybeUser.getValue());
    }
  }

  return users;
}

// Good: stay functional
function processUsers(ids: string[]): User[] {
  return ids
    .map(findUser)
    .filter(m => m.isSome())
    .map(m => m.getValue());
}

// Or better: traverse
function processUsers(ids: string[]): Maybe<User[]> {
  return traverse(ids, findUser);
}
```

### ❌ Antipattern 6: Over-abstracting

```typescript
// Bad: abstraction for abstraction's sake
class UserMonad<T> implements Monad<T> {
  constructor(private value: T, private userId: string) {}

  map<U>(fn: (value: T) => U): UserMonad<U> {
    return new UserMonad(fn(this.value), this.userId);
  }

  flatMap<U>(fn: (value: T) => UserMonad<U>): UserMonad<U> {
    const result = fn(this.value);
    return new UserMonad(result.value, this.userId);
  }
}

// Good: use existing monads
type UserContext<T> = Reader<User, T>;

function withUser<T>(fn: (user: User) => T): UserContext<T> {
  return Reader.asks(fn);
}
```

### ❌ Antipattern 7: Forcing Monad Laws on Non-Monads

```typescript
// Bad: claiming something is a monad when it isn't
class Cache<T> {
  constructor(private cache: Map<string, T>) {}

  flatMap<U>(fn: (value: T) => Cache<U>): Cache<U> {
    // This doesn't make sense - Cache isn't a monad!
    // What does it mean to flatMap over all cached values?
  }
}

// Good: use appropriate abstractions
class Cache<T> {
  get(key: string): Maybe<T> {
    return Maybe.fromNullable(this.cache.get(key));
  }

  set(key: string, value: T): void {
    this.cache.set(key, value);
  }
}
```

## Decision Tree

Use this to decide which monad to reach for:

```
Is the value optional?
├─ Yes: Absence self-explanatory?
│  ├─ Yes → Maybe
│  └─ No → Either
└─ No: Multiple values?
   ├─ Yes → List/Array
   └─ No: Involves I/O?
      ├─ Yes → IO/Task
      └─ No: Needs state?
         ├─ Yes → State
         └─ No: Needs config?
            ├─ Yes → Reader
            └─ No: Needs logging?
               ├─ Yes → Writer
               └─ No: Is async?
                  ├─ Yes → Promise/Task
                  └─ No: Maybe you don't need a monad!
```

## Performance Considerations

### When Monads Hurt Performance

```typescript
// Bad: tight loop with monad overhead
function sumArray(numbers: number[]): Maybe<number> {
  let result = Maybe.of(0);
  for (const n of numbers) {
    result = result.flatMap(sum => Maybe.of(sum + n));
  }
  return result;
}

// Good: use plain values in hot paths
function sumArray(numbers: number[]): number {
  let result = 0;
  for (const n of numbers) {
    result += n;
  }
  return result;
}

// Wrap only at boundaries
function sumArraySafe(numbers: Maybe<number[]>): Maybe<number> {
  return numbers.map(sumArray);
}
```

### When Monads Help Performance

```typescript
// Lazy evaluation with Task
const expensiveOperation = Task.of(() => {
  // This doesn't run yet!
  return computeExpensiveResult();
});

// Only executed when needed
if (shouldCompute) {
  expensiveOperation.run();
}

// Short-circuiting with Either
function validateAll(data: Data): Either<Error, ValidData> {
  return validateEmail(data.email)
    .flatMap(() => validatePassword(data.password))  // Skipped if email fails
    .flatMap(() => validateAge(data.age));           // Skipped if either fails
}
```

## Composition Patterns

### Sequential Composition

```typescript
// Chain dependent operations
const program = fetchUser(id)
  .flatMap(user => fetchPosts(user.id))
  .flatMap(posts => fetchComments(posts[0].id));
```

### Parallel Composition

```typescript
// Independent operations
const results = Promise.all([
  fetchUser(id),
  fetchPosts(id),
  fetchSettings(id)
]);
```

### Conditional Composition

```typescript
// Branch based on values
const result = checkPermissions(user)
  .flatMap(hasPermission =>
    hasPermission
      ? performOperation()
      : Either.left({ type: 'PermissionDenied' })
  );
```

## Testing Patterns

```typescript
// Test monadic code by inspecting results
describe('findUser', () => {
  it('returns Some when user exists', () => {
    const result = findUser('existing-id');
    expect(result.isSome()).toBe(true);
  });

  it('returns None when user does not exist', () => {
    const result = findUser('nonexistent-id');
    expect(result.isNone()).toBe(true);
  });
});

// Test pure logic separately from effects
describe('validateEmail', () => {
  it('accepts valid emails', () => {
    const result = validateEmail('user@example.com');
    expect(result.isRight()).toBe(true);
  });
});

// Mock monadic dependencies
const mockDatabase = {
  findUser: (id: string) => Maybe.of(testUser)
};
```

## What We've Learned

- Use the right monad for the job
- Maybe for optional, Either for errors with details
- IO for effects, State for stateful computations
- Reader for config, Writer for logging
- Avoid unnecessary nesting
- Stay in the monad until boundaries
- Don't over-abstract
- Consider performance in hot paths
- Test monadic code by inspecting results

## Coming Up

You know when to use monads. But what about building your own? When should you create a custom monad, and how do you do it correctly?

---

**Next: [Chapter 17 - Building Your Own Monads](17-custom-monads.md)**
