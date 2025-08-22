# Mantra Language Specification v0.1

## Table of Contents
1. [Overview](#overview)
2. [Lexical Elements](#lexical-elements)
3. [Data Types](#data-types)
4. [Syntax](#syntax)
5. [Type System](#type-system)
6. [Memory Model](#memory-model)
7. [Concurrency](#concurrency)
8. [Module System](#module-system)

## Overview

Mantra is a statically-typed, compiled programming language designed to combine the safety of Rust, the simplicity of Python, and the performance of C++. It features gradual typing, memory safety, built-in concurrency, and cross-platform compilation.

## Lexical Elements

### Keywords
```
and       async      await      break      case       catch
class     const      continue   def        else       enum
export    false      finally    fn         for        from
if        import     in         let        loop       match
mod       mut        not        null       or         pub
return    self       struct     super      throw      trait
true      try        type       union      use        var
where     while      with       yield
```

### Operators
```
Arithmetic: +, -, *, /, %, **
Comparison: ==, !=, <, <=, >, >=
Logical: and, or, not
Bitwise: &, |, ^, ~, <<, >>
Assignment: =, +=, -=, *=, /=, %=
Other: ->, =>, ?, !!, ?., !!
```

### Literals
```zenith
// Numeric literals
42          // Integer
42.0        // Float  
0x2A        // Hexadecimal
0b101010    // Binary
0o52        // Octal

// String literals
"Hello"     // String
'c'         // Character
r"raw\string"  // Raw string
f"Hello {name}"  // Format string

// Boolean and null
true, false, null
```

## Data Types

### Primitive Types
```zenith
// Integer types
i8, i16, i32, i64, i128    // Signed integers
u8, u16, u32, u64, u128    // Unsigned integers
int                        // Platform-dependent signed integer
uint                       // Platform-dependent unsigned integer

// Floating point
f32, f64                   // IEEE 754 floating point
float                      // Default floating point (f64)

// Other primitives
bool                       // Boolean
char                       // Unicode character
string                     // UTF-8 string
```

### Composite Types
```zenith
// Arrays (fixed size)
Array<T, N>               // Fixed-size array
[i32; 5]                  // Array of 5 i32s

// Slices (dynamic size)
Slice<T>                  // Dynamic array slice
[i32]                     // Slice of i32s

// Tuples
(i32, string, bool)       // Tuple type

// Structs
struct Point {
    x: f64,
    y: f64
}

// Enums
enum Color {
    Red,
    Green, 
    Blue,
    RGB(u8, u8, u8)
}

// Optional types
Option<T>                 // T or null
Result<T, E>              // T or error E
```

## Syntax

### Variable Declarations
```zenith
// Immutable by default
let x = 42
let name: string = "Alice"

// Mutable variables
mut count = 0
count += 1

// Constants
const PI = 3.14159
const MAX_SIZE: int = 1000
```

### Functions
```zenith
// Basic function
fn add(a: i32, b: i32) -> i32 {
    return a + b
}

// Expression function
fn multiply(a: i32, b: i32) -> i32 => a * b

// Generic function
fn swap<T>(a: mut T, b: mut T) {
    let temp = a
    a = b
    b = temp
}

// Async function
async fn fetch_data(url: string) -> Result<string, Error> {
    let response = http.get(url).await?
    return response.text().await
}
```

### Control Flow
```zenith
// If expressions
let result = if condition {
    "true branch"
} else {
    "false branch"
}

// Pattern matching
match value {
    0 => "zero",
    1..=5 => "small",
    x if x > 100 => "large",
    _ => "other"
}

// Loops
for item in collection {
    print(item)
}

loop {
    if should_break() { break }
}

while condition {
    // do something
}
```

### Error Handling
```zenith
// Result type for recoverable errors
fn divide(a: f64, b: f64) -> Result<f64, string> {
    if b == 0.0 {
        return Err("Division by zero")
    }
    return Ok(a / b)
}

// Using ? operator for error propagation
fn calculate() -> Result<f64, string> {
    let x = divide(10.0, 2.0)?
    let y = divide(x, 3.0)?
    return Ok(y)
}

// Try-catch for exceptions
try {
    risky_operation()
} catch e: NetworkError {
    handle_network_error(e)
} catch e: ParseError {
    handle_parse_error(e)
} finally {
    cleanup()
}
```

## Type System

### Type Inference
```zenith
// Compiler infers types when possible
let numbers = [1, 2, 3, 4]        // Array<i32>
let pairs = numbers.map(x => (x, x * 2))  // Array<(i32, i32)>

// Explicit typing when needed
let data: Map<string, Any> = load_config()
```

### Gradual Typing
```zenith
// Start with dynamic types
let data = parse_json(input)
data.process()  // Dynamic dispatch

// Add types for safety
let config: Config = data.as<Config>()
config.validate()  // Compile-time checks
```

### Traits (Interfaces)
```zenith
trait Display {
    fn display(self) -> string
}

trait Add<T> {
    type Output
    fn add(self, other: T) -> Self::Output
}

// Implementation
impl Display for Point {
    fn display(self) -> string {
        return f"({self.x}, {self.y})"
    }
}
```

### Generics
```zenith
// Generic structs
struct Container<T> {
    value: T
}

// Generic functions with constraints
fn process<T: Display + Clone>(item: T) -> T {
    print(item.display())
    return item.clone()
}

// Associated types
trait Iterator {
    type Item
    fn next(mut self) -> Option<Self::Item>
}
```

## Memory Model

### Ownership System
```zenith
// Ownership transfer
let data = String::from("hello")
let moved_data = data  // data is no longer accessible

// Borrowing (references)
fn process(text: &string) {  // Immutable borrow
    print(text.length())
}

fn modify(text: &mut string) {  // Mutable borrow
    text.push("!")
}

let mut message = String::from("hello")
process(&message)      // Borrow
modify(&mut message)   // Mutable borrow
print(message)         // Ownership returned
```

### Lifetimes
```zenith
// Explicit lifetimes when needed
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.length() > y.length() { x } else { y }
}

// Lifetime elision in common cases
fn first_word(s: &str) -> &str {  // Lifetime inferred
    // implementation
}
```

### Memory Management
```zenith
// Stack allocation (default)
let point = Point { x: 1.0, y: 2.0 }

// Heap allocation when needed
let boxed = Box::new(large_data)

// Reference counting for shared ownership
let shared = Rc::new(data)
let another_ref = shared.clone()

// Automatic cleanup
// No manual memory management required
```

## Concurrency

### Async/Await
```zenith
// Async functions return Future<T>
async fn fetch_url(url: string) -> Result<string, Error> {
    let client = HttpClient::new()
    let response = client.get(url).await?
    return response.text().await
}

// Concurrent execution
async fn fetch_multiple(urls: Array<string>) -> Array<Result<string, Error>> {
    let futures = urls.map(|url| fetch_url(url))
    return futures.join_all().await
}
```

### Channels and Message Passing
```zenith
// Channel communication
let (sender, receiver) = channel<i32>()

spawn {
    for i in 0..10 {
        sender.send(i).await
    }
}

spawn {
    while let Some(value) = receiver.recv().await {
        print(value)
    }
}
```

### Parallel Computing
```zenith
// Parallel iteration
let results = data
    .par_iter()
    .filter(|x| x.is_valid())
    .map(|x| x.process())
    .collect()

// GPU computing
#[gpu]
fn matrix_multiply(a: &Tensor<f32>, b: &Tensor<f32>) -> Tensor<f32> {
    // Automatically compiled to GPU kernel
    a.dot(b)
}
```

## Module System

### Module Declaration
```mantra
// In file: math/vector.mantra
mod vector {
    pub struct Vec3 {
        pub x: f64,
        pub y: f64, 
        pub z: f64
    }
    
    impl Vec3 {
        pub fn new(x: f64, y: f64, z: f64) -> Vec3 {
            Vec3 { x, y, z }
        }
        
        pub fn magnitude(self) -> f64 {
            (self.x * self.x + self.y * self.y + self.z * self.z).sqrt()
        }
    }
}
```

### Importing
```mantra
// Import specific items
import math.vector.Vec3
import std.collections.{HashMap, HashSet}

// Import with alias
import very.long.module.name as short

// Import all public items
import math.vector.*

// Conditional imports
#[cfg(feature = "gpu")]
import gpu.compute.*
```

### Package Management
```mantra
// Package.mantra file
package {
    name: "my-awesome-library",
    version: "1.0.0",
    author: "Developer Name",
    license: "MIT",
    
    dependencies: {
        "http-client": "^2.0",
        "json": "1.5.0"
    },
    
    dev_dependencies: {
        "test-framework": "^1.0"
    }
}
```

This specification provides the foundation for implementing the Mantra programming language according to the roadmap outlined in the README.
