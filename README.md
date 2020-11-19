# Exploring Epoll, Kqueue, IOCP

## How to build

* Install gitbook

    ```
    $ npm i gitbook-cli -g
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

-----

## Troubleshooting

If you have issues trying to run `gitbook`:

```
if (cb) cb.apply(this, arguments)
         ^
TypeError: cb.apply is not a function
```

Try commenting these lines in

 `(path from the error message)/polyfills.js`:

```
//fs.stat = statFix(fs.stat)
//fs.fstat = statFix(fs.fstat)
//fs.lstat = statFix(fs.lstat)
```

That worked for me!
