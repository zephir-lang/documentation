# Installation

To install Zephir, please follow these steps:

## Prerequisites

Before building a PHP extension with Zephir, ensure you have the following requirements:

* [Zephir parser][parser] >= 1.3.0
* A C compiler such as [gcc][gcc] >= 4.4 or an alternative such as [clang][clang] >= 3.0, [Visual C++][visual_c] >= 11 or [Intel C++][intel_c]. It is recommended to use `gcc` 4.4 or later
* [re2c][re2c] 0.13.6 or later
* PHP development headers and tools

For Linux based systems you'll need also:
* [GNU make][gnu_make] 3.81 or later
* [autoconf][autoconf] 2.31 or later
* [automake][automake] 1.14 or later
* libpcre3
* The `build-essential` package when using `gcc` on Ubuntu (and likely in other distributions as well)

On Ubuntu, you can install the required packages with:

```bash
sudo apt-get update
sudo apt-get install git gcc make re2c php php-json php-dev libpcre3-dev build-essential
```

Ensure you have a recent version of PHP installed and available in your console:

```bash
php -v
PHP 8.0.30 (cli) (built: Nov 21 2023 16:16:21) ( NTS )
Copyright (c) The PHP Group
Zend Engine v4.0.30, Copyright (c) Zend Technologies
    with Xdebug v3.3.1, Copyright (c) 2002-2023, by Derick Rethans
```

Additionally, check that the PHP development libraries are installed:

```bash
phpize -v
Configuring for:
PHP Api Version:         20200930
Zend Module Api No:      20200930
Zend Extension Api No:   420200930
```

You don't have to necessarily see the exact above output, but it's important that these commands are available to start developing with Zephir.

## Installing Zephir

First make sure that the Zephir parser extension is installed and activated. You can find installation instructions in the [Zephir Parser repository][parser_repo].

### Composer

The recommended installation method is using composer:

```bash
composer require phalcon/zephir
```

Make sure that `${COMPOSER_HOME}/vendor/bin` is in your `$PATH`, to have Zephir should be available as `zephir` on the command line.

#### Project

Use `composer exec zephir` within the project you installed Zephir in, above, to run it. (Alternately, you can still run `vendor/bin/zephir`.)

#### Global

For a global installation, you can use:

```bash
composer global require phalcon/zephir
```

### Release PHAR

You can also download the latest release PHAR from [GitHub][zephir_releases]. Place it in your `$PATH` and consider renaming it to `zephir` for convenience.


### Git Clone

Clone the latest tag from GitHub, install dependencies, and run Zephir from there:

```bash
git clone --depth 1 -b $(git ls-remote https://github.com/zephir-lang/zephir 0.17.* | sort -t/ -k3 -Vr | head -n1 | awk -F/ '{ print $NF }') https://github.com/zephir-lang/zephir
composer install
```

Either use the path to `zephir/zephir` or create a symlink in a directory in your `$PATH` to run Zephir using this option.

## Testing the Installation

Check if Zephir is available from any directory by executing:

```bash
zephir help
```


[parser]: https://github.com/zephir-lang/php-zephir-parser
[parser_repo]: https://github.com/zephir-lang/php-zephir-parser
[gcc]: https://gcc.gnu.org/
[clang]: https://clang.llvm.org/
[visual_c]: https://support.microsoft.com/en-us/help/2977003/the-latest-supported-visual-c-downloads
[intel_c]: https://software.intel.com/en-us/c-compilers
[re2c]: https://re2c.org/
[gnu_make]: https://www.gnu.org/software/make/
[autoconf]: https://www.gnu.org/software/autoconf/autoconf.html
[automake]: https://www.gnu.org/software/automake/
[zephir_releases]: https://github.com/zephir-lang/zephir/releases/latest
