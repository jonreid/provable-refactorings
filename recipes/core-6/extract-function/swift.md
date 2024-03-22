{{< Production >}}

# Extract Function

```mermaid
graph TD;
    Extract_Function["Extract Function"] --> Introduce_lambda["Introduce a lambda"];
    Introduce_lambda --> Move_lambda_to_outer_scope;
    Move_lambda_to_outer_scope --> |It compiles| Convert_to_function;
    Move_lambda_to_outer_scope --> |It does not compile| Notice_first_variable_that_doesnot_compile;
    Notice_first_variable_that_doesnot_compile --> Move_lambda_back_to_inner_scope;
    Move_lambda_back_to_inner_scope --> Introduce_variable;
    Introduce_variable --> |It compiles| Move_lambda_to_outer_scope;
    Introduce_variable --> |It does not compile| Add_return_clause;
    Add_return_clause --> Move_lambda_to_outer_scope;
```

If you get to the end of step 1, the refactoring is possible - it will produce a valid result.

This recipe is designed to do Extract Function and nothing else. If you want something else, there are other refactorings for that.

You can commit at the end of any numbered step. At each such point, the code will compile and produce exactly the same results as the original code.

# Constraints

This recipe only works on rvalue expressions or on whole blocks (surrounded by braces) or a single for/while/if statement. It does not work on single statements, due to variable lifetime changes. If you need to extract statements but not a block, first execute the Extract Block refactoring

# Recipe

## 1. Introduce a lambda

For a block/for/while/if, surround with:
 
```cpp
[&]() {
    // ...
}();
```

For an expression, surround it with:

```cpp
[&]() { return
    // ...
;}()
```

Compile single file. Possible errors:

* `not all control paths return a value`. You have an early return. Back up and either [Eliminate Early Return/Continue/Break (fix link)](#) or extract something different.
* `a break/continue statement may only be used within ...`.  You have a break/continue. Back up and either [Eliminate Early Return/Continue/Break (fix link)](#) or extract something different. 

Search the new lambda for any return statements (using find). If there are any returns:

If it's obvious that all code paths return, then add a `return` before the lambda:

```cpp
return [&]() { 
    //...
}();
```

If it's not obvious that all code paths return, then back up and either [Eliminate Early Return/Continue/Break (fix link)](#) or try something different.

## 2. Extract Variable

1. Assign the lambda to `Applesauce` and call it.  ([Extract Variable](/recipes/core-6/extract-variable/cpp.md))

For example,

```cpp
[&]() { 
    // ...
}();
```

becomes:

```cpp hl_lines="1 3"
auto Applesauce = [&]() {
    // ...
};  Applesauce();
```

Compile to make sure you didn't typo.

## 3. Set the return type

Set the return type on the lambda, even if it's `void`. In Visual Studio, the IntelliSense tooltip over `auto` will tell you the type.

```cpp
auto Applesauce = [&]() -> SOMETYPE {
    // ...
};
```

Compile to make sure you got the return type correct.

## 4. Capture explicitly

Replace `[&]` with `[this]` (or `[]` in a free function) and compile.

For each error about a variable that must be captured:
1. Copy the variable name
2. Paste it in to the capture list, prefixed with `&`. 
3. Repeat until green.

For example,

```cpp
auto Applesauce = [this, &foo]() -> bool {
    cout << foo;
};
```

The order of the capture list will influence the order of the parameters of the final function. If you want the parameters in a particular order, now is a good time to reorder the capture list.

## 5. Convert captures to parameters

1. Select the capture list (but not 'this') and Cut
2. Paste into the argument list, removing `&`s.
3. Paste into the parameter list and inject `const auto` in front of each parameter
4. Compile. 
5. If you get an error because the variable is modified, make it non-const

For example,

```cpp
auto Applesauce = [this, &i, &s]() -> void
{
	cout << ++i << s;
};
Applesauce();
```

becomes:

```cpp
auto Applesauce = [this](auto &i, const auto &s) -> void
{
	cout << ++i << s;
};
Applesauce(i, s);
```

## 6. Explicitly specify parameter types 

For each argument:

1. Go-to-definiton on the argument
2. Copy the variable type
3. Paste into the lambda parameter list
4. Compile

For example,

```cpp
int i = ...
std::string s = ...

auto Applesauce = [this](auto &i, const auto &s) -> void
{
    std::cout << ++i << s;
};
Applesauce(i, s);
```

becomes:

```cpp
int i = ...
std::string s = ...

auto Applesauce = [this](int &i, const std::string &s) -> void
{
    std::cout << ++i << s;
};
Applesauce(i, s);
```

### About pointer parameters and the `const` keyword

When the parameter is a pointer type, it's important to understand how the placement of the const keyword affects the parameter
type. 

For example, given a local variable in `Column* pCol = ...`, the original type is "Pointer to `Column`", so the pointer is what
must be const (so we know it's not reseated to point to another object). It's also important to specify that the `Column` itself is 
const to avoid losing the implicit constness in the original code as we make the data flow explicit.

```cpp
const Column* // WRONG - this is a non-const pointer to const column
Column const* // WRONG - same as above
Column* const // BETTER - this is a const pointer to non-const column
Column const * const // BEST - this is a const pointer to const column
```

To avoid ambiguity, always place the const keyword to the right of what you want to make const.

## 7. Try to eliminate `this` capture

1. Remove `this` from the capture list
2. Compile
3. If the compile fails, undo

For example,

```cpp
auto Applesauce = [this]() -> void {
    // ...
};
```

becomes:

```cpp
auto Applesauce = []() -> void {
    // ...
};
```

## 8. Convert lambda to function

* If `this` is captured, use 8A.
* If `this` is not captured, use 8B.

### 8A. Convert `this`-bound lambda to member function

1. Cut the lambda statement and paste it outside the current function.
2. Remove `= [this]`.
3. Mark the new method as `const`.
4. Copy the signature line.
5. Add `SomeClass::`
6. Paste the signature in to the class declaration in a private section.
7. Compile and remove `const` if necessary.

For example,

```cpp
class SomeClass
{
private:
    auto Applesauce () const -> void;
}

auto SomeClass::Applesauce () const -> void {
    // ...
};
```

### 8B. Convert non-`this` Lambda to free function

1. Cut the lambda statement and paste it above the current function.
2. Remove `= []`
3. Wrap it in an anonymous namespace
4. Compile and resolve any errors:
5. If the free function uses typedefs/aliases or classes nested in the original class, convert the free function to a private static function of the original class.

For example,

```cpp
namespace {
auto Applesauce () -> void {
    ...
};
}
```

### 9. Convert return value from lambda-like syntax to traditional syntax
 1. Select the return value (after the `->`) and cut it.
2. Delete the `->`
3. Select the word `auto` and paste the return value over it.
4. Remove the trailing semicolon.

 For example,
 ```cpp
namespace {
auto Applesauce () -> void {
    ...
};
}
```
 becomes:
 ```cpp
namespace {
void Applesauce () {
    ...
}
}
```
