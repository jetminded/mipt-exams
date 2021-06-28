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

## Пример 1.2
```
#include <iostream>

int main() {
  int i = 42;
  int j = 1;
  std::cout << i / --j;
}
```
Ответ: UB  
Пояснение: Integer division by zero is undefined behaviour. According to §[expr.mul]¶4 in the standard: "If the second operand of / or % is zero the behavior is undefined."

## Пример 1.3
```
#include <iostream>

struct C {
    C() = default;
    int i;
};

int main() {
    const C c;
    std::cout << c.i;
}
```
Ответ: CE  
Пояснение: We're trying to default-initialize _c_. This is not allowed since it is const and _C_ has a defaulted (not user-provided) constructor.





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



## Пример 2.3: Конструкторы при ромбовидном наследовании

```
#include <iostream>
using namespace std;

class A
{
public:
    A() { cout << "A"; }
    A(const A &) { cout << "a"; }
};

class B: public virtual A
{
public:
    B() { cout << "B"; }
    B(const B &) { cout<< "b"; }
};

class C: public virtual A
{
public:
    C() { cout<< "C"; }
    C(const C &) { cout << "c"; }
};

class D:B,C
{
public:
    D() { cout<< "D"; }
    D(const D &) { cout << "d"; }
};

int main()
{
    D d1;
    D d2(d1);
}
```
Ответ: ABCDABCd
Пояснение:
On the first line of main(), d1 is initialized, in the order A, B, C, D. That order is defined by §[class.base.init]¶13:
"
— First, and only for the constructor of the most derived class (§ 4.5), virtual base classes are initialized in the order they appear on a depth-first left-to-right traversal of the directed acyclic graph of base classes, where “left-to-right” is the order of appearance of the base classes in the derived class base-specifier-list.
— Then, direct base classes are initialized in declaration order as they appear in the base-specifier-list
(...)
— Finally, the compound-statement of the constructor body is executed.
"
So the output is ABCD.

On the second line, d2 is initialized. But why are the constructors (as opposed to the copy constructors) for the base classes, called? Why do we see ABCd instead of abcd?

As it turns out, an implicitly-defined copy constructor would have called the copy constructor of its bases (§[class.copy.ctor]¶14: "The implicitly-defined copy/move constructor for a non-union class X performs a memberwise copy/move of its bases and members."). But when you provide a user-defined copy constructor, this is something you have to do explicitly.

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


# Шаблончеки
```
#include <iostream>

using namespace std;

template <class T> void f(T) {
  static int i = 0;
  cout << ++i;
}

int main() {
  f(1);
  f(1.0);
  f(1);
}
```
Ответ: 112, для каждого типа реализуется своя void f(T) со своим static int i.
