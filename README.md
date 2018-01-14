# Building the lambda

According to the general C++ knowledge the lambda expression is just the function object underneath.

The code example:
```cpp
int main() {
    auto func = [](){return 0;};
    return func();
}
``` 
https://godbolt.org/g/VzgM7Q

Should be the same as the following one:
```cpp
struct __some_name{
    auto operator()(){
        return 0;
    }
};

int main() {
    auto func = __some_name{};
    return func();
}
```
https://godbolt.org/g/XghiRY

And it seems it is - according to the compiler output (gcc 7.2).

There are two differences in the operator() signature: `main::{lambda()#1}::operator()() const` vs `__some_name::operator()()` :

 - The lambda expression's operator() is const by default (unless the lambda is declared as `mutable`)
 - The lambda object is declared in the scope of the lambda expression

```cpp
int main() {
    struct __some_name{
        auto operator()() const{
            return 0;
        }
    };
    auto func = __some_name{};
    return func();
}
```
https://godbolt.org/g/uRPoKa

Our lambda seems to be complete, but the non-capturing lambda expression has one interesting feature - it can be assigned to function pointer!

Our lambda-to-be doesn't provide this one.

```cpp
int main () {
    auto func = []{return 0;};
    int (*fptr)() = func;    
    return fptr();
}
```
https://godbolt.org/g/fyksCq

In the assembly output we can see two new methods within the lambda object: 
- The conversion operator `main::{lambda()#1}::operator int (*)()() const`
- The helper funcion `main::{lambda()#1}::_FUN()`.

The conversion operator returns the offset of the `_FUN()` method.

The static method can be assigned to function pointer, so the lambda object can look like in this snippet.
```cpp
int main () {
    struct __some_name {
        typedef int(*fptr)();
        operator fptr () const {
            return _FUN;
        }
        int operator()() const {
            return _FUN();
        }
        static int _FUN() {
            return 0;
        }
    };
    __some_name func{};
    int (*fptr)() = func;
    return fptr();
}
```
https://godbolt.org/g/8X2JZ5

Analysing the assembly output we can observe that our snippet generate less instructions than the regular lambda generated code (25 vs 33 lines of assembly code). In our example the `operator() const` is removed because it is not used, while the regular lambda call it from the `_FUN()` function.

Unfortunately, it generates the compiler error because it's not possible to call non-static member function from the static member function.
We can't make the `operator()` static either.

```cpp
int main () {
    struct __some_name {
        typedef int(*fptr)();
        operator fptr () const {
            return _FUN;
        }
        int operator()() const {
            return 0;
        }
        static int _FUN() {
            return operator()(); // not possible
        }
    };
    __some_name func{};
    int (*fptr)() = func;
    return fptr();
}
```
https://godbolt.org/g/nxZ1Ty

To make it work and to generate the same assembly code we need to do some hacking.

**Don't try this at home!**

```cpp
int main () {
    struct __some_name {
        typedef int(*fptr)();
        operator fptr () const {
            return _FUN;
        }
        int operator()() const {
            return 0;
        }
        static int _FUN() {
            return ((__some_name*)nullptr)->operator()();
        }
    };
    __some_name func{};
    int (*fptr)() = func;
    return fptr();
}
```
https://godbolt.org/g/RNVYCE
