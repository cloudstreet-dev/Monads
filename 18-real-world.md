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

**Note:** Java's `Optional` doesn't strictly follow the monad laws. It disallows `null` values inside (throws `NullPointerException`), which breaks some monad equivalences. For example, `Optional.of(null).flatMap(f)` throws rather than propagating empty. It's a practical choice for Java's ecosystem but technically not a true monad.

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

Hooks have monad-like properties:

**Caveat:** While React hooks share some conceptual similarities with monads (encapsulating state, effects, composability), they don't compose the same way true monads do. `useState` returns `[state, setState]`, not a monad with `flatMap`. Hooks follow React's Rules of Hooks rather than monad laws. The comparison is conceptually useful but shouldn't be taken literally.

```javascript
// useState is conceptually like State monad
function Counter() {
  const [count, setCount] = useState(0);  // State-like encapsulation

  const increment = () => setCount(count + 1);

  return <button onClick={increment}>{count}</button>;
}

// useEffect chains IO-like operations
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    // IO-like behavior: describing effects
    fetchUser(userId)
      .then(setUser)
      .catch(console.error);
  }, [userId]);

  return user ? <div>{user.name}</div> : <div>Loading...</div>;
}

// Custom hooks compose patterns similar to monads
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

// Thunks are IO-like: describing async effects
const fetchUserAndPosts = (id) => async (dispatch, getState) => {
  dispatch({ type: 'FETCH_USER_START' });

  try {
    const user = await api.fetchUser(id);
    dispatch({ type: 'FETCH_USER_SUCCESS', payload: user });

    // Now fetch posts for this user
    const posts = await api.fetchPosts(user.id);
    dispatch({ type: 'FETCH_POSTS_SUCCESS', payload: posts });
  } catch (error) {
    dispatch({ type: 'FETCH_FAILURE', error });
  }
};
```

## Kotlin

### Arrow

Functional programming library for Kotlin:

```kotlin
import arrow.core.*
import arrow.fx.coroutines.*

// Option (Maybe)
fun findUser(id: String): Option<User> =
    database.users.firstOrNull { it.id == id }.toOption()

fun getUserEmail(userId: String): Option<String> =
    findUser(userId)
        .flatMap { it.email.toOption() }
        .map { it.lowercase() }

// Either for errors
sealed class UserError {
    object NotFound : UserError()
    data class InvalidEmail(val email: String) : UserError()
}

fun validateUser(user: User): Either<UserError, ValidUser> =
    user.email.validateEmail()
        .flatMap { email ->
            user.age.validateAge()
                .map { age -> ValidUser(user.id, email, age) }
        }

// Validated for accumulating errors
import arrow.core.raise.either
import arrow.core.raise.zipOrAccumulate

fun validateUserData(data: UserData): Either<Nel<String>, ValidUser> = either {
    zipOrAccumulate(
        { validateName(data.name).bind() },
        { validateEmail(data.email).bind() },
        { validateAge(data.age).bind() }
    ) { name, email, age ->
        ValidUser(name, email, age)
    }
}

// Effect for async operations (built on coroutines)
suspend fun fetchUserData(id: String): Either<ApiError, UserData> = either {
    val user = fetchUser(id).bind()
    val posts = fetchPosts(user.id).bind()
    val comments = fetchComments(posts.first().id).bind()
    UserData(user, posts, comments)
}

// For-comprehension style with either block
suspend fun processUser(id: String): Either<AppError, Result> = either {
    val user = findUser(id).bind()
    val validated = validateUser(user).bind()
    val saved = saveUser(validated).bind()
    Result(saved)
}
```

**Key features:**
- Integrates with Kotlin coroutines
- Type-safe error handling with Either and Validated
- Option for null safety beyond Kotlin's built-in nullability
- Popular in Android development

## Swift

### Result Type

Built-in since Swift 5:

```swift
// Result is a built-in Either
enum UserError: Error {
    case notFound
    case invalidEmail
    case networkError(String)
}

func findUser(id: String) -> Result<User, UserError> {
    guard let user = database.users.first(where: { $0.id == id }) else {
        return .failure(.notFound)
    }
    return .success(user)
}

// Chain with flatMap
func getUserEmail(userId: String) -> Result<String, UserError> {
    findUser(id: userId)
        .flatMap { user in
            guard let email = user.email else {
                return .failure(.invalidEmail)
            }
            return .success(email.lowercased())
        }
}

// Compose operations
func processUser(id: String) -> Result<ProcessedUser, UserError> {
    findUser(id: id)
        .flatMap { user in validateUser(user) }
        .flatMap { validUser in saveUser(validUser) }
        .map { savedUser in ProcessedUser(savedUser) }
}
```

### Combine Framework

Reactive programming with Publishers (like RxJS):

```swift
import Combine

// Publisher is a monad over time
func fetchUser(id: String) -> AnyPublisher<User, Error> {
    URLSession.shared
        .dataTaskPublisher(for: URL(string: "/api/users/\(id)")!)
        .map(\.data)
        .decode(type: User.self, decoder: JSONDecoder())
        .eraseToAnyPublisher()
}

// Chain with flatMap
func loadUserData(id: String) -> AnyPublisher<UserData, Error> {
    fetchUser(id: id)
        .flatMap { user in
            fetchPosts(userId: user.id)
                .map { posts in UserData(user: user, posts: posts) }
        }
        .eraseToAnyPublisher()
}

// Combine independent operations
func loadDashboard(userId: String) -> AnyPublisher<Dashboard, Error> {
    Publishers.Zip(
        fetchUser(id: userId),
        fetchSettings(userId: userId)
    )
    .map { user, settings in
        Dashboard(user: user, settings: settings)
    }
    .eraseToAnyPublisher()
}

// Usage with async/await (Swift 5.5+)
let cancellable = loadUserData(id: "123")
    .sink(
        receiveCompletion: { completion in
            switch completion {
            case .finished:
                print("Done")
            case .failure(let error):
                print("Error: \(error)")
            }
        },
        receiveValue: { userData in
            print("User: \(userData.user.name)")
        }
    )
```

**Key features:**
- Result type built into language
- Combine for reactive streams
- Integration with Swift concurrency (async/await)
- Popular in iOS/macOS development

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
