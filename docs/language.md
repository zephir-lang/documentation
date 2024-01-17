# Basic Syntax

In this section, we'll delve into the fundamental aspects of Zephir's syntax, covering topics such as file and namespace organization, variable declarations, syntax conventions, and other essential concepts.

## Organizing Code in Files and Namespaces

Zephir introduces a structured approach to code organization. Each file should contain only one class, and every class must be within a namespace. The directory structure aligns with class and namespace names, similar to PSR-4 autoloading conventions.

For example:

```bash
mylibrary/
    router/
        exception.zep # MyLibrary\Router\Exception
    router.zep # MyLibrary\Router
```

Class in `mylibrary/router.zep`:

```zephir
namespace MyLibrary;

class Router
{

}
```

Class in `mylibrary/router/exception.zep`:

```zephir
namespace MyLibrary\Router;

class Exception extends \Exception
{

}
```

The file system structure must mirror the namespaces, ensuring consistency.

## Instruction separation

In Zephir, semicolons can be used to separate statements and expressions, akin to Java, C/C++, and PHP:

```zephir
myObject->myMethod(1, 2, 3); echo "world";
```

## Comments

Zephir supports 'C'/'C++' style comments - single-line with `// ...` and multi-line with `/* ... */`:

```zephir
// this is a one line comment

/**
 * multi-line comment
 */
```

Multi-line comments also serve as docblocks, influencing the generated code and providing documentation.

## Variable Declarations

In Zephir, all variables used in a given scope must be declared. This gives important information to the compiler to perform optimizations and validations. Variables must be unique identifiers, and they cannot be reserved words.

```zephir
// Declaring variables for the same type	in the same instruction
var a, b, c;

// Declaring each variable in separate lines
var a;
var b;
var c;
```

Variables can have optional initial default values:

```zephir
// Declaring variables with default values
var a = "hello", b = 0, c = 1.0;
int d = 50; bool some = true;
```

Variable names are case-sensitive, the following variables are different:

```zephir
// Different variables
var somevalue, someValue, SomeValue;
```

## Variable Scope

Variables declared are locally scoped to the method where they are defined:

```zephir
namespace Test;

class MyClass
{
    public function someMethod1()
    {
        int a = 1, b = 2;
        return a + b;
    }

    public function someMethod2()
    {
        int a = 3, b = 4;
        return a + b;
    }
}
```

## Super Globals

Global variables are not supported in Zephir. However, access to PHP's super-globals is possible:

```zephir
// Getting a value from _POST
let price = _POST["price"];

// Read a value from _SERVER
let requestMethod = _SERVER["REQUEST_METHOD"];
```

## Local Symbol Table

Zephir does not implement the dynamic variable behavior found in PHP. The language does not have a local symbol table, as all variables are compiled into low-level variables without context awareness. To create a variable in the PHP symbol table, a specific syntax is used:

```zephir
// Set variable $name in PHP
let {"name"} = "hello";

// Set variable $price in PHP
let name = "price";
let {name} = 10.2;
```
