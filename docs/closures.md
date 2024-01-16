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
