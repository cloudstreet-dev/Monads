# Chapter 15: Do Notation - Imperative-Looking Functional Code

We've been writing a lot of this:

```typescript
findUser(id)
  .flatMap(user =>
    getProfile(user.id)
      .flatMap(profile =>
        getEmail(profile)
          .map(email =>
            email.toLowerCase()
          )
      )
  );
```

It works, but it's... not beautiful. The nesting grows rightward. Naming intermediate values is awkward.

**Do-notation** (and its equivalents) lets you write monadic code that looks imperative:

```haskell
do
  user <- findUser id
  profile <- getProfile (userId user)
  email <- getEmail profile
  return (map toLower email)
```

Same meaning, clearer syntax.

## The Problem with Nested flatMap

```typescript
// Get user's latest post's first comment
getUser("123")
  .flatMap(user =>
    getPosts(user.id)
      .flatMap(posts =>
        posts.length > 0
          ? getComments(posts[0].id)
              .flatMap(comments =>
                comments.length > 0
                  ? Maybe.of(comments[0].text)
                  : Maybe.none()
              )
          : Maybe.none()
      )
  );
```

Problems:
- Rightward drift
- Callback hell (again!)
- Hard to read the flow
- Intermediate names get awkward

## Haskell's Do-Notation

Haskell invented do-notation as sugar for flatMap chains:

```haskell
-- With do-notation
getUserPost :: String -> Maybe Comment
getUserPost userId = do
  user <- findUser userId
  posts <- getPosts (userId user)
  guard (not $ null posts)
  comments <- getComments (postId $ head posts)
  guard (not $ null comments)
  return (head comments)

-- Desugars to:
getUserPost userId =
  findUser userId >>= \user ->
  getPosts (userId user) >>= \posts ->
  guard (not $ null posts) >>
  getComments (postId $ head posts) >>= \comments ->
  guard (not $ null comments) >>
  return (head comments)
```

The `<-` arrow unwraps the monadic value. Regular `let` bindings are for non-monadic values.

```haskell
do
  x <- monadicFunction   -- x has type A from m A
  let y = pureFunction x -- y is just a value
  z <- anotherMonadic y  -- z has type B from m B
  return z
```

## Scala's For-Comprehension

Scala's for-comprehension is do-notation:

```scala
// With for-comprehension
def getUserPost(userId: String): Option[Comment] = for {
  user <- findUser(userId)
  posts <- getPosts(user.id)
  if posts.nonEmpty  // guard
  comments <- getComments(posts.head.id)
  if comments.nonEmpty
} yield comments.head

// Desugars to:
def getUserPost(userId: String): Option[Comment] =
  findUser(userId).flatMap { user =>
    getPosts(user.id).flatMap { posts =>
      if (posts.nonEmpty) {
        getComments(posts.head.id).flatMap { comments =>
          if (comments.nonEmpty) {
            Some(comments.head)
          } else {
            None
          }
        }
      } else {
        None
      }
    }
  }
```

Works with any type that has `map`, `flatMap`, and optionally `withFilter`:

```scala
// Works with Option
for {
  x <- Some(5)
  y <- Some(10)
} yield x + y  // Some(15)

// Works with List
for {
  x <- List(1, 2, 3)
  y <- List(10, 20)
} yield x + y  // List(11, 21, 12, 22, 13, 23)

// Works with Either
for {
  x <- Right(5)
  y <- Right(10)
} yield x + y  // Right(15)

// Works with Future
for {
  user <- fetchUser(id)
  posts <- fetchPosts(user.id)
} yield (user, posts)  // Future[(User, List[Post])]
```

## F#'s Computation Expressions

F# generalizes this to computation expressions:

```fsharp
// Maybe/Option
let getUserPost userId = maybe {
    let! user = findUser userId
    let! posts = getPosts user.id
    do! guard (not (List.isEmpty posts))
    let! comments = getComments posts.Head.id
    do! guard (not (List.isEmpty comments))
    return comments.Head
}

// Async
let fetchData userId = async {
    let! user = fetchUser userId
    let! posts = fetchPosts user.id
    return (user, posts)
}

// Result/Either
let validateUser user = result {
    let! email = validateEmail user.email
    let! password = validatePassword user.password
    let! age = validateAge user.age
    return { user with email = email; password = password; age = age }
}
```

The `let!` operator unwraps monadic values, regular `let` is for pure values.

## JavaScript's Async/Await

`async/await` is do-notation for Promises!

```javascript
// With async/await (do-notation)
async function getUserPost(userId) {
  const user = await findUser(userId);      // user <- findUser userId
  const posts = await getPosts(user.id);    // posts <- getPosts (userId user)
  if (posts.length === 0) return null;       // guard
  const comments = await getComments(posts[0].id);
  if (comments.length === 0) return null;
  return comments[0];                        // return (head comments)
}

// Desugars to:
function getUserPost(userId) {
  return findUser(userId).then(user =>
    getPosts(user.id).then(posts => {
      if (posts.length === 0) return Promise.resolve(null);
      return getComments(posts[0].id).then(comments => {
        if (comments.length === 0) return Promise.resolve(null);
        return Promise.resolve(comments[0]);
      });
    })
  );
}
```

`await` = `<-` in Haskell = `let!` in F# = `<-` in Scala's for.

**Important observation**: JavaScript specialized do-notation for one monad (Promise). This is why JavaScript doesn't have general do-notation like Haskell - `async/await` only works with Promises. Languages like Haskell, Scala, and F# provide do-notation for *all* monads, not just one specific type. This is more flexible but requires more language support.

## The Pattern

All do-notations share the same structure. Here's a side-by-side comparison:

| Operation | Haskell | Scala | F# | JavaScript |
|-----------|---------|-------|-----|------------|
| **Bind monadic value** | `x <- action` | `x <- action` | `let! x = action` | `const x = await action` |
| **Let pure value** | `let y = expr` | `val y = expr` | `let y = expr` | `const y = expr` |
| **Guard/filter** | `guard condition` | `if condition` | `do! guard condition` | `if (!condition) throw/return` |
| **Return final value** | `return x` | `yield x` | `return x` | `return x` |

Despite different syntax, the structure is identical across all languages.

## Desugaring Rules

### Simple Sequence

```haskell
-- Haskell
do
  x <- action1
  action2

-- Desugars to:
action1 >>= \x -> action2
```

```scala
// Scala
for {
  x <- action1
  result <- action2
} yield result

// Desugars to:
action1.flatMap(x => action2)
```

### Multiple Binds

```haskell
-- Haskell
do
  x <- action1
  y <- action2
  return (x + y)

-- Desugars to:
action1 >>= \x ->
  action2 >>= \y ->
    return (x + y)
```

### Let Bindings

```haskell
-- Haskell
do
  x <- action1
  let y = f x
  action2 y

-- Desugars to:
action1 >>= \x ->
  let y = f x in
    action2 y
```

### Pattern Matching

```haskell
-- Haskell
do
  Just x <- action1
  return x

-- Desugars to:
action1 >>= \case
  Just x -> return x
  Nothing -> fail "Pattern match failed"
```

## Implementing Do-Notation in TypeScript

TypeScript doesn't have built-in do-notation, but we can hack something similar using generators:

```typescript
// Helper to run generator as monadic do-notation
// How it works:
// 1. yield* passes monadic values to the generator
// 2. We extract those values and pass them through flatMap
// 3. The result of flatMap is fed back into the generator via next()
function Do<M>(generator: () => Generator<M, any, any>): M {
  const gen = generator();
  let result = gen.next();

  function step(value?: any): M {
    if (result.done) {
      return result.value;
    }

    const monad = result.value;
    return monad.flatMap((val: any) => {
      result = gen.next(val);  // Feed unwrapped value back to generator
      return step(val);
    });
  }

  return step();
}

// Usage with Maybe
const getUserPost = (userId: string) => Do(function* () {
  const user = yield* findUser(userId);
  const posts = yield* getPosts(user.id);

  if (posts.length === 0) return Maybe.none();

  const comments = yield* getComments(posts[0].id);

  if (comments.length === 0) return Maybe.none();

  return Maybe.of(comments[0]);
});
```

**Important caveats about this approach:**
- TypeScript's type system doesn't fully understand this pattern - you lose type safety
- The `any` types are necessary because generators weren't designed for this
- It's a clever hack, not a first-class language feature
- Libraries like `fp-ts` provide better-typed versions using generator comprehensions
- For production code, consider using a library's implementation rather than rolling your own

This demonstrates the concept, but JavaScript's lack of general do-notation is a real limitation compared to Haskell or Scala.

## Real Example: Database Transaction

### Haskell

```haskell
createOrder :: UserId -> [Item] -> DB Order
createOrder userId items = do
  -- Begin transaction
  beginTransaction

  -- Check user exists
  user <- findUser userId
  guard (isActive user)

  -- Check inventory
  forM_ items $ \item -> do
    stock <- getStock (itemId item)
    guard (stock >= itemQuantity item)

  -- Create order
  orderId <- insertOrder userId

  -- Add items to order
  forM_ items $ \item -> do
    insertOrderItem orderId item
    decrementStock (itemId item) (itemQuantity item)

  -- Commit
  commitTransaction

  -- Return order
  loadOrder orderId
```

### Scala

```scala
def createOrder(userId: String, items: List[Item]): DB[Order] = for {
  // Begin transaction
  _ <- beginTransaction

  // Check user exists
  user <- findUser(userId)
  _ <- if (user.isActive) DB.pure(()) else DB.fail("User inactive")

  // Check inventory
  _ <- items.traverse { item =>
    for {
      stock <- getStock(item.id)
      _ <- if (stock >= item.quantity) DB.pure(()) else DB.fail("Out of stock")
    } yield ()
  }

  // Create order
  orderId <- insertOrder(userId)

  // Add items
  _ <- items.traverse { item =>
    for {
      _ <- insertOrderItem(orderId, item)
      _ <- decrementStock(item.id, item.quantity)
    } yield ()
  }

  // Commit
  _ <- commitTransaction

  // Return order
  order <- loadOrder(orderId)
} yield order
```

### F#

```fsharp
let createOrder userId items = db {
    // Begin transaction
    do! beginTransaction

    // Check user exists
    let! user = findUser userId
    do! guard user.IsActive

    // Check inventory
    for item in items do
        let! stock = getStock item.Id
        do! guard (stock >= item.Quantity)

    // Create order
    let! orderId = insertOrder userId

    // Add items
    for item in items do
        do! insertOrderItem orderId item
        do! decrementStock item.Id item.Quantity

    // Commit
    do! commitTransaction

    // Return order
    let! order = loadOrder orderId
    return order
}
```

### JavaScript (async/await)

```javascript
async function createOrder(userId, items) {
  // Begin transaction
  await beginTransaction();

  // Check user exists
  const user = await findUser(userId);
  if (!user.isActive) throw new Error('User inactive');

  // Check inventory
  for (const item of items) {
    const stock = await getStock(item.id);
    if (stock < item.quantity) throw new Error('Out of stock');
  }

  // Create order
  const orderId = await insertOrder(userId);

  // Add items
  for (const item of items) {
    await insertOrderItem(orderId, item);
    await decrementStock(item.id, item.quantity);
  }

  // Commit
  await commitTransaction();

  // Return order
  return await loadOrder(orderId);
}
```

Notice how similar they all are! Do-notation makes monadic code look imperative.

## Benefits

### 1. Readability

```scala
// Without do-notation
findUser(id).flatMap(user =>
  getProfile(user.id).flatMap(profile =>
    getEmail(profile).map(email =>
      email.toLowerCase
    )
  )
)

// With do-notation
for {
  user <- findUser(id)
  profile <- getProfile(user.id)
  email <- getEmail(profile)
} yield email.toLowerCase
```

The second version reads top-to-bottom, like procedural code.

### 2. Variable Scoping

```scala
// Without: variables only available in inner scope
findUser(id).flatMap(user =>         // user available here
  getProfile(user.id).flatMap(profile =>  // user and profile available
    // Can use both user and profile
    combineInfo(user, profile)
  )
)

// With: all bound variables in scope
for {
  user <- findUser(id)        // user now in scope
  profile <- getProfile(user.id)  // user and profile in scope
  // Can use both easily
} yield combineInfo(user, profile)
```

### 3. Error Handling

```haskell
-- Either monad with do-notation
validateUser :: User -> Either Error ValidUser
validateUser user = do
  email <- validateEmail user.email
  password <- validatePassword user.password
  age <- validateAge user.age
  return $ ValidUser user email password age

-- Errors propagate automatically - clean!
```

## Common Patterns

### Optional Chaining

```scala
// Get deeply nested optional value
val street: Option[String] = for {
  user <- findUser(id)
  address <- user.address
  street <- address.street
} yield street

// vs null checks:
val street = {
  val user = findUser(id)
  if (user != null) {
    val address = user.address
    if (address != null) {
      address.street
    } else null
  } else null
}
```

### List Comprehension

```haskell
-- Generate all pairs
pairs :: [Int] -> [Char] -> [(Int, Char)]
pairs nums chars = do
  n <- nums
  c <- chars
  return (n, c)

-- Same as:
[(n, c) | n <- nums, c <- chars]
```

### Sequential Operations

```javascript
// Fetch data sequentially
async function loadPage(userId) {
  const user = await fetchUser(userId);
  const posts = await fetchPosts(user.id);
  const comments = await fetchComments(posts[0].id);

  return {
    user,
    posts,
    comments
  };
}
```

## When Not to Use Do-Notation

Do-notation can make code *too* imperative-looking, hiding opportunities for better abstractions:

```haskell
-- Bad: too imperative, hiding the monad
badFunction = do
  x <- getX
  y <- getY
  z <- getZ
  let a = f x
  let b = g y
  let c = h z
  return (a + b + c)

-- Better: use Applicative (from Chapter 13/19)
betterFunction =
  (+) <$> fmap f getX
      <*> fmap g getY
      <*> fmap h getZ

-- Or with liftA3
betterFunction = liftA3 (\x y z -> f x + g y + h z) getX getY getZ
```

**Why Applicative is better here:**
- The operations (`getX`, `getY`, `getZ`) are *independent* - none depends on the result of another
- With do-notation, they must run sequentially
- With Applicative, they can potentially run in parallel
- The type clearly shows independence: `Applicative` vs `Monad`

**Rule of thumb:**
- Use **do-notation** when operations are sequential and dependent
- Use **Applicative** when operations are independent (see Chapter 13 on Validation for another example)

For instance, form validation should use Applicative to collect all errors:

```haskell
-- Good: Applicative for independent validations
validateUser = User
  <$> validateName name
  <*> validateEmail email
  <*> validateAge age

-- Bad: Monad/do stops at first error
validateUser = do
  validName <- validateName name      -- If this fails,
  validEmail <- validateEmail email   -- these never run
  validAge <- validateAge age
  return $ User validName validEmail validAge
```

## What We've Learned

- Do-notation is syntactic sugar for flatMap chains
- Makes monadic code read like imperative code
- Appears in many languages: Haskell (do), Scala (for), F# (computation expressions), JS (async/await)
- Improves readability and variable scoping
- Same structure across all implementations
- Use it when operations are sequential
- Use Applicative when operations are independent

## Coming Up

Do-notation makes monadic code readable, but knowing when to use it is just as important as knowing how. Next, we'll explore practical patterns and antipatterns - when to reach for monads, when simpler abstractions suffice, and common mistakes to avoid.

---

**Next: [Chapter 16 - Common Patterns and Antipatterns](16-patterns.md)**
