# Buffer internals

For people writing C against a [contiguous buffer](buffer.md), and for whoever adds the next kernel class. If you are only using the class from `.zep` or PHP, [the feature page](buffer.md) is the one you want.

Source: `kernel/buffer.h` and `kernel/buffer.c` in the [zephir](https://github.com/zephir-lang/zephir) repository. All paths on this page are relative to that repository.

## Object layout

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

## The C API for extension authors

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

### Worked example

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

## Registration

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

## The opt-in path, end to end

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

### Why an explicit switch rather than inference

The compiler cannot infer the need. There is no syntax that implies a buffer, and a class reference can be dynamic (`new {var}`), so there is nothing to detect. Leaving it explicit also guarantees that every existing extension gains nothing it did not ask for.

A project that declares its own `<Ns>\Buffer` while the switch is on gets a `CompilerException` rather than a silent shadowing.

## The `kernel/array.c` fast path

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

## Porting traps

Each of these cost time during implementation. They are version boundaries, so they do not show up until a different container runs the build.

- **`zend_object_count_elements_t` changed return type.** `int` up to PHP 8.1, `zend_result` from 8.2. These are different types, and the handler is a function pointer, so a mismatch is a compile error rather than a warning. Hence the `ZEPHIR_BUFFER_COUNT_RESULT` macro.
- **`zend_ce_arrayiterator` is not exported; `spl_ce_ArrayIterator` is.** The symbol you want lives in `ext/spl/spl_array.h`.
- **Implementing `zend_ce_traversable` alone is `E_CORE_ERROR`.** For an internal class, either implement `IteratorAggregate` (what this class does) or implement `Iterator`; `IteratorAggregate` plus an explicitly assigned `ce->get_iterator` is also accepted. This class took neither exotic route — `getIterator()` returns an `ArrayIterator` over a materialised copy, because iterating one element at a time through the VM is the slow path the class exists to avoid. A lazy custom iterator would additionally have to cope with the iterator `valid` function changing from `int` to `zend_result` at 8.4.
- **A stub fixture's filename must match its class.** `Stub\BufferOps` lives in `stub/bufferops.zep`, not `stub/BufferOps.zep`.
- **`zend_illegal_container_offset()` is 8.3+.** The three-regime ladder in `zephir_buffer_offset_to_long()` exists because there is no single call that works across 8.0 → 8.5.

## Why a kernel class and not a new Zephir type

The obvious-looking alternative — a first-class Zephir type, `double[] x;` — was rejected. Recorded here so it does not get relitigated.

A new type keyword needs:

- the keyword added to **both** parsers: `src/Parser/Php/{TokenType,Lexer,PhpParser}.php` and the separate [php-zephir-parser](https://github.com/zephir-lang/php-zephir-parser) C extension;
- a parser release, or a bump of `Manager::MINIMUM_PARSER_VERSION`, before CI can use it;
- entries in roughly eight parallel type tables scattered through the compiler;
- a shape in `generateInitCode()` for a pointer-to-scalar local, which does not exist — every existing local is a zval or a C scalar, not a pointer to a heap array.

A kernel class needs none of that. It is a `.c`/`.h` pair, a MINIT call, and a config flag. It is also strictly more capable: it can be stored in a property, passed between methods, refcounted and garbage-collected, and reached from PHP userland — none of which a bare `double *` local would have given.

## Side fix carried by the same change

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

## Testing

```bash
php vendor/bin/phpunit -c phpunit.ext.xml
```

Two suites, with different jobs:

- `tests/Extension/BufferTest.php` asserts **PHP parity**. Offset diagnostics moved three times between 8.0 and 8.5, so rather than hard-coding message strings it evaluates the same statement against a live `SplFixedArray` and compares transcripts, rewriting the class name in the message. The one place it cannot agree — `isset()` on a fresh instance — is asserted separately.
- `tests/Extension/BufferZephirTest.php` asserts **equivalence**. The fast path is a speed change only, so every test there compares the compiled `buf[i]` path against the PHP `$buffer[$i]` path and requires them to agree, exceptions included.

The lvalue rules have their own oracle. `Issue2721Overloaded` in `tests/fixtures/mocks/` is the plainest possible `ArrayAccess` container — `offsetGet()` returns a value, never a reference, which is the position a buffer element is in — and `BufferTest` runs `$b[0]++`, `--$b[0]`, `$r =& $b[0]` and a by-reference argument against both, asserting that the diagnostics and the resulting element agree. The container name inside the notice is normalised away; the *wording* is deliberately not asserted, since the premise of that file is that PHP moves its diagnostics. The fixture is named rather than anonymous because PHP truncates an anonymous class name at the NUL byte when printing it in that notice, so `get_class()` would not match the text.

Both suites also carry a heap-growth probe: construct, convert, clone, read, write and fail in a loop, and require `memory_get_usage()` not to move.

## See also

- [Contiguous buffer](buffer.md) — the user-facing feature.
- [Buffer performance](buffer-performance.md) — what the fast path buys, measured.
