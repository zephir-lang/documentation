# Operator Precedence

**Operator precedence** is crucial in determining how expressions are parsed and evaluated. It dictates the order in which operators are applied when multiple operators are present in an expression. For example, in the expression `1 + 5 * 3`, the answer is 16 and not 18 because the multiplication (`*`) operator has a higher precedence than the addition (`+`) operator. Parentheses may be used to force precedence, if necessary. For instance: `(1 + 5) * 3` evaluates to 18.

Here's a breakdown of operator precedence in Zephir:

## Precedence Table

The table below lists operators in order of precedence, from highest to lowest. Operators on the same line have equal precedence.

| Precedence | Operators                 | Operator type                       | Associativity   |
|------------|---------------------------|-------------------------------------|-----------------|
| 1          | `->`                      | Member Access                       | right-to-left   |
| 2          | `~`                       | Bitwise NOT                         | right-to-left   |
| 3          | `!`                       | Logical NOT                         | right-to-left   |
| 4          | `new`                     | new                                 | non-associative |
| 5          | `clone`                   | clone                               | right-to-left   |
| 6          | `typeof`                  | Type-of                             | right-to-left   |
| 7          | `..`, `...`               | Inclusive/exclusive range           | left-to-right   |
| 8          | `isset`, `fetch`, `empty` | Exclusive range                     | right-to-left   |
| 9          | `*`, `/`, `%`             | Multiplication, Division, Remainder | left-to-right   |
| 10         | `+`, `-`, `.`             | Addition, Subtraction, Concat       | left-to-right   |
| 11         | `<<`, `>>`                | Bitwise shift left/right            | left-to-right   |
| 12         | `<`, `<=`, `=>`, `>`      | Comparison                          | left-to-right   |
| 13         | `==`, `!==`, `===`, `!==` | Comparison                          | left-to-right   |
| 14         | `&`                       | Bitwise AND, references             | left-to-right   |
| 15         | `^`                       | Bitwise XOR                         | left-to-right   |
| 16         | `|`                       | Bitwise OR                          | left-to-right   |
| 17         | `instanceof`              | Instance-of                         | left-to-right   |
| 18         | `&&`                      | Logical AND                         | left-to-right   |
| 19         | `||`                      | Logical OR                          | left-to-right   |
| 20         | `likely`, `unlikely`      | Branch prediction                   | right-to-left   |
| 21         | `?`                       | Logical                             | right-to-left   |
| 21         | `=>`                      | Closure Arrow                       | right-to-left   |


## Associativity

Associativity determines how operators with equal precedence are grouped. It specifies the order in which operators are evaluated when they have the same precedence. The associativity is either left-to-right or right-to-left.

For example, `-` is left-associative, so `1 - 2 - 3` is grouped as `(1 - 2) - 3` and evaluates to `-4`. On the other hand, `=` is right-associative, so `let a = b = c` is grouped as `let a = (b = c)`.

### Non-Associative Operator
In Zephir, `new` is a non-associative operator. This means that using it consecutively without explicit grouping is not allowed. For instance:

```zep
new new Foo();
```

This expression is illegal in Zephir because the `new` operator is non-associative.

Understanding operator precedence and associativity is essential for writing correct and predictable expressions in Zephir. Explicitly using parentheses when needed can improve code readability.
