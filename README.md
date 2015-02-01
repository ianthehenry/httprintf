# httpure

There are lots of ways to start up a local server when you're doing web client development. [`SimpleHTTPServer`](https://docs.python.org/2/library/simplehttpserver.html), [`knod`](https://github.com/moserrya/knod), [gulp](https://github.com/gulpjs/gulp), [brunch](https://github.com/brunch/brunch), countless others.

Use them.

This is my personal invented-here server that scratches an itch that only I have.

`httpure` has very few dependencies. It expects no programming language except shell, and no programs except netcat. And, like, `tr` and `file` and stuff. Things you have. I assume.

It half-heartedly wishes it were fully portable to any POSIX system, but it probably isn't, because figuring out how to do things in a POSIX-compliant way feels a lot like autoflagellation.

# Example

It works like this:

    TODO

# Reference

`httpure` consists of two things:

- a usually HTTP-like TCP server
- a set of shell scripts for generating HTTP requests

## The server

TODO

## The HTTP-generating shell scripts

Sorted roughly in order of usefulness.

### [`http-file`](http-file) (helper)

Prints an entire HTTP response to stdout based on the filepath provided.

    http-file path/to/index.html

Sends `200` if the file exists, `404` otherwise. Attempts to guess the correct `Content-Type` using `http-guess-mime-type`.

A future version may send `301`s for symlinks because that would be pretty whimsical.

### [`http-status`](http-status) (primitive)

Prints an HTTP status line to stdout.

    http-status 200
    http-status 418 "I'm a teapot"

The status code is required. If the reason is omitted, `http-status` will supply a reason based on [RFC 7231](http://tools.ietf.org/html/rfc7231#page-49). It is an error to call `http-status` without a status code, or to call it with an unknown status code without specifying a reason.

### [`http-header`](http-header) (primitive)

Prints an HTTP header line to stdout.

    http-header "Content-Type" "application/json"

It is an error to omit either the header name or value.

### [`http-body`](http-body) (helper)

Reads stdin and prints a `Content-Length` header, the header separator line, and the entire contents of stdin to stdout.

    http-body <index.html

May be useful in cases where `http-file` is not sufficient.

### [`http-guess-mime-type`](http-guess-mime-type) (helper)

Prints the guessed MIME type of the specified file to stdout, followed by a newline.

    http-guess-mime-type index.html
    http-guess-mime-type path/to/layout.css
    http-guess-mime-type app.js

Tries to guess based on the file's extension, with a dictionary of commonly served types. If that fails, it falls back to using `file -I`.

The file doesn't need to exist so long as the extension is recognized, but the fallback detection requires the file to exist.

### [`http-begin-body`](http-begin-body) (primitive)

Prints `\r\n` to stdout.

    http-begin-body

A trivial helper for separating the response headers from the response body. Unnecessary if using `http-body`. Takes no arguments.
