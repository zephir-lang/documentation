# Inline C and external libraries

Zephir is designed so that almost everything you need can be expressed in Zephir itself, and the compiler can reason about it (type-infer, optimize, validate). For the rare cases where you really do need to drop into raw C (calling a system library, declaring a `static` C helper, defining a macro shared by several methods), Zephir offers two escape hatches:

- **`%{ … }%` blocks** (commonly called *cblocks*): embed raw C inside a `.zep` file.
- **`extra-cflags` / `extra-libs` / `extra-sources` / `extra-classes`** in `config.json`: tell the build system about headers, libraries, additional `.c` files, and pre-existing PHP-extension class entries.

!!! warning "Last-resort escape hatch"
    Code inside a `%{ … }%` block is opaque to the Zephir compiler. Type inference, the read/write/mutation detectors, the `nonexistent-class`/`unused-variable` warnings, and the call-graph analyses all stop at the block boundary. Errors and warnings about it come from the C compiler (GCC/Clang/MSVC), not from Zephir, so debugging is rougher. Prefer a [custom optimizer](optimizers.md) whenever the target functionality can be expressed as a function call. Optimizers participate fully in Zephir's analysis pipeline.

## `%{ … }%` cblocks

A cblock is a fragment of literal C, opened with `%{` and closed with `}%`. Its contents are copied verbatim into the generated `.c` file. The legal positions are:

### 1. File-scope cblocks (before / after `namespace`)

Use these for `#include`s, `#define`s, file-scope `static` helpers, and other top-level declarations. They are emitted right after Zephir's own `#include` block in the generated `.zep.c`, so they can rely on the PHP / Zend / Zephir-kernel headers being already in scope.

```zep
%{
// Emitted at the top of the generated .c file, right after the
// auto-generated PHP / Zend / kernel includes.
#define MAX_FACTOR 10
}%

namespace Stub;

%{
// You can have several file-scope cblocks; they accumulate in source order.
#include "kernel/require.h"

// File-scope C helpers are fine. They're invisible to PHP, callable from
// any cblock inside the methods of this file.
static long fibonacci(long n) {
    if (n < 2) return n;
    else return fibonacci(n - 2) + fibonacci(n - 1);
}
}%

class Cblock
{
    // ... see below for statement-scope cblocks
}
```

### 2. Statement-scope cblocks (inside a method body)

Statement-scope cblocks read and write the surrounding Zephir variables directly. You reference them by their declared Zephir name. Built-in scalar types map naturally to their C counterparts:

| Zephir type | C type the cblock sees |
|---|---|
| `int` / `uint` / `long` / `ulong` | `zend_long` |
| `char` / `uchar` | `char` |
| `double` | `double` |
| `bool` | `zend_bool` |
| `string` | `zend_string *` (or `zval`, depending on context, see [Method.php's parameter handling](https://github.com/zephir-lang/zephir/blob/development/src/Class/Method/Method.php)) |
| `var` / `array` / object | `zval` |

A short example mixing the two scopes:

```zep
class Cblock
{
    public function fibonacci10()
    {
        long a = 0;

        %{
            // Calls the file-scope C helper declared above.
            // `a` is a `zend_long` here. Zephir's `long` maps to it directly.
            a = fibonacci(MAX_FACTOR);
        }%

        return a;
    }
}
```

What you **cannot** do inside a statement-scope cblock:

- Declare new Zephir-tracked variables. Declare any C locals you need *inside* the cblock. They don't escape.
- Return from the enclosing Zephir method. Compute a value into a Zephir-scoped variable and `return` it from Zephir code after the block.
- Rely on Zephir's read/write detectors. If you mutate a Zephir variable from inside a cblock, mark the change visible to Zephir by reading the variable from a `let` statement afterwards, or the compiler may have optimized away the load.

### Build-time vs. parse-time errors

Syntax errors *inside* a cblock are reported by the C compiler with a confusing line number. The offset is from the start of the generated `.c` file, not the `.zep` file. When debugging a cblock failure, open the corresponding `ext/<namespace>/<class>.zep.c` and look at the absolute line number from the C-compiler error. The cblock content lives at roughly the position you'd expect, surrounded by Zephir-generated scaffolding.

## Linking against external libraries

When a cblock `#include`s a header from a library that isn't part of PHP itself, you need to tell the build system how to find the header and how to link the library. Four `config.json` keys cover this:

| Key | Purpose |
|---|---|
| **[`extra-cflags`](config.md#extra-cflags)** | Compiler flags applied during the C build, typically `-I<dir>` to add a header search path. |
| **[`extra-libs`](config.md#extra-libs)** | Linker flags, typically `-L<dir>` for library search paths plus `-l<name>` for the library itself. |
| **[`extra-sources`](config.md#extra-sources)** | Extra `.c` files you want compiled and linked into the same extension. Paths are resolved relative to the `ext/` directory. |
| **[`extra-classes`](config.md#extra-classes)** | Pre-existing PHP-extension classes implemented in C that you want exposed alongside your Zephir classes. Each entry names the header, source file, init function and class-entry symbol. |

A typical *libevent*-using configuration:

```json
{
    "extra-cflags":  "-I/usr/local/Cellar/libevent/2.0.21_1/include",
    "extra-libs":    "-L/usr/local/Cellar/libevent/2.0.21_1/lib -levent",
    "extra-sources": ["utils/event_helpers.c"]
}
```

Then a Zephir file can `#include <event2/event.h>` from a cblock, call `event_base_new()` etc. directly, and the linker will find `libevent`. See the [`config.md`](config.md) page for the full schema of each key.

## When to use a [custom optimizer](optimizers.md) instead

A custom optimizer is a small PHP class registered in `config.json` under `optimizer-dirs` that the compiler invokes whenever it sees a particular function name. The optimizer's `optimize()` method returns a `CompiledExpression` whose `code` is the C expression the compiler should emit at the call site. The Zephir-side code calls the function normally:

```zep
let result = hello("world");        // looks like a regular function call
```

…and your optimizer emits whatever C it wants (including a call into the same external library) *as a tracked Zephir expression*. The compiler sees a normal function call, can reason about the return type, applies its analyses, and you don't pay the debuggability cost of an opaque cblock.

Use a cblock when:

- The C code is *declarative* (`#include`s, `#define`s, file-scope `static` helpers) and not callable as a function from Zephir.
- You're prototyping and don't yet want to write the optimizer scaffolding.
- The C-side intermediate state genuinely can't be modeled as a function (e.g. allocating into a `static` cache).

Use an optimizer when:

- The cblock would be a one-liner function call.
- You'd otherwise have to write the same cblock in several methods.
- You want Zephir to participate in type inference and warning generation around the call.

## Historical note

Cblocks were introduced in [PR #21](https://github.com/zephir-lang/zephir/pull/21) (inspired by Go's [cgo](https://blog.golang.org/c-go-cgo)), and the `extra-libs` config key landed in [PR #636](https://github.com/zephir-lang/zephir/pull/636). The feature has been silently supported for many years. See [issue #654](https://github.com/zephir-lang/zephir/issues/654) for the original "why isn't this documented?" report.
