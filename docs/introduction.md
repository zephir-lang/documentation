# Introducing Zephir

Welcome to Zephir, an open-source, high-level/domain-specific language designed to facilitate the creation and maintainability of extensions for PHP, with a focus on type and memory safety.

Zephir is a language that addresses the major needs of a PHP developer trying to write and compile code that can be executed by PHP. It supports both dynamic and static typing, and some of its features will be familiar to PHP developers.

The name Zephir is a contraction of the words Z(end) E(ngine)/PH(P)/I(nte)r(mediate). While this suggests that the pronunciation should be `zephyr`, the creators of Zephir actually pronounce it [zaefire][zaefire].

## Features

!!! info "NOTE"

    This version supports PHP 8.0, 8.1, 8.2 and 8.3

Zephir's main features include:

| Feature            | Description                                           |
|--------------------|-------------------------------------------------------|
| Type system        | dynamic/static                                        |
| Memory safety      | pointers or direct memory management are not allowed  |
| Compilation model  | ahead of time                                         |
| Memory model       | task-local garbage collection                         |


## Hello World!

Every language has its own "Hello World!" sample. In Zephir, this introductory example showcases some important features of the language.

In Zephir, code must be placed in classes, as the language is intended for creating object-oriented libraries/frameworks. Code outside a class is not allowed. Additionally, a namespace is required:

```zephir
namespace Test;

/**
 * This is a sample class
 */
class Hello
{
    /**
     * This is a sample method
     */
    public function say()
    {
        echo "Hello World!";
    }
}
```

Once this class is compiled, it will produce the following code, transparently compiled by gcc/clang/vc++:

```c
#ifdef HAVE_CONFIG_H
#include "config.h"
#endif

#include "php.h"
#include "php_test.h"
#include "test.h"

#include "kernel/main.h"

/**
 * This is a sample class
 */
ZEPHIR_INIT_CLASS(Test_Hello) {
    ZEPHIR_REGISTER_CLASS(Test, Hello, hello, test_hello_method_entry, 0);
    return SUCCESS;
}

/**
 * This is a sample method
 */
PHP_METHOD(Test_Hello, say) {
    php_printf("%s", "Hello World!");
}
```

We do not expect that a developer using Zephir must know or even understand C. However, if you have experience with compilers, PHP internals, or the C language, that will provide a clearer understanding of what's going on internally when working with Zephir.

## A Taste of Zephir

In the following examples, we'll describe just enough of the details to understand what's going on. The goal is to give you a sense of what programming in Zephir is like. We'll explore the details of the features in subsequent chapters.

The following example is very simple; it implements a class and a method, with a small program that checks the types of an array.

Let's examine the code in detail, so we can begin to learn Zephir syntax. There are a lot of details in just a few lines of code! We'll explain the general ideas here:

```zephir
namespace Test;

/**
 * MyTest (test/mytest.zep)
 */
class MyTest
{
    public function someMethod()
    {
        /* Variables must be declared */
        var myArray;
        int i = 0, length;

        /* Create an array */
        let myArray = ["hello", 0, 100.25, false, null];

        /* Count the array into a 'int' variable */
        let length = count(myArray);

        /* Print value types */
        while i < length {
            echo typeof myArray[i], "\n";
            let i++;
        }

        return myArray;
    }
}
```

In the method, the first lines use the `var` and `int` keywords. There are used to declare a variable in the local scope. Every variable used in a method must be declared with its respective type. This declaration is not optional - it helps the compiler warn you about mistyped variables, or about the use of variables out of scope, which usually ends in runtime errors.

Dynamic variables are declared with the keyword `var`. These variables can be assigned and reassigned to different types. On the other hand, the `i` and `length` variables are statically typed integer variables, that can only have integer values in the entire program execution.

In contrast with PHP, you are not required to put a dollar sign ($) in front of variable names.

Zephir follows the same comment conventions as Java, C#, C++, etc. A `// comment` goes to the end of a line, while a `/* comment */` can cross line boundaries.

Variables are, by default, immutable. This means that Zephir expects that most variables will stay unchanged. Variables that maintain their initial value can be optimized down by the compiler to static constants. When the variable value needs to be changed, the keyword `let` must be used:

```zephir
/* Create an array */
let myArray = ["hello", 0, 100.25, false, null];
```

By default, arrays are dynamically typed like in PHP - they may contain values of different types. Functions from the PHP userland can be called in Zephir code. In the next example, the function `count` is called, but the compiler can perform optimizations like avoiding this call, because it already knows the size of the array:

```zephir
/* Count the array into a 'int' variable */
let length = count(myArray);
```

Parentheses in control flow statements are optional. You can use them if you feel more comfortable doing so, but you aren't required to.

```zephir
while i < length {
    echo typeof myArray[i], "\n";
    let i++;
}
```

Since PHP only works with dynamic variables, methods always return dynamic variables. This means that if a statically typed variable is returned, in the PHP side you will get a dynamic variable that can be used in PHP code. Note that memory is automatically managed by the compiler, similarly to how PHP does it, so you don't need to allocate or free memory like in C.

## External Links

Some external resources that might be of interest:

- [Type system][type_system]
- [Memory safety][memory_safety]
- [Ahead-of-time compilation][ahead_of_time_compilation]
- [Memory management][memory_management]


[zaefire]: https://translate.google.com/#en/en/zaefire
[type_system]: https://en.wikipedia.org/wiki/Type_system
[memory_safety]: https://en.wikipedia.org/wiki/Memory_safety
[ahead_of_time_compilation]: https://en.wikipedia.org/wiki/Ahead-of-time_compilation
[memory_management]: https://en.wikipedia.org/wiki/Memory_management
