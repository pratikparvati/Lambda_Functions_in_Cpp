# Lambda Functions in C++

C++ Lambdas are an important feature in modern C++, Lambdas are fancy name for *anonymous* function object which are used widely for writing in-place functions (i.e, right at the location where it is invoked or passed as an argument to a function) and generally enhances the functionality of C++. 

A lambda consists typically of three parts: a capture list `[]`, an optional parameter list `()` and a body `{}`; the expression `[](){}();` declares an empty lambda and immediately executes it, of course this is completely useless expression. The basic prototype of lambda expressions is the following.

```
[capture clause] (parameters) -> return-type
{       
    // body
}
```
- *capture clause* - these are variables that are copied inside the lambda to be used in the code.
- *parameters* - these are the arguments that are passed to the lambda at execution time.
- *body*  - the body of the function, which is the same as in regular functions.
- *return-type* - return type of the function.

Here’s a simple example:

```C++ 
auto addition = [](int x, int y) { return x + y; };
```
where, `[](int x, int y) { return x + y; }` is the lambda expression.

### Motivation behind lambda functions

The primary motivation for the lambda functions is to provide flexible and convenient means to define unnamed function objects for STL algorithms. Typically STL algorithms operate on container elements via function objects, these function objects are passed as arguments to the algorithms.

> **_NOTE:_** If you are not aware of function objects in C++, I highly recommend reading [this](https://pratikparvati.com/html/blogview.html?id=-MbS5_6Fzyc2WKvmWpml&lan=cpp) article before going ahead.

Consider the following code as an example

```C++
class addVecFunctor
{
private:
    int m_x;

public:
    addVecFunctor(int x) : m_x(x) {}

    void operator()(int x)
    {
        std::cout << x + m_x << ' ';
    }
};

int main()
{
    std::vector<int> vec{10, 20, 30, 40, 50};
    std::for_each(vec.begin(), vec.end(), addVecFunctor(100));
    return 0;
}
```
If you only use `addVecFunctor()` once and in that specific place, it doesn't make sense to write an entire class just to do something trivial and one-time. C++11 introduces lambdas allow you to write an inline, anonymous functor to replace the `class addVecFunctor`.

The above code can be replaced using lambda function as

```C++
int main()
{
    std::vector<int> vec{10, 20, 30, 40, 50};
    int addNum = 100;
    std::for_each(vec.begin(), vec.end(), [&addNum](int x)
                  { std::cout << addNum + x << ' '; });
    return 0;
}
```

##### Lambda under the hood
The compiler replaces the line `std::for_each(vec.begin(), vec.end(), [](int x) { std::cout << x << ' ';});` for the above code as:

```C++
class anonymous
{
private:
    int &addNum;

public:
    anonymous(int &_addNum) : addNum(_addNum) {}
    inline auto operator()(int x) const
    {
        std::cout << addNum + x << ' ';
    }
};

int addNum = 100;
std::for_each(vec.begin(), vec.end(), anonymous(addNum));
```

The same is observed in cppinsight(displays compiler view of the code) [here](https://cppinsights.io/s/3f1fcae1). The compiler generates an unique **closure object** (or function object) to be replaced as an argument inside `for_each` function. **Capture list** will become a **constructor argument** inside the closure class and the **lambda parameters**(as received from the vector iterator) will become an **argument to `operator()`**.

### Lambda capture list
These are some examples of typical capture lists:

| Capture lists   | Description                                                                                                      |
|-----------------|------------------------------------------------------------------------------------------------------------------|
| `[]`            | Empty capture list, nothing will be captured.                                                                    |
| `[foo]`         | Capture foo by copy.                                                                                             |
| `[&bar]`        | Capture bar by reference.                                                                                        |
| `[foo, &bar]`   | Capture foo by-copy and bar by-reference.                                                                        |
| `[=]`           | Capture anything named from the enclosing scope by-copy.                                                         |
| `[&]`           | Capture anything named from the enclosing scope by-reference.                                                    |
| `[&, foo, bar]` | Capture anything named from the enclosing scope by reference, except `foo` and `bar` which must be captured by-copy. |
| `[=, &foo]`     | Capture anything named from the enclosing scope by copy, except foo which must be captured by-reference

#### Capturing member variables inside lambda function
If you try to capture member variable directly by value or reference, then it will not work

```C++
class addVector
{
    // vector sum
    int sum = 0;
public:
    int getSum()
    {
        return sum;
    }
    void update(std::vector<int>& vec)
    {
        // Capturing member variable by value will not work
        // Will result in Compile Error
        std::for_each(vec.begin(), vec.end(), [sum](int element){
                sum += element; // Accessing member variable from outer scope
        });
    }
};
```
since `sum` is a non-static member variable, It will result in compile error. Hence, to capture the member variables inside lambda function, capture the `this` pointer by value.

```C++
std::for_each(vec.begin(), vec.end(), [this](int element){
    sum += element;
}
```
Capturing `this` pointer inside lambda function will automatically capture all the member variables for this object inside lambda.

### Lambda return type

Often, the return type does not need to be specified because the compiler can deduce it. The compiler can easily deduce the return type if you only have one return statement. If a lambda body does not have a return statement, the lambda’s return type is `void`. If a lambda body consists of just a single return-statement, the lambda’s return type is the type of the return’s expression. It may be necessary or preferable to specify it explicitly for more complex lambdas.

```C++
auto func = [](int i){ return i; }; // return type is int
```

### `mutable` keyword for lambda

The lambda function call operator is `const` expression; which means lambda requires `mutable` keyword to modify any capture by value variable inside a capture list.

For instance, the below code throws compiler error
```C++
int val;
auto bad_lambda = [val]() { val += 100; };
```
The copy captures can be made writable if the lambda is declared as `mutable`; any change you do to the copy capture will be carried over to the next execution of the same lambda.

```C++
int val{100};

auto good_lambda = [val]() mutable {
    std::cout << val++ << std::endl;
};

// the lambda remembers the state of the closure object
good_lambda(); // prints 100
good_lambda(); // prints 101
good_lambda(); // prints 102

// the original variable outside of the lambda is unchanged
std::cout << val << std::endl; // prints 100
```

### Naming a Lambda Function

There are three possible ways to give lambda a name.

##### 1. Using `std::function<>`

we can use `std::function<>` to hold the function pointer. The below code snippet uses `std::function<>` which takes integer as function argument.

```C++
std::function<void(int)> print = [](int val)
{
    std::cout << val << std::endl;
};
// calling lambda function with explicit std::function handler
print(100);
```
##### 2. Using `auto` keyword
The lambda can be simply assigned a name by using C++ `auto` keyword; this is the simplest way of assigning a name to a lambda.

```C++
auto print = [](int val)
{
    std::cout << val << std::endl;
};
// calling lambda function 
print(100);
```
##### 3. Using C function pointers
Lambdas are just functions and can be assigned to a 'C' style function pointers; this is seriously discouraged.

```C++
void(*print)(int) = [](int val)
{
    std::cout << val << std::endl;
};
// calling function pointer
print(100);
```
### Immediately invoke a C++ lambda (IIFE)

An immediately invoked function expression (IIFE) is a function, which runs as soon as it is created. Since lambdas are kept anonymous, they are used and forgotten immediately. It is good way of declaring variables and executing code without polluting the global namespace. The parenthesis immediately following the closing braces invokes the function immediately and hence the name immediately invoking function expression. 

```C++
#include <iostream>

auto main() -> int {

  [](){
    std::cout <<"Hello IIFE" <<"\n";
  }(); // Invoking the lambda immediately, so it'll print "Hello IIFE"

  return 1;
}
```

The above code works perfectly fine and it is evident that C++ IIFE useful for converting a series of statements into an expression. This technique uses an anonymous function inside parentheses to convert it from a declaration to an expression, which is executed immediately. Also note that, everything that you define within the IIFE can only be accessed within the IIFE.

### Assembly code generated by the lambda
Whether or not you capture variables, the assembler code is the same as for a regular class. The only exception is that when capturing variables, the function Object() is *inlined* because it will not be used again.

Please read [this](https://web.mst.edu/~nmjxv3/articles/lambdas.html) article for deeper dive.

Finally, Lambda functions are very useful, even necessary, for effective use of modern C++ libraries and offer a clean and concise alternative to writing functions. They can be declared in various ways based on the needs of the users by utilizing capture clauses, parameters and return type(which are optional).






