NAME
====

stap++ - Simple macro language extensions to systemtap

Synopsis
========

    $ stap++ -I ./tapset -x 12345 --arg limit=10 samples/ngx-upstream-post-conn.sxx
    $ stap++ -e 'probe begin { println("hello") exit() }'

Description
===========

This interpreter adds some simple macro language extensions to the systemtap scripting language.

Efforts has been made to ensure that this macro language expansion does
not affect the source line numbers so that the line numbers reported by `stap` are exactly the same in the original `.sxx` source files.

Features
========

Standard Macro Variables
------------------------

### $^exec_path

The variable `$^exec_path` is always evaluated to the path to the executable file
for the pid specified by the `-x` option.

Here is an example:

    probe process("$^exec_path").function("blah") { ... }

### $^libNAME_path

This variable expands to the absolute path of the DSO library file specified by a pattern.

`stap++` automatically scans all the loaded DSO files in the running process (if the `-x PID` option is specified) to find a match. If it fails to find a match, this variable will take the value of `$^exec_path`, that is, assuming the library is statically linked.

Below is an example for tracing a user-land function in the libpcre library:

    probe process("$^libpcre_path").statement("pcre_exec")
    {
        println("pcre_exec called")
        print_ubacktrace()
    }

### $^arg_NAME

This variable can evaluate to the value of a specified command-line argument. For example, `$^arg_limit` is evaluated to the value of the command line argument `limit` specified like this:

    stap++ --arg limit=1000

### Default values

It's possible to specify a default value for a macro variable by means of the `default` trait, as in

    foreach (key in stats- limit $^arg_limit :default(1000)) {
        ...
    }

where `$^arg_limit` takes the default value 1000 when the user does not specify the `--arg limit=N` command-line option while invoking `stap++`.

User-defined Macro Variables
----------------------------

It's possible to bind a `@cast()` expression to a user-defined macro variable of the form `$*NAME`. Here is an example,

    sock = sockfd_lookup(fd)
    $*sock := @cast(sock, "socket", "kernel")

    printf(", sock->state:%d", $*sock->state)
    state = $*sock->sk->__sk_common->skc_state
    printf(", sock->sk->sk_state:%d (%s)\n", state, tcp_sockstate_str(state))

Note that we used the `:=` operator to bind a `@cast()` expression to user variable `$*sock`, and later we reference it whenever we need that `@cast()` expression.

The scope of user variables is always limited to the current `.sxx` source file.

Tapset Modules
--------------

One can use the stap++ language to define new tapset module files and later use the `@use` directive to load the module in a main stap++ program file.

For example, we can have a module file located at `./tapset/kernel/socket.sxx`:

    // module kernel.socket
    function socketfd_lookup(fd)
    {
        ...
    }

And then in a stap++ script file, `foo.sxx`, we can import this library like this

    @use kernel.socket

and in `foo.sxx`, we are now free to call the `socketfd_lookup` function defined in the `kernel.socket` module.

Finally, we should invoke the `stap++` interpreter like this:

    stap++ -I ./tapset foo.sxx ...

Note the `-I ./tapset` option that specifies the search path for the stap++ tapset modules. The default module search paths are `.`, and `<bin-dir>/tapset`, where `<bin-dir>` is the directory where `stap++` sits in.

Unlike `stap`, only the used stapset modules are processed so as to reduce startup time.

One can `@use` multiple modules like this

    @use kernel.socket
    @use nginx.upstream

or equivalently,

    @use kernel.socket, nginx.upstream

All those macro variables are free to use in the tapset module files.

Shorthands
----------

### @pfunc(FUNCTION)

This is equivalent to `process("$^exec_path").function("FUNCTION").

For example,

    probe @pfunc(ngx_http_upstream_finalize_request),
          @pfunc(ngx_http_upstream_send_request)
    {
        ...
    }

is equivalent to

    probe process("$^exec_path").function("ngx_http_upstream_finalize_request"),
          process("$^exec_path").function("ngx_http_upstream_send_request")
    {
        ...
    }

Samples
=======

ngx-rps
-------

Calculate the current number of requests per second handled by the Nginx
worker process specified by its pid:

    # making the ./stap++ tool visible in PATH:
    $ export PATH=$PWD:PATH

    # assuming one nginx worker process has the pid 19647.
    $ ./samples/ngx-rps.sxx -x 19647
    WARNING: Tracing process 19647.
    Hit Ctrl-C to end.
    [1376939543] 300 req/sec
    [1376939544] 235 req/sec
    [1376939545] 235 req/sec
    [1376939546] 166 req/sec
    [1376939547] 238 req/sec
    [1376939548] 234 req/sec
    ^C

The numbers in the leading square brackets are the current timestamp (seconds since the Epoch).

Behind the scene, the Nginx main requests' completion events are traced.

Author
======

Yichun Zhang (agentzh), <agentzh@gmail.com>, CloudFlare Inc.

Copyright and License
=====================

This module is licensed under the BSD license.

Copyright (C) 2013, by Yichun "agentzh" Zhang (章亦春) <agentzh@gmail.com>, CloudFlare Inc.

All rights reserved.

Redistribution and use in source and binary forms, with or without modification, are permitted provided that the following conditions are met:

* Redistributions of source code must retain the above copyright notice, this list of conditions and the following disclaimer.

* Redistributions in binary form must reproduce the above copyright notice, this list of conditions and the following disclaimer in the documentation and/or other materials provided with the distribution.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

See Also
========
* SystemTap Wiki Home: http://sourceware.org/systemtap/wiki
* Nginx Systemtap Toolkit: https://github.com/agentzh/nginx-systemtap-toolkit
