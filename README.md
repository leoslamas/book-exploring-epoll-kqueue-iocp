# Exploring Epoll, Kqueue, IOCP

## How to build

* Install gitbook

    ```
    $ npm i gitbook-cli
    ```

* Install gitbook plugins

    Inside `book` folder:

    ```
    $ gitbook install
    ```

* Build

    ```
    # Generate a PDF file
    $ gitbook pdf ./ ./mybook.pdf

    # Generate an ePub file
    $ gitbook epub ./ ./mybook.epub

    # Generate a Mobi file
    $ gitbook mobi ./ ./mybook.mobi
    ```

### Installing ebook-convert

`ebook-convert` is required to generate ebooks (epub, mobi, pdf).

#### OS X

Download the [Calibre application](https://calibre-ebook.com/download). After moving the `calibre.app` to your Applications folder create a symbolic link to the ebook-convert tool:

```
$ sudo ln -s ~/Applications/calibre.app/Contents/MacOS/ebook-convert /usr/bin
```

You can replace `/usr/bin` with any directory that is in your $PATH.