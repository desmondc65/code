Static Polymorphism (also called **Compile-Time Polymorphism**) is a technique where the decision of which function to execute is made by the compiler **during compilation**, not while the program is running.

This is the cornerstone of high-performance C++ (like the standard library `std::sort` or Eigen library) because it avoids the overhead of **Virtual Tables (v-tables)** and enables aggressive compiler optimizations like inlining.

There are two primary ways to achieve this: **Function Overloading** (basic) and **Templates** (architectural).

-----

### 1\. Function Overloading (The Basic Form)

This is the simplest form. You define multiple functions with the same name but different parameters. The compiler picks the correct one based on the arguments passed.

```cpp
void process(int i) { 
    std::cout << "Processing Int: " << i << "\n"; 
}

void process(double d) { 
    std::cout << "Processing Double: " << d << "\n"; 
}

int main() {
    process(10);   // Compiler binds to process(int)
    process(3.14); // Compiler binds to process(double)
}
```

-----

### 2\. Templates (The "Duck Typing" Form)

This is the architectural version of static polymorphism. Unlike Java or C\# interfaces (which are dynamic), C++ templates rely on "Duck Typing": *If it walks like a duck (has the right method name), the compiler treats it like a duck.*

We write a generic function that accepts a type `T`. As long as `T` has the methods we call, the code compiles.

```cpp
#include <iostream>

class Dog {
public:
    void speak() const { std::cout << "Woof!\n"; }
};

class Cat {
public:
    void speak() const { std::cout << "Meow!\n"; }
};

// STATIC POLYMORPHISM
// The compiler generates two separate versions of this function:
// one for T=Dog, one for T=Cat.
template <typename T>
void makeItSpeak(const T& animal) {
    animal.speak(); // Resolved at compile time
}

int main() {
    Dog d;
    Cat c;
    
    makeItSpeak(d); // Calls Dog::speak directly (fast)
    makeItSpeak(c); // Calls Cat::speak directly (fast)
}
```

-----

### 3\. The CRTP (Curiously Recurring Template Pattern)

**This is the "Google Interview" level implementation.**

If you want the structure of inheritance (Base class defines interface, Derived class defines implementation) but **without** the cost of `virtual` functions, you use CRTP.

**The Trick:** The Base class is a template, and the Derived class inherits from `Base<Derived>`. This allows the Base class to cast `this` to the `Derived` type at compile time.

[Image of C++ CRTP inheritance diagram]

```cpp
#include <iostream>

// 1. The Base Template
template <typename Derived>
class ShapeBase {
public:
    // The Interface
    void draw() {
        // static_cast is safe here because we know Derived inherits from ShapeBase<Derived>
        static_cast<Derived*>(this)->drawImplementation();
    }
};

// 2. Derived Class A
class Circle : public ShapeBase<Circle> {
public:
    void drawImplementation() {
        std::cout << "Drawing Circle\n";
    }
};

// 3. Derived Class B
class Square : public ShapeBase<Square> {
public:
    void drawImplementation() {
        std::cout << "Drawing Square\n";
    }
};

// 4. A generic processor
template <typename T>
void render(ShapeBase<T>& shape) {
    shape.draw();
}

int main() {
    Circle c;
    Square s;

    render(c); // Compiler inlines the call. No v-table lookup.
    render(s);
}
```

-----

### 4\. Static vs. Dynamic Polymorphism (Comparison)

| Feature | Static Polymorphism (Templates/CRTP) | Dynamic Polymorphism (Virtual Functions) |
| :--- | :--- | :--- |
| **Binding Time** | Compile Time | Runtime |
| **Performance** | **High.** Can be inlined. No pointer chasing. | **Lower.** Requires V-Table lookup + Indirect function call. |
| **Memory** | **Code Bloat.** Generates new code for every type `T`. | **Compact.** One function handles all pointers. |
| **Flexibility** | Rigid. Types must be known at compile time. | Flexible. Can handle heterogeneous lists (e.g., `vector<Shape*>`). |
| **Errors** | Ugly, verbose template errors (pre-C++20). | Cleaner link/runtime errors. |

### Critical Critique

Static polymorphism is **not** a silver bullet.

1.  **Code Bloat:** If you use `makeItSpeak<T>` with 50 different types, the compiler generates 50 distinct functions. This increases binary size and instruction cache pressure.
2.  **Compilation Time:** Heavy template usage slows down compilation significantly.
3.  **Heterogeneous Collections:** You cannot put different static types in the same `std::vector` easily. You can't do `std::vector<ShapeBase>` because `ShapeBase` is a template, not a concrete type. You would need `std::variant` or type erasure to manage that.

**Would you like to see how `std::variant` and `std::visit` bridge the gap between static and dynamic polymorphism in modern C++ (C++17)?**