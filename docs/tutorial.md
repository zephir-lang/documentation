# Tutorial

Zephir, and this manual, are intended for PHP developers who want to create C extensions, with a lower complexity.

We assume that you are experienced in one or more other programming languages. We draw parallels to features in PHP, C, Javascript, and other languages. We'll point out features in Zephir that are similar to these other languages, as well as many features that are new or different. If you are familiar with these specific languages, you'll pick up on these comparisons more quickly.

In this guide, we will use the standard Linux terminal commands. If you are a Windows user, you need to replace these commands with their counterparts.

## Checking the Installation
If you have successfully installed Zephir, you will be able to execute the following command in your console:

```bash
zephir help
```

If everything is well, you should see the following help (or something very similar):

```
 _____              __    _
/__  /  ___  ____  / /_  (_)____
  / /  / _ \/ __ \/ __ \/ / ___/
 / /__/  __/ /_/ / / / / / /
/____/\___/ .___/_/ /_/_/_/
         /_/

Zephir 0.17.0 by the Phalcon Team
Thanks to the work by: Andres Gutierrez and Serghei Iakovlev (source)

Usage:
  command [options] [arguments]

Options:
      --dumpversion  Print the version of the compiler and don't do anything else (also works with a single hyphen)
  -h, --help         Print this help message
      --no-ansi      Disable ANSI output
  -v, --verbose      Displays more detail in error messages from exceptions generated by commands (can also disable with -V)
      --vernum       Print the version of the compiler as integer
      --version      Print compiler version information and quit

Available commands:
  api        Generates a HTML API based on the classes exposed in the extension
  build      Generates/Compiles/Installs a Zephir extension
  clean      Cleans any object files created by the extension
  compile    Compile a Zephir extension
  fullclean  Cleans any object files created by the extension (including files generated by phpize)
  generate   Generates C code from the Zephir code without compiling it
  help       Display help for a command
  init       Initializes a Zephir extension
  install    Installs the extension in the extension directory (may require root password)
  stubs      Generates stubs that can be used in a PHP IDE
```

If something went wrong, please return back to the [installation][installation] page.

## Extension Skeleton
The first thing we have to do is generate an extension skeleton. This will provide to our extension the basic structure we need to start working. In our case, we're going to create an extension called `utils`:

```bash
zephir init utils
```

After this, a directory called "utils" is created on the current working directory:

```bash
utils/
   ext/
   utils/
```

The directory `ext/` (inside utils) contains the code that is going to be used by the compiler to produce the extension. Another directory created is `utils` - this directory has the same name as our extension. We will place Zephir code there.

We need to change the working directory to "utils" to start compiling our code:

```bash
cd utils
ls
ext/ utils/ config.json
```

The directory listing will also show us a file called `config.json`. This file contains configuration settings we can use to alter the behavior of Zephir and/or the extension itself.

## Adding our first class
Zephir is designed to generate object-oriented extensions. To start developing functionality, we need to add our first class to the extension.

As in many languages/tools, the first thing we want to do is see a `hello world` generated by Zephir, and check that everything is well. So our first class will be called `Utils\Greeting`, and contain a method printing `hello world!`.

The code for this class must be placed in `utils/utils/greeting.zep`:

```zephir
namespace Utils;

class Greeting
{

    public static function say()
    {
        echo "hello world!";
    }

}
```

Now, we need to tell Zephir that our project must be compiled and the extension generated:

```bash
zephir build
```

Initially, and only for the first time, a number of internal commands are executed producing the necessary code and configurations to export this class to the PHP extension. If everything goes well, you will see the following message at the end of the output:

```bash
    ...
Extension installed!
Add extension=utils.so to your php.ini
Don't forget to restart your web server
```

At the above step, it's likely that you would need to supply your root password in order to install the extension.

Finally, the extension must be added to the `php.ini` in order to be loaded by PHP. This is achieved by adding the initialization directive: `extension=utils.so` to it.

NOTE: You can also load it on the command line with `-d extension=utils.so`, but it will only load for that single request, so you'd need to include it every time you want to test your extension in the CLI. Adding the directive to the `php.ini` will ensure it is loaded for every request from then on.

## Initial Testing
Now that the extension was added to your `php.ini`, check whether the extension is being loaded properly by executing the following:

```bash
php -m
[PHP Modules]
Core
date
libxml
pcre
Reflection
session
SPL
standard
tokenizer
utils
xdebug
xml
```

Extension `utils` should be part of the output, indicating that the extension was loaded correctly. Now, let's see our `hello world` directly executed by PHP. To accomplish this, you can create a simple PHP file calling the static method we have just created:

```php
<?php

echo Utils\Greeting::say(), "\n";
```

Congratulations! Уou have your first extension running in PHP.

## A useful class
The `Utils\Greeting::say` method was fine to check if our environment was right. Now, let's create some more useful classes.

The first useful class we are going to add to this extension will provide filtering facilities to users. This class is called `Utils\Filter` and its code must be placed in `utils/utils/filter.zep`:

A basic skeleton for this class is the following:

```zephir
namespace Utils;

class Filter
{

}
```

The class contains filtering methods that help users to filter unwanted characters from strings. The first method is called `alpha`, and its purpose is to filter only those characters that are ASCII basic letters. To begin, we are just going to traverse the string, printing every byte to the standard output:

```zephir
namespace Utils;

class Filter
{

    public function alpha(string str)
    {
        char ch;

        for ch in str {
            echo ch, "\n";
        }
    }
}
```

When invoking this method:

```php
<?php

$f = new Utils\Filter();
$f->alpha("hello");
```

You will see:

    h
    e
    l
    l
    o

Checking every character in the string is straightforward. Now we'll create another string with the right filtered characters:

```zephir
class Filter
{

    public function alpha(string str) -> string
    {
        char ch; string filtered = "";

        for ch in str {
            if (ch >= 'a' && ch <= 'z') || (ch >= 'A' && ch <= 'Z') {
                let filtered .= ch;
            }
        }

        return filtered;
    }
}
```

The complete method can be tested as before:

```php
<?php

$f = new Utils\Filter();
echo $f->alpha("!he#02l3'121lo."); // prints "hello"
```

In the following screencast you can watch how to create the extension explained in this tutorial:

<div class="video-iframe-wrapper">
    <div class="video-iframe-item">
        <iframe title="video" src="//player.vimeo.com/video/84180223" width="500" height="313" allowfullscreen></iframe>
    </div>
</div>

## Conclusion
This is a very simple tutorial, and as you can see, it's easy to start building extensions using Zephir. We invite you to continue reading the manual so that you can discover additional features offered by Zephir!

[installation]: installation.md
