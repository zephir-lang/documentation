
# Arrays

Array manipulation in Zephir provides a way to leverage PHP arrays. An array in Zephir corresponds to the concept of a [hash table][hash_table] in other programming languages.

## Declaring Array Variables

Array variables can be declared using the keywords `var` or `array`:

```zephir
var a   = []; // array variable, its type can be changed
array b = []; // array variable, its type cannot be changed across execution
```

## Creating Arrays

Arrays are created by enclosing elements in square brackets:

##### Empty array

```zephir
let elements = [];
```

##### Array with elements

```zephir
let elements = [1, 3, 4];
```

##### Array with elements of different types

```zephir
let elements = ["first", 2, true];
```

##### Multi-dimensional array

```zephir
let elements = [[0, 1], [4, 5], [2, 3]];
```

Zephir supports hashes or dictionaries, similar to PHP:

##### Hash with string keys

```zephir
let elements = ["foo": "bar", "bar": "foo"];
```

##### Hash with numeric keys

```zephir
let elements = [4: "bar", 8: "foo"];
```

##### Hash with mixed string and numeric keys

```zephir
let elements = [4: "bar", "foo": 8];
```

## Updating arrays

Arrays are updated using square brackets:

##### String key

```zephir
let elements["foo"] = "bar";
```

##### Numeric key

```zephir
let elements[0] = "bar";
```

##### Multi-dimensional array

```zephir
let elements[0]["foo"] = "bar";
let elements["foo"][0] = "bar";
```

## Appending elements

Elements can be appended at the end of the array:

```zephir
let elements[] = "bar";
```

## Reading elements from arrays

Retrieve array elements using either string or numeric keys:

##### Using the string key

```zephir
let foo = elements["foo"];
```

##### Using the numeric key

```zephir
let foo = elements[0];
```

[array]: https://www.php.net/manual/en/language.types.array.php
[hash_table]: https://en.wikipedia.org/wiki/Hash_table
