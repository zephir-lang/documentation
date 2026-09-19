# Contiguous buffer

A `Buffer` is a fixed-size, contiguous array of C scalars — either `double` or `int` — held inside an ordinary refcounted PHP object. A PHP array of a million floats costs a zval plus a hash-table slot per element, and the only way to hand it to a C numeric library is to walk it into a scratch buffer and walk the result back out. A `Buffer` is the place to keep that data *between* operations instead: the elements live in one `emalloc`'d C array, C code reads and writes them through a raw pointer, and a PHP array is materialised only at the boundary, by `toArray()`.

The governing rule is that **the payoff scales with the length of the chain, not with a single call**. Converting in, doing one operation, and converting straight back out is usually a net loss. See [Buffer performance](buffer-performance.md).

!!! warning "Not in a release yet"
    The buffer landed after 1.5.0 and is only on the `development` branch of [zephir-lang/zephir](https://github.com/zephir-lang/zephir). It was added for [issue #2721](https://github.com/zephir-lang/zephir/issues/2721).

File paths on this page are relative to the [zephir](https://github.com/zephir-lang/zephir) repository, not to this documentation repository.

## Turning it on

The class is **off by default** — an extension never gains a class it did not ask for. Opt in from your project's `config.json` with the top-level `kernel-classes` section:

```json
{
    "namespace": "myext",
    "kernel-classes": {
        "buffer": true
    }
}
```

!!! warning "Two ways to switch it on and get nothing"
    The switch is a strict identity check against `true` (`Compiler::bufferDefines()`). JSON `1`, `"true"` or `"yes"` all leave the class compiled out, with no warning.

    A top-level section from `config.json` **replaces** the corresponding default outright, it does not merge into it (`Config::offsetSet()`). Listing one kernel class in `kernel-classes` therefore drops the defaults of every other key in that section.

Turning the switch on changes the **generated** extension, not just the compiled binary, so regenerate before rebuilding:

```bash
php zephir fullclean
php zephir build
```

Confirm it took by looking for the two defines in the generated header:

```bash
grep ZEPHIR_BUFFER ext/php_myext.h
```

```c
#define ZEPHIR_BUFFER_ENABLED 1
#define ZEPHIR_BUFFER_NAMESPACE "Myext"
```

## What the class is called

The class is registered as `<RootNamespace>\Buffer`, where the root namespace is taken **proper-cased from a compiled class**, not from the lower-case `namespace` key in `config.json` (`Compiler::properCaseRootNamespace()`). Zephir's own stub extension declares `"namespace": "stub"` and gets `Stub\Buffer`.

If your project already declares a class with that name, the build stops with a `CompilerException` rather than shadowing it:

```
Class "Myext\Buffer" collides with the compiler-provided buffer class registered by
`kernel-classes.buffer`. Rename the class, or turn the option off in config.json.
```

## From Zephir

The compiler has no definition to check a `<Buffer>` type hint against — it is a hand-written kernel class, not a compiled `.zep` file — so buffer parameters are declared `var`. Indexing still takes the kernel fast path; the speed does not depend on the hint.

```zephir
namespace Stub;

class BufferOps
{
    public function readAt(var buf, int index)
    {
        return buf[index];
    }

    public function writeAt(var buf, int index, var value) -> void
    {
        let buf[index] = value;
    }

    public function has(var buf, int index) -> bool
    {
        return isset buf[index];
    }

    public function size(var buf) -> int
    {
        return count(buf);
    }
}
```

A read loop — this is the shape the class exists for:

```zephir
public function sum(var buf) -> double
{
    double total = 0.0;
    int i = 0, n = 0;

    let n = count(buf);

    while i < n {
        let total += (double) buf[i];
        let i++;
    }

    return total;
}
```

And a read-modify-write loop:

```zephir
public function scale(var buf, double factor) -> void
{
    int i = 0, n = 0;

    let n = count(buf);

    while i < n {
        let buf[i] = ((double) buf[i]) * factor;
        let i++;
    }
}
```

`buf[i]`, `let buf[i] = v` and `isset buf[i]` are the three operations that bypass `offsetGet()` / `offsetSet()` entirely. `count(buf)` does not need the fast path — it goes through the `count_elements` object handler, which is already a field read.

## From PHP

```php
use Stub\Buffer;

$buffer = new Buffer(3);              // 3 doubles, zero-filled
$buffer[0] = 1.5;
$buffer[1] = -2.5;

$buffer[0];                           // 1.5
count($buffer);                       // 3
$buffer->type();                      // Buffer::TYPE_DOUBLE
$buffer->toArray();                   // [1.5, -2.5, 0.0]

$ints = new Buffer(2, Buffer::TYPE_LONG);
$ints[0] = 42;
$ints->toArray();                     // [42, 0]

$fromArray = Buffer::fromArray([1.0, 2.0, 3.0]);
$fromArray->fill(7.5);
$fromArray->toArray();                // [7.5, 7.5, 7.5]
```

`fromArray()` is positional: keys are discarded and values are taken in iteration order.

```php
Buffer::fromArray(['b' => 2.0, 'a' => 1.0, 7 => 3.0])->toArray();   // [2.0, 1.0, 3.0]
```

## Choosing the element type

Two kinds, fixed at construction and never changed afterwards:

| Constant | C type | `toArray()` yields |
| --- | --- | --- |
| `Buffer::TYPE_DOUBLE` (default) | `double` | `float` |
| `Buffer::TYPE_LONG` | `int` (`zend_long`) | `int` |

A write converts with the engine's own cast — `zval_get_double()` or `zval_get_long()` — so an element ends up holding exactly what `(float)` or `(int)` would have produced, PHP's own notices included. There is **no type check and no `TypeError`**:

```php
$doubles = new Buffer(1);
$doubles[0] = '2.5';                  // 2.5
$doubles[0] = true;                   // 1.0
$doubles[0] = null;                   // 0.0

$longs = new Buffer(1, Buffer::TYPE_LONG);
$longs[0] = 3.9;                      // 3
$longs[0] = '12abc';                  // 12, with PHP's usual notice
```

If you need rejection rather than coercion, validate before writing.

## Traps

These are the places where a `Buffer` does not behave like a PHP array. None of them are bugs; all of them follow from the elements being raw C scalars rather than zvals.

### An element is not an lvalue

`zephir_buffer_read_dimension()` returns the value in a temporary, never a pointer into the buffer, because the elements are raw C scalars with no zval to point at. Compound assignment still works, because the engine implements it on an object as a read followed by a write:

```php
$buffer[0] += 5.0;                    // works: offsetGet, add, offsetSet
```

But anything that needs a *reference* to the element does not, and raises PHP's standard notice instead — *Indirect modification of overloaded element of `<Ns>\Buffer` has no effect*:

```php
$buffer[0]++;                         // notice; $buffer[0] unchanged
--$buffer[0];                         // notice; $buffer[0] unchanged
$ref =& $buffer[0];                   // notice; writes through $ref never land
settype($buffer[0], 'integer');       // notice; any by-reference parameter

$buffer[0] = $buffer[0] + 1.0;        // do this instead of ++
```

This is not specific to `Buffer` — it is what any `ArrayAccess` object does when it does not hand back a reference. In Zephir the same rule applies: read into a local, compute, write back, which is what `scale()` above does.

### `foreach` materialises the whole buffer

`getIterator()` builds an `ArrayIterator` over a full PHP array copy. That is deliberate — stepping one element at a time through the VM is the slow path by definition — but it means `foreach` over a million-element buffer allocates the million-element array you were trying to avoid. In hot code, index:

```php
for ($i = 0, $n = count($buffer); $i < $n; $i++) {
    // $buffer[$i]
}
```

### Every in-range element is set, including zero

A buffer holds numbers, and no number reads as absent, so `isset()` is true for any in-range index on a freshly constructed buffer. This is the one place where the class deliberately disagrees with `SplFixedArray`, whose slots start out `null`.

```php
$buffer = new Buffer(2);
isset($buffer[0]);                    // true  (SplFixedArray: false)
isset($buffer[2]);                    // false (out of range)
$buffer[9] ?? 'fallback';             // 'fallback', no exception
```

### `unset()` zeroes, it does not remove

There is no hole to make:

```php
$buffer = Buffer::fromArray([1.5, 2.5]);
unset($buffer[0]);
$buffer->toArray();                   // [0.0, 2.5]
count($buffer);                       // still 2
```

### `==` compares nothing useful

No `compare` handler is installed, so two buffers fall back to the engine's default object comparison — which compares declared properties, and a buffer has none. Every buffer therefore compares equal to every other buffer, whatever its contents, length or element type:

```php
Buffer::fromArray([1.0]) == Buffer::fromArray([2.0]);          // true (!)
Buffer::fromArray([1.0]) == Buffer::fromArray([1.0, 2.0]);     // true (!)

Buffer::fromArray([1.0])->toArray() == Buffer::fromArray([2.0])->toArray();   // false
```

Compare `toArray()` results, or compare element by element. `===` still means identity, so it behaves as expected.

### `(array)` casts to an empty array

There is no `cast_object` handler and there are no properties, so `(array) $buffer` is `[]`. Use `toArray()`. `var_dump()` does show the elements — that goes through `get_debug_info`.

### Fixed size, positional, numeric

No `$buffer[] = $v` append, no string keys, no resizing, and `fromArray()` discards keys. A `Buffer` is `final` and rejects dynamic properties.

## Cookbook

**Hold a buffer in a property.** It is an ordinary object: refcounted, storable, garbage-collected.

```zephir
namespace Myext;

class Signal
{
    protected buf;

    public function __construct(int size)
    {
        let this->buf = new \Myext\Buffer(size);
    }

    public function at(int i) -> double
    {
        var buf;

        let buf = this->buf;

        return (double) buf[i];
    }
}
```

Copy the property into a local before a hot loop. A property read returns a fresh temporary each time; a local does not.

**Convert only at the boundary.** Accept and return PHP arrays in your public API if you must, but keep the buffer alive across the intermediate steps:

```php
$buf = Buffer::fromArray($input);     // one conversion in
$math->normalize($buf);               // no conversion
$math->scale($buf, 2.0);              // no conversion
$math->clamp($buf, 0.0, 1.0);         // no conversion
return $buf->toArray();               // one conversion out
```

**Serialise and encode.** Both work, and both go through the element list:

```php
json_encode(Buffer::fromArray([1.5, 2.5]));      // '[1.5,2.5]'
unserialize(serialize($buffer))->toArray();      // round-trips type and values
```

`json_encode()` emits a JSON list rather than `{}` because the class implements `JsonSerializable`.

**Clone is a deep copy.** The element array is copied, not shared:

```php
$copy = clone $original;
$copy[0] = 99.0;                      // $original[0] unchanged
```

## API reference

### Synopsis

```php
final class <Ns>\Buffer implements ArrayAccess, Countable, IteratorAggregate, JsonSerializable
{
    public const TYPE_DOUBLE = 1;
    public const TYPE_LONG   = 2;

    public function __construct(int $size, int $type = self::TYPE_DOUBLE);
    public static function fromArray(array $values, int $type = self::TYPE_DOUBLE);

    public function toArray(): array;
    public function type(): int;
    public function count(): int;
    public function fill(mixed $value): void;

    public function offsetExists(mixed $offset): bool;
    public function offsetGet(mixed $offset): mixed;
    public function offsetSet(mixed $offset, mixed $value): void;
    public function offsetUnset(mixed $offset): void;

    public function getIterator(): Traversable;
    public function jsonSerialize(): mixed;
    public function __serialize(): array;
    public function __unserialize(array $data): void;
}
```

Two details the synopsis smooths over: `__construct()` and `fromArray()` declare **no return type** in their arg-info, and the `$type` default is applied in C rather than in arg-info, so reflection reports the parameter as optional but shows no default-value expression.

The class is `final` and carries `ZEND_ACC_NO_DYNAMIC_PROPERTIES`, so it can neither be extended nor grow ad-hoc properties.

### Methods

| Method | Behaviour |
| --- | --- |
| `__construct(int $size, int $type = self::TYPE_DOUBLE)` | Allocates `$size` zero-filled elements. `$size === 0` is legal and allocates nothing. `$size < 0` raises a `ValueError` on argument #1; an unknown `$type` raises one on argument #2. |
| `static fromArray(array $values, int $type = self::TYPE_DOUBLE)` | Builds a buffer from the **values** of `$values` in iteration order. Keys are discarded — a buffer is positional. Each value is converted with the same cast a write uses. |
| `toArray(): array` | Materialises the elements as a packed PHP list. O(n), and the main boundary cost. |
| `type(): int` | Returns `TYPE_DOUBLE` or `TYPE_LONG`. |
| `count(): int` | Element count. `count($buffer)` reaches the same value through the `count_elements` object handler, without a method call. |
| `fill(mixed $value): void` | Converts `$value` once, then writes it to every element. |
| `offsetExists(mixed $offset): bool` | True for every in-range index, including one holding zero. |
| `offsetGet(mixed $offset): mixed` | Returns `float` or `int` according to the buffer's kind. |
| `offsetSet(mixed $offset, mixed $value): void` | Coerces the value; never rejects on type. |
| `offsetUnset(mixed $offset): void` | Writes the zero element. It does not shrink the buffer or create a hole. |
| `getIterator(): Traversable` | Returns an `ArrayIterator` over a **fully materialised** copy of the elements. There is no lazy iterator. |
| `jsonSerialize(): mixed` | Returns the element list, so `json_encode($buffer)` emits `[1.5,2.5]`. Without it the engine would serialise an object with no properties as `{}` and drop every element. |
| `__serialize(): array` | The payload is a two-element list, `[kind, elements]`. |
| `__unserialize(array $data): void` | Validates that payload and throws on anything else. |

The four `offset*` methods are the `ArrayAccess` surface; `$buffer[$i]` reaches the same code through the dimension handlers rather than through a method call.

### Offset conversion

Any offset is converted to an index by `zephir_buffer_offset_to_long()`, which follows php-src's own `spl_offset_convert_to_long()`:

| Offset | Becomes |
| --- | --- |
| `int` | itself |
| numeric string (`'1'`) | that integer |
| `float` | truncated toward zero |
| `false` / `true` | `0` / `1` |
| reference | dereferenced, then re-examined |
| resource | its handle (deprecated from PHP 8.1) |
| anything else | illegal — see below |

The range check is a single unsigned compare, so a negative index is out of range rather than counted from the end.

### Errors

| Situation | Raises | Message |
| --- | --- | --- |
| Index out of range (read, write, unset) | `OutOfBoundsException` on PHP ≥ 8.4, `RuntimeException` below | `Index invalid or out of range` |
| Illegal offset **type**, PHP ≥ 8.3 | `TypeError` from `zend_illegal_container_offset()` | engine text, naming the container, e.g. `Cannot access offset of type string on <Ns>\Buffer` |
| Illegal offset type, PHP 8.1 / 8.2 | `TypeError` | `Illegal offset type` |
| Illegal offset type, PHP 8.0 | *(no type check)* | the offset becomes `-1` and is reported by the range check instead |
| `$buffer[] = $v`, PHP ≥ 8.1 | `Error` | `[] operator not supported for <Ns>\Buffer` |
| `$buffer[] = $v`, PHP 8.0 | `RuntimeException` | `Index invalid or out of range` |
| `$buffer[]` in a read context | `Error` | `Cannot use [] for reading` |
| `new Buffer(-1)` | `ValueError` | `<Ns>\Buffer::__construct(): Argument #1 ($size) must be greater than or equal to 0` |
| Invalid `$type` | `ValueError` | `... Argument #2 ($type) must be Buffer::TYPE_DOUBLE or Buffer::TYPE_LONG` |
| Bad `__unserialize()` payload | `Exception` | `Invalid serialization data for <Ns>\Buffer object` |
| `$buffer->anything = 1` | `Error` | engine text for a class with no dynamic properties |
| Writing a value of the "wrong" type | *(nothing)* | there is no type check; the value is cast |

### PHP-version matrix

Three diagnostics moved in php-src, and a `Buffer` follows each of them so that it reports on every version whatever `SplFixedArray` reports there.

| Behaviour | 8.0 | 8.1 | 8.2 | 8.3 | 8.4 | 8.5 |
| --- | --- | --- | --- | --- | --- | --- |
| Out-of-range class | `RuntimeException` | `RuntimeException` | `RuntimeException` | `RuntimeException` | `OutOfBoundsException` | `OutOfBoundsException` |
| Illegal offset type | no check | `TypeError: Illegal offset type` | same | `TypeError` naming the container | same | same |
| `$buffer[] =` | `RuntimeException` | `Error: [] operator not supported` | same | same | same | same |
| Resource as offset | accepted | accepted, deprecated | same | same | same | same |

`OutOfBoundsException` extends `RuntimeException`, so a `catch (RuntimeException)` written for 8.3 keeps working on 8.4.

!!! warning "`SplFixedArray` is the test oracle, but not a safe reference for `isset()`"
    `BufferTest` asserts parity by evaluating the same statement against a live `SplFixedArray` and comparing transcripts. That is the right oracle for offset diagnostics and the reason the table above tracks php-src at all.

    It is the wrong oracle for one thing. `SplFixedArray` slots start out `null`, so `isset($fixed[0])` is **false** on a freshly constructed instance. A numeric buffer's zero is a value like any other, so `isset($buffer[0])` is **true**. If you are comparing the two, fill both sides first and assert the `isset()` difference on its own.

### Standard operations

| Operation | Result |
| --- | --- |
| `count($b)` | element count, via the `count_elements` handler |
| `foreach ($b as $i => $v)` | index ⇒ value, in order, over a materialised copy |
| `clone $b` | deep copy; the element array is duplicated |
| `var_dump($b)` / `print_r($b)` | shows the elements, via `get_debug_info` |
| `json_encode($b)` | a JSON list |
| `serialize($b)` / `unserialize()` | round-trips kind, values and count |
| `$b == $b2` | **always `true`** between any two buffers — no `compare` handler, and no properties for the engine's default comparison to look at. Compare `toArray()` results instead |
| `(array) $b` | `[]` — no `cast_object` handler and no properties. Use `toArray()` |
| `$b[0] += 1`, `$b[0] *= 2` | works — the engine does a read followed by a write |
| `$b[0]++`, `--$b[0]`, `$r =& $b[0]`, a by-reference argument | *Indirect modification of overloaded element* notice, no effect |

## See also

- [Buffer performance](buffer-performance.md) — where the win comes from and how to measure it.
- [Buffer internals](buffer-internals.md) — object layout and the raw-pointer C API for extension authors.
