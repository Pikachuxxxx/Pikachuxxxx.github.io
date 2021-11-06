---
title: Object Slicing in C++
tags: [Software Engineering, C++]
style:
color:
description: Learn about Object slicing and why it can save your precious data and help master polymorphism.
blog: [Graphics]
published: true
permalink: /cpp/:title
---

Source: [Object Slicing Wikipedia](https://en.wikipedia.org/wiki/Object_slicing)


## What is Object Slicing?

**Simply put:** When a derived class object is assigned to a base class object, additional attributes of a derived class object are sliced off to form the base class object.

**More complexly put:** In C++, any derived type is also of the type base class, while this provides way for very powerful behaviour, it does come with it's fair share of problems, i.e. Object Slicing. Assume, like the other answers, that you're dealing with two classes A and B, where B derives (publicly) from A.

In this situation, C++ lets you pass an instance of B to  A's assignment operator (and also to the copy constructor). This works because an instance of B can be converted to a const A&, which is what assignment operators and copy-constructors expect their arguments to be.

```C++
B b;
A a = b;
```

Nothing bad happens there - you asked for an instance of A which is a copy of B, and that's exactly what you get. Sure, a won't contain some of b's members, but how should it? It's an A, after all, not a B, so it hasn't even heard about these members, let alone would be able to store them.

When you create a container, its elements are full objects of their own. For example:

```C++
std::vector<int> v(3);  // define a vector of 3 integers
int i = 42;
v[0] = i;  // copy 42 into v[0]
```

The same thing happens with classes:

```C++
class Base { ... };
std::vector<Base> v(3);  // instantiates 3 Base objects
Base x(42);
v[0] = x;
```

then v[0] = x tries to copy the contents of a Derived object into a Base object. What happens in this case is that all members declared in Derived are ignored. Only the data members declared in the base class Base are copied, because that's all v[0] has room for. The fundamental issue is copying an object (which is not an issue in languages where classes are "reference types", but in C++ the default is to pass things by value, i.e. making a copy). "Slicing" means copying the value of a bigger object (of type B, which derives from A) into a smaller object (of type A). Because A is smaller, only a partial copy is made.


## How do we resolve this issue?

What a pointer gives you is the ability to avoid copying. When you do

```cpp
T x;
T *ptr = &x;
```
`ptr` is not a copy of `x`, it just points to `x`.

Similarly, you can do

```cpp
Derived obj;
Base *ptr = &obj;
```

`&obj` and `ptr` have different types (`Derived *` and `Base *`, respectively), but C++ allows this code anyway. Because Derived objects contain all members of Base, it's OK to let a Base pointer point at a Derived instance.

What this gives you is essentially a reduced interface to obj. When accessed through ptr, it only has the methods declared in Base. But because no copying was done, all data (including the Derived specific parts) are still there and can be used internally.

### What about virtual functions?

As for `virtual` Normally, when you call a method foo through an object of type Base, it will invoke exactly Base::foo (i.e. the method defined in Base). This happens even if the call is made through a pointer that actually points at a derived object (as described above) with a different implementation of the method:

```cpp
class Base {
    public:
    void foo() const { std::cout << "hello from Base::foo\n"; }
};

class Derived : public Base {
    public:
    void foo() const { std::cout << "hello from Derived::foo\n"; }
};

Derived obj;
Base *ptr = &obj;
obj.foo();  // calls Derived::foo
ptr->foo();  // calls Base::foo, even though ptr actually points to a Derived object
```

By marking `foo` as `virtual`, we force the method call to use the actual type of the object, instead of the declared type of the pointer the call is made through:

```cpp
class Base {
    public:
    virtual void foo() const { std::cout << "hello from Base::foo\n"; }
};

class Derived : public Base {
    public:
    void foo() const { std::cout << "hello from Derived::foo\n"; }
};

Derived obj;
Base *ptr = &obj;
obj.foo();  // calls Derived::foo
ptr->foo();  // also calls Derived::foo
```

`virtual` has no effect on normal objects because there the declared type and the actual type are always the same. It only affects method calls made through pointers (and references) to objects, because those have the ability to refer to other objects (of potentially different types).  


#### And that is another reason to store a collection of pointers!  


When you have several different subclasses of `Base`, all of which implement their own custom virtual methods overrides, you want the code to pay attention to the actual types of the objects, so the right method gets called in each case.  
