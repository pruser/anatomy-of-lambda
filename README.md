# Building the lambda

According to the general C++ knowledge the lambda expression is just the function object underneath.
The C++ standard call these objects closures.

```
A lambda-expression is a prvalue whose result object is called the closure object.
``` 
§ 8.1.5 Lambda expressions - expr.prim.lambda

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

 - The closure operator() is const by default (unless the lambda is declared as `mutable`)
 - The closure object is declared in the scope where the lambda expression is declared

```
The closure type is declared in the smallest block scope, class scope, or namespace scope that contains the
corresponding lambda-expression.
``` 
§ 8.1.5.1 Closure types - expr.prim.lambda.closure

```
The function call operator or operator template is declared const (12.2.2) if and only if the lambda-expression’s
parameter-declaration-clause is not followed by mutable. It is neither virtual nor declared volatile. Any
noexcept-specifier specified on a lambda-expression applies to the corresponding function call operator or
operator template. An attribute-specifier-seq in a lambda-declarator appertains to the type of the corresponding
function call operator or operator template. The function call operator or any given operator template
specialization is a constexpr function if either the corresponding lambda-expression’s parameter-declaration clause
is followed by constexpr, or it satisfies the requirements for a constexpr function (10.1.5).
``` 
§ 8.1.5.1 Closure types - expr.prim.lambda.closure

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

Our closure object seems to be complete, but the non-capturing lambda expression has one interesting feature - it can be assigned to function pointer!

```
 The closure type for a non-generic lambda-expression with no lambda-capture has a conversion function to
pointer to function with C++ language linkage (10.5) having the same parameter and return types as the
closure type’s function call operator. The conversion is to “pointer to noexcept function” if the function
call operator has a non-throwing exception specification. The value returned by this conversion function is
the address of a function F that, when invoked, has the same effect as invoking the closure type’s function
call operator. F is a constexpr function if the function call operator is a constexpr function. For a generic
lambda with no lambda-capture, the closure type has a conversion function template to pointer to function.
The conversion function template has the same invented template-parameter-list, and the pointer to function
has the same parameter types, as the function call operator template. The return type of the pointer to
function shall behave as if it were a decltype-specifier denoting the return type of the corresponding function
call operator template specialization.
``` 
§ 8.1.5.1 Closure types - expr.prim.lambda.closure

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

It may seem strange, but the outline of this design is proposed in the ISO C++ Standard.
```cpp
struct Closure {
    template<class T> auto operator()(T t) const { ... }
    template<class T> static auto lambda_call_operator_invoker(T a) {
        // forwards execution to operator()(a) and therefore has
        // the same return type deduced
        ...
    }
    template<class T> using fptr_t =
        decltype(lambda_call_operator_invoker(declval<T>())) (*)(T);
    template<class T> operator fptr_t<T>() const
    { 
        return &lambda_call_operator_invoker; }
    };
}
``` 
§ 8.1.5.1 Closure types - expr.prim.lambda.closure

