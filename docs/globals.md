# Extension Globals

In Zephir, extension globals provide a way to define and manage global variables within an extension. These globals are designed for simple scalar types such as `int`, `bool`, `double`, `char`, etc. It's a mechanism to set up configuration options that can influence the behavior of your library.

To enable extension globals, you need to add a specific structure to your `config.json`:

```json
{
    "globals": {
        "allow_some_feature": {
            "type": "bool",
            "default": true,
            "module": true
        },
        "number_times": {
            "type": "int",
            "default": 10
        },
        "some_component.my_setting_1": {
            "type": "bool",
            "default": true
        },
        "some_component.my_setting_2": {
            "type": "int",
            "default": 100
        }
    }
}
```

Each global has the following structure:

```json
"<global-name>": {
    "type": "<some-valid-type>",
    "default": <some-compatible-default-value>
}
```

For compound (namespaced) globals:

```json
"<namespace>.<global-name>": {
    "type": "<some-valid-type>",
    "default": <some-compatible-default-value>
}
```

The optional `module` key, if present, places the global's initialization process into the module-wide `GINIT` lifecycle event. This means it is set up only once per PHP process, rather than being reinitialized for every request, which is the default.

```json
{
    "globals": {
        "allow_some_feature": {
            "type": "bool",
            "default": true,
            "module": true
        },
        "number_times": {
            "type": "int",
            "default": 10
        }
    }
}
```

In the example above, `allow_some_feature` is set up only once at startup; `number_times` is set up at the start of each request.

Inside any method, you can read/write extension globals using the built-in functions `globals_get`/`globals_set`:

```zephir
globals_set("allow_some_feature", true);
let someFeature = globals_get("allow_some_feature");
```

To modify these globals from PHP, you can create a method for this purpose:

```zephir
namespace Test;

class MyOptions
{
    public static function setOptions(array options)
    {
        boolean someOption, anotherOption;

        if fetch someOption, options["some_option"] {
            globals_set("some_option", someOption);
        }

        if fetch anotherOption, options["another_option"] {
            globals_set("another_option", anotherOption);
        }
    }
}
```

Note that extension globals cannot be dynamically accessed, as the C code generated by the `globals_get`/`globals_set` optimizers must be resolved at compilation time:

```zephir
let myOption = "someOption";

// will throw a compiler exception
let someOption = globals_get(myOption);
```
