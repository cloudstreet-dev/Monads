# Chapter 13: Validation - Collecting All Errors

Here's a frustrating user experience:

```typescript
// Submit form
// Error: "Email is invalid"
// Fix email, submit again
// Error: "Password too short"
// Fix password, submit again
// Error: "Username already taken"
// Fix username, submit again
// Success!
```

Four round trips for three errors. Annoying!

With Either, we stop at the first error:

```typescript
validateEmail(data.email)          // Left("Invalid email")
  .flatMap(() => validatePassword(data.password))  // Never runs!
  .flatMap(() => validateUsername(data.username))  // Never runs!
```

`flatMap` is sequential - it short-circuits on error. But for validation, we want to collect *all* errors at once.

Enter **Validation** - an Applicative Functor that accumulates errors instead of short-circuiting.

## The Difference

```typescript
// Either: stop at first error (Monad)
Either<Error, A>
  .flatMap(f)  // If Left, stop here

// Validation: collect all errors (Applicative)
Validation<Error[], A>
  .apply(f)    // If Left, keep going and collect
```

## Building Validation

```typescript
type Validation<E, A> = Success<A> | Failure<E>;

class Success<A> {
  readonly tag = 'success';
  constructor(readonly value: A) {}
}

class Failure<E> {
  readonly tag = 'failure';
  constructor(readonly errors: E[]) {}
}

class Validation<E, A> {
  private constructor(private data: Success<A> | Failure<E>) {}

  // Wrap a value
  static success<E, A>(value: A): Validation<E, A> {
    return new Validation(new Success(value));
  }

  // Create a failure
  static failure<E, A>(error: E): Validation<E, A> {
    return new Validation(new Failure([error]));
  }

  // Transform the value (if success)
  map<B>(fn: (value: A) => B): Validation<E, B> {
    if (this.data.tag === 'success') {
      return Validation.success(fn(this.data.value));
    }
    return new Validation(this.data) as any;
  }

  // Apply a function wrapped in Validation
  // This is where we collect errors!
  ap<B>(
    vfn: Validation<E, (value: A) => B>
  ): Validation<E, B> {
    if (this.data.tag === 'failure' && vfn.data.tag === 'failure') {
      // Both failed - combine errors!
      return new Validation(
        new Failure([...vfn.data.errors, ...this.data.errors])
      ) as any;
    }

    if (this.data.tag === 'failure') {
      return new Validation(this.data) as any;
    }

    if (vfn.data.tag === 'failure') {
      return new Validation(vfn.data) as any;
    }

    // Both succeeded
    return Validation.success(vfn.data.value(this.data.value));
  }

  // Extract the result
  fold<B>(
    onFailure: (errors: E[]) => B,
    onSuccess: (value: A) => B
  ): B {
    return this.data.tag === 'failure'
      ? onFailure(this.data.errors)
      : onSuccess(this.data.value);
  }

  isSuccess(): boolean {
    return this.data.tag === 'success';
  }

  isFailure(): boolean {
    return this.data.tag === 'failure';
  }
}
```

## Form Validation Example

```typescript
interface RegistrationForm {
  username: string;
  email: string;
  password: string;
  age: number;
}

type ValidationError =
  | { field: 'username'; message: string }
  | { field: 'email'; message: string }
  | { field: 'password'; message: string }
  | { field: 'age'; message: string };

// Individual field validators
function validateUsername(username: string): Validation<ValidationError, string> {
  if (username.length < 3) {
    return Validation.failure({
      field: 'username',
      message: 'Username must be at least 3 characters'
    });
  }
  if (!/^[a-zA-Z0-9_]+$/.test(username)) {
    return Validation.failure({
      field: 'username',
      message: 'Username can only contain letters, numbers, and underscores'
    });
  }
  return Validation.success(username);
}

function validateEmail(email: string): Validation<ValidationError, string> {
  if (!email.includes('@')) {
    return Validation.failure({
      field: 'email',
      message: 'Invalid email address'
    });
  }
  return Validation.success(email);
}

function validatePassword(password: string): Validation<ValidationError, string> {
  const errors: ValidationError[] = [];

  if (password.length < 8) {
    errors.push({
      field: 'password',
      message: 'Password must be at least 8 characters'
    });
  }

  if (!/[A-Z]/.test(password)) {
    errors.push({
      field: 'password',
      message: 'Password must contain an uppercase letter'
    });
  }

  if (!/[0-9]/.test(password)) {
    errors.push({
      field: 'password',
      message: 'Password must contain a number'
    });
  }

  if (errors.length > 0) {
    return new Validation(new Failure(errors)) as any;
  }

  return Validation.success(password);
}

function validateAge(age: number): Validation<ValidationError, number> {
  if (age < 13) {
    return Validation.failure({
      field: 'age',
      message: 'Must be at least 13 years old'
    });
  }
  if (age > 120) {
    return Validation.failure({
      field: 'age',
      message: 'Invalid age'
    });
  }
  return Validation.success(age);
}

// Combine validators using applicative style
function validateRegistration(
  form: RegistrationForm
): Validation<ValidationError, RegistrationForm> {
  // Helper to build objects applicatively
  const validUsername = validateUsername(form.username);
  const validEmail = validateEmail(form.email);
  const validPassword = validatePassword(form.password);
  const validAge = validateAge(form.age);

  // Build the result object
  return Validation.success(
    (username: string) => (email: string) => (password: string) => (age: number) => ({
      username,
      email,
      password,
      age
    })
  )
    .ap(validUsername)
    .ap(validEmail)
    .ap(validPassword)
    .ap(validAge);
}

// Usage
const form = {
  username: 'ab',       // Too short
  email: 'notanemail',  // Invalid
  password: 'weak',     // Too short, no uppercase, no number
  age: 10               // Too young
};

const result = validateRegistration(form);

result.fold(
  errors => {
    console.log('Validation failed with errors:');
    errors.forEach(error =>
      console.log(`  ${error.field}: ${error.message}`)
    );
    // ALL errors are reported at once!
  },
  validForm => {
    console.log('Registration successful:', validForm);
  }
);
```

Output:
```
Validation failed with errors:
  username: Username must be at least 3 characters
  email: Invalid email address
  password: Password must be at least 8 characters
  password: Password must contain an uppercase letter
  password: Password must contain a number
  age: Must be at least 13 years old
```

All six errors reported at once!

## Helper for Cleaner Syntax

```typescript
// Applicative builder for better ergonomics
class ValidationBuilder<E> {
  static combine<E, A1, A2, R>(
    v1: Validation<E, A1>,
    v2: Validation<E, A2>,
    fn: (a1: A1, a2: A2) => R
  ): Validation<E, R> {
    return Validation.success<E, (a1: A1) => (a2: A2) => R>(
      a1 => a2 => fn(a1, a2)
    )
      .ap(v1)
      .ap(v2);
  }

  static combine3<E, A1, A2, A3, R>(
    v1: Validation<E, A1>,
    v2: Validation<E, A2>,
    v3: Validation<E, A3>,
    fn: (a1: A1, a2: A2, a3: A3) => R
  ): Validation<E, R> {
    return Validation.success<E, (a1: A1) => (a2: A2) => (a3: A3) => R>(
      a1 => a2 => a3 => fn(a1, a2, a3)
    )
      .ap(v1)
      .ap(v2)
      .ap(v3);
  }

  static combine4<E, A1, A2, A3, A4, R>(
    v1: Validation<E, A1>,
    v2: Validation<E, A2>,
    v3: Validation<E, A3>,
    v4: Validation<E, A4>,
    fn: (a1: A1, a2: A2, a3: A3, a4: A4) => R
  ): Validation<E, R> {
    return Validation.success<E, (a1: A1) => (a2: A2) => (a3: A3) => (a4: A4) => R>(
      a1 => a2 => a3 => a4 => fn(a1, a2, a3, a4)
    )
      .ap(v1)
      .ap(v2)
      .ap(v3)
      .ap(v4);
  }
}

// Now validation is cleaner
function validateRegistration(
  form: RegistrationForm
): Validation<ValidationError, RegistrationForm> {
  return ValidationBuilder.combine4(
    validateUsername(form.username),
    validateEmail(form.email),
    validatePassword(form.password),
    validateAge(form.age),
    (username, email, password, age) => ({
      username,
      email,
      password,
      age
    })
  );
}
```

## Validation vs Either

When to use which?

### Use Either when:
- You want fail-fast behavior
- Errors are unexpected (exceptional cases)
- Operations depend on each other
- Example: Database operations, file I/O

```typescript
// Database operations - stop if user not found
fetchUser(id)
  .flatMap(user => fetchProfile(user.id))
  .flatMap(profile => updateProfile(profile, newData));
// If user doesn't exist, no point checking profile
```

### Use Validation when:
- You want to collect all errors
- Errors are expected (validation)
- Operations are independent
- Example: Form validation, configuration validation

```typescript
// Form validation - check all fields
validateForm({
  name: validateName(form.name),
  email: validateEmail(form.email),
  phone: validatePhone(form.phone)
});
// Get all validation errors at once
```

## Real-World Example: Config Validation

```typescript
interface DatabaseConfig {
  host: string;
  port: number;
  database: string;
  username: string;
  password: string;
}

type ConfigError = { field: string; issue: string };

function validateHost(host: string): Validation<ConfigError, string> {
  if (!host) {
    return Validation.failure({ field: 'host', issue: 'Host is required' });
  }
  if (host === 'localhost' && process.env.NODE_ENV === 'production') {
    return Validation.failure({
      field: 'host',
      issue: 'Cannot use localhost in production'
    });
  }
  return Validation.success(host);
}

function validatePort(port: number): Validation<ConfigError, number> {
  if (port < 1 || port > 65535) {
    return Validation.failure({
      field: 'port',
      issue: 'Port must be between 1 and 65535'
    });
  }
  return Validation.success(port);
}

function validateDatabaseName(name: string): Validation<ConfigError, string> {
  if (!name) {
    return Validation.failure({
      field: 'database',
      issue: 'Database name is required'
    });
  }
  if (!/^[a-zA-Z0-9_]+$/.test(name)) {
    return Validation.failure({
      field: 'database',
      issue: 'Database name can only contain letters, numbers, and underscores'
    });
  }
  return Validation.success(name);
}

function validateCredentials(
  username: string,
  password: string
): Validation<ConfigError, { username: string; password: string }> {
  const errors: ConfigError[] = [];

  if (!username) {
    errors.push({ field: 'username', issue: 'Username is required' });
  }

  if (!password) {
    errors.push({ field: 'password', issue: 'Password is required' });
  }

  if (password.length < 12 && process.env.NODE_ENV === 'production') {
    errors.push({
      field: 'password',
      issue: 'Password must be at least 12 characters in production'
    });
  }

  if (errors.length > 0) {
    return new Validation(new Failure(errors)) as any;
  }

  return Validation.success({ username, password });
}

function validateDatabaseConfig(
  config: DatabaseConfig
): Validation<ConfigError, DatabaseConfig> {
  return ValidationBuilder.combine4(
    validateHost(config.host),
    validatePort(config.port),
    validateDatabaseName(config.database),
    validateCredentials(config.username, config.password),
    (host, port, database, credentials) => ({
      host,
      port,
      database,
      ...credentials
    })
  );
}

// Load and validate config at startup
const config = loadConfigFromEnv();
const validation = validateDatabaseConfig(config);

validation.fold(
  errors => {
    console.error('Configuration validation failed:');
    errors.forEach(error =>
      console.error(`  ${error.field}: ${error.issue}`)
    );
    process.exit(1);
  },
  validConfig => {
    console.log('Configuration valid');
    startServer(validConfig);
  }
);
```

## Validation in Other Languages

### Haskell (using Validation from validation package)

```haskell
import Data.Validation

data ValidationError = ValidationError String
  deriving (Show)

validateUsername :: String -> Validation [ValidationError] String
validateUsername name
  | length name < 3 = Failure [ValidationError "Username too short"]
  | otherwise = Success name

validateEmail :: String -> Validation [ValidationError] String
validateEmail email
  | '@' `elem` email = Success email
  | otherwise = Failure [ValidationError "Invalid email"]

validateAge :: Int -> Validation [ValidationError] Int
validateAge age
  | age < 13 = Failure [ValidationError "Too young"]
  | age > 120 = Failure [ValidationError "Invalid age"]
  | otherwise = Success age

-- Applicative combination
validateUser :: String -> String -> Int -> Validation [ValidationError] User
validateUser name email age =
  User <$> validateUsername name
       <*> validateEmail email
       <*> validateAge age
```

### Scala (using cats Validated)

```scala
import cats.data.Validated
import cats.data.ValidatedNel  // Validated Non-Empty List
import cats.implicits._

type ValidationResult[A] = ValidatedNel[String, A]

def validateUsername(name: String): ValidationResult[String] =
  if (name.length >= 3) name.validNel
  else "Username too short".invalidNel

def validateEmail(email: String): ValidationResult[String] =
  if (email.contains("@")) email.validNel
  else "Invalid email".invalidNel

def validateAge(age: Int): ValidationResult[Int] =
  if (age >= 13 && age <= 120) age.validNel
  else "Invalid age".invalidNel

// Applicative combination
case class User(name: String, email: String, age: Int)

def validateUser(name: String, email: String, age: Int): ValidationResult[User] =
  (validateUsername(name),
   validateEmail(email),
   validateAge(age)).mapN(User)

// Usage
validateUser("ab", "notanemail", 10) match {
  case Validated.Valid(user) =>
    println(s"Valid user: $user")
  case Validated.Invalid(errors) =>
    println("Validation errors:")
    errors.toList.foreach(err => println(s"  - $err"))
}
```

### F# (using FsToolkit.ErrorHandling)

```fsharp
type ValidationError = ValidationError of string

let validateUsername name =
    if String.length name >= 3 then
        Ok name
    else
        Error [ValidationError "Username too short"]

let validateEmail email =
    if String.contains "@" email then
        Ok email
    else
        Error [ValidationError "Invalid email"]

let validateAge age =
    if age >= 13 && age <= 120 then
        Ok age
    else
        Error [ValidationError "Invalid age"]

// Validation.ofResult combines results
type User = { Name: string; Email: string; Age: int }

let validateUser name email age =
    validation {
        let! validName = validateUsername name
        and! validEmail = validateEmail email
        and! validAge = validateAge age
        return { Name = validName; Email = validEmail; Age = validAge }
    }
```

## Why Not Monad?

Validation is an Applicative, not a Monad. Here's why:

```typescript
// With Monad (Either), this is possible:
Either.of("Alice")
  .flatMap(name => {
    // Can use 'name' to decide what to validate next
    if (name === "Alice") {
      return validateAdminPassword(password);
    } else {
      return validateUserPassword(password);
    }
  });

// With Applicative (Validation), this is NOT possible
// All validations must be independent!
Validation.success("Alice")
  .ap(/* can't branch based on name */)
```

Monad's `flatMap` allows the second operation to depend on the first. Applicative's `ap` requires independence.

For validation, this is perfect - fields shouldn't depend on each other. We want to check them all in parallel.

## The Applicative Pattern

```typescript
// Applicative: combine independent computations
Validation.success((a: A) => (b: B) => (c: C) => result)
  .ap(computeA)  // Independent
  .ap(computeB)  // Independent
  .ap(computeC)  // Independent

// Monad: chain dependent computations
monadA
  .flatMap(a => computeB(a))      // Depends on a
  .flatMap(b => computeC(a, b))   // Depends on a and b
```

Applicative is less powerful but more composable - we can run operations in parallel since they don't depend on each other.

## What We've Learned

- Validation accumulates errors instead of short-circuiting
- Perfect for form validation and configuration checking
- Uses Applicative instead of Monad
- `ap` combines independent computations
- Either for fail-fast, Validation for collect-all
- Independence enables parallelism

## Coming Up

We've seen many monads, but what if you need to combine them? What if you need Maybe + IO, or Either + State? That's where Monad Transformers come in - they stack monads together.

---

**Next: [Chapter 14 - Monad Transformers: Stacking Effects](14-monad-transformers.md)**
