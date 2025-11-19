# Chapter 5: List - The Monad You Already Know

Plot twist: You've been using monads this whole time.

## The Revelation

Every time you've written code like this:

```typescript
const numbers = [1, 2, 3];
const doubled = numbers.map(x => x * 2);
// [2, 4, 6]
```

You were using a functor.

Every time you've written:

```typescript
const nested = [[1, 2], [3, 4], [5, 6]];
const flattened = nested.flatMap(x => x);
// [1, 2, 3, 4, 5, 6]
```

You were using a monad.

Arrays are monads. Lists are monads. They've been hiding in plain sight.

## Array as a Container

Think about what an array is:
- It's a container that holds zero or more values
- You can transform values inside it (`map`)
- You can chain operations that produce arrays (`flatMap`)

Sound familiar?

```typescript
// Maybe<T>: zero or one value
Some(5) or None

// Either<L, R>: one value (left or right)
Left(error) or Right(value)

// Array<T>: zero or more values
[], [1], [1, 2, 3]
```

Arrays are just the "multiple values" version of the same pattern.

## The Monad Interface

Let's verify arrays have the monad operations:

```typescript
// 1. Wrap a value
const wrapped = [5];  // or Array.of(5)

// 2. Transform values inside
const doubled = [1, 2, 3].map(x => x * 2);
// [2, 4, 6]

// 3. Chain operations that return arrays
const nested = [1, 2, 3].flatMap(x => [x, x * 10]);
// [1, 10, 2, 20, 3, 30]
```

That's it. Arrays are monads.

## What flatMap Really Does

`flatMap` is the key to understanding list as a monad. It:

1. Maps a function over each element
2. Flattens the result one level

```typescript
// Step by step
const numbers = [1, 2, 3];

// Map produces nested arrays
const mapped = numbers.map(x => [x, x * 10]);
// [[1, 10], [2, 20], [3, 30]]

// Flatten removes one level of nesting
const flattened = mapped.flat();
// [1, 10, 2, 20, 3, 30]

// flatMap does both at once
const result = numbers.flatMap(x => [x, x * 10]);
// [1, 10, 2, 20, 3, 30]
```

Why is this useful? Because functions that return arrays can now chain cleanly:

```typescript
function getChildren(person: Person): Person[] {
  return person.children || [];
}

function getGrandchildren(person: Person): Person[] {
  return person.children?.flatMap(getChildren) || [];
}

function getGreatGrandchildren(person: Person): Person[] {
  return person.children
    ?.flatMap(getChildren)
    ?.flatMap(getChildren) || [];
}

// Without flatMap, you'd have nested arrays:
// [[[grandchild1, grandchild2], [grandchild3]], [...]]
```

## Nondeterminism and Choice

Here's where it gets philosophical. Lists represent **nondeterministic computation** - calculations with multiple possible outcomes.

```typescript
// All possible pairs from two lists
const colors = ["red", "blue"];
const sizes = ["small", "large"];

const products = colors.flatMap(color =>
  sizes.map(size => ({ color, size }))
);

// [
//   { color: "red", size: "small" },
//   { color: "red", size: "large" },
//   { color: "blue", size: "small" },
//   { color: "blue", size: "large" }
// ]
```

The list monad explores **all possible paths**. Each `flatMap` branches into multiple possibilities.

Think of it as a search space:

```typescript
// Find all ways to make change for 50 cents
// using quarters (25¢), dimes (10¢), and nickels (5¢)

type Coin = { name: string; value: number; };
const coins: Coin[] = [
  { name: "quarter", value: 25 },
  { name: "dime", value: 10 },
  { name: "nickel", value: 5 }
];

function makeChange(amount: number, availableCoins: Coin[]): Coin[][] {
  if (amount === 0) return [[]];  // One way: no coins
  if (amount < 0 || availableCoins.length === 0) return [];  // No way

  const [firstCoin, ...restCoins] = availableCoins;

  // Try using the first coin
  const withFirst = makeChange(amount - firstCoin.value, availableCoins)
    .map(coins => [firstCoin, ...coins]);

  // Try without the first coin
  const withoutFirst = makeChange(amount, restCoins);

  return [...withFirst, ...withoutFirst];
}

const solutions = makeChange(50, coins);
// All possible combinations of coins that sum to 50
```

The list monad naturally handles exploring all possibilities.

## List Comprehensions

Many languages have special syntax for working with lists monadically:

### Python

```python
# List comprehension
doubled = [x * 2 for x in [1, 2, 3]]
# [2, 4, 6]

# Nested comprehension (like flatMap)
pairs = [(x, y) for x in [1, 2] for y in ['a', 'b']]
# [(1, 'a'), (1, 'b'), (2, 'a'), (2, 'b')]

# With filtering
evens = [x for x in range(10) if x % 2 == 0]
# [0, 2, 4, 6, 8]
```

### Haskell

```haskell
-- List comprehension
doubled = [x * 2 | x <- [1, 2, 3]]
-- [2, 4, 6]

-- Multiple generators (like flatMap)
pairs = [(x, y) | x <- [1, 2], y <- ['a', 'b']]
-- [(1,'a'), (1,'b'), (2,'a'), (2,'b')]

-- With guards (filtering)
evens = [x | x <- [0..9], even x]
-- [0, 2, 4, 6, 8]

-- This desugars to monadic operations:
pairs = do
  x <- [1, 2]
  y <- ['a', 'b']
  return (x, y)

-- Which is really:
pairs = [1, 2] >>= (\x ->
  ['a', 'b'] >>= (\y ->
    return (x, y)))
```

### Scala

```scala
// For-comprehension
val doubled = for (x <- List(1, 2, 3)) yield x * 2
// List(2, 4, 6)

// Multiple generators
val pairs = for {
  x <- List(1, 2)
  y <- List('a', 'b')
} yield (x, y)
// List((1,'a'), (1,'b'), (2,'a'), (2,'b'))

// With filtering
val evens = for {
  x <- (0 to 9).toList
  if x % 2 == 0
} yield x
// List(0, 2, 4, 6, 8)

// Desugars to flatMap and map:
val pairs = List(1, 2).flatMap(x =>
  List('a', 'b').map(y => (x, y))
)
```

All of these are syntactic sugar for monadic operations on lists!

## Real-World Examples

### Parsing Ambiguous Input

```typescript
// Parse a number that might be in different formats
function parseNumber(s: string): number[] {
  const results: number[] = [];

  // Try parsing as decimal
  const decimal = parseFloat(s);
  if (!isNaN(decimal)) results.push(decimal);

  // Try parsing as hex (if starts with 0x)
  if (s.startsWith("0x")) {
    const hex = parseInt(s, 16);
    if (!isNaN(hex)) results.push(hex);
  }

  // Try parsing as binary (if starts with 0b)
  if (s.startsWith("0b")) {
    const binary = parseInt(s.slice(2), 2);
    if (!isNaN(binary)) results.push(binary);
  }

  return results;
}

// Now we can work with all possible interpretations
parseNumber("10")     // [10]
parseNumber("0x10")   // [16, 16]  (decimal and hex)
parseNumber("0b10")   // [10, 2]   (decimal and binary)

// Find all valid calculations
const inputs = ["10", "0x10", "5"];
const results = inputs.flatMap(parseNumber);
// [10, 16, 16, 5]

const doubled = results.map(x => x * 2);
// [20, 32, 32, 10]
```

### Generating Test Data

```typescript
interface User {
  name: string;
  age: number;
  role: string;
}

const names = ["Alice", "Bob", "Charlie"];
const ages = [25, 30, 35];
const roles = ["admin", "user"];

// Generate all combinations
const testUsers: User[] = names.flatMap(name =>
  ages.flatMap(age =>
    roles.map(role => ({ name, age, role }))
  )
);

// 3 names × 3 ages × 2 roles = 18 test users
console.log(testUsers.length); // 18
```

### Graph Traversal

```typescript
interface Node {
  id: string;
  neighbors: Node[];
}

// Find all nodes reachable in N steps
function reachableIn(start: Node, steps: number): Node[] {
  if (steps === 0) return [start];

  return start.neighbors.flatMap(neighbor =>
    reachableIn(neighbor, steps - 1)
  );
}

// Find all paths of length N
function pathsOfLength(start: Node, length: number): Node[][] {
  if (length === 0) return [[start]];

  return start.neighbors.flatMap(neighbor =>
    pathsOfLength(neighbor, length - 1).map(path =>
      [start, ...path]
    )
  );
}
```

## The List Monad in Different Languages

### JavaScript/TypeScript

```typescript
// Native support with flatMap
[1, 2, 3].flatMap(x => [x, x * 10])
// [1, 10, 2, 20, 3, 30]

// Comprehension-style with generators (kind of)
function* comprehension() {
  for (const x of [1, 2, 3]) {
    for (const y of ['a', 'b']) {
      yield [x, y];
    }
  }
}
Array.from(comprehension())
// [[1,'a'], [1,'b'], [2,'a'], [2,'b'], [3,'a'], [3,'b']]
```

### Haskell

```haskell
-- Using >>= (bind/flatMap)
[1, 2, 3] >>= \x -> [x, x * 10]
-- [1,10,2,20,3,30]

-- Using do-notation
result = do
  x <- [1, 2, 3]
  y <- ['a', 'b']
  return (x, y)
-- [(1,'a'),(1,'b'),(2,'a'),(2,'b'),(3,'a'),(3,'b')]

-- The List type is defined:
-- data List a = Nil | Cons a (List a)
-- instance Monad List where
--   return x = [x]
--   xs >>= f = concat (map f xs)
```

### Scala

```scala
// Using flatMap
List(1, 2, 3).flatMap(x => List(x, x * 10))
// List(1, 10, 2, 20, 3, 30)

// Using for-comprehension
val result = for {
  x <- List(1, 2, 3)
  y <- List('a', 'b')
} yield (x, y)
// List((1,a), (1,b), (2,a), (2,b), (3,a), (3,b))
```

### Rust

```rust
// Rust doesn't have built-in list monad, but we can use iterators

// Using flat_map
let result: Vec<i32> = vec![1, 2, 3]
    .into_iter()
    .flat_map(|x| vec![x, x * 10])
    .collect();
// [1, 10, 2, 20, 3, 30]

// Cartesian product
let result: Vec<(i32, char)> = vec![1, 2, 3]
    .into_iter()
    .flat_map(|x| vec!['a', 'b'].into_iter().map(move |y| (x, y)))
    .collect();
// [(1, 'a'), (1, 'b'), (2, 'a'), (2, 'b'), (3, 'a'), (3, 'b')]
```

### F#

```fsharp
// Using List.collect (flatMap)
[1; 2; 3] |> List.collect (fun x -> [x; x * 10])
// [1; 10; 2; 20; 3; 30]

// Using list comprehension
let result = [
    for x in [1; 2; 3] do
        for y in ['a'; 'b'] do
            yield (x, y)
]
// [(1, 'a'); (1, 'b'); (2, 'a'); (2, 'b'); (3, 'a'); (3, 'b')]
```

## Filtering: The Guard Operation

Lists add one more operation: filtering.

```typescript
const numbers = [1, 2, 3, 4, 5, 6];

// Keep only evens
const evens = numbers.filter(x => x % 2 === 0);
// [2, 4, 6]

// Combine with flatMap
const result = numbers
  .filter(x => x % 2 === 0)
  .flatMap(x => [x, x * 10]);
// [2, 20, 4, 40, 6, 60]
```

In monadic terms, `filter` with an always-empty list acts as a "guard" - it prunes branches in the nondeterministic computation.

```haskell
-- Haskell has guard function
import Control.Monad (guard)

result = do
  x <- [1..6]
  guard (even x)  -- Skip if x is odd
  return (x, x * 10)
-- [(2,20),(4,40),(6,60)]

-- guard is defined as:
-- guard True  = [()]  -- Continue
-- guard False = []    -- Stop this branch
```

## Why This Matters

Understanding lists as monads reveals that:

1. **flatMap is fundamental** - It's not just for arrays, it's a pattern for chaining
2. **Comprehensions are monadic** - List comprehensions in Python, Haskell, Scala are all sugar for flatMap
3. **Nondeterminism is natural** - Multiple results aren't special, they're just another monad
4. **You already know this** - You've been thinking monadically whenever you've used arrays

## The Pattern Solidifies

Let's compare our three monads so far:

```typescript
// Maybe: zero or one value
Some(5).flatMap(x => Some(x * 2))     // Some(10)
None.flatMap(x => Some(x * 2))        // None

// Either: one value (left or right)
Right(5).flatMap(x => Right(x * 2))   // Right(10)
Left("error").flatMap(x => Right(x * 2))  // Left("error")

// Array: zero or more values
[5].flatMap(x => [x * 2])             // [10]
[].flatMap(x => [x * 2])              // []
[1, 2, 3].flatMap(x => [x, x * 2])    // [1, 2, 2, 4, 3, 6]
```

Same pattern, different contexts:
- Maybe: possibility of absence
- Either: possibility of error
- Array: possibility of multiple results

All use `flatMap` to chain operations. All handle their context automatically.

## Exercises

1. Implement `flatMap` for arrays from scratch:

```typescript
function flatMap<T, U>(
  array: T[],
  fn: (value: T) => U[]
): U[] {
  // Your implementation
}
```

2. Generate all possible 2-character passwords from lowercase letters:

```typescript
const letters = "abcdefghijklmnopqrstuvwxyz".split("");
// Result should be ["aa", "ab", "ac", ..., "zz"]
```

3. Find all ways to partition a number into sums:

```typescript
function partitions(n: number): number[][] {
  // e.g., partitions(3) = [[3], [2,1], [1,2], [1,1,1]]
  // Your implementation
}
```

## What We've Learned

- Arrays/Lists are monads
- `flatMap` chains operations that produce arrays
- List monad represents nondeterministic computation
- List comprehensions are syntactic sugar for monadic operations
- You've been using monads every time you've used arrays

## Coming Up

We've seen three specific monads. Now it's time to step back and see the pattern that connects them all. What makes a monad a monad?

---

## Exercise Answers

```typescript
// 1. flatMap implementation
function flatMap<T, U>(
  array: T[],
  fn: (value: T) => U[]
): U[] {
  const result: U[] = [];
  for (const item of array) {
    const mapped = fn(item);
    for (const value of mapped) {
      result.push(value);
    }
  }
  return result;
}

// Or more concisely:
function flatMap<T, U>(
  array: T[],
  fn: (value: T) => U[]
): U[] {
  return array.reduce(
    (acc, item) => [...acc, ...fn(item)],
    [] as U[]
  );
}

// 2. All 2-character passwords
const letters = "abcdefghijklmnopqrstuvwxyz".split("");
const passwords = letters.flatMap(first =>
  letters.map(second => first + second)
);
// Or with nested flatMap:
const passwords2 = letters.flatMap(first =>
  letters.flatMap(second =>
    [first + second]
  )
);

// 3. Number partitions
function partitions(n: number): number[][] {
  if (n === 0) return [[]];
  if (n < 0) return [];

  // Try starting with each number from 1 to n
  return Array.from({ length: n }, (_, i) => i + 1)
    .flatMap(first =>
      partitions(n - first).map(rest =>
        [first, ...rest]
      )
    );
}
```

---

**Next: [Chapter 6 - The Pattern Emerges](06-the-pattern.md)**
