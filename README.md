<p align="center">
    <a href="https://zephir-lang.com/" target="_blank">
        <img src="https://assets.phalcon.io/zephir/zephir_logo-105x36.svg" height="100" alt="Zephir"/>
    </a>
</p>

# Zephir - Documentation [site](https://docs.zephir-lang.com/latest/introduction/) source

This is the official documentation site source code.

## How to try it out

```sh
$ git clone git@github.com:zephir-lang/documentation.git
$ cd zephir-docs-app
```

You will need to build a docker image, to be able to work with the documents locally.

```sh
docker build -t zephir-mike .
```

Once the image is built, run the `./serve` command

```sh
$ ./serve
```

This will get you in the dockerized environment. Launch the server by issuing the following command:

```sh
$ mkdocs serve
```

Check the documents on your browser by visiting `http://localhost:8000/`



### Easy way pull latest of all submodules
## License

This project is open-sourced software licensed under the MIT License. See the [LICENSE](https://github.com/zephir-lang/documentation/blob/master/LICENSE) file for more information.

## Thank you

A big thank you to:
- [Cloudflare](https://cloudflare.com) for hosting the site
- All of our supporters and users!
