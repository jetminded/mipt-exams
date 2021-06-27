## Пример 1
```#include <iostream>
#include <exception>

int x = 0;

class A {
public:
  A() {
    std::cout << 'a';
    if (x++ == 0) {
      throw std::exception();
    }
  }
  ~A() { std::cout << 'A'; }
};

class B {
public:
  B() { std::cout << 'b'; }
  ~B() { std::cout << 'B'; }
  A a;
};

void foo() { static B b; }

int main() {
  try {
    foo();
  }
  catch (std::exception &) {
    std::cout << 'c';
    foo();
  }
}
```
Ответ: acabBA  
Пояснение: Static local variables are initialized the first time control passes through their declaration. The first time `foo()` is called, `b` is attempted initialized. Its constructor is called, which first constructs all member variables. This means `A::A()` is called, printing _a_. `A::A()` then throws an exception, the constructor is aborted, and neither `b` or `B::a` are actually considered constructed. In the catch-block, _c_ is printed, and then `foo()` is called again. Since `b` was never initialized the first time, it tries again, this time succeeding, printing _ab_. When main() exits, the static variable `b` is destroyed, first calling the destructor printing _B_, and then destroying member variables, printing _A_.

## Пример 2
```
#include <iostream>

class A;

class B {
public:
  B() { std::cout << "B"; }
  friend B A::createB();
};

class A {
public:
  A() { std::cout << "A"; }

  B createB() { return B(); }
};

int main() {
  A a;
  B b = a.createB();
}
```
Ответ: CE  
Пояснение: There is a compilation error when attempting to declare `A::createB()` a friend of `B`. To declare `A::createB()` a friend of `B`, the compiler needs to know that that function exists. Since it has only seen the declaration of `A` so far, not the full definition, it cannot know this.
