---
date: "2023-11-04T18:13:39+02:00"
title: Rust - quick start for C++ developers
---

### Basic types

    In order to fully understand what

When all writes are visible instantly to all subsequent reads in other threads then this is strict memory-ordering.


### Project structure

With rust naming we have (source: doc.rust-lang.org):
- Packages: A Cargo feature that lets you build, test, and share crates
- Crates: A tree of modules that produces a library or executable
- Modules and use: Let you control the organization, scope, and privacy of paths
- Paths: A way of naming an item, such as a struct, function, or module

Packages are set of crates providing sets o functionalities and may be considered as a project or a package depending on what type of dependency management you use when programming in C++.

A crate may be considered as a 'compilation unit' but can be formed from one or more modules. Rust does not differentiate between interface files (hpp) and implementation files (cpp) but have a way to tell that a function is public (exported using C++20 terms)

Modules that are declared in crate root file (`src/lib.rs` or `src/main.rs`) will be searched in root directory under `src/module_name.rs` or in `src/module_name/mod.rs`. Modules declared in other than root file are submodules that similar to modules partitions in C++20.

Rust introduces visibility/privacy scopes - module cannot se what other module have defined until explicitly annotated by `pub` keyword. All definitions without the keyword are private by default.

Interesting fact is that parent modules cannot access child module code until it is `pub`. But it is possible in opposite direction: child modules can access what is defined in parent module.

```rust
mod dog {
    fn move() {
        tail::move() // OK - as expected (use of public interface)
    }
    fn jump() {}

    mod tail {
        pub fn move() {
            crate::dog::jump() // More specific code has access to it's parent
        }
        // Private stuff...
    }
}
```

### Safety

Rust is claimed to be safe by default - there are features that have to be disabled to lose safety net Rust provides. How does it work in practice?

#### Fundamental types - overflow

In `C++` code as following results with type overflow with no warning  
```c++
uint8_t var = 250;
var = var + 10;
std::cout << +var << "\n";
```
compiled with: `-std=c++17 -Wall -Wpedantic`

Produces *no warnings* and outputs: `4` when executed.

Similar code in Rust also produces no warning during compilation but panics when compiled in `debug` mode:

```rust
let mut var : u8 = 250;
var = var + 10;
println!("{}", var);
return;
```

... but when compiled in release the same behaviour as in C++ - outputs `4`

#### Fundamental types - integral promotion

Integral types can be implicitly converted to another wider integral types (a type that can represent a larger set of values)

"The arguments of the following arithmetic operators undergo implicit conversions for the purpose of obtaining the common real type, which is the type in which the calculation is performed" - https://en.cppreference.com/w/c/language/conversion:

- binary arithmetic *, /, %, +, -
- relational operators <, >, <=, >=, ==, !=
- binary bitwise arithmetic &, ^, |,
- the conditional operator ?:

How does it work in practice?

```c++
int var1 = -10;
unsigned int var2 = 250;
std::cout << (var1 < var2) << "\n";
```
Outputs: `0` and when compiling with gcc with flag `-Wall` it produces warning:
```c++
warning: comparison of integer expressions of different signedness: 'int' and 'unsigned int' [-Wsign-compare]
   24 |     std::cout << (var1 < var2) << "\n";
```

Similar code in Rust won't compile either in `debug` and in `release` mode:
```c++
let var1 : i8 = -10;
let var2 : u8 = 250;
println!("{}", (var1 <  var2));  // Won't compile (expected `i8`, found `u8`)
```

#### Fundamental types - default argument promotion

C++ functions arguments are promoted according to the rules mentioned above, sometimes leading to confusion:

```c++
void foo(int var1) {printf("int");}
void foo(unsigned int var2) {printf("unsigned");}

int main() {
    char v1{};
    foo(v1);  // Which overload will be called? (will output int)
}
```

Rust however is a little bit safe here as similar code won't compile:
```rust
fn foo(var: i32) { println!("{var}"); }
// fn foo(var: u32) { println!("{var}"); } - Rust does not support overloading

fn main() {
    let var1 : u8 = 10;
    foo(var1);  // Won't compile (cannot convert 'u8' to 'i32')
    return;
```

Rust provides no default argument promotion!

#### Fundamental types - boolean conversion

"A value of any scalar type can be implicitly converted to bool ..."

```c++
int var = 1;
if (var) {printf("converted!");}
```

outputs `converted` as value `1` is implicitly converted to `int`!

However, following Rust code won't compile:
```rust
let var1 : u8 = 1;
if var1 {println!("converted!");}  // expected `bool`, found `u8`
```

Rust provides no implicit boolean conversion!

#### Statements and expressions

Rust introduces distinction between `statements` and `expressions` (from doc.rust-lang.org):
- Statements are instructions that perform some action and do not return a value.
- Expressions evaluate to a resultant value. Let’s look at some examples.

```rust
let var1 = 1;  // Statement
let var2 = 2;  // Statement
var1 = var2 = 3; // Won't compile (expected integer, found `()`)
```

But in C++ assignment returns the value of the assignment, so it's possible to write:

```c++
int var1 = 0;
int var2 = 1;
var1 = var2 = 3; // Valid code
```

Rust also allow block scope to return value:
```rust
fn foo(var: i32) -> i32 {
    let v = {
        let v = 10;  // Statement
        v // Expression
    };
    v  // Expression
}
```

### Enums can hold values


