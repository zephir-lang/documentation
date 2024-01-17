# Phpinfo() sections

To add more directives and information to the [phpinfo()][phpinfo] output of your Zephir extension, you can make use of the configuration options in the config.json file. This allows you to provide additional details, environment data, and INI options related to your extension.

Here is an example of how to configure this in your `config.json` file:

```json
{
    "namespace": "your_extension",
    "name": "Your Extension",
    "author": "Your Name",
    "version": "1.0.0",
   
    "ini": {
        "your_extension.some_ini_directive": "default_value",
        "your_extension.another_ini_directive": "another_default_value"
    },
   
    "info": {
        "some_information": "This is additional information about your extension",
        "another_information": "You can add more information here"
    }
}
```

## Adding INI Directives
The `ini` section allows you to specify INI directives related to your extension. In the example above, two INI directives (`your_extension.some_ini_directive` and `your_extension.another_ini_directive`) are defined with their default values.

## Adding Additional Information

The `info` section allows you to provide additional information that will be displayed in the `phpinfo()` output. You can add any key-value pairs under this section to include relevant details about your extension.

After making these configurations, when users run `phpinfo()`, your extension's section will include the specified INI directives and additional information.

Adjust the values and keys according to your extension's needs. This customization helps users understand the configuration options and details related to your Zephir extension.

This information will be shown as follows:

![](assets/images/content/info.png)

[phpinfo]: https://php.net/manual/en/function.phpinfo.php
