# Buffers

A `Buffer` is a fixed-size, contiguous array of C scalars — either `double` or `int` — held inside an ordinary refcounted PHP object. A PHP array of a million floats costs a zval plus a hash-table slot per element, and the only way to hand it to a C numeric library is to walk it into a scratch buffer and walk the result back out. A `Buffer` is the place to keep that data *between* operations instead: the elements live in one `emalloc`'d C array, C code reads and writes them through a raw pointer, and a PHP array is materialised only at the boundary, by `toArray()`.

The governing rule is that **the payoff scales with the length of the chain, not with a single call**. Converting in, doing one operation, and converting straight back out is usually a net loss. See [Performance](#performance).

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

## Performance

Where a buffer actually pays off, how to measure it, and what the numbers looked like when it was built.

### Where the win comes from

Two independent mechanisms. They compound, but it is worth keeping them apart when reasoning about whether a given workload will benefit.

**Storage.** A PHP array of `n` floats costs a zval plus a hash-table slot per element. A buffer is one `ecalloc(n, sizeof(double))` — roughly half the bytes for `double` elements, and contiguous, so it is cache- and prefetcher-friendly, and it can be handed to a C numeric library as a pointer with no packing step at all.

**Access.** `buf[i]` in Zephir source compiles to a `zephir_array_*` kernel call. Without the fast path that call would see an ordinary `ArrayAccess` object and dispatch a full `offsetGet()` or `offsetSet()` **method call** per element. The fast path in `kernel/array.c` puts a class-entry pointer compare ahead of the `ArrayAccess` branch and, on a match, does a direct scalar load or store without leaving C.

The second mechanism is why `buf[i]` from Zephir is worth using at all. Without the fast path a Zephir-side `buf[i]` read was measurably *slower* than the same read written in plain PHP — PHP goes through the `read_dimension` object handler, while Zephir's kernel was dispatching a full method call. With it, a compiled `buf[i]` read beats a compiled PHP-array read.

#### No type inference is involved

This is worth stating because it is the usual first question. The compiler cannot prove that `this->buf` holds a `Buffer`: every property read is typed `undefined` (`src/Expression/PropertyAccess.php`), and `Buffer` is a hand-written kernel class with no `.zep` definition to hint against anyway.

It does not matter. The dispatch is a runtime check — `Z_OBJCE_P(arr) == zephir_buffer_ce` — one predictable compare, placed where the kernel was already going to test the container's type. Declaring the parameter `var` costs nothing.

### The boundary cost

`fromArray()` and `toArray()` are both O(n) conversions, and they are not cheap in absolute terms. That produces the single most important rule for using the class:

!!! warning "The rule"
    The payoff scales with the length of the chain between conversions, not with the speed of any one operation.

Convert in, do one operation, convert out, and you will usually lose. Convert in, do five operations, convert out, and the conversions amortise away.

### Measuring it

`tests/Benchmark/BufferBench.php` drives the Zephir-side workloads in `stub/bench.zep`. Subjects:

| Subject | Measures |
| --- | --- |
| `benchZephirBufferSum` | compiled `buf[i]` read via the kernel fast path |
| `benchZephirArraySum` | compiled read over a PHP array (control) |
| `benchPhpArraySum` | interpreted PHP array read (baseline) |
| `benchPhpBufferSum` | interpreted `$buffer[$i]` — the `read_dimension` handler, no fast path |
| `benchZephirBufferWrite` | compiled `let buf[i] = v` |
| `benchBufferToArray` | the materialisation cost the buffer exists to avoid |
| `benchBufferFromArray` | the ingest cost |

Run them inside one of the prepared containers (`zephir-8.0` … `zephir-8.5`), where `/srv` is bind-mounted to the repository:

```bash
cd /srv
php zephir fullclean && php zephir build
php -d extension=ext/modules/stub.so vendor/bin/phpbench run --report=aggregate
```

The two comparisons that answer the two questions above:

- **Is the fast path working?** `benchZephirBufferSum` against `benchZephirArraySum`. Both are the same compiled loop; only the container differs. If the buffer subject is not at least as fast as the array one, the fast path is not being taken — check that `ZEPHIR_BUFFER_ENABLED` is actually defined in the generated header.
- **Is a buffer worth it for my data?** Take that difference and weigh it against `benchBufferFromArray` + `benchBufferToArray` for your `n` and your number of operations per round trip.

`benchPhpBufferSum` is the userland control rather than a fast-path measurement: from PHP, `$buffer[$i]` goes through the `read_dimension` object handler, not through `offsetGet()`, and the kernel fast path is not involved at all.

For an A/B across two builds, use the `--tag=base` / `--ref=base` flow documented in [tests/Benchmark/README.md](https://github.com/zephir-lang/zephir/blob/development/tests/Benchmark/README.md). Note that the reported percentage is **time**, so a negative delta means faster.

#### Observed numbers

One run on a development host, PHP 8.3.31, `n = 1000` per subject. Reproduce with the command above rather than quoting these — they depend on the CPU, the PHP build and the element count.

| Subject | Throughput | Per 1000-element loop |
| --- | --- | --- |
| `benchZephirBufferWrite` | 0.219 ops/μs | 4.6 μs |
| `benchZephirBufferSum` | 0.147 ops/μs | 6.8 μs |
| `benchZephirArraySum` | 0.125 ops/μs | 8.0 μs |
| `benchPhpArraySum` | 0.032 ops/μs | 31.7 μs |
| `benchPhpBufferSum` | 0.026 ops/μs | 37.8 μs |
| `benchBufferFromArray` | 0.680 ops/μs | 1.5 μs |
| `benchBufferToArray` | 0.155 ops/μs | 6.5 μs |

Three things to read out of it:

- **A compiled `buf[i]` read beats a compiled PHP-array read** — 6.8 μs against 8.0 μs. That is the fast path doing its job; a buffer element costs less than a hash lookup.
- **Doing the loop in Zephir is worth far more than the container choice** — 6.8 μs against 37.8 μs for the same loop written in PHP over the same buffer, about 5.6×. From PHP, indexing a buffer is *slower* than indexing an array (37.8 vs 31.7 μs), because PHP's array access is heavily optimised and the buffer goes through an object handler. A buffer pays off in compiled code and at the C boundary, not in userland loops.
- **The boundary costs are the size of an operation, not a rounding error** — `toArray()` at 6.5 μs costs about the same as one entire 1000-element read loop, and `fromArray()` at 1.5 μs about a fifth of one. Which is the arithmetic behind the rule above.

!!! note "Numbers that are not reproducible from the repository"
    During development the fast path was measured against a build with it disabled — where each `buf[i]` becomes a real `offsetGet()` / `offsetSet()` method call — at roughly **27× for reads and 48× for writes**, and a Zephir-side `buf[i]` read was about **5× slower than the same read written in PHP** before the fast path existed. Reproducing those requires reverting the `kernel/array.c` guards, so treat them as background on why the fast path exists rather than as numbers you can check.

A whole-suite sweep with the buffer compiled out versus compiled in showed every other subject inside noise, which is the expected result: every fast-path site is behind `#ifdef` and, when present, behind an `UNEXPECTED()` pointer compare.

### Case study: Tensor

!!! warning "External, and not reproducible from the Zephir repository"
    These numbers come from a prototype `Tensor\BufferVector` built against [Tensor](https://github.com/RubixML/Tensor) during the design of this feature, compared with the existing `Tensor\Vector` compiled into the **same `.so`**, so the comparison is apples to apples. One prototype, one host, one workload — it illustrates the shape of the win, it is not a promise about yours.

At `n = 1e6`:

| Operation | PHP-array path | Buffer path |
| --- | --- | --- |
| `add` | 32.9 ms | 3.9 ms |
| `multiply` | 35.6 ms | 3.7 ms |
| `dot` | 16.8 ms | 0.40 ms |
| chain `a.add(b).multiply(c)` | 69.5 ms | 8.3 ms |

`dot` is the outlier at ~42× because the array path never reached `cblas_ddot` at all — the packing step dominated it so thoroughly that the BLAS call was never worth making. That is the general lesson: a contiguous buffer does not just make the existing path faster, it makes a different path viable.

At `n = 1e5`, where the data fits in cache, the ratios rose to 50–70×. Storage for 1e6 doubles fell from **16.8 MB to 8.0 MB**.

And the boundary costs, which are what cap the win on a *single* operation:

| Boundary | Cost at n = 1e6 |
| --- | --- |
| `build` (array → buffer) | 3.9 ms |
| `asArray` (buffer → array) | 11.2 ms |

Put those next to the per-operation figures and the rule falls out: one `add` costs 3.9 ms against a 15.1 ms round trip, so a single operation loses. Four operations win comfortably.

### Measuring on your own host

One trap is worth knowing because it silently poisons results rather than failing loudly.

On a CPU without AVX — a QEMU vCPU advertising only up to `sse4_2`, for instance — OpenBLAS may dispatch a kernel built for instructions the CPU does not have, and the process dies with `SIGILL` (exit status 132). This happens on pristine sources too, so it is easy to misread as a regression in whatever you are testing. Pin the kernel:

```bash
export OPENBLAS_CORETYPE=NEHALEM
```

And do not hide it behind a pipe. `phpbench ... | tail` exits 0 even when the left-hand side died, so the failure disappears:

```bash
php -d extension=ext/modules/stub.so vendor/bin/phpbench run --report=aggregate | tail -20
echo "${PIPESTATUS[0]}"    # this is the exit status that matters
```

## Internals

For people writing C against a buffer, and for whoever adds the next kernel class. The source is `kernel/buffer.h` and `kernel/buffer.c`.

### Object layout

```c
typedef struct _zephir_buffer_object {
	union {
		double    *d;
		zend_long *l;
		void      *raw;
	} data;

	zend_long len;
	uint8_t   kind;

	/* MUST stay last: handlers.offset is XtOffsetOf(..., std). */
	zend_object std;
} zephir_buffer_object;
```

Three things matter here:

- **`zend_object std` must stay last.** The handler table sets `handlers.offset = XtOffsetOf(zephir_buffer_object, std)`, which is how the engine walks back from a `zend_object *` to the containing struct. Move the member and every access silently reads the wrong memory.
- **The union is the point.** The elements are never zvals. `kind` selects which arm is live; `data.raw` is `NULL` when `len == 0`.
- **The allocation is single and permanent.** One `ecalloc(len, elem_size)` at construction; the buffer is fixed-size, so it is never reallocated, which is what lets the raw pointer be handed out safely.

Zero-filling relies on all-bits-zero being `0.0` and `0` — true on every platform PHP supports.

### The C API for extension authors

This is the actual payoff of the class. Include `kernel/buffer.h` and you get:

```c
int        zephir_buffer_create(zval *ret, zend_long len, uint8_t kind);
int        zephir_buffer_create_from_array(zval *ret, zval *arr, uint8_t kind);
int        zephir_buffer_to_array(zval *ret, const zval *obj);

uint8_t    zephir_buffer_kind(const zval *obj);
zend_long  zephir_buffer_len(const zval *obj);

double    *zephir_buffer_doubles(const zval *obj);
zend_long *zephir_buffer_longs(const zval *obj);

static inline int zephir_is_buffer(const zval *zv);
```

Contract:

- `zephir_buffer_doubles()` / `zephir_buffer_longs()` return `NULL` for a non-buffer, for a buffer of the **other** kind, and for an empty buffer. Asking for the wrong arm is always a `NULL`, never a silently reinterpreted pointer.
- The pointer stays valid until the object is destroyed. Fixed size, no reallocation.
- The `create` helpers report failure with `FAILURE` **and** `ZVAL_NULL(ret)` — they do not throw. The PHP-visible constructor throws; the C entry points do not.
- `zephir_buffer_kind()` and `zephir_buffer_len()` return `0` for a non-buffer, so a length of zero is not by itself proof that you were handed a buffer.

#### Worked example

Handing the elements straight to BLAS, with no packing step:

```c
#include "kernel/buffer.h"
#include <cblas.h>

/* y := alpha * x + y, over two <Ns>\Buffer objects of doubles. */
PHP_METHOD(MyExt_Blas, axpy)
{
	zval      *x, *y;
	double     alpha;
	double    *xp, *yp;
	zend_long  n;

	ZEND_PARSE_PARAMETERS_START(3, 3)
		Z_PARAM_DOUBLE(alpha)
		Z_PARAM_OBJECT_OF_CLASS(x, zephir_buffer_ce)
		Z_PARAM_OBJECT_OF_CLASS(y, zephir_buffer_ce)
	ZEND_PARSE_PARAMETERS_END();

	n = zephir_buffer_len(x);

	if (n != zephir_buffer_len(y)) {
		zend_throw_error(NULL, "axpy() needs two buffers of the same length");
		RETURN_THROWS();
	}

	if (n == 0) {
		return;
	}

	xp = zephir_buffer_doubles(x);
	yp = zephir_buffer_doubles(y);

	if (!xp || !yp) {
		zend_throw_error(NULL, "axpy() needs two TYPE_DOUBLE buffers");
		RETURN_THROWS();
	}

	cblas_daxpy((int) n, alpha, xp, 1, yp, 1);
}
```

!!! warning "Check the length before the pointers"
    An empty buffer legitimately yields `NULL`, so a `!xp` guard placed first would reject a valid zero-length argument as a type error.

### Registration

`zephir_buffer_module_init()` does the whole job:

```c
INIT_NS_CLASS_ENTRY(ce, ZEPHIR_BUFFER_NAMESPACE, "Buffer", zephir_buffer_methods);
zephir_buffer_ce = zend_register_internal_class(&ce);
zephir_buffer_ce->ce_flags |= ZEND_ACC_FINAL | ZEND_ACC_NO_DYNAMIC_PROPERTIES;
zephir_buffer_ce->create_object = zephir_buffer_create_object;
```

then declares the two constants, copies `std_object_handlers` and overrides nine slots, and calls `zend_class_implements()` with the four interfaces.

It is called **unconditionally** from `zephir_module_init()` in `kernel/main.c`. The prototype is declared outside the enable guard in `kernel/buffer.h` precisely so that this one call site compiles whether the project opted in or not; when it did not, the whole translation unit collapses to an empty function.

Handlers overridden: `offset`, `free_obj`, `clone_obj`, `get_debug_info`, `count_elements`, `read_dimension`, `write_dimension`, `has_dimension`, `unset_dimension`.

Deliberately **not** overridden: `compare` (so `==` is the default object comparison, not element-wise), `cast_object`, `get_iterator` (iteration goes through `IteratorAggregate` instead), and the serialize handlers (`__serialize` / `__unserialize` magic methods instead).

`kernel/generator.c` was the template for all of this — same shape: a struct with `zend_object std` last, a `handlers.offset`, a `create_object`, and a MINIT hook. Copy that pair when adding the next kernel class.

### The opt-in path, end to end

```
config.json  "kernel-classes": { "buffer": true }
    -> Config::$defaults              default false
    -> Compiler::bufferDefines()      strict `true !==` check
                                      emits the two #defines
    -> %BUFFER_DEFINES%               in templates/engine/php_project.h
    -> ext/php_<project>.h            #define ZEPHIR_BUFFER_ENABLED 1
                                      #define ZEPHIR_BUFFER_NAMESPACE "<Ns>"
    -> kernel/buffer.c                whole class behind #ifdef
    -> kernel/array.c                 six fast-path sites behind #ifdef
```

Two properties of this design are worth stating explicitly, because they look like oversights otherwise:

**The source file is always compiled.** `kernel/buffer.c` is in the unconditional source list in both `templates/engine/config.m4` and `templates/engine/config.w32`, and both `.c`/`.h` files are always copied into the generated `ext/kernel/`. The compile-out happens in the preprocessor, not in the build files. A project that never opts in still builds the file — it just produces an empty `zephir_buffer_module_init()`.

**`ZEPHIR_BUFFER_NAMESPACE` is mandatory alongside `ZEPHIR_BUFFER_ENABLED`.** `kernel/buffer.c` has an `#error` for the half-configured case, so a partial define fails at compile time rather than registering a class in the wrong namespace.

The namespace itself comes from `Compiler::properCaseRootNamespace()`: the root segment of the first compiled class definition, falling back to `ucfirst()` of the `namespace` config key. That is why `"namespace": "stub"` produces `Stub\Buffer` rather than `stub\Buffer`.

#### Why an explicit switch rather than inference

The compiler cannot infer the need. There is no syntax that implies a buffer, and a class reference can be dynamic (`new {var}`), so there is nothing to detect. Leaving it explicit also guarantees that every existing extension gains nothing it did not ask for.

A project that declares its own `<Ns>\Buffer` while the switch is on gets a `CompilerException` rather than a silent shadowing.

### The `kernel/array.c` fast path

Six insertion points, each `#ifdef ZEPHIR_BUFFER_ENABLED`, each wrapped in `UNEXPECTED()`, and each placed as the **first** container test — ahead of the generic `Z_TYPE_P(arr) == IS_OBJECT && zephir_instance_of_ev(..., zend_ce_arrayaccess)` branch:

| Kernel function | Delegates to |
| --- | --- |
| `zephir_array_isset` | `zephir_buffer_dim_isset` |
| `zephir_array_isset_long` | `zephir_buffer_dim_isset_long` |
| `zephir_array_fetch` | `zephir_buffer_dim_read` |
| `zephir_array_fetch_long` | `zephir_buffer_dim_read_long` |
| `zephir_array_update_zval` | `zephir_buffer_dim_write` |
| `zephir_array_update_long` | `zephir_buffer_dim_write_long` |

They raise the same diagnostics as the object handlers, so `buf[i]` from Zephir and `$buffer[$i]` from PHP report identically — which is exactly what `tests/Extension/BufferZephirTest.php` asserts.

Three design points:

**The test is a pointer compare, not `instanceof`.** `zephir_is_buffer()` is `Z_TYPE_P(zv) == IS_OBJECT && Z_OBJCE_P(zv) == zephir_buffer_ce`. `zephir_instance_of_ev()` walks a class hierarchy; this does not. A subclass would therefore miss the fast path — moot, since the class is `final`.

**It goes ahead of the `ArrayAccess` branch, not inside it.** Placing it after would mean paying the `instanceof` walk first, which is most of what the fast path is trying to avoid.

**A write-context fetch deliberately falls through.** Both `fetch` guards exclude `PH_WRITE`:

```c
if (UNEXPECTED(zephir_is_buffer(arr)) && (flags & PH_WRITE) != PH_WRITE) {
	return zephir_buffer_dim_read_long(return_value, arr, index);
}
```

A buffer element is a raw C scalar with no zval to point at, so there is no lvalue to return. Falling through means the engine's own *Indirect modification of overloaded element* notice still fires for `$b[0]++`, `$r =& $b[0]` and by-reference arguments, matching what any other `ArrayAccess` object does; taking the fast path there would suppress the diagnostic and silently do nothing.

Plain compound assignment (`$b[0] += 1`) is *not* affected — the engine implements it on an object as a read plus a write, so it goes through the read fast path and then the write fast path and works normally. Only the reference-taking forms lose.

### Porting traps

Each of these cost time during implementation. They are version boundaries, so they do not show up until a different container runs the build.

- **`zend_object_count_elements_t` changed return type.** `int` up to PHP 8.1, `zend_result` from 8.2. These are different types, and the handler is a function pointer, so a mismatch is a compile error rather than a warning. Hence the `ZEPHIR_BUFFER_COUNT_RESULT` macro.
- **`zend_ce_arrayiterator` is not exported; `spl_ce_ArrayIterator` is.** The symbol you want lives in `ext/spl/spl_array.h`.
- **Implementing `zend_ce_traversable` alone is `E_CORE_ERROR`.** For an internal class, either implement `IteratorAggregate` (what this class does) or implement `Iterator`; `IteratorAggregate` plus an explicitly assigned `ce->get_iterator` is also accepted. This class took neither exotic route — `getIterator()` returns an `ArrayIterator` over a materialised copy, because iterating one element at a time through the VM is the slow path the class exists to avoid. A lazy custom iterator would additionally have to cope with the iterator `valid` function changing from `int` to `zend_result` at 8.4.
- **A stub fixture's filename must match its class.** `Stub\BufferOps` lives in `stub/bufferops.zep`, not `stub/BufferOps.zep`.
- **`zend_illegal_container_offset()` is 8.3+.** The three-regime ladder in `zephir_buffer_offset_to_long()` exists because there is no single call that works across 8.0 → 8.5.

### Why a kernel class and not a new Zephir type

The obvious-looking alternative — a first-class Zephir type, `double[] x;` — was rejected. Recorded here so it does not get relitigated.

A new type keyword needs:

- the keyword added to **both** parsers: `src/Parser/Php/{TokenType,Lexer,PhpParser}.php` and the separate [php-zephir-parser](https://github.com/zephir-lang/php-zephir-parser) C extension;
- a parser release, or a bump of `Manager::MINIMUM_PARSER_VERSION`, before CI can use it;
- entries in roughly eight parallel type tables scattered through the compiler;
- a shape in `generateInitCode()` for a pointer-to-scalar local, which does not exist — every existing local is a zval or a C scalar, not a pointer to a heap array.

A kernel class needs none of that. It is a `.c`/`.h` pair, a MINIT call, and a config flag. It is also strictly more capable: it can be stored in a property, passed between methods, refcounted and garbage-collected, and reached from PHP userland — none of which a bare `double *` local would have given.

### Side fix carried by the same change

php-src stopped declaring `HAVE_JSON` in `php_config.h` at PHP 8.4. The old probe in `templates/engine/config.m4` was `AC_CHECK_DECL([HAVE_JSON])`, so on 8.4 and 8.5 it silently failed and left `ZEPHIR_USE_PHP_JSON` undefined — which demoted `zephir_json_encode()` from calling `php_json_encode()` in-process to dispatching the **userland** `json_encode()` function. The fix probes the header the code actually includes:

```m4
AC_CHECK_HEADERS(
        [ext/json/php_json.h],
        [
                PHP_ADD_EXTENSION_DEP([%PROJECT_LOWER%], [json])
                AC_DEFINE([ZEPHIR_USE_PHP_JSON], [1], [Whether PHP json extension is present at compile time])
        ],
        ,
        [[#include "main/php.h"]]
)
```

The neighbouring PCRE probe still uses the older `AC_CHECK_DECL` shape and was left alone.

!!! warning "Never gate a class interface on a build-time probe"
    This is why `kernel/buffer.c` includes `<ext/json/php_json.h>` ungated. `JsonSerializable` is not an optimisation — without it, `json_encode()` on a buffer emits `{}` and drops every element. A probe that fails in some environment would turn that into silent data loss rather than a slower path.

### Testing

```bash
php vendor/bin/phpunit -c phpunit.ext.xml
```

Two suites, with different jobs:

- `tests/Extension/BufferTest.php` asserts **PHP parity**. Offset diagnostics moved three times between 8.0 and 8.5, so rather than hard-coding message strings it evaluates the same statement against a live `SplFixedArray` and compares transcripts, rewriting the class name in the message. The one place it cannot agree — `isset()` on a fresh instance — is asserted separately.
- `tests/Extension/BufferZephirTest.php` asserts **equivalence**. The fast path is a speed change only, so every test there compares the compiled `buf[i]` path against the PHP `$buffer[$i]` path and requires them to agree, exceptions included.

The lvalue rules have their own oracle. `Issue2721Overloaded` in `tests/fixtures/mocks/` is the plainest possible `ArrayAccess` container — `offsetGet()` returns a value, never a reference, which is the position a buffer element is in — and `BufferTest` runs `$b[0]++`, `--$b[0]`, `$r =& $b[0]` and a by-reference argument against both, asserting that the diagnostics and the resulting element agree. The container name inside the notice is normalised away; the *wording* is deliberately not asserted, since the premise of that file is that PHP moves its diagnostics. The fixture is named rather than anonymous because PHP truncates an anonymous class name at the NUL byte when printing it in that notice, so `get_class()` would not match the text.

Both suites also carry a heap-growth probe: construct, convert, clone, read, write and fail in a loop, and require `memory_get_usage()` not to move.
