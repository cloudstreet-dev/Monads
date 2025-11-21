# Chapter 14: Monad Transformers - Stacking Effects

You've hit the wall. You need both Maybe AND IO. Or Either AND State. Or Reader AND Task.

```typescript
// This is ugly
function findUser(id: string): IO<Maybe<User>> {
  return new IO(() => {
    const user = database.findUser(id);
    return user ? Maybe.of(user) : Maybe.none();
  });
}

// Now you have nested monads
findUser("123").map(maybeUser =>
  maybeUser.map(user =>
    // Getting deeply nested...
  )
);
```

The problem: **monads don't compose automatically**. When you have `Maybe<IO<T>>` or `IO<Maybe<T>>`, you're stuck with nested structure.

**Monad Transformers** solve this by stacking monads into a single, cohesive type.

## The Problem in Detail

```typescript
// Find user (might not exist)
function findUser(id: string): IO<Maybe<User>> {
  return new IO(() => Maybe.fromNullable(db.findUser(id)));
}

// Get email (might not exist)
function getEmail(user: User): Maybe<string> {
  return Maybe.fromNullable(user.email);
}

// Try to compose them
const program = findUser("123")
  .map(maybeUser =>              // IO layer
    maybeUser.flatMap(user =>    // Maybe layer - nested!
      getEmail(user)
    )
  );
// Type: IO<Maybe<Maybe<string>>> - Yuck!

// To use it:
program.unsafeRun()  // Maybe<Maybe<string>>
  .flatMap(x => x)   // Flatten manually
  .fold(
    () => console.log("Not found"),
    email => console.log(email)
  );
```

Too much nesting! We want to treat `IO<Maybe<T>>` as a single monad.

## Introducing MaybeT (Maybe Transformer)

```typescript
// MaybeT<M, A> wraps M<Maybe<A>>
class MaybeT<M, A> {
  constructor(private value: M<Maybe<A>>) {}

  // Wrap a value
  static of<M, A>(value: A, monad: MonadLike<M>): MaybeT<M, A> {
    return new MaybeT(
      monad.of(Maybe.of(value))
    );
  }

  // Wrap a None
  static none<M, A>(monad: MonadLike<M>): MaybeT<M, A> {
    return new MaybeT(
      monad.of(Maybe.none<A>())
    );
  }

  // Transform the value
  map<B>(fn: (value: A) => B, monad: MonadLike<M>): MaybeT<M, B> {
    return new MaybeT(
      monad.map(
        this.value,
        (maybe: Maybe<A>) => maybe.map(fn)
      )
    );
  }

  // Chain MaybeT operations
  flatMap<B>(
    fn: (value: A) => MaybeT<M, B>,
    monad: MonadLike<M>
  ): MaybeT<M, B> {
    return new MaybeT(
      monad.flatMap(
        this.value,
        (maybe: Maybe<A>) =>
          maybe.fold(
            () => monad.of(Maybe.none<B>()),
            value => fn(value).run()
          )
      )
    );
  }

  // Get the wrapped value
  run(): M<Maybe<A>> {
    return this.value;
  }

  // Lift an M<A> into MaybeT<M, A>
  static lift<M, A>(m: M<A>, monad: MonadLike<M>): MaybeT<M, A> {
    return new MaybeT(
      monad.map(m, (value: A) => Maybe.of(value))
    );
  }
}

// Helper interface for monad operations
interface MonadLike<M> {
  of<A>(value: A): M<A>;
  map<A, B>(m: M<A>, fn: (a: A) => B): M<B>;
  flatMap<A, B>(m: M<A>, fn: (a: A) => M<B>): M<B>;
}

// IO monad instance
const IOMonad: MonadLike<IO> = {
  of: <A>(value: A) => IO.of(value),
  map: <A, B>(io: IO<A>, fn: (a: A) => B) => io.map(fn),
  flatMap: <A, B>(io: IO<A>, fn: (a: A) => IO<B>) => io.flatMap(fn)
};
```

## Using MaybeT

```typescript
// Now we can compose cleanly!
function findUser(id: string): MaybeT<IO, User> {
  return MaybeT.lift(
    new IO(() => database.findUser(id)),
    IOMonad
  ).flatMap(
    user => user ? MaybeT.of(user, IOMonad) : MaybeT.none(IOMonad),
    IOMonad
  );
}

function getEmail(user: User): MaybeT<IO, string> {
  return user.email
    ? MaybeT.of(user.email, IOMonad)
    : MaybeT.none(IOMonad);
}

function sendEmail(email: string, message: string): MaybeT<IO, void> {
  return MaybeT.lift(
    new IO(() => emailService.send(email, message)),
    IOMonad
  );
}

// Compose naturally!
const program = findUser("123")
  .flatMap(user => getEmail(user), IOMonad)
  .flatMap(email => sendEmail(email, "Hello!"), IOMonad);

// Run both layers at once
const result = program.run().unsafeRun();
// Type: Maybe<void>

result.fold(
  () => console.log("User not found or no email"),
  () => console.log("Email sent!")
);
```

No more nesting!

## EitherT (Either Transformer)

For error handling + other effects:

```typescript
// EitherT<M, E, A> wraps M<Either<E, A>>
class EitherT<M, E, A> {
  constructor(private value: M<Either<E, A>>) {}

  static right<M, E, A>(value: A, monad: MonadLike<M>): EitherT<M, E, A> {
    return new EitherT(
      monad.of(Either.right<E, A>(value))
    );
  }

  static left<M, E, A>(error: E, monad: MonadLike<M>): EitherT<M, E, A> {
    return new EitherT(
      monad.of(Either.left<E, A>(error))
    );
  }

  map<B>(fn: (value: A) => B, monad: MonadLike<M>): EitherT<M, E, B> {
    return new EitherT(
      monad.map(
        this.value,
        (either: Either<E, A>) => either.map(fn)
      )
    );
  }

  flatMap<B>(
    fn: (value: A) => EitherT<M, E, B>,
    monad: MonadLike<M>
  ): EitherT<M, E, B> {
    return new EitherT(
      monad.flatMap(
        this.value,
        (either: Either<E, A>) =>
          either.fold(
            error => monad.of(Either.left<E, B>(error)),
            value => fn(value).run()
          )
      )
    );
  }

  run(): M<Either<E, A>> {
    return this.value;
  }

  static lift<M, E, A>(m: M<A>, monad: MonadLike<M>): EitherT<M, E, A> {
    return new EitherT(
      monad.map(m, (value: A) => Either.right<E, A>(value))
    );
  }
}
```

## Real Example: Database Operations with Error Handling

```typescript
type DbError = { type: 'NotFound' | 'QueryError' | 'ConnectionError'; message: string };

// Type alias makes signatures much more readable
type DbIO<A> = EitherT<IO, DbError, A>;

// Database operations return DbIO<A>
function findUser(id: string): DbIO<User> {
  return EitherT.lift(
    new IO(() => database.query('SELECT * FROM users WHERE id = ?', [id])[0]),
    IOMonad
  ).flatMap(
    user => user
      ? EitherT.right(user, IOMonad)
      : EitherT.left({ type: 'NotFound', message: `User ${id} not found` }, IOMonad),
    IOMonad
  );
}

function findPosts(userId: string): DbIO<Post[]> {
  return EitherT.lift(
    new IO(() => database.query('SELECT * FROM posts WHERE user_id = ?', [userId])),
    IOMonad
  );
}

function updatePostCount(userId: string, count: number): DbIO<void> {
  return EitherT.lift(
    new IO(() => {
      database.query('UPDATE users SET post_count = ? WHERE id = ?', [count, userId]);
    }),
    IOMonad
  );
}

// Compose database operations
const program = findUser("123")
  .flatMap(user => findPosts(user.id), IOMonad)
  .flatMap(posts =>
    updatePostCount("123", posts.length).map(() => posts, IOMonad),
    IOMonad
  );

// Execute
const result = program.run().unsafeRun();
// Type: Either<DbError, Post[]>

result.fold(
  error => console.error(`Database error: ${error.message}`),
  posts => console.log(`Found ${posts.length} posts`)
);
```

## ReaderT (Reader Transformer)

Combine configuration with other effects:

```typescript
// ReaderT<R, M, A> wraps R => M<A>
class ReaderT<R, M, A> {
  constructor(private runReader: (env: R) => M<A>) {}

  static of<R, M, A>(value: A, monad: MonadLike<M>): ReaderT<R, M, A> {
    return new ReaderT(() => monad.of(value));
  }

  static ask<R, M>(monad: MonadLike<M>): ReaderT<R, M, R> {
    return new ReaderT((env: R) => monad.of(env));
  }

  map<B>(fn: (value: A) => B, monad: MonadLike<M>): ReaderT<R, M, B> {
    return new ReaderT((env: R) =>
      monad.map(this.runReader(env), fn)
    );
  }

  flatMap<B>(
    fn: (value: A) => ReaderT<R, M, B>,
    monad: MonadLike<M>
  ): ReaderT<R, M, B> {
    return new ReaderT((env: R) =>
      monad.flatMap(
        this.runReader(env),
        value => fn(value).run(env)
      )
    );
  }

  run(env: R): M<A> {
    return this.runReader(env);
  }

  static lift<R, M, A>(m: M<A>): ReaderT<R, M, A> {
    return new ReaderT(() => m);
  }
}

// Example: Config + IO
interface AppConfig {
  database: { url: string };
  email: { apiKey: string };
}

function getDatabaseUrl(): ReaderT<AppConfig, IO, string> {
  return ReaderT.ask<AppConfig, IO>(IOMonad).map(
    config => config.database.url,
    IOMonad
  );
}

function connectToDatabase(url: string): ReaderT<AppConfig, IO, Connection> {
  return ReaderT.lift(
    new IO(() => database.connect(url))
  );
}

function getEmailApiKey(): ReaderT<AppConfig, IO, string> {
  return ReaderT.ask<AppConfig, IO>(IOMonad).map(
    config => config.email.apiKey,
    IOMonad
  );
}

// Compose
const program = getDatabaseUrl()
  .flatMap(url => connectToDatabase(url), IOMonad)
  .flatMap(conn =>
    getEmailApiKey().map(apiKey => ({ conn, apiKey }), IOMonad),
    IOMonad
  );

// Run with config
const config: AppConfig = {
  database: { url: 'postgresql://localhost/mydb' },
  email: { apiKey: 'secret-key' }
};

const result = program.run(config).unsafeRun();
// { conn: Connection, apiKey: string }
```

## StateT (State Transformer)

Combine state with other effects:

```typescript
// StateT<S, M, A> wraps S => M<[A, S]>
class StateT<S, M, A> {
  constructor(private runState: (state: S) => M<[A, S]>) {}

  static of<S, M, A>(value: A, monad: MonadLike<M>): StateT<S, M, A> {
    return new StateT(state => monad.of([value, state]));
  }

  static get<S, M>(monad: MonadLike<M>): StateT<S, M, S> {
    return new StateT(state => monad.of([state, state]));
  }

  static put<S, M>(newState: S, monad: MonadLike<M>): StateT<S, M, void> {
    return new StateT(() => monad.of([undefined as void, newState]));
  }

  map<B>(fn: (value: A) => B, monad: MonadLike<M>): StateT<S, M, B> {
    return new StateT(state =>
      monad.map(
        this.runState(state),
        ([value, newState]) => [fn(value), newState]
      )
    );
  }

  flatMap<B>(
    fn: (value: A) => StateT<S, M, B>,
    monad: MonadLike<M>
  ): StateT<S, M, B> {
    return new StateT(state =>
      monad.flatMap(
        this.runState(state),
        ([value, newState]) => fn(value).run(newState)
      )
    );
  }

  run(initialState: S): M<[A, S]> {
    return this.runState(initialState);
  }

  static lift<S, M, A>(m: M<A>, monad: MonadLike<M>): StateT<S, M, A> {
    return new StateT(state =>
      monad.map(m, value => [value, state])
    );
  }
}
```

### Example: Stateful Logging with IO

Combining state (for a counter) with IO (for side effects):

```typescript
// Type alias for clarity
type CounterIO<A> = StateT<number, IO, A>;

// Stateful operations that also do IO
function logWithCount(message: string): CounterIO<void> {
  return StateT.get<number, IO>(IOMonad).flatMap(count =>
    StateT.lift(
      new IO(() => console.log(`[${count}] ${message}`)),
      IOMonad
    ).flatMap(() =>
      StateT.put(count + 1, IOMonad),
      IOMonad
    ),
    IOMonad
  );
}

function getUserWithLogging(id: string): CounterIO<User> {
  return logWithCount(`Fetching user ${id}`)
    .flatMap(() =>
      StateT.lift(
        new IO(() => database.findUser(id)),
        IOMonad
      ),
      IOMonad
    )
    .flatMap(user => {
      return logWithCount(`Found user: ${user.name}`)
        .map(() => user, IOMonad);
    }, IOMonad);
}

// Compose stateful IO operations
const program = logWithCount("Starting program")
  .flatMap(() => getUserWithLogging("123"), IOMonad)
  .flatMap(user => logWithCount(`Processing user ${user.name}`), IOMonad);

// Run with initial state
const result = program.run(0).unsafeRun();
// Console output:
// [0] Starting program
// [1] Fetching user 123
// [2] Found user: Alice
// [3] Processing user Alice

// result = [undefined, 4] - final value and final counter
```

This combines:
- **State**: Automatic counter incrementing
- **IO**: Actual console logging
- **Clean composition**: No manual state threading

## Practical Pattern: The MTL Stack

In real applications, you often stack multiple transformers. Let's build this step by step.

### Step 1: Define the Error Type

```typescript
type AppError =
  | { type: 'DatabaseError'; message: string }
  | { type: 'EmailError'; message: string }
  | { type: 'ValidationError'; errors: string[] };
```

### Step 2: Build the Stack with Type Aliases

```typescript
// Start with IO at the base
// Add error handling: EitherT<IO, AppError, A>
type IOWithErrors<A> = EitherT<IO, AppError, A>;

// Add configuration: ReaderT<Config, IOWithErrors, A>
type App<A> = ReaderT<AppConfig, IOWithErrors, A>;

// This is equivalent to:
// App<A> = ReaderT<AppConfig, EitherT<IO, AppError, A>>
// Which desugars to: Config => IO<Either<AppError, A>>
```

### Step 3: Create Helper Functions

Rather than working with the raw stack, create helpers:

```typescript
class AppHelpers {
  // Wrap a pure value
  static pure<A>(value: A): App<A> {
    const eitherMonad = /* EitherT monad instance */;
    return ReaderT.of(value, eitherMonad);
  }

  // Get the config
  static getConfig(): App<AppConfig> {
    const eitherMonad = /* EitherT monad instance */;
    return ReaderT.ask(eitherMonad);
  }

  // Perform IO
  static io<A>(effect: IO<A>): App<A> {
    const lifted = EitherT.lift(effect, IOMonad);
    return ReaderT.lift(lifted);
  }

  // Fail with an error
  static fail<A>(error: AppError): App<A> {
    const failed = EitherT.left<IO, AppError, A>(error, IOMonad);
    return ReaderT.lift(failed);
  }
}
```

### Step 4: Write Business Logic

Now the code is much clearer:

```typescript
function saveUser(user: User): App<User> {
  return AppHelpers.io(new IO(() => database.save(user)));
}

function sendWelcomeEmail(user: User): App<void> {
  return AppHelpers.io(new IO(() => emailService.send(user.email, "Welcome!")));
}

// Compose operations - this reads much better!
function registerUser(name: string, email: string): App<User> {
  const user = { name, email };

  return saveUser(user).flatMap(savedUser =>
    sendWelcomeEmail(savedUser).map(() =>
      savedUser
    )
  );
}
```

### Step 5: Run the Stack

```typescript
const config: AppConfig = loadConfig();
const program = registerUser("Alice", "alice@example.com");

// Unwrap each layer
const result = program
  .run(config)      // ReaderT layer -> EitherT<IO, AppError, User>
  .run()            // EitherT layer -> IO<Either<AppError, User>>
  .unsafeRun();     // IO layer -> Either<AppError, User>

// Handle the result
result.fold(
  error => console.error(`Error: ${error.message}`),
  user => console.log(`Created user: ${user.name}`)
);
```

**Key insight**: Type aliases (`App<A>`) and helper functions make transformer stacks manageable. Without them, the types become unreadable.

## Why Transformers Are Hard

Monad transformers have a reputation for being difficult:

1. **Type complexity**: `ReaderT<R, StateT<S, IO, A>>` is a mouthful
2. **Performance**: Each layer adds overhead
3. **Lift proliferation**: Need to lift through each layer
4. **Order matters**: `ReaderT<R, StateT<S, M, A>>` ≠ `StateT<S, ReaderT<R, M, A>>`

But they solve a real problem: composing effects.

## Alternatives to Transformers

### Effect Systems (Modern Approach)

Libraries like ZIO (Scala), Polysemy (Haskell), or Eff (various languages) provide effect systems that avoid transformer stacks:

```typescript
// Hypothetical effect system
type App<A> = Effect<Config | Database | Email, AppError, A>;

function createUser(name: string, email: string): App<User> {
  return Effect.gen(function* () {
    const config = yield* Effect.config<Config>();
    const user = yield* Effect.database.insert({ name, email });
    yield* Effect.email.sendWelcome(email);
    return user;
  });
}

// Run with handlers
const result = createUser("Alice", "alice@example.com")
  .provideConfig(config)
  .provideDatabase(db)
  .provideEmail(emailService)
  .runPromise();
```

This is the cutting edge of functional effect handling.

## What We've Learned

- Monads don't compose automatically
- Transformers stack monads into a single type
- MaybeT, EitherT, ReaderT, StateT are common transformers
- Lift operations between layers
- Transformers add complexity but solve real problems
- Modern effect systems are an alternative approach

## Coming Up

We've been writing a lot of nested `flatMap` calls. Languages have better syntax for this. Let's explore do-notation and its equivalents - how to write monadic code that looks imperative.

---

**Next: [Chapter 15 - Do Notation: Imperative-Looking Functional Code](15-do-notation.md)**
