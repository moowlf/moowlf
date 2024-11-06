---
title: "C++ and the && operator"
date: 2024-11-06T11:14:46Z
slug: 2024-11-06-move-semantics
type: posts
summary: "A brief presentation of move semantics and perfect forwarding"
draft: false
categories:
  - Programming
tags:
  - c++
---


In this article, we explore the various definitions and their meanings of the operator && in the c++ language.

# Move Semantics

Move semantics were added to the C++ standard in 2011 with the release of C++11. In this topic, we discuss some of the drawbacks it aimed to address, as well as how it is being used.

### Before C++11

Before the introduction of move semantics, the ABCs of a C++ programmer stated that one should follow the rule of three: *if you need to define one of copy constructor, copy assignment or destructor, you must define all of them*. Sometimes, these constructors were not enough, and the programmer had to program additional constructors to further optimize code. For that, the user had to implement both new constructors (that received by reference only, instead of const reference) and possibly declare new swap methods as seen below. 

As one can see, it was possible to use workarounds to reduce copies in your programs. However, the programs lacked clarity. Did the constructor steal any data? Does the function that receives an argument by reference modify it or outright steal its content? Having this in mind, a proposal to introduce move semantics was made and eventually accepted. 

Below you can see a possible implementation of a class using the dynamic pre-move semantics.

```c++
class Feathers {
    friend void swap(Feathers& a, Feathers& b) {
        using std::swap;
        // ...
    }
};

class Dinosaur {

  public:
	// Default constructor
	Dinosaur() {}
	
	// User defined constructor
	Dinosaur(Feathers feathers) { mFeathers = feathers; }

	// Copy Constructor
	Dinosaur(Dinosaur const& rhs) { mFeathers = rhs.mFeathers; }

	// Copy Constructor that allow stealing
	Dinosaur(Dinosaur& rhs) { std::swap(mFeathers, rhs.mFeathers); }

	// Copy operator assignment
	Dinosaur& operator=(Dinosaur const& rhs) { 
		mFeathers = rhs.mFeathers;
		return *this;
	}

	// Copy operator assignment that allow stealing
	Dinosaur& operator=(Dinosaur& rhs) { 
		std::swap(mFeathers, rhs.mFeathers);
		return *this;
	}

	// Destructor
	~Dinosaur() = default;
	
  private:
	Feathers mFeathers;
};
```

### C++11 changes

The move semantics introduced in the standard 11, made use of some other changes introduced with it. One of them was the need to change the type of expressions available. The other one was the introduction of the operator && to represent rvalue-references.
#### Expressions

In the beginning of the C++ standards,  there were two different types of values. An expression was either an **lvalue** or an **rvalue**. You either represented something in memory and were a **lvalue**, or were a **rvalue**.

```c++
int lvalue = 27;
//    ^       ^______ rvalue
//    lvalue 

lvalue + 30; // > This was considered an rvalue expression
```

Their names are related to where one could find them in expressions. The **lvalue** was found at the left of the equal sign and thus its name **L**eft-**value**. On the other hand, the **rvalue** was found by the right side of equal sign. With the introduction of the C++11, these categories were expanded a little. Instead of only two types of expressions, we got two primary categories: **glvalue** and **rvalue** and three *subcategories*: **lvalue**, **xvalue** and **prvalue**. The **glvalue** category comprises both **lvalue** and **xvalue**. Similarly, the **rvalue** category had two subcategories: **xvalue** and **prvalue**. You can find the definitions provided by a paper for the committee [1](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2010/n3055.pdf). The definitions are as follows:

* **lvalue**: Expressions that designate a function or an object;
* **xvalue**: Expressions that designate an object nearing the end of its current lifetime. Its name comes from e**x**piring **value**;
* **prvalue**: All expressions that are rvalues but are not at the end of their lifetime.
* **glvalue**: Expressions that are either an **lvalue** or an **xvalue**. Its name comes from **g**eneralized values.
* **rvalue**: Expressions that are either **xvalue**, a temporary object or a value that is not associated with an object

Why is this important to know ? To understand what can and what cannot be moved

#### Move semantics

What exactly means to move an object ? Below is a transcript from a paper submitted to the committee. 

> C and C++ are built on copy semantics. This is a Good Thing. **Move semantics is not an attempt to supplant copy semantics**, nor undermine it in any way. Rather this proposal seeks to augment copy semantics. **A general user defined class might be both copyable and movable**, one or the other, or neither.
   **The difference between a copy and a move is that a copy leaves the source unchanged.** A move on the other hand leaves the source in a state defined differently for each type. The state of the source may be unchanged, or it may be radically different. The only requirement is that the object remain in a self consistent state (all internal invariants are still intact). From a client code point of view, choosing move instead of copy means that you don't care what happens to the state of the source.
   **For PODs, move and copy are identical operations (right down to the machine instruction level).**

With the introduction of move semantics, one more constructor and one more assignment operator were introduced.

```c++
class Dinosaur {
  public:

		// ...

		// Move constructor
		Dinosaur(Dinosaur&&) noexcept;
	
		// Move assignment
		Dinosaur& operator=(Dinosaur&&) noexcept;

		// ...
}
```

One question that may appear to the most aware is where is the *const* in the constructor akin the copy constructor. By definition, it makes no sense to have a move constructor with *const*. How would you perform move operations from object A to object B if A is *const* ? That would be the same as performing copy operations.

Returning to the above example, the introduction of move semantics brought new tokens into what was already a complex language. I am talking about the double amps "&&". While it is similar to "&", the **rvalue reference** token only binds to **r-values**. Other thing that was introduced were the move constructor and move assignment operator. With them, the doubt that existed before, whether the object was stolen or simply modified, was eliminated. When the programmer sees the object did its purpose and can be stripped of its contents, a move constructor/operator is used. To do that, the programmer has to make sure the compiler knows the object can be moved.  Below are some examples easy-to-understand.

```c++

#include "Dinosaur.hpp"


void print_dinosaur(Dinosaur && dino) {
	std::cout << "Received an rvalue reference to a dino" << '\n';
}

void print_dinosaur(Dinosaur const& dino) {
	std::cout << "Received an const lvalue reference to a dino" << '\n';
}


int main() {
	Dinosaur dino;

	// Dino here is an lvalue, therefore it can bind to const&
	print_dinosaur(dino);
	// > Received an const lvalue reference to a dino

	// Dino here is being told to convert to an rvalue
	print_dinosaur(std::move(dino));
	// > Received an rvalue reference to a dino
	
	print_dinosaur(Dinosaur{});
	// > Received an rvalue reference to a dino
	
	// <-------------------------------------------------------->
	
	const Dinosaur dino_1;

	print_dinosaur(dino_1);
	// -> Received an const lvalue reference to a dino

	print_dinosaur(std::move(dino_1));
	// > Received an const lvalue reference to a dino
}
```

Here, it is important to note one of the biggest misnomers in the C++ standard. The **std::move** function does not **move** anything; It merely converts an **lvalue** into a **rvalue** reference! 

# Universal references

The `&&` implies almost always that the current type is a **rvalue-reference**. However there are some cases in which it does not. in this section, we think in what could possibly have lead the usage of `&&` to mean other thing other than a rvalue-reference.

### The problem

```c++
#include "Dinosaur.hpp"

void print_dinosaur(Dinosaur& dino) {
    std::cout << "I am a reference to dino" << '\n';
}

void print_dinosaur(Dinosaur const& dino) {
    std::cout << "I am a const reference to dino" << '\n';
}

void print_dinosaur(Dinosaur&& dino) {
    std::cout << "I am a rvalue reference to dino" << '\n';
}

template<typename T>
void print(T obj) {
    print_dinosaur(obj);
}

int main() {

    Dinosaur dino;
    const Dinosaur dino_1;

    print(dino); // I am a reference to dino
    print(dino_1); // I am a reference to dino
    print(std::move(dino)); // I am a reference to dino
}
```

This problem arises from the fact that template deduction ignores *const*-ness and *reference*-ness. Before trying to solve this problem, I must show you some of the rules that templates follow in order to deduce their type.

```c++

// If T is neither & or pointer:
template<typename T>
void func(T param);

// Array types decay for the underlying type
int arr[2] = {0, 0};
func(arr); // T = int*

// const volatiles are throw away
const int i = 0;
func(i); // T = int


// If T is a reference or pointer
template <typename T>
void func_a(T& param);

const int j = 0;
func_a(j); // T is now const int

// If T is rvalue reference
template <typename T>
void func_b(T&& param);

int k = 1;
func_b(k); // Since K is an lvalue reference, T -> int&

const int l = 1;
func_b(l); // L is const lvalue reference, T -> const int&

```

#### Trying to solve the problem #1

We've seen that having only one function `print` as it is will not work. First, we're losing the correct types that we're passing - passing rvalue reference is the same as passing a lvalue reference. Additionally, they way the things stand, we're making copies all over the place even when not needed. One first approach would be to add a `&`to the function so copies would not be made.

```c++
template<typename T>
void print(T& obj) {
    print_dinosaur(obj);
}

int main() {

    Dinosaur dino;
    const Dinosaur dino_1;

    print(dino); // I am a reference to dino
    print(dino_1); // I am a const reference to dino
    print(std::move(dino)); // cannot bind non-const lvalue reference of type...
}
```

Adding a `&` appears to work for *const* objects, however it completely fails for r-value references. C++ standard forbids non const lvalue references to bind to temporary variables. 
#### Trying to solve the problem #2

One of the possible solutions would be to overload the function template as seen below. Adding the previous *print* to the code would fail miserably since it would give ambiguous calls, that is, the compiler would not know which function should call for a particular type.


```c++
template<typename T>
void print(T obj) {
    print_dinosaur(obj);
}

template<typename T>
void print(T& obj) {
    print_dinosaur(obj);
}

// would be "transformed" to
void print(Dinosaur obj) {
//...
}

void print(Dinosaur& obj) {
//...
}

Dinossaur dino;
print(dino); // ambiguous call
```

#### Trying to solve the problem above #3

We've seen that overloading the methods will not work. So **universal references** were created. By adding **&&**, we can pass it and have them deduced *correctly* (more on that later). 

```c++
template<typename T>
void print(T&& obj) {
    print_dinosaur(obj);
}
```

This change in the code would make the compiler create the following functions:

```c++
template<>
void print<Dinosaur &>(Dinosaur & obj)
{
  print_dinosaur(obj);
}

template<>
void print<const Dinosaur &>(const Dinosaur & obj)
{
  print_dinosaur(obj);
}

template<>
void print<Dinosaur>(Dinosaur && obj)
{
  print_dinosaur(obj);
}
```

OK, now it seems the compiler is working correctly. Running it would prove us wrong very quickly. It appears that our *Dinosaur &&* is losing its rvalue-reference and choosing the function that receives the parameter by lvalue-reference. This occurs because inside a function, *obj* would be a **lvalue** reference. In order to fix that, we need to correctly convert it to an rvalue-reference.

```c++
template<typename T>
void print(T&& obj) {
    print_dinosaur(static_cast<T&&>(obj));
}
```

This will compile and make our code run as intended. But what if we pass an **lvalue** reference? This was thought and the solution was called *reference colapsing*. Below one can see how references collapse:

- T& -> & =  **T&**
- T& -> && = **T&**
- T&&-> & = **T&**
- T&& -> &&= **T&&**

By leveraging the rules of the reference collapsing and the, the programmer can now forward the correct type for other functions. Instead of *static_cast* the standard gave us the *std::forward\<T\>* function. But in general, they serve the same purpose.

### References
- [Lvalues and Rvalues (C++) @ Microsoft](https://learn.microsoft.com/en-us/cpp/cpp/lvalues-and-rvalues-visual-cpp?view=msvc-170)
- [lvalue and rvalue in C language @ GeeksforGeeks](https://www.geeksforgeeks.org/lvalue-and-rvalue-in-c-language/)
- [move-proposal @ open-std](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2002/n1377.htm#Introduction)
- [Understanding Move Semantics and Perfect Forwarding: Part 3 @ drewcampbell92](https://drewcampbell92.medium.com/understanding-move-semantics-and-perfect-forwarding-part-3-65575d523ff8)
