---
layout: default
language: 'en'
version: '0.12'
---

# Compiler Warnings
The compiler raises warnings when it finds situations where the code can be improved, or a potential error can be avoided.

Warnings can be enabled via command line parameters, or can be added to the `config.json` to enable or disable them more permanently.

You can enable warnings by passing their name prefixed by `-w`:

```bash
zephir -wunused-variable -wnonexistent-function
```

Warnings can be disabled by passing their name prefixed by `-W`:

```bash
zephir -Wunused-variable -Wnonexistent-function
```

The following warnings are supported:

## unused-variable
Raised when a variable is declared but it is not used within a method. This warning is enabled by default.

```zephir
public function some()
{
    var e; // declared but not used

    return false;
}
```

## unused-variable-external
Raised when a parameter is declared but it is not used within a method.

```zephir
public function sum(a, b, c) // c is not used
{
    return a + b;
}
```

## possible-wrong-parameter-undefined
Raised when a method is called with a wrong type for a parameter:

```zephir
public function some()
{
    return this->sum("a string", "another");  // wrong parameters passed
}

public function sum(int a, int b)
{
    return a + b;
}
```

## nonexistent-function
Raised when a function is called that does not exist at compile time:

```zephir
public function some()
{
    someFunction(); // someFunction does not exist
}
```

## nonexistent-class
Raised when a class is used that does not exist at compile time:

```zephir
public function some()
{
    var a;

    let a = new \MyClass(); // MyClass does not exist
}
```

## non-valid-isset
Raised when the compiler detects that an 'isset' operation is being made on a non-array or -object value:

```zephir
public function some()
{
    var b = 1.2;

    return isset b[0]; // variable integer 'b' used as array
}
```

## non-array-update
Raised when the compiler detects that an array update operation is being made on a non-array value:

```zephir
public function some()
{
    var b = 1.2;
    let b[0] = true; // variable 'b' cannot be used as array
}
```

## non-valid-objectupdate
Raised when the compiler detects that an object update operation is being made on a non-object value:

```zephir
public function some()
{
    var b = 1.2;
    let b->name = true; // variable 'b' cannot be used as object
}
```

## non-valid-fetch
Raised when the compiler detects that a 'fetch' operation is being made on a non-array or -object value:

##### variable integer 'b' used as array

```zephir
public function some()
{
    var b = 1.2, a;
    fetch a, b[0];
}
```

## invalid-array-index
Raised when the compiler detects that an invalid array index is used:

```zephir
public function some(var a)
{
    var b = [];
    let a[b] = true;
}
```

## non-array-append
Raised when the compiler detects that an element is being appended to a non-array variable:

```zephir
public function some()
{
    var b = false;
    let b[] = "some value";
}
```
