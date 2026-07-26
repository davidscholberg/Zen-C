<div align="center">
  <h1>Zen C</h1>
  <h3>Modern Ergonomics. Zero Overhead. Pure C.</h3>
  <br>
  <p><em>Write like a high-level language, run like C.</em></p>
</div>

---

## Overview

**Zen C** is a modern systems programming language that compiles to `C89` for maximum portability. It provides a rich feature set including pattern matching, generics, traits, and manual memory management with deferred cleanup, all while maintaining 100% C ABI compatibility.

## Community

Join the discussion, share demos, ask questions, or report bugs in the official Zen C Discord server!

- Discord: [Join here](https://discord.com/invite/q6wEsCmkJP)

## Quick Start

### Installation

Use cmake to build the Release target or use the following make targets:

```bash
git clone https://github.com/davidscholberg/Zen-C.git
cd Zen-C
make gen-release
make release
```

On Windows, use Visual Studio to build the project, or use the following console commands (you must have cmake installed and be inside an MSVC dev console):

```cmd
git clone https://github.com/davidscholberg/Zen-C.git
cd Zen-C
build.bat gen-release
build.bat release
```

The resulting executable will be in `build/Release/`.

### Usage

```bash
# Compile and run
zc run hello.zc

# Build executable
zc build hello.zc -o hello
```

### Environment Variables

You can set the `ZC_ROOT` environment variable to specify the location of the Standard Library (standard imports like `import "std/vec.zc"`). This allows you to run `zc` from any directory.

```bash
export ZC_ROOT=/path/to/Zen-C
```

---

## Language Reference

### Guiding Principles

The following principles are meant to guide the design of the language rather than be absolute rules that must be adhered to at all times.

* Intuitive syntax.
* Reduce boilerplate while remaining maximally expressive.
* No hidden memory allocations.
* Distinct language features should not share syntax with eachother.
    * For example, operator overloading allows operators to behave differently (either as a primitive operator or as a function call) while having identical syntax. As a rule we want to avoid this sort of shared syntax.

### Variables and Constants

#### Constants
`#define` preprocessor macros are passed through to the generated C.

```zc
#define MAX_SIZE 1024
let buffer: char[MAX_SIZE];
```

#### Variables
Variables are named storage locations in memory. Can be mutable or read-only (`const`).

```zc
let x: int = 10;        // Mutable
x = 20;                 // OK

let y: const int = 10;  // Read-only (const qualified type)
// y = 20;              // Error: cannot assign to const
```

### Primitive Types
All primitive types available in `C89` (i.e. `int`, `short`, `char`, etc.) are available in Zen C. Additionally, you may use types from newer C revisions if you plan on compiling the generated C code with a later C standard.

Zen C provides some convenient type aliases as well:

| Type | C Equivalent | Description |
|:---|:---|:---|
| `uchar` | 'unsigned char` | unsigned char (Interop) |
| `ushort` | `unsigned short` | unsigned short (Interop) |
| `uint` | `unsigned int` | unsigned int (Interop) |
| `ulong` | `unsigned long` | unsigned long (Interop) |
| `ulonglong` | `unsigned long long` | unsigned long long (Interop) |
| `i8`, `i16`, `i32`, `i64` | `int8_t`, etc. | Signed fixed-width integers (only defined on supported arches) |
| `u8`, `u16`, `u32`, `u64` | `uint8_t`, etc. | Unsigned fixed-width integers (only defined on supported arches) |
| `f32`, `f64`  | `float`, `double` | Fixed-width floating point numbers (only defined on supported arches) |
| `bool` | `bool` | `true` or `false` |
| `isize`, `usize` | `ptrdiff_t`, `size_t` | Pointer-sized integers |

#### Literals
- **Integers**: Decimal (`123`), Hex (`0xFF`), Octal (`0o755`), Binary (`0b1011`).
  - *Note*: Numbers with leading zeros are treated as decimal (`0123` is `123`), unlike C.
  - *Note*: Numbers can contain underscores for readability (`1_000_000`, `0b_1111_0000`).
- **Floats**: Standard (`3.14`), Scientific (`1e-5`, `1.2E3`). Floating point numbers also support underscores (`3_14.15_92`).

> [!IMPORTANT]
> **Best Practices for Portable Code**
>
> - Only use fixed-width integer/float types when writing code for specific arches, otherwise stick to standard C integer/float types.
> - Use `usize` for array indexing and pointer addition, and use `isize` for pointer subtraction.

### Aggregate Types

#### Arrays
Fixed-size arrays with value semantics.
```zc
#define SIZE 5;
let ints: int[SIZE] = [1, 2, 3, 4, 5];
```

#### Structs
Data structures with optional bitfields.
```zc
struct Point {
    x: int;
    y: int;
}

// Struct initialization
let p: Point = { x: 10, y: 20 };

// Bitfields (only supported for `int`, both signed and unsigned)
struct Flags {
    valid: int : 1;
    mode:  int : 3;
}
```

> [!NOTE]
> Fields can be accessed via `.` even on pointer to a struct object (Auto-Dereference).

#### Opaque Structs
You can define a struct as `opaque` to restrict access to its fields to the defining module only, while still allowing the struct to be allocated on the stack (size is known).

```zc
// In user.zc
opaque struct User {
    id: int;
    name: string;
}

fn new_user(name: string) -> User {
    return {id: 1, name: name}; // OK: Inside module
}

// In main.zc
import "user.zc";

fn main() {
    let u: User = new_user("Alice");
    // let id: int = u.id; // Error: Cannot access private field 'id'
}
```

#### Enums
Tagged unions (Sum types) capable of holding data.
```zc
struct Rect {
    width: float;
    height: float;
}

enum Shape {
    Circle(float),      // Holds radius
    Rect(Rect),         // Holds struct which contains width, height
    Point               // No data
}
```

#### Unions
Standard C unions (unsafe access).
```zc
union Data {
    i: int;
    f: float;
}
```

#### Function Pointers
Function pointers hold addresses to callable functions and may be "called" directly.

```zc
fn add(a: int, b: int) -> int {
    return a + b;
}

fn main() {
    let f: fn(int, int) -> int = add;
    println "{f(3, 2)}"; // output: 5
}
```

#### Type Aliases
Create a new name for an existing type.
```zc
alias ID = int;
alias PointMap = Map<string, Point>
alias OpFunc = fn(int, int) -> int
```

#### Opaque Type Aliases
You can define a type alias as `opaque` to create a new type that is distinct from its underlying type outside of the defining module. This provides strong encapsulation and type safety without the runtime overhead of a wrapper struct.

```zc
// In library.zc
opaque alias Handle = int;

fn make_handle(v: int) -> Handle {
    return v; // Implicit conversion allowed inside module
}

// In main.zc
import "library.zc";

fn main() {
    let h: Handle = make_handle(42);
    // let i: int = h; // Error: Type validation failed
    // let h2: Handle = 10; // Error: Type validation failed
}
```

### Functions

Defines a callable function with input parameters and an output.

```zc
fn add(a: int, b: int) -> int {
    return a + b;
}

#### Const Arguments
Function arguments can be marked as `const` to enforce read-only semantics.

```zc
fn print_val(v: const int) {
    // v = 10; // Error: Cannot assign to const variable
    println "{v}";
}
```

#### Default Arguments
Functions can define default values for trailing arguments. These can be literals, expressions, or valid Zen C code (like struct constructors).
```zc
// Simple default value
fn increment(val: int, amount: int = 1) -> int {
    return val + amount;
}

// Expression default value (evaluated at call site)
fn offset(val: int, pad: int = 10 * 2) -> int {
    return val + pad;
}

// Struct default value
struct Config { debug: bool; }
fn init(cfg: Config = { debug: true }) {
    if cfg.debug { println "Debug Mode"; }
}

fn main() {
    increment(10);      // 11
    offset(5);          // 25
    init();             // Prints "Debug Mode"
}
```

#### Variadic Functions
Functions can accept a variable number of arguments using `...` and the `va_list` type.
```zc
fn log(lvl: int, fmt: char*, ...) {
    let ap: va_list;
    va_start(ap, fmt);
    vprintf(fmt, ap); // Use C stdio
    va_end(ap);
}
```

### Control Flow

#### Conditionals
```zc
if x > 10 {
    println "Large";
} else if x > 5 {
    println "Medium";
} else {
    println "Small";
}

// Ternary
let y = x > 10 ? 1 : 0;

// If-Expression (for complex conditions)
let category = if (x > 100) { "huge" } else if (x > 10) { "large" } else { "small" };
```

#### Pattern Matching
Powerful alternative to `switch`.
```zc
match val {
    1         => { print "One" },
    6, 7, 8   => { print "Six to Eight" },    // OR with comma
    10 ..< 15 => { print "10 to 14" },        // Exclusive range
    20 ..= 25 => { print "20 to 25" },        // Inclusive range
    _         => { print "Other" },           // Default match
}

// Destructuring Enums
match shape {
    Shape::Circle(r)   => { println "Radius: {r}" },
    Shape::Rect(rect)  => { println "Area: {rect.w * rect.h}" },
    Shape::Point       => { println "Point" },
}
```

#### Loops
```zc
// Range
for i in 0..<10 { ... }     // Exclusive (0 to 9)
for i in 0..=10 { ... }     // Inclusive (0 to 10)
for i in 0..<10 step 2 { ... }
for i in 10..=0 step -1 { ... }  // Descending loop

// Repeat N times
for _ in 0..<5 { ... }

// While
while x < 10 { ... }

// Do-While
do { ... } while x < 10;

// Infinite loop with label
loop outer {
    if done { break outer; }
}
```

### Operators

#### Standard Operators

| Category | Operator |
|:---|:---|
| **Arithmetic** | `+`, `-`, `*`, `/`, `%`, `**` |
| **Comparison** | `==`, `!=`, `<`, `>`, `<=`, `>=` |
| **Bitwise** | `&`, `\|`, `^`, `<<`, `>>` |
| **Unary** | `-`, `!`, `~` |

#### Syntactic Sugar

| Operator | Name | Description |
|:---|:---|:---|
| `??` | Null Coalescing | `val ?? default` returns `default` if `val` is NULL (pointers) |
| `??=` | Null Assignment | `val ??= init` assigns if `val` is NULL |
| `?` | Try Operator | `res?` returns error if present (Result/Option types) |

**Auto-Dereference**:
Pointer field access (`ptr.field`) and method calls (`ptr.method()`) automatically dereference the pointer, equivalent to `(*ptr).field`.

### Printing and String Interpolation

Zen C provides shorthands for printing interpolated strings to the console.

| Keyword | Description |
|:---|:---|
| `print "..."` | Prints to `stdout` without a trailing newline. |
| `println "..."` | Prints to `stdout` **with** a trailing newline. |
| `eprint "..."` | Prints to `stderr` without a trailing newline. |
| `eprintln "..."` | Prints to `stderr` **with** a trailing newline. |

You can embed expressions directly into the string literals by prefixing the string literal with `f` and using `{}` syntax inside the string. Note that this interpolation method will only work with the above printing shorthands.

```zc
let x: int = 42;
let name: char* = "Zen";
println f"Value: {x}, Name: {name}";
```

**Escaping Braces**: Use `{{` to produce a literal `{` and `}}` for a literal `}`:

```zc
println f"JSON: {{\"key\": \"value\"}}";
// Output: JSON: {"key": "value"}
```

To print a string with no interpolation, omit the `f` prefix:

```zc
println "JSON: {\"key\": \"value\"}"
// Output: JSON: {"key": "value"}
```

### Resource Management

In Zen C, resources (memory, file descriptors, etc) are manually managed. Zen C offers the `defer` keyword to defer the execution of resource cleanup to block exit.

Defer statements are executed in LIFO (last-in, first-out) order.

```zc
let f: FILE = fopen("file.txt", "r");
defer fclose(f);
```

> [!WARNING]
> To prevent undefined behavior, control flow statements (`return`, `break`, `continue`, `goto`) are **not allowed** inside a `defer` block.

### Object Oriented Programming

#### Methods
Define methods on types using `impl`.
```zc
impl Point {
    fn dist(self) -> float {
        return sqrt(self.x * self.x + self.y * self.y);
    }
}
```

#### Traits
Define shared behavior.
```zc
struct Circle { radius: f32; }

trait Drawable {
    fn draw(self);
}

impl Drawable for Circle {
    fn draw(self) { ... }
}

let circle: Circle = { radius: 1 };
let drawable: Drawable = maketrait &circle;
drawable.draw();
let circle_ptr: Circle* = (Circle*)drawable.obj_ptr;
```

Note that trait objects are structs with two members: a void pointer to an object of a concrete type that implements the trait, and a pointer to the object type's vtable. The keyword `maketrait` is a special constructor for trait objects that automatically sets the correct vtable for the type of the provided object.

### Generics

Type-safe templates for Structs and Functions.

```zc
// Generic Struct
struct Box<T> {
    item: T;
}

// Generic Function
fn identity<T>(val: T) -> T {
    return val;
}

// Multi-parameter Generics
struct Pair<K, V> {
    key: K;
    value: V;
}
```

### C Interoperability

Zen C allows you to include C headers and use symbols defined within them. Currently Zen C does not implicitly type check symbols included from a C header, but if type checking of a C symbol is desired, you can define the type of the symbol with the keyword `ffidef`.

```zc
#include <stdio.h> // Includes are emitted directly to the generated C

// Define signature for type checking of C symbol.
ffidef printf: fn(char*, ...) -> int;

fn main() {
    printf("Hello FFI: %d\n", 42); // Type checked by Zen C
}
```

### Unit Testing Framework

Zen C features a built-in testing framework with **per-test isolation**, **named output**, and **non-fatal assertions**.

#### Syntax
A `test` block contains a descriptive name and a body of code to execute. Tests do not require a `main` function to run.

```zc
test "descriptive name" {
    let a = 3;
    assert(a > 0, "a should be positive");
}
```

#### Running Tests
```bash
zc run my_file.zc
```

Output shows each test by name:
```
  TEST: descriptive name ... OK
  TEST: another test ... FAIL

1 test(s) failed
```

#### Assertions
| Function | Behavior |
|:---|:---|
| `assert(cond, msg)` | Records failure, continues to next test (no longer aborts) |
| `expect(cond, msg)` | Non-fatal — records failure but continues within the same test |

Use `assert` for critical checks that should stop the current test, and `expect` when you want to verify multiple conditions without short-circuiting:

```zc
test "example" {
    expect(result != null, "result should not be null");
    expect(result.code == 200, "status should be 200");
    // both run even if the first fails
}
```

#### Exit Code
The binary exits with the number of failed tests (0 = all passed).

---

## Standard Library

Zen C includes a standard library (`std`) covering essential functionality.

### Key Modules

| Module | Description | Docs |
| :--- | :--- | :--- |
| **`std/vec.zc`** | Growable dynamic array `Vec<T>`. | [Docs](docs/std/vec.md) |
| **`std/string.zc`** | Heap-allocated `String` type with UTF-8 support. | [Docs](docs/std/string.md) |
| **`std/queue.zc`** | FIFO queue (Ring Buffer). | [Docs](docs/std/queue.md) |
| **`std/map.zc`** | Generic Hash Map `Map<V>`. | [Docs](docs/std/map.md) |
| **`std/fs.zc`** | File system operations. | [Docs](docs/std/fs.md) |
| **`std/io.zc`** | Standard Input/Output (`print`/`println`). | [Docs](docs/std/io.md) |
| **`std/option.zc`** | Optional values (`Some`/`None`). | [Docs](docs/std/option.md) |
| **`std/result.zc`** | Error handling (`Ok`/`Err`). | [Docs](docs/std/result.md) |
| **`std/path.zc`** | Cross-platform path manipulation. | [Docs](docs/std/path.md) |
| **`std/env.zc`** | Process environment variables. | [Docs](docs/std/env.md) |
| **`std/time.zc`** | Time measurement and sleep. | [Docs](docs/std/time.md) |
| **`std/json.zc`** | JSON parsing and serialization. | [Docs](docs/std/json.md) |
| **`std/stack.zc`** | LIFO Stack `Stack<T>`. | [Docs](docs/std/stack.md) |
| **`std/set.zc`** | Generic Hash Set `Set<T>`. | [Docs](docs/std/set.md) |
| **`std/process.zc`** | Process execution and management. | [Docs](docs/std/process.md) |

---

## Compiler Support & Compatibility

Zen C is designed to produce C89 compatible code for maximum portability. You can of course include headers that conform to other standards and compile the generated code with whatever C standard you need.

---

<div align="center">
  <p>
    Copyright © 2026 Zen C Programming Language.<br>
    Start your journey today.
  </p>
  <p>
    <a href="https://discord.com/invite/q6wEsCmkJP">Discord</a> •
    <a href="https://github.com/zenc-lang/zenc">GitHub</a> •
  </p>
</div>
