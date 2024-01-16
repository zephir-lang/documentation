# Calling Functions

In Zephir, you can call PHP functions directly within your code. Here's an example:

```zephir
namespace MyLibrary;

class Encoder
{
    public function encode(var text)
    {
        if strlen(text) != 0 {
            return base64_encode(text);
        }
        return false;
    }
}
```

You can also call functions that are expected to exist in the PHP userland, but are not necessarily built in to PHP itself:

```zephir
namespace MyLibrary;

class Encoder
{

    public function encode(var text)
    {
        if strlen(text) != 0 {
            if function_exists("my_custom_encoder") {
                return my_custom_encoder(text);
            } else {
                return base64_encode(text);
            }
        }
        return false;
    }
}
```

Remember that PHP functions expect and return dynamic variables. If you pass a statically typed variable, Zephir automatically creates a temporary dynamic variable to facilitate the function call:

```zephir
namespace MyLibrary;

class Encoder
{
    public function encode(string text)
    {
        if strlen(text) != 0 {
            // an implicit dynamic variable is created to
            // pass the static typed 'text' as parameter
            return base64_encode(text);
        }
        return false;
    }
}
```

When functions return dynamic values, explicit casting is needed to assign them to static variables:

```zephir
namespace MyLibrary;

class Encoder
{
    public function encode(string text)
    {
        string encoded = "";

        if strlen(text) != 0 {
            let encoded = (string) base64_encode(text);
            return "(" . encoded . ")";
        }
        return false;
    }
}
```

Zephir also supports dynamic function calls:

```zephir
namespace MyLibrary;

class Encoder
{
    public function encode(var callback, string text)
    {
        return {callback}(text);
    }
}
```
