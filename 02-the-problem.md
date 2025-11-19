# Chapter 2: The Problem - Programming is Messy

Before we talk about solutions, let's talk about pain.

## The Billion-Dollar Mistake

In 1965, Tony Hoare invented null references. In 2009, he apologized:

> "I call it my billion-dollar mistake... My goal was to ensure that all use of references should be absolutely safe, with checking performed automatically by the compiler. But I couldn't resist the temptation to put in a null reference, simply because it was so easy to implement."

Look at this seemingly innocent function:

```typescript
function getUserEmail(userId: string): string {
  const user = database.findUser(userId);
  const profile = user.getProfile();
  const email = profile.getPrimaryEmail();
  return email.toLowerCase();
}
```

How many ways can this fail?

1. `database.findUser()` might return `null` (user doesn't exist)
2. `user.getProfile()` might return `null` (profile not set up)
3. `profile.getPrimaryEmail()` might return `null` (no email on file)
4. `email.toLowerCase()` might explode if `email` is somehow `null`

That's four potential `NullPointerException` bombs. And this is a simple function!

## The Defensive Programming Trap

So you write defensive code:

```typescript
function getUserEmail(userId: string): string | null {
  const user = database.findUser(userId);
  if (user === null) return null;

  const profile = user.getProfile();
  if (profile === null) return null;

  const email = profile.getPrimaryEmail();
  if (email === null) return null;

  return email.toLowerCase();
}
```

This works, but:
- It's verbose (more code to write, read, and maintain)
- It's repetitive (the same null-check pattern over and over)
- It's error-prone (forget one check and boom)
- It obscures the happy path (what the function *actually* does)

The actual business logic (`toLowerCase()`) is drowning in error handling scaffolding.

## The Error Handling Pyramid

Error handling is even worse:

```typescript
async function processOrder(orderId: string): Promise<void> {
  try {
    const order = await fetchOrder(orderId);
    try {
      const payment = await processPayment(order);
      try {
        const shipment = await scheduleShipment(order);
        try {
          await sendConfirmation(order, shipment);
          console.log("Order processed successfully");
        } catch (e) {
          console.error("Failed to send confirmation", e);
          await rollbackShipment(shipment);
          throw e;
        }
      } catch (e) {
        console.error("Failed to schedule shipment", e);
        await refundPayment(payment);
        throw e;
      }
    } catch (e) {
      console.error("Failed to process payment", e);
      await cancelOrder(order);
      throw e;
    }
  } catch (e) {
    console.error("Failed to fetch order", e);
    throw e;
  }
}
```

This is the "pyramid of doom" - nested error handling that:
- Grows rightward with each step
- Makes rollback logic complex
- Becomes unmaintainable fast
- Makes you question your career choices

## The Exception Problem

Some languages use exceptions for control flow:

```java
public String getUserEmail(String userId) {
    try {
        User user = database.findUser(userId);
        Profile profile = user.getProfile();
        Email email = profile.getPrimaryEmail();
        return email.toLowerCase();
    } catch (NullPointerException e) {
        return null;
    }
}
```

But exceptions are:
- **Invisible**: Not in the type signature (did you know this throws?)
- **Expensive**: Stack unwinding has performance costs
- **Non-local**: Control flow jumps around unpredictably
- **Easy to ignore**: Catch blocks get swallowed or forgotten

Plus, using exceptions for expected cases (like "user not found") mixes error handling with actual exceptional situations (like "database is on fire").

## The Side Effect Shuffle

Then there are side effects - functions that do more than just compute:

```typescript
let config: Config | null = null;

function loadConfig(): void {
  config = readFromDisk("config.json");
}

function getApiKey(): string {
  if (config === null) {
    loadConfig();
  }
  return config.apiKey;
}
```

Problems:
- Global state makes testing hard
- Order of operations matters
- Race conditions in concurrent code
- Hard to reason about what's happening when

You can't just look at `getApiKey()` and know what it does. You have to trace through the state mutations.

## The Composition Problem

The real issue is that none of these approaches **compose** well.

Say you have three functions:

```typescript
function getUser(id: string): User | null { /* ... */ }
function getEmail(user: User): string | null { /* ... */ }
function validateEmail(email: string): boolean { /* ... */ }
```

Composing them elegantly is hard:

```typescript
// Option 1: Nested nulls
const user = getUser("123");
if (user !== null) {
  const email = getEmail(user);
  if (email !== null) {
    if (validateEmail(email)) {
      // finally, do something
    }
  }
}

// Option 2: Early returns (only works in some contexts)
function processUser(id: string): boolean {
  const user = getUser(id);
  if (user === null) return false;

  const email = getEmail(user);
  if (email === null) return false;

  return validateEmail(email);
}

// Option 3: The ugly chain
const isValid =
  (user => user !== null ? getEmail(user) : null)(getUser("123"))
    ?.let(email => email !== null ? validateEmail(email) : false)
  ?? false;
```

None of these are satisfying. We want to write:

```typescript
// Dream code (doesn't work in plain TypeScript)
getUser("123")
  .andThen(getEmail)
  .andThen(validateEmail)
```

A clean chain where each step automatically handles the "might be null" case.

## What We Actually Want

Looking at all these problems, here's what we want:

1. **Type safety**: The compiler should catch potential nulls
2. **Explicit failure**: The type signature should show that something might fail
3. **Composability**: Chain operations without nesting or repetition
4. **Clarity**: The happy path should be obvious
5. **Automatic handling**: Failure propagation without manual checks

In other words, we want a pattern that:
- Wraps potentially absent values
- Lets us chain operations on them
- Automatically short-circuits on failure
- Keeps our code clean and readable

## Enter: Monads

All of those problems - null handling, error management, side effects, composition - have something in common. They're all about **values in a context**:

- A value that might not exist (`User | null`)
- A computation that might fail (`Result<User, Error>`)
- An operation with side effects (`IO<User>`)
- A list of possible values (`Array<User>`)

Monads are a pattern for:
1. Putting values in a context
2. Transforming values while respecting that context
3. Chaining operations without nested boilerplate

Let's see how they solve each problem, starting with the simplest: the Maybe monad.

## A Glimpse of the Solution

Here's that email function again, but with a Maybe monad:

```typescript
// Before: Nested null checks
function getUserEmail(userId: string): string | null {
  const user = database.findUser(userId);
  if (user === null) return null;

  const profile = user.getProfile();
  if (profile === null) return null;

  const email = profile.getPrimaryEmail();
  if (email === null) return null;

  return email.toLowerCase();
}

// After: Monadic chain
function getUserEmail(userId: string): Maybe<string> {
  return Maybe.of(() => database.findUser(userId))
    .flatMap(user => user.getProfile())
    .flatMap(profile => profile.getPrimaryEmail())
    .map(email => email.toLowerCase());
}
```

The second version:
- Has no manual null checks
- Is flat, not nested
- Clearly shows the transformation pipeline
- Automatically handles the failure case at each step
- Returns a type that says "this might not have a value"

That's the power of monads. They turn error handling from scattered defensive code into a composable pipeline.

Ready to see how it works?

---

**Next: [Chapter 3 - Maybe: Your First Monad](03-maybe-monad.md)**
