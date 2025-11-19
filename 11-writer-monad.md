# Chapter 11: Writer - Logging Along the Way

You're computing a value, but you also want to collect information along the way - logs, audit trails, debug messages, warnings, metrics.

The naive approach:

```typescript
let logs: string[] = [];

function compute(x: number): number {
  logs.push(`Computing with ${x}`);
  const result = x * 2;
  logs.push(`Result: ${result}`);
  return result;
}

const value = compute(5);
console.log(value);  // 10
console.log(logs);   // ["Computing with 5", "Result: 10"]
```

But global state strikes again! Hard to test, not composable, not thread-safe.

The Writer monad lets you accumulate values (like logs) alongside your computation, without globals.

## The Core Idea

Return both the value AND the accumulated log:

```typescript
function compute(x: number): [number, string[]] {
  return [x * 2, [`Computing with ${x}`, `Result: ${x * 2}`]];
}

const [value, logs] = compute(5);
```

Writer automates combining logs from chained operations.

## Building Writer

```typescript
// Writer<W, A> represents:
// A pair of (value A, accumulated log W)
// W must be a Monoid (has empty value and combine operation)
class Writer<W, A> {
  constructor(
    private value: A,
    private log: W
  ) {}

  // Wrap a value with empty log
  static of<W, A>(value: A, empty: W): Writer<W, A> {
    return new Writer(value, empty);
  }

  // Create a Writer with just a log entry
  static tell<W>(log: W): Writer<W, void> {
    return new Writer(undefined, log);
  }

  // Transform the value
  map<B>(fn: (value: A) => B): Writer<W, B> {
    return new Writer(fn(this.value), this.log);
  }

  // Chain Writer computations (combines logs)
  flatMap<B>(
    fn: (value: A) => Writer<W, B>,
    combine: (w1: W, w2: W) => W
  ): Writer<W, B> {
    const next = fn(this.value);
    return new Writer(
      next.value,
      combine(this.log, next.log)
    );
  }

  // Extract the pair
  run(): [A, W] {
    return [this.value, this.log];
  }

  // Get only the value
  getValue(): A {
    return this.value;
  }

  // Get only the log
  getLog(): W {
    return this.log;
  }
}
```

## Simple Example: Logging Calculations

```typescript
type Log = string[];

// Helper to combine logs
const combineLogs = (l1: Log, l2: Log): Log => [...l1, ...l2];
const emptyLog: Log = [];

// Create a Writer helper class for strings
class LogWriter<A> extends Writer<Log, A> {
  static of<A>(value: A): LogWriter<A> {
    return new LogWriter(value, emptyLog);
  }

  static tell(message: string): LogWriter<void> {
    return new LogWriter(undefined, [message]);
  }

  flatMap<B>(fn: (value: A) => LogWriter<B>): LogWriter<B> {
    const next = fn(this.value);
    return new LogWriter(
      next.value,
      combineLogs(this.getLog(), next.getLog())
    ) as LogWriter<B>;
  }
}

// Functions that log
function add(a: number, b: number): LogWriter<number> {
  const result = a + b;
  return LogWriter.tell(`Adding ${a} + ${b} = ${result}`)
    .flatMap(() => LogWriter.of(result));
}

function multiply(a: number, b: number): LogWriter<number> {
  const result = a * b;
  return LogWriter.tell(`Multiplying ${a} * ${b} = ${result}`)
    .flatMap(() => LogWriter.of(result));
}

function divide(a: number, b: number): LogWriter<number> {
  if (b === 0) {
    return LogWriter.tell(`Cannot divide ${a} by zero`)
      .flatMap(() => LogWriter.of(NaN));
  }
  const result = a / b;
  return LogWriter.tell(`Dividing ${a} / ${b} = ${result}`)
    .flatMap(() => LogWriter.of(result));
}

// Compose operations
const computation = add(10, 5)
  .flatMap(sum => multiply(sum, 2))
  .flatMap(product => divide(product, 3));

const [result, logs] = computation.run();
console.log('Result:', result);  // 10
console.log('Logs:');
logs.forEach(log => console.log('  -', log));
// Logs:
//   - Adding 10 + 5 = 15
//   - Multiplying 15 * 2 = 30
//   - Dividing 30 / 3 = 10
```

All logs collected automatically!

## Real Example: Audit Trail

Track all operations for compliance:

```typescript
interface AuditEntry {
  timestamp: Date;
  operation: string;
  userId: string;
  details: string;
}

type AuditLog = AuditEntry[];

class AuditWriter<A> extends Writer<AuditLog, A> {
  static of<A>(value: A): AuditWriter<A> {
    return new AuditWriter(value, []);
  }

  static audit(operation: string, userId: string, details: string): AuditWriter<void> {
    return new AuditWriter(undefined, [{
      timestamp: new Date(),
      operation,
      userId,
      details
    }]);
  }

  flatMap<B>(fn: (value: A) => AuditWriter<B>): AuditWriter<B> {
    const next = fn(this.value);
    return new AuditWriter(
      next.value,
      [...this.getLog(), ...next.getLog()]
    ) as AuditWriter<B>;
  }
}

// Business operations with auditing
function createAccount(
  userId: string,
  accountType: string
): AuditWriter<Account> {
  const account = {
    id: generateId(),
    userId,
    type: accountType,
    balance: 0
  };

  return AuditWriter.audit(
    'CREATE_ACCOUNT',
    userId,
    `Created ${accountType} account ${account.id}`
  ).flatMap(() => AuditWriter.of(account));
}

function deposit(
  account: Account,
  amount: number,
  userId: string
): AuditWriter<Account> {
  const updated = {
    ...account,
    balance: account.balance + amount
  };

  return AuditWriter.audit(
    'DEPOSIT',
    userId,
    `Deposited ${amount} to account ${account.id}. New balance: ${updated.balance}`
  ).flatMap(() => AuditWriter.of(updated));
}

function withdraw(
  account: Account,
  amount: number,
  userId: string
): AuditWriter<Account> {
  if (account.balance < amount) {
    return AuditWriter.audit(
      'WITHDRAW_FAILED',
      userId,
      `Insufficient funds in account ${account.id}. Attempted: ${amount}, Available: ${account.balance}`
    ).flatMap(() => AuditWriter.of(account));
  }

  const updated = {
    ...account,
    balance: account.balance - amount
  };

  return AuditWriter.audit(
    'WITHDRAW',
    userId,
    `Withdrew ${amount} from account ${account.id}. New balance: ${updated.balance}`
  ).flatMap(() => AuditWriter.of(updated));
}

// Execute operations
const bankingOperation = createAccount('user123', 'checking')
  .flatMap(account => deposit(account, 1000, 'user123'))
  .flatMap(account => withdraw(account, 200, 'user123'))
  .flatMap(account => withdraw(account, 900, 'user123'));

const [finalAccount, auditTrail] = bankingOperation.run();

console.log('Final balance:', finalAccount.balance);  // 800 (fails last withdrawal)

console.log('\nAudit Trail:');
auditTrail.forEach(entry => {
  console.log(`[${entry.timestamp.toISOString()}] ${entry.operation}`);
  console.log(`  User: ${entry.userId}`);
  console.log(`  ${entry.details}`);
});
```

Perfect for compliance and debugging!

## Accumulating Different Types

Writer isn't just for strings - accumulate anything that can be combined:

### Metrics

```typescript
interface Metrics {
  operationCount: number;
  totalTime: number;
  errorsCount: number;
}

const emptyMetrics: Metrics = {
  operationCount: 0,
  totalTime: 0,
  errorsCount: 0
};

const combineMetrics = (m1: Metrics, m2: Metrics): Metrics => ({
  operationCount: m1.operationCount + m2.operationCount,
  totalTime: m1.totalTime + m2.totalTime,
  errorsCount: m1.errorsCount + m2.errorsCount
});

class MetricsWriter<A> extends Writer<Metrics, A> {
  static of<A>(value: A): MetricsWriter<A> {
    return new MetricsWriter(value, emptyMetrics);
  }

  static recordOperation(time: number, error: boolean = false): MetricsWriter<void> {
    return new MetricsWriter(undefined, {
      operationCount: 1,
      totalTime: time,
      errorsCount: error ? 1 : 0
    });
  }

  flatMap<B>(fn: (value: A) => MetricsWriter<B>): MetricsWriter<B> {
    const next = fn(this.value);
    return new MetricsWriter(
      next.value,
      combineMetrics(this.getLog(), next.getLog())
    ) as MetricsWriter<B>;
  }
}

// Usage
function processRequest(req: Request): MetricsWriter<Response> {
  const start = Date.now();
  const response = handleRequest(req);
  const duration = Date.now() - start;

  return MetricsWriter.recordOperation(duration, response.error)
    .flatMap(() => MetricsWriter.of(response));
}

const requests = [req1, req2, req3];
const results = requests.map(processRequest);

// Combine all metrics
const totalMetrics = results.reduce(
  (acc, writer) => {
    const [_, metrics] = writer.run();
    return combineMetrics(acc, metrics);
  },
  emptyMetrics
);

console.log(`Processed ${totalMetrics.operationCount} requests`);
console.log(`Total time: ${totalMetrics.totalTime}ms`);
console.log(`Errors: ${totalMetrics.errorsCount}`);
```

### Set of Dependencies

```typescript
type Dependencies = Set<string>;

class DependencyWriter<A> extends Writer<Dependencies, A> {
  static of<A>(value: A): DependencyWriter<A> {
    return new DependencyWriter(value, new Set());
  }

  static require(dep: string): DependencyWriter<void> {
    return new DependencyWriter(undefined, new Set([dep]));
  }

  flatMap<B>(fn: (value: A) => DependencyWriter<B>): DependencyWriter<B> {
    const next = fn(this.value);
    return new DependencyWriter(
      next.value,
      new Set([...this.getLog(), ...next.getLog()])
    ) as DependencyWriter<B>;
  }
}

// Analyze code dependencies
function parseImport(statement: string): DependencyWriter<Import> {
  const match = statement.match(/import .* from ['"](.*)['"]/)!;
  const module = match[1];

  return DependencyWriter.require(module)
    .flatMap(() => DependencyWriter.of({ module, statement }));
}

function analyzeFile(file: string): DependencyWriter<FileInfo> {
  const imports = file.match(/import .* from ['"](.*)['"]/g) || [];

  let writer = DependencyWriter.of<Import[]>([]);

  for (const imp of imports) {
    writer = writer.flatMap(list =>
      parseImport(imp).map(parsed => [...list, parsed])
    );
  }

  return writer.map(imports => ({ file, imports }));
}

const [fileInfo, dependencies] = analyzeFile(sourceCode).run();
console.log('Dependencies:', Array.from(dependencies));
```

## Writer in Other Languages

### Haskell

```haskell
import Control.Monad.Writer

-- Writer with string accumulation
compute :: Int -> Writer String Int
compute x = do
  tell $ "Computing with " ++ show x
  let result = x * 2
  tell $ "Result: " ++ show result
  return result

-- Writer with list accumulation
loggedAdd :: Int -> Int -> Writer [String] Int
loggedAdd a b = do
  let result = a + b
  tell ["Adding " ++ show a ++ " + " ++ show b ++ " = " ++ show result]
  return result

-- Run the computation
main :: IO ()
main = do
  let (result, logs) = runWriter $ do
        x <- loggedAdd 10 5
        y <- loggedAdd x 3
        return y

  print result  -- 18
  mapM_ putStrLn logs
  -- "Adding 10 + 5 = 15"
  -- "Adding 15 + 3 = 18"
```

### Scala

```scala
case class Writer[W, A](value: A, log: W) {
  def map[B](f: A => B): Writer[W, B] =
    Writer(f(value), log)

  def flatMap[B](f: A => Writer[W, B])(implicit m: Monoid[W]): Writer[W, B] = {
    val next = f(value)
    Writer(next.value, m.combine(log, next.log))
  }
}

object Writer {
  def pure[W, A](a: A)(implicit m: Monoid[W]): Writer[W, A] =
    Writer(a, m.empty)

  def tell[W](w: W): Writer[W, Unit] =
    Writer((), w)
}

// Usage with List[String] as log
implicit val listMonoid: Monoid[List[String]] = new Monoid[List[String]] {
  def empty: List[String] = List.empty
  def combine(a: List[String], b: List[String]): List[String] = a ++ b
}

def add(a: Int, b: Int): Writer[List[String], Int] = {
  val result = a + b
  Writer.tell(List(s"Adding $a + $b = $result"))
    .flatMap(_ => Writer.pure(result))
}

val computation = for {
  x <- add(10, 5)
  y <- add(x, 3)
} yield y

val Writer(result, logs) = computation
println(s"Result: $result")
logs.foreach(println)
```

### F#

```fsharp
type Writer<'W, 'A> = Writer of 'A * 'W

module Writer =
    let run (Writer (a, w)) = (a, w)

    let pure (monoid: 'W -> 'W -> 'W) (empty: 'W) a =
        Writer (a, empty)

    let tell w = Writer ((), w)

    let bind (monoid: 'W -> 'W -> 'W) f (Writer (a, w1)) =
        let (Writer (b, w2)) = f a
        Writer (b, monoid w1 w2)

    let map f (Writer (a, w)) =
        Writer (f a, w)

// Usage
let listMonoid = (@)
let emptyList = []

let add a b =
    let result = a + b
    Writer.tell [sprintf "Adding %d + %d = %d" a b result]
    |> Writer.bind listMonoid (fun () -> Writer.pure listMonoid emptyList result)

let computation = writer {
    let! x = add 10 5
    let! y = add x 3
    return y
}

let (result, logs) = Writer.run computation
printfn "Result: %d" result
logs |> List.iter (printfn "%s")
```

## Practical Example: Expression Evaluator with Trace

```typescript
type Expr =
  | { type: 'num'; value: number }
  | { type: 'add'; left: Expr; right: Expr }
  | { type: 'mul'; left: Expr; right: Expr }
  | { type: 'var'; name: string };

type Env = Map<string, number>;

function evaluate(expr: Expr, env: Env): LogWriter<number> {
  switch (expr.type) {
    case 'num':
      return LogWriter.tell(`Evaluating number: ${expr.value}`)
        .flatMap(() => LogWriter.of(expr.value));

    case 'var':
      const value = env.get(expr.name) ?? 0;
      return LogWriter.tell(`Looking up variable ${expr.name} = ${value}`)
        .flatMap(() => LogWriter.of(value));

    case 'add':
      return evaluate(expr.left, env).flatMap(left =>
        evaluate(expr.right, env).flatMap(right => {
          const result = left + right;
          return LogWriter.tell(`Adding ${left} + ${right} = ${result}`)
            .flatMap(() => LogWriter.of(result));
        })
      );

    case 'mul':
      return evaluate(expr.left, env).flatMap(left =>
        evaluate(expr.right, env).flatMap(right => {
          const result = left * right;
          return LogWriter.tell(`Multiplying ${left} * ${right} = ${result}`)
            .flatMap(() => LogWriter.of(result));
        })
      );
  }
}

// Build expression: (x + 5) * 2
const expr: Expr = {
  type: 'mul',
  left: {
    type: 'add',
    left: { type: 'var', name: 'x' },
    right: { type: 'num', value: 5 }
  },
  right: { type: 'num', value: 2 }
};

const env = new Map([['x', 10]]);
const [result, trace] = evaluate(expr, env).run();

console.log('Result:', result);  // 30
console.log('\nExecution trace:');
trace.forEach((step, i) => console.log(`${i + 1}. ${step}`));
// 1. Looking up variable x = 10
// 2. Evaluating number: 5
// 3. Adding 10 + 5 = 15
// 4. Evaluating number: 2
// 5. Multiplying 15 * 2 = 30
```

Perfect for debugging complex evaluators!

## Combining Writer with Other Monads

Writer often needs to work with Maybe, Either, IO:

```typescript
// Writer + Maybe: Computation that might fail, but logs either way
function safeDivide(a: number, b: number): LogWriter<Maybe<number>> {
  if (b === 0) {
    return LogWriter.tell(`Cannot divide ${a} by ${b}`)
      .flatMap(() => LogWriter.of(Maybe.none()));
  }

  const result = a / b;
  return LogWriter.tell(`Dividing ${a} / ${b} = ${result}`)
    .flatMap(() => LogWriter.of(Maybe.of(result)));
}

// Writer + Either: Computation with error details and logging
function validateAge(age: number): LogWriter<Either<string, number>> {
  if (age < 0) {
    return LogWriter.tell(`Validation failed: negative age ${age}`)
      .flatMap(() => LogWriter.of(Either.left('Age cannot be negative')));
  }

  if (age > 150) {
    return LogWriter.tell(`Validation failed: unrealistic age ${age}`)
      .flatMap(() => LogWriter.of(Either.left('Age seems unrealistic')));
  }

  return LogWriter.tell(`Validation passed: age ${age}`)
    .flatMap(() => LogWriter.of(Either.right(age)));
}
```

## When to Use Writer

**Use Writer when:**
- You want to accumulate information during computation
- Logging, audit trails, metrics collection
- Dependency analysis
- You need pure logging (no side effects)
- Accumulation is independent of the main computation

**Don't use Writer when:**
- You need immediate logging (use IO instead)
- Logs are huge (memory concerns)
- You need conditional logging based on later results
- Simple `console.log` would suffice

## Performance Note

Writer accumulates everything in memory. For large logs:

```typescript
// Bad: Logs get very large
function process(items: Item[]): LogWriter<Result> {
  return items.reduce(
    (acc, item) => acc.flatMap(result =>
      processItem(item).map(processed => ({
        ...result,
        items: [...result.items, processed]
      }))
    ),
    LogWriter.of({ items: [] })
  );
}

// Better: Batch logging or use IO
```

## What We've Learned

- Writer accumulates values alongside computation
- Perfect for logging, audit trails, metrics
- Logs collected automatically through chains
- Works with any monoid (combinable type)
- Keeps logging pure and testable
- Composes with other monads

## Coming Up

We've seen Reader (read), State (read/write), and Writer (write). Now let's tackle one of the most common monads in modern programming: Promise/Async - monads for computations in time.

---

**Next: [Chapter 12 - Async/Promise: Monads in Time](12-async-monad.md)**
