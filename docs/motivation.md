# Why Zephir?

In the ever-evolving landscape of PHP applications, developers face challenges in achieving a delicate balance between stability, performance, and functionality. The core components, often in the form of libraries or frameworks, serve as the foundation for many applications. Zephir emerges as a solution to address the need for fast, robust extensions that enhance performance and resource efficiency.

## For PHP Programmers...

PHP, renowned for web application development, provides high productivity through its dynamically typed and interpreted nature. Zephir's integration with PHP running on the Zend Engine empowers developers to create extensions, offering a balance between the flexibility of PHP and the efficiency of a compiled language. While Zephir introduces a compilation step, it adheres to strict standards, providing improved performance.

## For C Programmers...

C, a powerful and widely-used language, serves as the backbone for PHP. Zephir, inheriting some characteristics from C, ensures a safer environment by eliminating pointers and manual memory management. While Zephir may feel less powerful than C to seasoned C programmers, it offers a friendlier experience, emphasizing safety and ease of use.

## Compilation vs Interpretation

Zephir's compilation step introduces a slight delay in development, requiring patience for code compilation before execution. However, this approach brings performance benefits without sacrificing the interpretative nature of PHP. Developers have the flexibility to choose which parts of their application to implement in Zephir, achieving a balance between speed and productivity.

## Statically Typed Versus Dynamically Typed Languages

In a statically typed language, a variable is bound to a particular type for its lifetime. Its type can't be changed and it can only reference type-compatible instances and operations. Languages like C/C++ were implemented with this scheme:

```zephir
int a = 0;
a = "hello"; // not allowed
```

In dynamic typing, the type is bound to the value, not the variable. So, a variable might refer to a value of one type, then be reassigned later to a value of an unrelated type. Javascript/PHP are examples of a dynamically typed languages:

```zephir
var a = 0;
a = "hello"; // allowed
```

Despite their productivity advantages, dynamic languages may not be the best choices for all applications, particularly for very large code bases and high-performance applications.

Optimizing the performance of a dynamic language like PHP is more challenging than for a static language like C. In a static language, optimizers can exploit the type information attached to variables themselves to make decisions. In a dynamic language, fewer such clues are available for the optimizer, making optimization choices harder.

While recent advancements in optimizations for dynamic languages are promising (like JIT compilation), they lag behind the state of the art for static languages. So, if you require very high performance, static languages are probably a safer choice.

Another small benefit of static languages is the extra checking the compiler performs. A compiler can't find logic errors, which are far more significant, but a compiler can find errors in advance that in a dynamic language only can be found in runtime.

Zephir is both statically and dynamically typed, allowing you to take advantage of both approaches where possible.

## Compilation Scheme

Zephir employs native code generation, compiling to C and utilizing compilers like gcc/clang/vc++ to optimize and produce machine code. This compilation scheme harnesses the capabilities of mature compilers, benefiting from various optimizations provided by these tools.

![compilation scheme](assets/images/content/scheme.png)

In addition to the ones provided by Zephir, over time, compilers have implemented and matured a number of optimizations that improve the performance of compiled applications:

* [GCC optimizations][gcc_optimizations]
* [LLVM passes][llvm_passes]
* [Visual C/C++ optimizations][c_optimizations]

## Code Protection

Compilation with Zephir not only enhances performance but also provides a level of intellectual protection. By producing native binaries, Zephir allows developers to "hide" the original code from users or customers, adding an extra layer of security.

## Conclusion

Zephir doesn't aim to replace PHP or C; instead, it complements them. It serves as a bridge, inviting PHP developers to explore code compilation and static typing while preserving the best aspects of both the C and PHP worlds. Zephir is a strategic tool, seeking opportunities to accelerate application development and enhance user experiences.

[gcc_optimizations]: https://gcc.gnu.org/onlinedocs/gcc-4.1.0/gcc/Optimize-Options.html
[llvm_passes]: https://llvm.org/docs/Passes.html
[c_optimizations]: https://msdn.microsoft.com/en-us/library/k1ack8f1.aspx
