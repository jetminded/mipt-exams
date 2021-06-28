# 1. Основы (cout'ы, поля видимости, etc)

## Пример 1.1
```
#include <iostream>

int a;

int main () {
    std::cout << a;
}
```
Ответ: 0  
Пояснение: Since _a_ has static storage duration and no initializer, it is guaranteed to be zero-initialized. Had a been defined as a local non-static variable inside `main()`, this would not have happened.

Note: int a has static storage duration because it is declared at namespace scope. It does not need to have static in front of it, that would only denote internal linkage.


# 2. Наследование и иже с ним

## Пример 2.1
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

## Пример 2.2
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


# 3. STL контейнеры


## Пример 3.1
```
#include <iostream>
#include <map>
using namespace std;

bool default_constructed = false;
bool constructed = false;
bool assigned = false;

class C {
public:
    C() { default_constructed = true; }
    C(int) { constructed = true; }
    C& operator=(const C&) { assigned = true; return *this;}
};

int main() {
    map<int, C> m;
    m[7] = C(1);

    cout << default_constructed << constructed << assigned;
}
```
Ответ: 111  
Пояснение: The `[]` operator inserts an element if the key is not present. In the case of a class, the element is default constructed. So doing `m[7]` calls the default constructor of `C` (no matter if we assign to it right after), setting `default_constructed` to `true`.

The expression `C(1)` constructs an instance of `C` using the constructor taking an `int`, setting `constructed` to `true`.

The `=` in `m[7] = C(1)` calls the copy assignment operator to copy assign the newly created `C(1)` to the previously default constructed `C` inside the map, setting `assigned` to `true`.
