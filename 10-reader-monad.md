# Chapter 10: Reader - Dependency Injection, Functionally

Ever written code like this?

```typescript
const API_URL = "https://api.example.com";
const API_TIMEOUT = 5000;
const DEBUG_MODE = true;

function fetchUser(id: string): Promise<User> {
  // Implicitly depends on global config
  return fetch(`${API_URL}/users/${id}`, { timeout: API_TIMEOUT })
    .then(r => r.json());
}
```

Global configuration is convenient but problematic:
- Hard to test (have to mock globals)
- Hard to change (what if you need different configs?)
- Hidden dependencies (where does `API_URL` come from?)
- Not composable (can't easily switch configs)

The Reader monad solves this: **dependency injection without the ceremony**.

## The Core Idea

Instead of accessing globals:

```typescript
function doSomething(): Result {
  return computeWith(GLOBAL_CONFIG);
}
```

Make dependencies explicit:

```typescript
function doSomething(config: Config): Result {
  return computeWith(config);
}
```

Reader automates threading configuration through function calls.

## Building Reader

```typescript
// Reader<R, A> represents:
// A function that, given environment R, produces value A
class Reader<R, A> {
  constructor(private runReader: (env: R) => A) {}

  // Wrap a pure value (ignores environment)
  static of<R, A>(value: A): Reader<R, A> {
    return new Reader(() => value);
  }

  // Access the environment
  static ask<R>(): Reader<R, R> {
    return new Reader(env => env);
  }

  // Run with a specific environment value
  static asks<R, A>(selector: (env: R) => A): Reader<R, A> {
    return new Reader(selector);
  }

  // Transform the value
  map<B>(fn: (value: A) => B): Reader<R, B> {
    return new Reader(env => {
      const value = this.runReader(env);
      return fn(value);
    });
  }

  // Chain reader computations
  flatMap<B>(fn: (value: A) => Reader<R, B>): Reader<R, B> {
    return new Reader(env => {
      const value = this.runReader(env);
      const nextReader = fn(value);
      return nextReader.runReader(env);
    });
  }

  // Run the computation with an environment
  run(env: R): A {
    return this.runReader(env);
  }

  // Transform the environment before running
  local<R2>(fn: (env: R2) => R): Reader<R2, A> {
    return new Reader(env2 => this.runReader(fn(env2)));
  }
}
```

## Simple Example: Configuration

```typescript
interface Config {
  apiUrl: string;
  apiTimeout: number;
  debugMode: boolean;
}

// Functions that need config
function getApiUrl(): Reader<Config, string> {
  return Reader.asks(config => config.apiUrl);
}

function getTimeout(): Reader<Config, number> {
  return Reader.asks(config => config.apiTimeout);
}

function isDebugMode(): Reader<Config, boolean> {
  return Reader.asks(config => config.debugMode);
}

// Build a URL with config
function buildUrl(endpoint: string): Reader<Config, string> {
  return getApiUrl().map(baseUrl => `${baseUrl}${endpoint}`);
}

// Fetch with config (conceptual - returns description)
function fetchUser(id: string): Reader<Config, Promise<User>> {
  return buildUrl(`/users/${id}`).flatMap(url =>
    getTimeout().map(timeout =>
      fetch(url, { timeout }).then(r => r.json())
    )
  );
}

// Compose operations that need config
const program = fetchUser("123").flatMap(userPromise =>
  isDebugMode().map(debug => {
    if (debug) {
      console.log("Fetching user 123...");
    }
    return userPromise;
  })
);

// Run with actual config
const config: Config = {
  apiUrl: "https://api.example.com",
  apiTimeout: 5000,
  debugMode: true
};

const userPromise = program.run(config);
```

No globals! Configuration is passed implicitly through the Reader chain.

## Real Example: Multi-Environment App

```typescript
interface AppConfig {
  database: {
    host: string;
    port: number;
    name: string;
  };
  api: {
    baseUrl: string;
    timeout: number;
  };
  features: {
    enableNewUI: boolean;
    enableBetaFeatures: boolean;
  };
}

// Environment-aware functions
function getDatabaseUrl(): Reader<AppConfig, string> {
  return Reader.asks(config =>
    `postgresql://${config.database.host}:${config.database.port}/${config.database.name}`
  );
}

function isFeatureEnabled(feature: keyof AppConfig['features']): Reader<AppConfig, boolean> {
  return Reader.asks(config => config.features[feature]);
}

// Business logic
function createUser(name: string, email: string): Reader<AppConfig, Promise<User>> {
  return getDatabaseUrl().flatMap(dbUrl =>
    isFeatureEnabled('enableBetaFeatures').map(betaEnabled => {
      const user = {
        name,
        email,
        betaTester: betaEnabled
      };

      return db.connect(dbUrl).then(conn =>
        conn.query('INSERT INTO users VALUES ($1, $2, $3)', [
          user.name,
          user.email,
          user.betaTester
        ])
      );
    })
  );
}

function renderUI(): Reader<AppConfig, JSX.Element> {
  return isFeatureEnabled('enableNewUI').map(newUI =>
    newUI ? <NewUserInterface /> : <LegacyUserInterface />
  );
}

// Different configs for different environments
const devConfig: AppConfig = {
  database: { host: 'localhost', port: 5432, name: 'app_dev' },
  api: { baseUrl: 'http://localhost:3000', timeout: 10000 },
  features: { enableNewUI: true, enableBetaFeatures: true }
};

const prodConfig: AppConfig = {
  database: { host: 'db.example.com', port: 5432, name: 'app_prod' },
  api: { baseUrl: 'https://api.example.com', timeout: 5000 },
  features: { enableNewUI: false, enableBetaFeatures: false }
};

// Run the same code with different configs
const devUser = createUser("Alice", "alice@example.com").run(devConfig);
const prodUser = createUser("Alice", "alice@example.com").run(prodConfig);
```

## Dependency Injection Without the Framework

Reader is lightweight DI:

```typescript
// Traditional DI (Spring-style)
@Injectable()
class UserService {
  constructor(
    @Inject('DatabaseService') private db: DatabaseService,
    @Inject('EmailService') private email: EmailService,
    @Inject('Config') private config: Config
  ) {}

  async createUser(name: string): Promise<User> {
    const user = await this.db.save({ name });
    await this.email.sendWelcome(user.email);
    return user;
  }
}

// Reader-based DI
interface Services {
  database: DatabaseService;
  email: EmailService;
  config: Config;
}

function createUser(name: string): Reader<Services, Promise<User>> {
  return Reader.ask<Services>().flatMap(services =>
    Reader.of(
      services.database.save({ name }).then(user =>
        services.email.sendWelcome(user.email).then(() => user)
      )
    )
  );
}

// No annotations, no container, just functions
const services: Services = {
  database: new DatabaseService(),
  email: new EmailService(),
  config: loadConfig()
};

const user = createUser("Alice").run(services);
```

## Local Environment Changes

Sometimes you need to temporarily modify the environment:

```typescript
interface Context {
  userId: string;
  permissions: string[];
  locale: string;
}

function getCurrentUserId(): Reader<Context, string> {
  return Reader.asks(ctx => ctx.userId);
}

function hasPermission(perm: string): Reader<Context, boolean> {
  return Reader.asks(ctx => ctx.permissions.includes(perm));
}

// Run with modified context
function asUser(userId: string): <A>(reader: Reader<Context, A>) => Reader<Context, A> {
  return reader => reader.local(ctx => ({ ...ctx, userId }));
}

function withPermissions(perms: string[]): <A>(reader: Reader<Context, A>) => Reader<Context, A> {
  return reader => reader.local(ctx => ({
    ...ctx,
    permissions: [...ctx.permissions, ...perms]
  }));
}

// Usage
const checkAccess = hasPermission('admin').flatMap(isAdmin =>
  getCurrentUserId().map(id => ({ id, isAdmin }))
);

const normalContext: Context = {
  userId: 'user123',
  permissions: ['read'],
  locale: 'en-US'
};

const normalResult = checkAccess.run(normalContext);
// { id: 'user123', isAdmin: false }

const adminResult = withPermissions(['admin'])(checkAccess).run(normalContext);
// { id: 'user123', isAdmin: true }

const otherUserResult = asUser('user456')(checkAccess).run(normalContext);
// { id: 'user456', isAdmin: false }
```

## Testing with Reader

Reader makes testing trivial - just supply test config:

```typescript
// Production code
function sendNotification(userId: string, message: string): Reader<Services, Promise<void>> {
  return Reader.ask<Services>().flatMap(services =>
    Reader.of(
      services.database.findUser(userId).then(user =>
        services.email.send(user.email, message)
      )
    )
  );
}

// Test with mocks
describe('sendNotification', () => {
  it('sends email to user', async () => {
    const mockServices: Services = {
      database: {
        findUser: jest.fn().mockResolvedValue({ email: 'test@example.com' })
      },
      email: {
        send: jest.fn().mockResolvedValue(undefined)
      },
      config: { /* test config */ }
    };

    await sendNotification('user123', 'Hello!').run(mockServices);

    expect(mockServices.database.findUser).toHaveBeenCalledWith('user123');
    expect(mockServices.email.send).toHaveBeenCalledWith(
      'test@example.com',
      'Hello!'
    );
  });
});
```

## Reader in Other Languages

### Haskell

```haskell
import Control.Monad.Reader

type Config = String  -- Simplified

-- Functions that need config
getConfig :: Reader Config String
getConfig = ask

useConfig :: String -> Reader Config String
useConfig prefix = do
  config <- ask
  return (prefix ++ ": " ++ config)

processData :: Reader Config String
processData = do
  config <- ask
  let processed = "Processing with " ++ config
  return processed

-- Compose and run
program :: Reader Config String
program = do
  result1 <- processData
  result2 <- useConfig "Result"
  return (result1 ++ " | " ++ result2)

main :: IO ()
main = do
  let result = runReader program "MyConfig"
  putStrLn result
  -- Output: "Processing with MyConfig | Result: MyConfig"
```

### Scala

```scala
case class Reader[R, A](run: R => A) {
  def map[B](f: A => B): Reader[R, B] =
    Reader(r => f(run(r)))

  def flatMap[B](f: A => Reader[R, B]): Reader[R, B] =
    Reader(r => f(run(r)).run(r))
}

object Reader {
  def pure[R, A](a: A): Reader[R, A] =
    Reader(_ => a)

  def ask[R]: Reader[R, R] =
    Reader(identity)

  def asks[R, A](f: R => A): Reader[R, A] =
    Reader(f)
}

// Usage
case class Config(apiUrl: String, timeout: Int)

def getApiUrl: Reader[Config, String] =
  Reader.asks(_.apiUrl)

def getTimeout: Reader[Config, Int] =
  Reader.asks(_.timeout)

def buildRequest(endpoint: String): Reader[Config, String] = for {
  url <- getApiUrl
  timeout <- getTimeout
} yield s"GET $url$endpoint (timeout: ${timeout}ms)"

val config = Config("https://api.example.com", 5000)
val request = buildRequest("/users").run(config)
// "GET https://api.example.com/users (timeout: 5000ms)"
```

### F#

```fsharp
type Reader<'R, 'A> = Reader of ('R -> 'A)

module Reader =
    let run (Reader f) env = f env

    let pure x = Reader (fun _ -> x)

    let ask<'R> = Reader id

    let asks f = Reader f

    let bind f (Reader run1) =
        Reader (fun env ->
            let a = run1 env
            let (Reader run2) = f a
            run2 env
        )

    let map f m = bind (f >> pure) m

// Usage
type Config = { ApiUrl: string; Timeout: int }

let getApiUrl = Reader.asks (fun (c: Config) -> c.ApiUrl)
let getTimeout = Reader.asks (fun (c: Config) -> c.Timeout)

let buildRequest endpoint = reader {
    let! url = getApiUrl
    let! timeout = getTimeout
    return sprintf "GET %s%s (timeout: %dms)" url endpoint timeout
}

let config = { ApiUrl = "https://api.example.com"; Timeout = 5000 }
let request = Reader.run (buildRequest "/users") config
```

## Combining Reader with Other Monads

Reader often needs to work with IO, Maybe, Either, etc:

```typescript
// Reader that returns Maybe
function findUser(id: string): Reader<Services, Maybe<User>> {
  return Reader.ask<Services>().map(services =>
    Maybe.fromNullable(services.database.findUser(id))
  );
}

// Reader that returns Either
function validateUser(id: string): Reader<Services, Either<Error, User>> {
  return Reader.ask<Services>().map(services => {
    const user = services.database.findUser(id);
    return user
      ? Either.right(user)
      : Either.left(new Error('User not found'));
  });
}

// Reader that returns IO
function saveUser(user: User): Reader<Services, IO<void>> {
  return Reader.ask<Services>().map(services =>
    new IO(() => services.database.save(user))
  );
}

// This nesting gets awkward - we'll fix it with Monad Transformers (Chapter 14)
```

## Real-World Pattern: Three-Layer Architecture

```typescript
// 1. Infrastructure layer (concrete implementations)
interface Database {
  query<T>(sql: string, params: any[]): Promise<T[]>;
}

interface Logger {
  info(message: string): void;
  error(message: string, error: Error): void;
}

// 2. Application layer (business logic with Reader)
interface AppServices {
  db: Database;
  logger: Logger;
}

function findUserById(id: string): Reader<AppServices, Promise<User | null>> {
  return Reader.ask<AppServices>().map(services => {
    services.logger.info(`Finding user ${id}`);
    return services.db
      .query<User>('SELECT * FROM users WHERE id = $1', [id])
      .then(users => users[0] || null)
      .catch(error => {
        services.logger.error('Database error', error);
        return null;
      });
  });
}

function createOrder(
  userId: string,
  items: OrderItem[]
): Reader<AppServices, Promise<Order>> {
  return Reader.ask<AppServices>().flatMap(services =>
    Reader.of(
      findUserById(userId).run(services).then(user => {
        if (!user) throw new Error('User not found');

        services.logger.info(`Creating order for user ${userId}`);
        return services.db.query<Order>(
          'INSERT INTO orders (user_id, items) VALUES ($1, $2) RETURNING *',
          [userId, JSON.stringify(items)]
        ).then(orders => orders[0]);
      })
    )
  );
}

// 3. Presentation layer (inject dependencies)
const services: AppServices = {
  db: new PostgresDatabase(),
  logger: new ConsoleLogger()
};

// All business logic is pure Reader computations
const order = createOrder('user123', [{ id: 'item1', qty: 2 }]).run(services);
```

## When to Use Reader

**Use Reader when:**
- Multiple functions need the same configuration
- You want to avoid globals
- You need different configs (dev/test/prod)
- Dependencies are read-only
- You want lightweight dependency injection

**Don't use Reader when:**
- Only one function needs the config (just pass it as a parameter)
- Configuration changes during execution (use State instead)
- You have complex DI needs (might want a DI framework)

## What We've Learned

- Reader threads read-only environment through computations
- Makes dependencies explicit without manual passing
- Perfect for configuration and dependency injection
- Easy to test with mock services
- Composes naturally with other monads
- Lightweight alternative to DI frameworks

## Coming Up

Reader reads from environment, State reads and writes state. But what about *write-only* state? What if you want to accumulate values (like logs) as you compute? That's the Writer monad.

---

**Next: [Chapter 11 - Writer: Logging Along the Way](11-writer-monad.md)**
