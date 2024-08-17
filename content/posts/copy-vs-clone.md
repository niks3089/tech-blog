---
title: "To copy or to clone? That's the question"
date: 2024-08-17T00:00:00+08:00
description: "A comparision of copy vs clone in rust and knowing when to use what"
tags: ["rust"]
type: post
weight: 25
showTableOfContents: true
---

I like having the rust-analyser fix issues for me. I hover over the code and see an issue like this 

```
cannot move out of `host.metal_type` which is behind a shared reference
move occurs because `host.metal_type` has type `std::option::Option<MetalType>`, which does not implement the `Copy trait`
```

and the first thing I do is `host.metal_type.clone()` and that fixes the problem. But why does it fix the problem? What was the problem? Should I add a copy trait to my struct? Lets deep dive. 

First thing is what is a `trait`? 

## `trait` in Rust

Before we jump in, we need to understand what a trait is. In Rust, a **trait** is a language feature that defines a set of methods that types can implement. Traits are somewhat similar to **interfaces** in languages like C++ but with some important distinctions and additional capabilities. Like in other blogs, I like comparing Rust features with C++

### Key Traits of Traits in Rust

1. **Method Definitions**:
   - A trait defines one or more method signatures, but it does not provide an implementation for those methods (except for default implementations, which are optional).
   - Types that implement the trait must provide concrete implementations for the methods defined in the trait.
   - **C++ Comparison**: This is akin to defining a pure virtual class (interface) in C++, where derived classes must provide implementations for the pure virtual functions.

2. **Implementing Traits**:
   - Types (like structs, enums, or even other traits) can implement one or more traits. When a type implements a trait, it agrees to provide the method implementations defined by that trait.
   - This allows you to define common behavior across different types in a consistent way.
   - **C++ Comparison**: In C++, when a class inherits from an interface (pure virtual class), it must implement the virtual functions, similar to how a type in Rust implements the methods of a trait.

3. **Trait Bounds**:
   - Traits can be used to define constraints on generic types. This is known as "trait bounds." It allows you to write generic functions or structs that work with any type that implements a specific trait.
   - **C++ Comparison**: This is similar to **template specialization** or **concepts** in C++20, where you can constrain a template to only work with types that fulfill certain conditions.

4. **Dynamic Dispatch**:
   - Traits can be used for polymorphism in Rust, allowing different types to be treated uniformly based on the trait they implement. This is often used in conjunction with trait objects (`&dyn Trait` or `Box<dyn Trait>`) to achieve dynamic dispatch.
   - **C++ Comparison**: This is analogous to **polymorphism** in C++ using pointers or references to base classes (interfaces), where virtual functions enable dynamic dispatch.

5. **Default Implementations**:
   - Traits can provide default implementations for some or all of their methods. When a type implements the trait, it can choose to use the default implementation or provide its own.
   - **C++ Comparison**: Similar to providing a default implementation for a virtual function in a base class in C++, which derived classes can override if needed.

### Example of a Trait in Rust

Here’s an example of defining and implementing a trait in Rust, with a comparison to how it might look in C++:

#### Rust Example

```rust
// Define a trait named `Describe`
pub trait Describe {
    fn describe(&self) -> String;
}

// Implement the `Describe` trait for a struct `Person`
struct Person {
    name: String,
    age: u32,
}

impl Describe for Person {
    fn describe(&self) -> String {
        format!("{} is {} years old.", self.name, self.age)
    }
}

// Implement the `Describe` trait for a struct `Car`
struct Car {
    make: String,
    model: String,
}

impl Describe for Car {
    fn describe(&self) -> String {
        format!("This is a {} {}.", self.make, self.model)
    }
}

fn main() {
    let person = Person {
        name: String::from("Alice"),
        age: 30,
    };
    let car = Car {
        make: String::from("Toyota"),
        model: String::from("Corolla"),
    };

    // Both `Person` and `Car` can use the `describe` method
    println!("{}", person.describe());
    println!("{}", car.describe());
}
```
The same example in C++

```cpp
#include <iostream>
#include <string>

// Define an interface (pure virtual class) named `Describe`
class Describe {
public:
    virtual std::string describe() const = 0;  // Pure virtual function
    virtual ~Describe() = default;             // Virtual destructor
};

// Implement the `Describe` interface for a class `Person`
class Person : public Describe {
public:
    std::string name;
    int age;

    Person(std::string n, int a) : name(n), age(a) {}

    std::string describe() const override {
        return name + " is " + std::to_string(age) + " years old.";
    }
};

// Implement the `Describe` interface for a class `Car`
class Car : public Describe {
public:
    std::string make;
    std::string model;

    Car(std::string m, std::string mo) : make(m), model(mo) {}

    std::string describe() const override {
        return "This is a " + make + " " + model + ".";
    }
};

int main() {
    Person person("Alice", 30);
    Car car("Toyota", "Corolla");

    // Both `Person` and `Car` can use the `describe` method
    std::cout << person.describe() << std::endl;
    std::cout << car.describe() << std::endl;
}
```

### Traits and Polymorphism

Traits in Rust enable polymorphism, where different types can be treated uniformly based on the trait they implement. This allows you to write code that can work with any type that implements a certain trait, promoting code reuse and flexibility.

For example, you could write a function that accepts any type that implements Describe:

```rust
fn print_description(item: &impl Describe) {
    println!("{}", item.describe());
}
```

## What is the `Copy` Trait in Rust?

In Rust, the `Copy` trait is a special marker trait that indicates a type can be duplicated by making a simple bitwise copy of its memory. This trait is designed for types that are inexpensive to copy, such as scalar values (integers, floating-point numbers, booleans) or types that contain only `Copy` types. When a type implements the `Copy` trait, its values are copied implicitly whenever they are assigned to another variable or passed to a function, rather than being moved. Sounds complicated? Lets simplify 

### Characteristics of the `Copy` Trait

1. **Implicit Copying**:
   - Types that implement `Copy` do not require an explicit call to `.clone()` to duplicate values. Instead, they are automatically copied when assigned to another variable or when passed as arguments to functions.

2. **Shallow Copy**:
   - The `Copy` trait performs a shallow copy, which is a direct bitwise duplication of the value. This is efficient but means that `Copy` types cannot own heap-allocated memory or other resources that require deep copying.
   - A shallow copy means copying the value exactly as it is stored in memory, without considering any deeper structures or references that the value might contain.
   ```rust
    fn main() {
        let x = 42; // `x` is an integer, which is a `Copy` type
        let y = x;  // This creates a shallow copy of `x`

        println!("x: {}, y: {}", x, y);
    }
   ```
   - Direct bitwise duplication refers to copying the raw memory representation of a value from one location to another. This process doesn't involve any deeper logic or traversal of structures—it's simply duplicating the exact binary data (the bits) as it exists in memory. For example, when you copy an integer, Rust simply duplicates the bits that represent that integer. The process is very fast and efficient because it doesn’t require any additional computation or memory management.


3. **Deriving `Copy`**:
   - The `Copy` trait can often be automatically derived by the compiler for simple types. For example, most primitive types in Rust, such as `i32`, `u32`, `f64`, and `bool`, implement `Copy` by default.

#### Example of a `Copy` Type

Here’s a simple example of how the `Copy` trait is used in Rust:

```rust
#[derive(Copy, Clone)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p1 = Point { x: 10, y: 20 };
    let p2 = p1;  // `p1` is copied to `p2`, not moved

    println!("p1: ({}, {}), p2: ({}, {})", p1.x, p1.y, p2.x, p2.y);
}
```

#### Explanation:
- Deriving Copy and Clone: In the example, the Point struct is marked with #[derive(Copy, Clone)]. This tells the compiler to automatically implement both the Copy and Clone traits for Point.
- Implicit Copying: When p1 is assigned to p2, p1 is not moved but copied. Both p1 and p2 contain the same data, and both remain valid after the assignment.
- No Drop Implementation: Because Point implements Copy, it cannot have a custom destructor (Drop implementation), which aligns with the expectation that Copy types are trivial to copy and do not require cleanup.

### What is the `Clone` Trait in Rust?

The `Clone` trait in Rust is a standard library trait that allows for the explicit duplication of an object. When a type implements the `Clone` trait, it provides a method for creating a deep copy of the object, meaning that all of the data is duplicated, rather than just copying the memory address (which would be a shallow copy).

### Key Characteristics of the `Clone` Trait

1. **Explicit Copying**:
   - The `Clone` trait is used for types where a deep copy is necessary or desired. This includes types that manage resources like heap-allocated memory, file handles, or network connections.
   - Unlike the `Copy` trait, which allows for implicit copying, cloning is always explicit. You must call the `.clone()` method to create a duplicate of the object.

2. **Deep Copy**:
   - The `Clone` trait usually involves a deep copy, meaning that all of the data associated with the object is duplicated. For example, if a type owns a heap-allocated array, `.clone()` will create a new array with the same elements, rather than just copying the pointer to the original array.

3. **Custom Implementation**:
   - Types that implement `Clone` can provide a custom implementation of the `.clone()` method. This allows you to control exactly how the type is duplicated.
   - If the type consists entirely of other types that implement `Clone`, you can often derive `Clone` automatically using `#[derive(Clone)]`.

4. **Combination with `Copy`**:
   - The `Clone` trait is often implemented alongside the `Copy` trait for simple types. However, when a type implements both, the `.clone()` method typically just returns `*self`, because the `Copy` trait guarantees that the type can be duplicated by copying bits.

#### Example of a `Clone` Implementation

Here’s a simple example of how the `Clone` trait is used in Rust:

```rust
#[derive(Clone)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p1 = Point { x: 10, y: 20 };
    let p2 = p1.clone();  // Explicitly clone `p1` to create `p2`

    println!("p1: ({}, {}), p2: ({}, {}))", p1.x, p1.y, p2.x, p2.y);
}
```

#### Explanation:

- **Deriving `Clone`**: In the example, the `Point` struct is marked with `#[derive(Clone)]`, which automatically implements the `Clone` trait for the struct. This is possible because `i32` (the type of the fields `x` and `y`) implements `Clone`.

- **Explicit Cloning**: When `p1.clone()` is called, it creates a new `Point` instance `p2` with the same values as `p1`. Unlike the `Copy` trait, which would simply duplicate `p1` implicitly when assigning it to `p2`, the `Clone` trait requires the explicit call to `.clone()`.

### How do you know if your data is on stack or the heap?

- In Rust, simple data types like integers, booleans, and fixed-size arrays are stored on the stack. These types can be copied quickly and easily using a shallow copy because they are small and don’t involve complex memory management.
- More complex data types, such as strings (String), vectors (Vec<T>), and other types that can grow dynamically, are stored on the heap. The heap is a region of memory used for dynamic allocations, where data can be stored that doesn’t fit within the fixed-size limits of the stack.
- When a type allocates memory on the heap, it typically involves a pointer stored on the stack that points to the actual data on the heap. A shallow copy of this pointer would just duplicate the pointer itself, not the data it points to. This could lead to issues like double-free errors or data corruption if both copies try to manage the same heap-allocated memory.
- To properly duplicate heap-allocated data, a deep copy is required, where the data on the heap is also copied to a new location, and the new pointer points to this new location

### How `Copy` and `Clone` are Implemented in Rust's Source Code

In Rust, the `Copy` and `Clone` are implemented within the standard library. The `Copy` trait is a marker trait that signifies a type can be duplicated by simply copying its bits, making it suitable for types that are small and inexpensive to copy, such as integers or floating-point numbers. In contrast, the `Clone` trait is more general and allows for deep copying of an object, which is necessary for types that manage resources like heap-allocated memory.

#### `Clone` Trait

```rust
pub trait Clone {
    fn clone(&self) -> Self;

    fn clone_from(&mut self, source: &Self) {
        *self = source.clone();
    }
}
```

The `Clone` trait defines the `clone` method, which is used to create a deep copy of an object. This method must be implemented by any type that wants to provide a custom way to duplicate its values. Additionally, the `Clone` trait includes a `clone_from` method, which is optional and provides a more efficient way to clone by reusing the resources of the existing value when possible.

#### `Copy` Trait

```rust
pub trait Copy: Clone {}
```

The `Copy` trait is defined as a subtrait of `Clone`. It does not define any methods and serves as a marker trait to indicate that a type can be copied with a simple bitwise copy. Since `Copy` is a subtrait of `Clone`, any type that implements `Copy` must also implement `Clone`. This ensures that `Copy` types can always be cloned, even though the `clone` method for `Copy` types is generally trivial and simply returns the value itself.

#### Auto-Deriving `Copy` and `Clone`

Rust allows types to automatically derive implementations of `Copy` and `Clone` if all of their fields also implement these traits. This auto-derivation simplifies the process of making simple structs and enums `Copy` and `Clone` by avoiding the need to manually implement these traits.

#### How `Copy` and `Clone` Work Together

The relationship between `Copy` and `Clone` ensures that any `Copy` type can also be cloned, but in a straightforward manner. When a type implements `Copy`, the `clone` method typically just returns a bitwise copy of the value, making the `clone` method trivial. This relationship allows for efficient handling of simple, small types while still providing the flexibility to perform more complex cloning operations when needed.


### Memory Implications of Using `Copy` and `Clone`

The `Copy` and `Clone` traits in Rust have different memory implications due to how they handle data duplication.

- **`Copy`**: The `Copy` trait performs a shallow bitwise copy of the data, which is very efficient in terms of both time and memory. Since `Copy` does not involve any heap allocation or complex data structures, it simply duplicates the data by copying its bits directly. This makes `Copy` ideal for small, simple types like integers, floating-point numbers, and other types that occupy a fixed, small amount of memory. The memory implication of using `Copy` is minimal because it doesn't require additional memory beyond the space needed for the original and copied values.

- **`Clone`**: The `Clone` trait, on the other hand, typically involves a deep copy of the data. This means that all elements of a compound data structure (like a vector or a string) are duplicated, including any heap-allocated memory. Consequently, `Clone` can be more memory-intensive because it allocates new memory for the copied data and duplicates everything, which may also include performing additional operations like reference counting or other forms of resource management.

### Given a Choice, Which Should You Use and Why?

- **Use `Copy` When Possible**: If your type can implement `Copy`, you should prefer it over `Clone` due to its efficiency. `Copy` is faster and uses less memory because it simply performs a shallow copy of the data. Additionally, the use of `Copy` leads to more predictable performance and fewer surprises, as it avoids the potential overhead associated with heap allocations and other complex operations that `Clone` might involve.

- **Use `Clone` When Necessary**: You should use `Clone` when you need to perform a deep copy of a type that manages heap-allocated memory or other resources that cannot be safely duplicated with a simple bitwise copy. Types like `String`, `Vec<T>`, and other heap-allocated structures typically implement `Clone` because they need to create entirely new instances of their data rather than just copying a pointer or reference.


### Summary

In summary, Rust's `Copy` and `Clone` traits offer different approaches to data duplication, each with its own advantages. The `Copy` trait enables efficient, shallow copying of small, simple types by directly duplicating their memory bits, making it ideal for types like integers and booleans. In contrast, the `Clone` trait provides a way to perform deep copying, necessary for more complex types that manage heap-allocated resources, such as `String` or `Vec<T>`. Understanding when to use `Copy` versus `Clone` is crucial for optimizing both performance and memory usage in Rust, with `Copy` being preferable for its speed and simplicity when applicable, and `Clone` being necessary for ensuring the safe duplication of more complex, heap-based data.
