# Closures

Zephir supports closures, also known as anonymous functions, which are PHP-compatible and can be seamlessly utilized in Zephir. These closures can be returned to the PHP userland.

Consider the following example:

```zephir
namespace MyLibrary;

class Functional
{

    public function map(array! data)
    {
        return function(number) {
            return number * number;
        };
    }
}
```

In this example, a closure is defined within the map method, taking a number as a parameter and returning its square.

Closures can also be executed directly within Zephir and passed as parameters to other functions/methods:

```zephir
namespace MyLibrary;

class Functional
{

    public function map(array! data)
    {
        return data->map(function(number) {
            return number * number;
        });
    }
}
```

Here, the closure is employed within the map function, applying the defined transformation to each element of the array.

Additionally, Zephir provides a concise syntax for defining closures:

```zephir
namespace MyLibrary;

class Functional
{

    public function map(array! data)
    {
        return data->map(number => number * number);
    }
}
```

The short syntax offers a more compact way to express closures, enhancing code readability.

## Passing methods as PHP callbacks

PHP's higher-order functions (`array_reduce`, `array_map`, `array_filter`, `array_walk`, `usort`, `preg_replace_callback`, `call_user_func`, etc.) validate their callable argument before invoking it. The validator checks visibility against the *calling scope*, which it determines by walking the call stack for the nearest user-code frame.

Zephir methods compile to `ZEND_INTERNAL_FUNCTION`. The scope walker skips internal-function frames, so when a Zephir method passes a `[ClassName, "method"]` array (or `[this, "method"]`) directly to one of those PHP functions, PHP can't see the Zephir method as the caller. If the referenced method is `protected` or `private`, the validator reports it as out-of-scope and the call fails:

```zephir
namespace MyLibrary;

class Matrix
{
    protected a;

    public function __toString() -> string
    {
        // FAILS at runtime with:
        //   TypeError: array_reduce(): Argument #2 ($callback) must be a
        //   valid callback, cannot access protected method
        //   MyLibrary\Matrix::implodeRow()
        return array_reduce(
            this->a,
            ["MyLibrary\\Matrix", "implodeRow"],
            ""
        );
    }

    protected static function implodeRow(string carry, array row) -> string
    {
        return carry . "[ " . implode(" ", row) . " ]";
    }
}
```

### Workaround: wrap the callback in a Zephir closure

A Zephir closure is automatically bound to the enclosing class scope, so a delegating call from inside the closure body passes the visibility check. Use the **fully-qualified absolute class name** for the delegating call, because `self::` and `static::` inside a closure body resolve to the anonymous closure class, not the enclosing class:

```zephir
namespace MyLibrary;

class Matrix
{
    protected a;

    public function __toString() -> string
    {
        return array_reduce(
            this->a,
            function (carry, row) {
                // Use the fully-qualified absolute name (leading `\`).
                // `self::` / `static::` inside a closure body would
                // resolve to the anonymous closure class.
                return \MyLibrary\Matrix::implodeRow(carry, row);
            },
            ""
        );
    }

    protected static function implodeRow(string carry, array row) -> string
    {
        return carry . "[ " . implode(" ", row) . " ]";
    }
}
```

The same workaround applies to the instance-method form (`[this, "method"]`):

```zephir
return array_map(
    function (item) {
        return this->transform(item);
    },
    this->items
);
```

If the method is already `public`, you can pass the `[ClassName, "method"]` array directly. The workaround is only needed for `protected` and `private` callables.

See [issue #2167](https://github.com/zephir-lang/zephir/issues/2167) and [#2321](https://github.com/zephir-lang/zephir/issues/2321) for the background discussion.
