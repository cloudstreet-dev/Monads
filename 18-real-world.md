# Chapter 18: Monads in the Wild

Let's look at how real libraries and frameworks use monads in production code.

## JavaScript/TypeScript

### fp-ts

The most comprehensive FP library for TypeScript:

```typescript
import * as O from 'fp-ts/Option';
import * as E from 'fp-ts/Either';
import * as TE from 'fp-ts/TaskEither';
import { pipe } from 'fp-ts/function';

// Option (Maybe)
const findUser = (id: string): O.Option<User> =>
  pipe(
    database.users.find(u => u.id === id),
    O.fromNullable
  );

// Either for errors
type AppError = { message: string; code: number };

const validateEmail = (email: string): E.Either<AppError, string> =>
  email.includes('@')
    ? E.right(email)
    : E.left({ message: 'Invalid email', code: 400 });

// TaskEither for async operations with errors
const fetchUser = (id: string): TE.TaskEither<AppError, User> =>
  TE.tryCatch(
    () => fetch(`/api/users/${id}`).then(r => r.json()),
    (error) => ({ message: String(error), code: 500 })
  );

// Compose them
const program = pipe(
  fetchUser('123'),
  TE.flatMap(user =>
    pipe(
      validateEmail(user.email),
      E.fold(
        error => TE.left(error),
        email => TE.right({ ...user, email })
      )
    )
  )
);

// Execute
program().then(
  E.fold(
    error => console.error(error),
    user => console.log(user)
  )
);
```

**Key features:**
- Full monad transformer stack
- Pipe-friendly API
- Comprehensive type safety
- Do-notation via generators

### RxJS (Observables)

Reactive programming with observables (list + async monad):

```typescript
import { Observable, of, from } from 'rxjs';
import { map, flatMap, filter, catchError } from 'rxjs/operators';

// Observable is a monad
const fetchUser$ = (id: string): Observable<User> =>
  from(fetch(`/api/users/${id}`).then(r => r.json()));

const program$ = pipe(
  fetchUser$('123'),
  flatMap(user => fetchPosts$(user.id)),  // flatMap!
  map(posts => posts.filter(p => p.published)),
  catchError(error => of([]))
);

// Subscribe to execute
program$.subscribe({
  next: posts => console.log('Posts:', posts),
  error: err => console.error('Error:', err),
  complete: () => console.log('Done')
});

// Combines List monad (multiple values) + Async monad (over time)
```

**Key features:**
- Streams as monads
- Powerful operators
- Time-based operations
- Backpressure handling

## Rust

### Option<T> and Result<T, E>

Built into the language:

```rust
// Option is a monad
fn find_user(id: &str) -> Option<User> {
    database.users.iter()
        .find(|u| u.id == id)
        .cloned()
}

// Chain with and_then (flatMap)
fn get_email(id: &str) -> Option<String> {
    find_user(id)
        .and_then(|user| user.email)
        .map(|email| email.to_lowercase())
}

// The ? operator is do-notation!
fn process_user(id: &str) -> Option<String> {
    let user = find_user(id)?;  // Early return if None
    let email = user.email?;
    Some(email.to_lowercase())
}

// Result for error handling
fn parse_config(path: &str) -> Result<Config, ConfigError> {
    let contents = fs::read_to_string(path)?;  // ? propagates errors
    let config: Config = serde_json::from_str(&contents)?;
    validate_config(&config)?;
    Ok(config)
}

// Combine Result and Option
fn load_user_setting(user_id: &str, key: &str) -> Result<Option<String>, DbError> {
    let user = find_user(user_id)?;
    Ok(user.and_then(|u| u.settings.get(key).cloned()))
}
```

**Key features:**
- Zero-cost abstractions
- Compile-time guarantees
- `?` operator for ergonomics
- No runtime overhead

### Futures (Async)

```rust
use futures::future::{self, Future};

async fn fetch_user(id: &str) -> Result<User, ApiError> {
    let response = reqwest::get(&format!("/api/users/{}", id)).await?;
    let user = response.json().await?;
    Ok(user)
}

// async/await is do-notation
async fn load_user_data(id: &str) -> Result<UserData, ApiError> {
    let user = fetch_user(id).await?;  // Monadic chain
    let posts = fetch_posts(&user.id).await?;
    let comments = fetch_comments(&posts[0].id).await?;

    Ok(UserData { user, posts, comments })
}
```

## Scala

### Standard Library

```scala
// Option built-in
def findUser(id: String): Option[User] =
  database.users.find(_.id == id)

// For-comprehension (do-notation)
def getUserPosts(userId: String): Option[List[Post]] = for {
  user <- findUser(userId)
  posts <- fetchPosts(user.id)
} yield posts

// Either for errors
def validateUser(user: User): Either[ValidationError, ValidUser] = for {
  email <- validateEmail(user.email)
  age <- validateAge(user.age)
} yield ValidUser(user.id, email, age)

// Future for async
import scala.concurrent.Future
import scala.concurrent.ExecutionContext.Implicits.global

def loadUserData(id: String): Future[UserData] = for {
  user <- fetchUser(id)
  posts <- fetchPosts(user.id)
  comments <- fetchComments(posts.head.id)
} yield UserData(user, posts, comments)
```

### Cats

Functional programming library:

```scala
import cats._
import cats.implicits._

// Option monad
val result: Option[Int] = for {
  a <- Some(5)
  b <- Some(10)
} yield a + b

// Validated for accumulating errors
import cats.data.Validated
import cats.data.ValidatedNel

def validateUser(user: User): ValidatedNel[String, ValidUser] = (
  validateEmail(user.email),
  validatePassword(user.password),
  validateAge(user.age)
).mapN(ValidUser)

// IO for effects
import cats.effect.IO

val program: IO[Unit] = for {
  _ <- IO(println("Enter your name:"))
  name <- IO(scala.io.StdIn.readLine())
  _ <- IO(println(s"Hello, $name!"))
} yield ()

program.unsafeRunSync()
```

### ZIO

Modern effect system:

```scala
import zio._

// ZIO[R, E, A] = requires R, fails with E, succeeds with A
type UserId = String

def findUser(id: UserId): ZIO[Database, DbError, User] =
  ZIO.serviceWithZIO[Database](_.findUser(id))

def sendEmail(user: User): ZIO[EmailService, EmailError, Unit] =
  ZIO.serviceWithZIO[EmailService](_.send(user.email, "Welcome!"))

// Compose effects
val program: ZIO[Database & EmailService, DbError | EmailError, Unit] = for {
  user <- findUser("123")
  _ <- sendEmail(user)
} yield ()

// Provide dependencies
val result = program.provide(
  DatabaseLive.layer,
  EmailServiceLive.layer
)
```

**Key features:**
- Built-in dependency injection
- Typed errors
- Concurrent & parallel primitives
- Resource safety

## Haskell

Everything is monadic in Haskell:

```haskell
import Control.Monad (when, unless)
import Data.Maybe (Maybe(..))
import Control.Monad.Reader
import Control.Monad.State
import Control.Monad.Except

-- Maybe monad
findUser :: String -> Maybe User
findUser id = lookup id users

getUserEmail :: String -> Maybe String
getUserEmail userId = do
  user <- findUser userId
  email <- userEmail user
  return email

-- Either for errors
validateUser :: User -> Either ValidationError ValidUser
validateUser user = do
  email <- validateEmail (userEmail user)
  age <- validateAge (userAge user)
  return $ ValidUser (userId user) email age

-- IO monad
main :: IO ()
main = do
  putStrLn "Enter your name:"
  name <- getLine
  putStrLn $ "Hello, " ++ name ++ "!"

-- Reader for configuration
type App = ReaderT Config IO

getApiUrl :: App String
getApiUrl = asks configApiUrl

fetchUser :: UserId -> App User
fetchUser userId = do
  apiUrl <- getApiUrl
  liftIO $ httpGet (apiUrl ++ "/users/" ++ userId)

-- Monad transformers
type AppM = ReaderT Config (ExceptT AppError IO)

runApp :: AppM a -> Config -> IO (Either AppError a)
runApp app config = runExceptT (runReaderT app config)
```

## F#

### Computation Expressions

```fsharp
// Option monad
let getUserEmail userId = option {
    let! user = findUser userId
    let! email = user.Email
    return email.ToLower()
}

// Result monad
type Result<'T,'TError> = Ok of 'T | Error of 'TError

let validateUser user = result {
    let! email = validateEmail user.Email
    let! password = validatePassword user.Password
    let! age = validateAge user.Age
    return { Email = email; Password = password; Age = age }
}

// Async monad
let loadUserData userId = async {
    let! user = fetchUser userId
    let! posts = fetchPosts user.Id
    let! comments = fetchComments posts.[0].Id
    return { User = user; Posts = posts; Comments = comments }
}

// List monad (comprehensions)
let pairs = [
    for x in [1; 2; 3] do
    for y in ['a'; 'b'] do
    yield (x, y)
]
```

### FsToolkit.ErrorHandling

```fsharp
open FsToolkit.ErrorHandling

// TaskResult (async + result)
let fetchUserSafely userId: TaskResult<User, ApiError> = taskResult {
    let! response = httpGet $"/api/users/{userId}"
    let! user = parseJson<User> response
    do! validateUser user
    return user
}

// Validation (accumulate errors)
let validateUserData data = validation {
    let! email = validateEmail data.Email
    and! password = validatePassword data.Password
    and! age = validateAge data.Age
    return { Email = email; Password = password; Age = age }
}
```

## Java

### Optional<T>

Built-in monad (kind of):

```java
Optional<User> findUser(String id) {
    return Optional.ofNullable(database.findUser(id));
}

Optional<String> getUserEmail(String userId) {
    return findUser(userId)
        .flatMap(user -> Optional.ofNullable(user.getEmail()))
        .map(String::toLowerCase);
}

// With streams
List<String> emails = users.stream()
    .map(User::getEmail)
    .flatMap(Optional::stream)  // Filter out empties
    .collect(Collectors.toList());
```

### CompletableFuture

Async monad:

```java
CompletableFuture<UserData> loadUserData(String userId) {
    return fetchUser(userId)
        .thenCompose(user -> fetchPosts(user.getId()))  // flatMap!
        .thenCompose(posts -> fetchComments(posts.get(0).getId()))
        .thenApply(comments -> new UserData(user, posts, comments));
}

// Handle errors
CompletableFuture<User> fetchUserSafe(String id) {
    return fetchUser(id)
        .exceptionally(error -> {
            logger.error("Failed to fetch user", error);
            return DEFAULT_USER;
        });
}
```

### Vavr

Functional library for Java:

```java
import io.vavr.control.Option;
import io.vavr.control.Either;
import io.vavr.control.Try;

// Option monad
Option<User> findUser(String id) {
    return Option.of(database.findUser(id));
}

// Either for errors
Either<ValidationError, ValidUser> validateUser(User user) {
    return validateEmail(user.getEmail())
        .flatMap(email -> validatePassword(user.getPassword())
            .map(password -> new ValidUser(user.getId(), email, password)));
}

// Try for exception handling
Try<Config> loadConfig(String path) {
    return Try.of(() -> readFile(path))
        .flatMap(contents -> Try.of(() -> parseJson(contents)));
}
```

## React (JavaScript)

Hooks are secretly monadic:

```javascript
// useState is like State monad
function Counter() {
  const [count, setCount] = useState(0);  // State<number>

  const increment = () => setCount(count + 1);

  return <button onClick={increment}>{count}</button>;
}

// useEffect chains IO operations
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    // IO monad-like behavior
    fetchUser(userId)
      .then(setUser)
      .catch(console.error);
  }, [userId]);

  return user ? <div>{user.name}</div> : <div>Loading...</div>;
}

// Custom hooks compose monadic patterns
function useRemoteData(fetchFn) {
  const [data, setData] = useState({ status: 'idle' });

  const load = () => {
    setData({ status: 'loading' });
    fetchFn()
      .then(result => setData({ status: 'success', data: result }))
      .catch(error => setData({ status: 'error', error }));
  };

  return [data, load];
}
```

## Redux (State Management)

Redux is a State monad:

```javascript
// Reducer: State -> Action -> State
const counterReducer = (state = 0, action) => {
  switch (action.type) {
    case 'INCREMENT':
      return state + 1;
    case 'DECREMENT':
      return state - 1;
    default:
      return state;
  }
};

// Compose reducers (monad composition)
const rootReducer = combineReducers({
  counter: counterReducer,
  user: userReducer,
  posts: postsReducer
});

// Thunks are like IO monad
const fetchUser = (id) => async (dispatch, getState) => {
  dispatch({ type: 'FETCH_USER_START' });

  try {
    const user = await api.fetchUser(id);
    dispatch({ type: 'FETCH_USER_SUCCESS', payload: user });
  } catch (error) {
    dispatch({ type: 'FETCH_USER_FAILURE', error });
  }
};
```

## Elm

Pure functional language for web:

```elm
-- Maybe built-in
findUser : String -> Maybe User
findUser id =
    List.head (List.filter (\u -> u.id == id) users)

-- Chain with andThen (flatMap)
getUserEmail : String -> Maybe String
getUserEmail userId =
    findUser userId
        |> Maybe.andThen .email
        |> Maybe.map String.toLower

-- Result for errors
type alias ValidationError = String

validateUser : User -> Result ValidationError ValidUser
validateUser user =
    Result.map2 ValidUser
        (validateEmail user.email)
        (validateAge user.age)

-- Cmd for effects (like IO)
fetchUser : String -> Cmd Msg
fetchUser id =
    Http.get
        { url = "/api/users/" ++ id
        , expect = Http.expectJson GotUser userDecoder
        }
```

## Common Patterns Across Libraries

### 1. Naming Conventions

- **flatMap**: Haskell `>>=`, Scala `flatMap`, Rust `and_then`, Java `flatMap`
- **map**: Universal
- **of/pure/return**: Create from value

### 2. Error Handling

- **Either/Result**: Scala `Either`, Rust `Result`, Haskell `Either`
- **Try**: Scala `Try`, Vavr `Try`
- **Validation**: Cats `Validated`, F# `Validation`

### 3. Async Operations

- **Promise**: JavaScript
- **Future**: Scala, Rust
- **Task**: Haskell, fp-ts
- **IO**: Haskell, Cats Effect, ZIO

### 4. Optional Values

- **Maybe/Option**: Haskell `Maybe`, Scala `Option`, Rust `Option`, fp-ts `Option`
- **Optional**: Java `Optional`

## What We've Learned

- Monads appear in most modern languages
- Naming varies but the pattern is consistent
- Production libraries rely heavily on monads
- They're used for errors, async, state, effects
- You've probably been using them without knowing
- The pattern transcends language boundaries

## Coming Up

We've focused on monads, but they're part of a bigger family. Let's zoom out and see where monads fit in the functional programming hierarchy.

---

**Next: [Chapter 19 - Beyond Monads](19-beyond.md)**
