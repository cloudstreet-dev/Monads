# Chapter 12: Async/Promise - Monads in Time

You use them every day:

```javascript
fetch('/api/user')
  .then(response => response.json())
  .then(user => fetch(`/api/posts/${user.id}`))
  .then(posts => console.log(posts))
  .catch(error => console.error(error));
```

Congratulations - you've been using monads all along.

**Promises are monads.** They represent "a value that will exist in the future."

## The Problem They Solve

Asynchronous code without Promises is callback hell:

```javascript
// The pyramid of doom
fetchUser(userId, (error, user) => {
  if (error) {
    handleError(error);
    return;
  }

  fetchPosts(user.id, (error, posts) => {
    if (error) {
      handleError(error);
      return;
    }

    fetchComments(posts[0].id, (error, comments) => {
      if (error) {
        handleError(error);
        return;
      }

      render(user, posts, comments);
    });
  });
});
```

This is the same problem we've been solving with monads - chaining operations that produce contexts (async results), with automatic error handling.

## Promise as a Monad

Let's verify Promises have the monad operations:

```javascript
// 1. Wrap a value (of/return)
const promise = Promise.resolve(42);

// 2. Transform the value (map)
promise.then(x => x * 2);  // Promise<number>

// 3. Chain operations that return Promises (flatMap)
promise.then(x => fetch(`/api/data/${x}`));  // Promise<Response>
```

The `.then()` method is `flatMap` - it automatically unwraps nested Promises!

```javascript
// If you return a Promise from .then(), it gets flattened
Promise.resolve(5)
  .then(x => Promise.resolve(x * 2))  // Returns Promise<number>
  // NOT Promise<Promise<number>>!
```

## Building Task (A Better Promise)

Promises have some quirks (they eagerly execute, can't be cancelled). Let's build a better version called Task:

```typescript
class Task<T> {
  constructor(private computation: () => Promise<T>) {}

  // Wrap a value
  static of<T>(value: T): Task<T> {
    return new Task(() => Promise.resolve(value));
  }

  // Create a failing task
  static reject<T>(error: Error): Task<T> {
    return new Task(() => Promise.reject(error));
  }

  // Transform the value
  map<U>(fn: (value: T) => U): Task<U> {
    return new Task(() =>
      this.computation().then(fn)
    );
  }

  // Chain tasks
  flatMap<U>(fn: (value: T) => Task<U>): Task<U> {
    return new Task(() =>
      this.computation().then(value => fn(value).run())
    );
  }

  // Handle errors
  catch(handler: (error: Error) => Task<T>): Task<T> {
    return new Task(() =>
      this.computation().catch(error => handler(error).run())
    );
  }

  // Run the task (lazy execution)
  run(): Promise<T> {
    return this.computation();
  }

  // Timeout
  timeout(ms: number): Task<T> {
    return new Task(() =>
      Promise.race([
        this.computation(),
        new Promise<T>((_, reject) =>
          setTimeout(() => reject(new Error('Timeout')), ms)
        )
      ])
    );
  }

  // Retry on failure
  retry(times: number, delay: number = 0): Task<T> {
    return new Task(async () => {
      for (let i = 0; i < times; i++) {
        try {
          return await this.computation();
        } catch (error) {
          if (i === times - 1) throw error;
          if (delay > 0) {
            await new Promise(resolve => setTimeout(resolve, delay));
          }
        }
      }
      throw new Error('Unreachable');
    });
  }
}
```

## Real Example: API Call Chain

```typescript
interface User {
  id: string;
  name: string;
  email: string;
}

interface Post {
  id: string;
  userId: string;
  title: string;
  content: string;
}

interface Comment {
  id: string;
  postId: string;
  text: string;
}

// API calls as Tasks
function fetchUser(id: string): Task<User> {
  return new Task(() =>
    fetch(`/api/users/${id}`).then(r => r.json())
  );
}

function fetchPosts(userId: string): Task<Post[]> {
  return new Task(() =>
    fetch(`/api/users/${userId}/posts`).then(r => r.json())
  );
}

function fetchComments(postId: string): Task<Comment[]> {
  return new Task(() =>
    fetch(`/api/posts/${postId}/comments`).then(r => r.json())
  );
}

// Compose with flatMap
function getUserWithPosts(userId: string): Task<{ user: User; posts: Post[] }> {
  return fetchUser(userId).flatMap(user =>
    fetchPosts(user.id).map(posts => ({ user, posts }))
  );
}

function getPostWithComments(postId: string): Task<{ post: Post; comments: Comment[] }> {
  return new Task(() =>
    fetch(`/api/posts/${postId}`).then(r => r.json())
  ).flatMap(post =>
    fetchComments(post.id).map(comments => ({ post, comments }))
  );
}

// Use the tasks
const task = getUserWithPosts('user123')
  .flatMap(({ user, posts }) => {
    console.log(`User: ${user.name}`);
    console.log(`Posts: ${posts.length}`);
    return getPostWithComments(posts[0].id);
  })
  .map(({ post, comments }) => ({
    title: post.title,
    commentCount: comments.length
  }))
  .timeout(5000)  // 5 second timeout
  .retry(3, 1000); // Retry 3 times with 1s delay

// Execute (lazy!)
task.run()
  .then(result => console.log(result))
  .catch(error => console.error(error));
```

## Parallel Execution

Monads are sequential. For parallel operations, we need different combinators:

```typescript
class Task<T> {
  // ... previous methods ...

  // Run two tasks in parallel
  static all<T1, T2>(t1: Task<T1>, t2: Task<T2>): Task<[T1, T2]> {
    return new Task(() =>
      Promise.all([t1.run(), t2.run()])
    );
  }

  // Run multiple tasks in parallel
  static allArray<T>(tasks: Task<T>[]): Task<T[]> {
    return new Task(() =>
      Promise.all(tasks.map(t => t.run()))
    );
  }

  // Race - first one to complete wins
  static race<T>(tasks: Task<T>[]): Task<T> {
    return new Task(() =>
      Promise.race(tasks.map(t => t.run()))
    );
  }

  // Run tasks in sequence (monadic)
  static sequence<T>(tasks: Task<T>[]): Task<T[]> {
    return tasks.reduce(
      (acc, task) =>
        acc.flatMap(results =>
          task.map(result => [...results, result])
        ),
      Task.of<T[]>([])
    );
  }
}

// Usage
const parallel = Task.all(
  fetchUser('user1'),
  fetchUser('user2')
).map(([user1, user2]) => [user1, user2]);

const sequential = Task.sequence([
  fetchUser('user1'),
  fetchUser('user2'),
  fetchUser('user3')
]);

// Fetch multiple posts in parallel
function fetchAllPosts(ids: string[]): Task<Post[]> {
  const tasks = ids.map(id =>
    new Task(() => fetch(`/api/posts/${id}`).then(r => r.json()))
  );
  return Task.allArray(tasks);
}
```

## Async/Await: Do-Notation for Promises

JavaScript's `async/await` is syntactic sugar for Promise chains:

```javascript
// With then (monadic style)
fetchUser('123')
  .then(user => fetchPosts(user.id))
  .then(posts => posts.filter(p => p.published))
  .then(published => console.log(published));

// With async/await (do-notation style)
async function getPublishedPosts(userId) {
  const user = await fetchUser(userId);
  const posts = await fetchPosts(user.id);
  const published = posts.filter(p => p.published);
  console.log(published);
  return published;
}
```

`await` is like the `<-` in Haskell's do-notation - it unwraps the monadic value.

```haskell
-- Haskell do-notation
getPublishedPosts :: String -> IO [Post]
getPublishedPosts userId = do
  user <- fetchUser userId
  posts <- fetchPosts (userId user)
  let published = filter published posts
  return published

-- JavaScript async/await (same structure!)
async function getPublishedPosts(userId) {
  const user = await fetchUser(userId);
  const posts = await fetchPosts(user.id);
  const published = posts.filter(p => p.published);
  return published;
}
```

## Error Handling

Promises have built-in error handling (Left/Right behavior):

```typescript
// Promise catches errors automatically
function fetchWithRetry(url: string): Task<Response> {
  return new Task(() => fetch(url))
    .catch(error => {
      console.error('Fetch failed, retrying...', error);
      return new Task(() => fetch(url));
    })
    .catch(error => {
      console.error('Retry failed');
      return Task.reject(error);
    });
}

// With async/await
async function fetchWithRetry(url: string): Promise<Response> {
  try {
    return await fetch(url);
  } catch (error) {
    console.error('Fetch failed, retrying...', error);
    try {
      return await fetch(url);
    } catch (error) {
      console.error('Retry failed');
      throw error;
    }
  }
}
```

## Combining Task with Other Monads

Often you need Task + Maybe or Task + Either:

```typescript
// Task that might not find a result
function findUser(id: string): Task<Maybe<User>> {
  return new Task(() =>
    fetch(`/api/users/${id}`)
      .then(r => r.json())
      .then(user => user ? Maybe.of(user) : Maybe.none())
      .catch(() => Maybe.none())
  );
}

// Task with explicit error type
function fetchUser(id: string): Task<Either<Error, User>> {
  return new Task(() =>
    fetch(`/api/users/${id}`)
      .then(r => r.json())
      .then(user => Either.right(user))
      .catch(error => Either.left(error))
  );
}

// Handle both layers
const result = await fetchUser('123').run();
result.fold(
  error => console.error('Failed:', error),
  user => console.log('Success:', user)
);
```

## Real-World Pattern: Data Fetching

```typescript
// Declarative data loading
interface PageData {
  user: User;
  posts: Post[];
  notifications: Notification[];
}

function loadPageData(userId: string): Task<PageData> {
  // Fetch in parallel for performance
  return Task.all(
    fetchUser(userId),
    Task.all(
      fetchPosts(userId),
      fetchNotifications(userId)
    )
  ).map(([user, [posts, notifications]]) => ({
    user,
    posts,
    notifications
  }));
}

// With loading states
type LoadingState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error };

function useAsyncData<T>(task: Task<T>): LoadingState<T> {
  const [state, setState] = useState<LoadingState<T>>({ status: 'idle' });

  useEffect(() => {
    setState({ status: 'loading' });

    task.run()
      .then(data => setState({ status: 'success', data }))
      .catch(error => setState({ status: 'error', error }));
  }, [task]);

  return state;
}

// Usage in React
function UserPage({ userId }: { userId: string }) {
  const data = useAsyncData(loadPageData(userId));

  switch (data.status) {
    case 'idle':
    case 'loading':
      return <Spinner />;

    case 'error':
      return <Error message={data.error.message} />;

    case 'success':
      return (
        <div>
          <UserProfile user={data.data.user} />
          <PostList posts={data.data.posts} />
          <Notifications items={data.data.notifications} />
        </div>
      );
  }
}
```

## Observable: The Async List Monad

Observables (from RxJS) are like async lists - streams of values over time:

```typescript
import { Observable } from 'rxjs';
import { map, flatMap, filter } from 'rxjs/operators';

// Observable is a monad
const numbers$ = new Observable<number>(subscriber => {
  subscriber.next(1);
  subscriber.next(2);
  subscriber.next(3);
  subscriber.complete();
});

// Transform with map
const doubled$ = numbers$.pipe(
  map(x => x * 2)
);

// Chain with flatMap
const fetched$ = numbers$.pipe(
  flatMap(id => fetch(`/api/users/${id}`).then(r => r.json()))
);

// Filter (guard operation)
const evens$ = numbers$.pipe(
  filter(x => x % 2 === 0)
);
```

Observables combine:
- List monad (multiple values)
- Async monad (values over time)
- Lazy evaluation (like Task)

## Async Generators: Monads Built-In

JavaScript async generators are monadic:

```typescript
async function* fetchPages(query: string) {
  let page = 1;
  let hasMore = true;

  while (hasMore) {
    const response = await fetch(`/api/search?q=${query}&page=${page}`);
    const data = await response.json();

    yield data.results;  // Like 'return' in monad

    hasMore = data.hasMore;
    page++;
  }
}

// Use with for-await (do-notation style!)
async function searchAll(query: string) {
  for await (const results of fetchPages(query)) {
    console.log('Got page:', results);
    // Process results
  }
}
```

## When Promises Break Monad Laws

JavaScript Promises almost follow the laws, but not quite:

```javascript
// Promises auto-flatten
Promise.resolve(Promise.resolve(5))  // Promise(5), not Promise(Promise(5))

// This breaks strict monad laws but is convenient in practice

// Compare to Task:
Task.of(Task.of(5))  // Task(Task(5)) - nested!
```

For most purposes, this doesn't matter. But technically, Promises are "lazy monads" or "tasks with eager evaluation."

## Best Practices

### 1. Keep Chains Flat

```typescript
// Bad: Nested promises
fetchUser(id).then(user => {
  return fetchPosts(user.id).then(posts => {
    return fetchComments(posts[0].id).then(comments => {
      return { user, posts, comments };
    });
  });
});

// Good: Flat chain
fetchUser(id)
  .then(user =>
    fetchPosts(user.id).then(posts => ({ user, posts }))
  )
  .then(({ user, posts }) =>
    fetchComments(posts[0].id).then(comments => ({ user, posts, comments }))
  );

// Better: async/await
async function getData(id) {
  const user = await fetchUser(id);
  const posts = await fetchPosts(user.id);
  const comments = await fetchComments(posts[0].id);
  return { user, posts, comments };
}
```

### 2. Handle Errors Explicitly

```typescript
// Bad: Unhandled rejections
fetchData()
  .then(process)
  .then(save);

// Good: Catch errors
fetchData()
  .then(process)
  .then(save)
  .catch(error => {
    logError(error);
    return fallbackValue;
  });
```

### 3. Parallel When Possible

```typescript
// Sequential (slow)
const user = await fetchUser(id);
const settings = await fetchSettings(id);
const notifications = await fetchNotifications(id);

// Parallel (fast)
const [user, settings, notifications] = await Promise.all([
  fetchUser(id),
  fetchSettings(id),
  fetchNotifications(id)
]);
```

## What We've Learned

- Promises/Tasks are monads for asynchronous computation
- `.then()` is `flatMap` with automatic unwrapping
- `async/await` is do-notation for Promises
- Task is a lazy, composable alternative to Promise
- Observables extend the pattern to streams
- Async is just another context for monads

## Coming Up

Sometimes we need to accumulate *all* errors, not just stop at the first one. Form validation is a perfect example. That's where Validation (an Applicative) comes in.

---

**Next: [Chapter 13 - Validation: Collecting All Errors](13-validation.md)**
