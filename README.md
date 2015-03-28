# httprintf

There are lots of ways to start up a local server when you're doing web client development. See [SimpleHTTPServer](https://docs.python.org/2/library/simplehttpserver.html), [knod](https://github.com/moserrya/knod), [gulp](https://github.com/gulpjs/gulp), [brunch](https://github.com/brunch/brunch), and countless others.

Use them, if you can.

This is my personal invented-here tool that scratches an itch that only I have.

`httprintf` has very few dependencies. It is written entirely in shell. It only uses common executables, like `file` and `awk`. Things you have. I assume.

It half-heartedly wishes it were fully portable to any POSIX system, but it isn't, because figuring out how to do things in a POSIX-compliant way feels a lot like autoflagellation.

# But why?

For fun! Though there is a more reasonable answer: I really liked the look of [`redo`](https://github.com/apenwarr/redo), and wanted to try it out for web stuff.

But I didn't want to manually recompile my assets, nor recompile on file save (to prevent the frustrating situation where you request an asset after saving the source file but before you finish generating the target).

Thus it fills a tiny gap between "static file server" and "dynamic HTTP server," and allows me to `redo-ifchanged` my files at request time.

# Reference

## The server

I recommend that you use this in conjunction with [`ncat`](http://nmap.org/ncat/), if you actually want to talk to it from a web browser:

    ncat -kl localhost 8080 -c http-handler

Linux netcats may work as well, but BSD netcats (including the one provided with OS X) will not. If you can get it working in a way that can respond to multiple requests in rapid succession, please tell me. I was unable to figure it out.

### [`http-handler`](scripts/http-handler)

This script expects an HTTP request on stdin, and will try to produce one on stdout.

First it parses the HTTP method and path, then it tries to find a handler file to invoke. It will look for a file matching the exact verb anywhere in the path, starting from the directory of the file specified. If that's not found, it will repeat the search for a file called `ANY`.

So, for a request like `DELETE /foo/bar/baz.json HTTP/1.1`, it will try to invoke:

    ./foo/bar/DELETE ./baz.json DELETE
    ./foo/DELETE ./bar/baz.json DELETE
    ./DELETE ./foo/bar/baz.json DELETE
    ./foo/bar/ANY ./baz.json DELETE
    ./foo/ANY ./bar/baz.json DELETE
    ./ANY ./foo/bar/baz.json DELETE

In that order. If it's a request for a directory, like `POST /foo/bars/ HTTP/1.1`, it will try:

    ./foo/bars/POST . POST
    ./foo/POST ./bars/ POST
    ./POST ./foo/bars/ POST
    ./foo/bars/ANY . POST
    ./foo/ANY ./bars/ POST
    ./ANY ./foo/bars/ POST

If no handler is found, it will send a `405` with an empty `Allow` header. If this is emotionally upsetting to you, provide a root `ANY` that sets the correct `Allow` header.

It will log some information about each request and response to stderr.

If it invokes a handler that returns a non-zero exit code, chaos will reign.

It won't currently send anything else to the script. Want to do something based on HTTP headers, or the request body? It's coming soon, but it isn't supported yet.

`http-handler` makes no attempt to protect against escaping the directory from which it's invoke. `GET /../../passwords.txt` will work just fine. This is by design. You could attempt to sanitize the path in your handler script, but if you're using `http-handler` for anything but local development you're doing it wrong.

### Examples

`http-handler` has no default implementations for any methods. A simple static server might look like this, with a single root `GET`:

    #!/usr/bin/sh
    http-file "$1"

## The HTTP-generating shell scripts

Sorted roughly in order of usefulness.

### [`http-file`](scripts/http-file) (helper)

Prints an entire HTTP response to stdout based on the filepath provided.

    http-file path/to/index.html

Sends `200` if the file exists and `404` otherwise. A future version may send `307`s for symlinks because that would be pretty whimsical.

If the file exists, it sets the `Content-Type` header using `http-guess-mime-type`.

### [`http-response`](scripts/http-response) (helper)

Prints an entire HTTP response to stdout using a body supplied via stdin.

    http-response application/json <foo.json

The first argument specifies the `Content-Type`, which is required. No `Content-Length` header is sent when using `http-response`.

### [`http-status`](scripts/http-status) (primitive)

Prints an HTTP status line to stdout.

    http-status 200
    http-status 418 "I'm a teapot"

The status code is required. If the reason is omitted, `http-status` will supply a reason based on [RFC 7231](http://tools.ietf.org/html/rfc7231#page-49). It is an error to call `http-status` without a status code, or to call it with an unknown status code without specifying a reason.

### [`http-header`](scripts/http-header) (primitive)

Prints an HTTP header line to stdout.

    http-header "Content-Type" "application/json"

It is an error to omit either the header name or value.

### [`http-body`](scripts/http-body) (helper)

Reads a file and prints a `Content-Length` header, the header separator line, and the entire contents of stdin to stdout.

    http-body index.html

May be helpful in cases where `http-file` is not sufficient.

I'd love for it to take input on stdin, but I can't figure out how to make that work with binary files without first creating a temporary file.

### [`http-guess-mime-type`](scripts/http-guess-mime-type) (helper)

Prints the guessed MIME type of the specified file to stdout, followed by a newline.

    http-guess-mime-type index.html
    http-guess-mime-type path/to/layout.css
    http-guess-mime-type app.js

Tries to guess based on the file's extension, with a dictionary of commonly served types. If that fails, it falls back to using `file -I`.

The file doesn't need to exist so long as the extension is recognized, but the fallback detection requires the file to exist.

### [`http-begin-body`](scripts/http-begin-body) (primitive)

Prints `\r\n` to stdout.

    http-begin-body

A trivial helper for separating the response headers from the response body. Unnecessary if using `http-body`. Takes no arguments.

# TODO

- clean up http-handler
- pass request headers and body to handler scripts
- test that anything actually works
- homebrew it, maybe
