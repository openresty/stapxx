NAME
====

stap++ - Simple macro language extensions to systemtap

Table of Contents
=================

* [NAME](#name)
* [Synopsis](#synopsis)
* [Description](#description)
* [Features](#features)
    * [Standard Macro Variables](#standard-macro-variables)
        * [$^exec_path](#exec_path)
        * [$^libNAME_path](#libname_path)
        * [$^arg_NAME](#arg_name)
        * [Default values](#default-values)
    * [User-defined Macro Variables](#user-defined-macro-variables)
    * [Tapset Modules](#tapset-modules)
    * [Shorthands](#shorthands)
        * [@pfunc(FUNCTION)](#pfuncfunction)
* [Samples](#samples)
    * [ngx-rps](#ngx-rps)
    * [ngx-req-latency-distr](#ngx-req-latency-distr)
    * [ctx-switches](#ctx-switches)
    * [ngx-lj-gc](#ngx-lj-gc)
    * [lj-gc](#lj-gc)
    * [ngx-lj-gc-objs](#ngx-lj-gc-objs)
    * [lj-gc-objs](#lj-gc-objs)
    * [ngx-lj-vm-states](#ngx-lj-vm-states)
    * [lj-vm-states](#lj-vm-states)
    * [ngx-lj-trace-exits](#ngx-lj-trace-exits)
    * [ngx-lj-lua-bt](#ngx-lj-lua-bt)
    * [lj-lua-bt](#lj-lua-bt)
    * [ngx-lj-lua-stacks](#ngx-lj-lua-stacks)
    * [lj-lua-stacks](#lj-lua-stacks)
    * [epoll-et-lt](#epoll-et-lt)
    * [epoll-loop-blocking-distr](#epoll-loop-blocking-distr)
    * [sample-bt-leaks](#sample-bt-leaks)
    * [ngx-lua-shdict-writes](#ngx-lua-shdict-writes)
    * [ngx-lua-shdict-info](#ngx-lua-shdict-info)
    * [ngx-single-req-latency](#ngx-single-req-latency)
    * [ngx-rewrite-latency-distr](#ngx-rewrite-latency-distr)
    * [ngx-lua-exec-time](#ngx-lua-exec-time)
    * [ngx-lua-tcp-recv-time](#ngx-lua-tcp-recv-time)
    * [ngx-lua-tcp-total-recv-time](#ngx-lua-tcp-total-recv-time)
    * [ngx-lua-udp-recv-time](#ngx-lua-udp-recv-time)
    * [ngx-lua-udp-total-recv-time](#ngx-lua-udp-total-recv-time)
    * [ngx-orig-resp-body-len](#ngx-orig-resp-body-len)
    * [zlib-deflate-chunk-size](#zlib-deflate-chunk-size)
    * [lj-str-tab](#lj-str-tab)
    * [func-latency-distr](#func-latency-distr)
    * [ngx-count-conns](#ngx-count-conns)
    * [ngx-lua-count-timers](#ngx-lua-count-timers)
    * [cpu-hogs](#cpu-hogs)
    * [cpu-robbers](#cpu-robbers)
    * [ngx-pcre-dist](#ngx-pcre-dist)
    * [ngx-pcre-top](#ngx-pcre-top)
    * [vfs-page-cache-misses](#vfs-page-cache-misses)
* [Installation](#installation)
* [Author](#author)
* [Copyright and License](#copyright-and-license)
* [See Also](#see-also)

Synopsis
========

```bash
    $ stap++ -I ./tapset -x 12345 --arg limit=10 samples/ngx-upstream-post-conn.sxx
    $ stap++ -e 'probe begin { println("hello") exit() }'
```

Description
===========

This interpreter adds some simple macro language extensions to the systemtap scripting language.

Efforts has been made to ensure that this macro language expansion does
not affect the source line numbers so that the line numbers reported by `stap` are exactly the same in the original `.sxx` source files.

Features
========

[Back to TOC](#table-of-contents)

Standard Macro Variables
------------------------

[Back to TOC](#table-of-contents)

### $^exec_path

The variable `$^exec_path` is always evaluated to the path to the executable file
for the pid specified by the `-x` or `--master` option.

Here is an example:

```stap
    probe process("$^exec_path").function("blah") { ... }
```

When you specify the `--exec PATH` option on the command line, then
this PATH is always used regardless of the presense of the `-x` or `--master` option.

[Back to TOC](#table-of-contents)

### $^libNAME_path

This variable expands to the absolute path of the DSO library file specified by a pattern.

`stap++` automatically scans all the loaded DSO files in the running process (if the `-x PID` option is specified) to find a match. If it fails to find a match, this variable will take the value of `$^exec_path`, that is, assuming the library is statically linked.

Below is an example for tracing a user-land function in the libpcre library:

```stap
    probe process("$^libpcre_path").function("pcre_exec")
    {
        println("pcre_exec called")
        print_ubacktrace()
    }
```

[Back to TOC](#table-of-contents)

### $^arg_NAME

This variable can evaluate to the value of a specified command-line argument. For example, `$^arg_limit` is evaluated to the value of the command line argument `limit` specified like this:

    stap++ --arg limit=1000

You can dump out all the available arguments in the stap++ script by specifying the --args option, for example:

    $ stap++ --args foo.sxx
    --arg method=VALUE (default: )
    --arg time=VALUE (default: 60)

[Back to TOC](#table-of-contents)

### Default values

It's possible to specify a default value for a macro variable by means of the `default` trait, as in

    foreach (key in stats- limit $^arg_limit :default(1000)) {
        ...
    }

where `$^arg_limit` takes the default value 1000 when the user does not specify the `--arg limit=N` command-line option while invoking `stap++`.

[Back to TOC](#table-of-contents)

User-defined Macro Variables
----------------------------

It's possible to bind a `@cast()` or `@var()` expression to a user-defined macro variable of the form `$*NAME`. Here is an example,

    sock = sockfd_lookup(fd)
    $*sock := @cast(sock, "socket", "kernel")

    printf(", sock->state:%d", $*sock->state)
    state = $*sock->sk->__sk_common->skc_state
    printf(", sock->sk->sk_state:%d (%s)\n", state, tcp_sockstate_str(state))

Note that we used the `:=` operator to bind a `@cast()` or `@var()` expression to user variable `$*sock`, and later we reference it whenever we need that `@cast()` or `@var()` expression.

The scope of user variables is always limited to the current `.sxx` source file.

[Back to TOC](#table-of-contents)

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

[Back to TOC](#table-of-contents)

Shorthands
----------

[Back to TOC](#table-of-contents)

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

[Back to TOC](#table-of-contents)

Samples
=======

[Back to TOC](#table-of-contents)

ngx-rps
-------

Calculate the current number of requests per second handled by the Nginx
worker process specified by its pid:

    # making the ./stap++ tool visible in PATH:
    $ export PATH=$PWD:$PATH

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

[Back to TOC](#table-of-contents)

ngx-req-latency-distr
---------------------

Calculates the distribution of the Nginx request latencies (excluding the request header reading time) in any specified
Nginx worker process at real time:

    # making the ./stap++ tool visible in PATH:
    $ export PATH=$PWD:$PATH

    $ ./samples/ngx-req-latency-distr.sxx -x 28078
    WARNING: Start tracing process 28078 (/path/to/some/program)...
    ^C
    Distribution of the main request latencies (in microseconds)
    (min/avg/max: 92/242181/42808832)
        value |-------------------------------------------------- count
           16 |                                                      0
           32 |                                                      0
           64 |                                                      8
          128 |                                                      1
          256 |                                                      3
          512 |@@@@@                                               274
         1024 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@  2474
         2048 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@                     1547
         4096 |@@@@@@@@@@@@@@@@@@@                                 952
         8192 |@@@@@@@@@@                                          500
        16384 |@@@@@@@                                             359
        32768 |@@@@@@@@                                            414
        65536 |@@@@@@@@@@@@                                        644
       131072 |@@@@@@@@@@@@@@@@@                                   851
       262144 |@@@@@@@@@@@@                                        614
       524288 |@@@@@@                                              334
      1048576 |@@                                                  147
      2097152 |                                                     46
      4194304 |                                                     24
      8388608 |@                                                    64
     16777216 |                                                      1
     33554432 |                                                      1
     67108864 |                                                      0
    134217728 |                                                      0

One can also filter out requests by a specified request method name via the `--arg method=METHOD` option. For instance,

    $ ./samples/ngx-req-latency-distr.sxx -x 5447 --arg method=POST --arg time=60
    Start tracing process 5447 (/opt/nginx/sbin/nginx)...
    Please wait for 60 seconds...
    (Tracing only POST request methods)

    Distribution of the main request latencies (in microseconds) for 52 samples:
    (min/avg/max: 1167/8373/28281)
    value |-------------------------------------------------- count
      256 |                                                    0
      512 |                                                    0
     1024 |@@                                                  2
     2048 |@@@@@@@@                                            8
     4096 |@@@@@@@@@@@@@@@@@@@@@@@                            23
     8192 |@@@@@@@@@@@@@@                                     14
    16384 |@@@@@                                               5
    32768 |                                                    0
    65536 |                                                    0

We can also see from the example above that we can limit the sampling period by specifying the `--arg time=SECONDS` option.

[Back to TOC](#table-of-contents)

ctx-switches
------------

Calculates the CPU context switching rate (number/second) in any specified user process at real time:

    # making the ./stap++ tool visible in PATH:
    $ export PATH=$PWD:$PATH

    # assuming the target process pid is 6254:
    $ ./samples/ctx-switches.sxx -x 6254
    WARNING: Tracing process 6254 (/path/to/some/process).
    Hit Ctrl-C to end.
    [1379631372] 13741 cs/sec
    [1379631373] 13330 cs/sec
    [1379631374] 14263 cs/sec
    [1379631375] 14424 cs/sec
    [1379631376] 14591 cs/sec
    [1379631377] 11108 cs/sec
    [1379631378] 12620 cs/sec
    [1379631379] 12519 cs/sec
    [1379631380] 13479 cs/sec
    [1379631381] 14614 cs/sec
    [1379631382] 14721 cs/sec
    [1379631383] 13408 cs/sec
    [1379631384] 14682 cs/sec

Both switch-in and switch-out are counted in this tool.

High context switching rate usually means higher overhead in the system. Ideally
we could keep the context switching rate low.

[Back to TOC](#table-of-contents)

ngx-lj-gc
---------

This tool has been renamed to [lj-gc](#lj-gc) because it is no longer specific to Nginx.

[Back to TOC](#table-of-contents)

lj-gc
-----

This tool analyses the LuaJIT 2.0/2.1 GC in the specified "luajit" utility program's process or the specified Nginx worker process via the [ngx_lua](http://wiki.nginx.org/HttpLuaModule) mdoule.

Other custom C processes with LuaJIT embedded can also be analyzed by this tool as long as the target C program saves the main Lua VM state (lua_State) pointer in a global C variable named `globalL`, just as in the standard `luajit` command-line utility program.

For now, it just prints out the total memory currently allocated in the LuaJIT GC. For example,

    # making the ./stap++ tool visible in PATH:
    $ export PATH=$PWD:$PATH

    # assuming the nginx worker process's pid is 4771:
    $ ./samples/lj-gc.sxx -x 4771
    Start tracing 4771 (/opt/nginx/sbin/nginx)
    Total GC count: 258618 bytes

[Back to TOC](#table-of-contents)

ngx-lj-gc-objs
--------------

This tool has been renamed to [lj-gc-objs](#lj-gc-objs) because it is no longer specific to Nginx.

[Back to TOC](#table-of-contents)

lj-gc-objs
----------

This tool dumps the GC objects' memory usage stats in any specified `luajit` utility program's process or any specified running Nginx worker process
according to the GC object's types.

Other custom C processes with LuaJIT embedded can also be analyzed by this tool as long as the target C program saves the main Lua VM state (lua_State) pointer in a global C variable named `globalL`, just as in the standard `luajit` command-line utility program.

This tool reveals exactly how the memory is distributed among all Lua value types, which is useful for optimizing Lua code's memory usage and debugging memory leak issues in the Lua programs.

For now, both LuaJIT 2.0 and LuaJIT 2.1 are supported.

Here is an example.

    # making the ./stap++ tool visible in PATH:
    $ export PATH=$PWD:$PATH

    # assuming the nginx worker pid is 5686:
    $ ./samples/lj-gc-objs.sxx -x 5686
    Start tracing 5686 (/opt/nginx/sbin/nginx)

    main machine code area size: 65536 bytes
    C callback machine code size: 4096 bytes
    GC total size: 922648 bytes
    GC state: sweep

    4713 table objects: max=6176, avg=90, min=32, sum=428600 (in bytes)
    3341 string objects: max=2965, avg=47, min=18, sum=159305 (in bytes)
    677 function objects: max=144, avg=29, min=20, sum=20224 (in bytes)
    563 userdata objects: max=8895, avg=82, min=24, sum=46698 (in bytes)
    306 proto objects: max=34571, avg=541, min=78, sum=165557 (in bytes)
    287 upvalue objects: max=24, avg=24, min=24, sum=6888 (in bytes)
    102 trace objects: max=928, avg=337, min=160, sum=34468 (in bytes)
    8 cdata objects: max=24, avg=17, min=16, sum=136 (in bytes)
    7 thread objects: max=1648, avg=1493, min=568, sum=10456 (in bytes)
    JIT state size: 6920 bytes
    global state tmpbuf size: 2948 bytes
    C type state size: 2520 bytes

    My GC walker detected for total 922648 bytes.
    5782 microseconds elapsed in the probe handler.

For LuaJIT instances with big memory usage, you need to increase the `MAXACTION` threshold, as in

    $ ./samples/lj-gc-objs.sxx -x 14378 -D MAXACTION=200000
    Start tracing 14378 (/opt/nginx/sbin/nginx)

    main machine code area size: 65536 bytes
    C callback machine code size: 4096 bytes
    GC total size: 9683407 bytes
    GC state: pause

    27948 table objects: max=131112, avg=106, min=32, sum=2983944 (in bytes)
    22343 string objects: max=1421562, avg=198, min=18, sum=4432482 (in bytes)
    12168 userdata objects: max=8916, avg=50, min=27, sum=619223 (in bytes)
    2837 function objects: max=148, avg=27, min=20, sum=78264 (in bytes)
    1200 upvalue objects: max=24, avg=24, min=24, sum=28800 (in bytes)
    650 proto objects: max=3860, avg=313, min=74, sum=203902 (in bytes)
    349 thread objects: max=1648, avg=774, min=424, sum=270464 (in bytes)
    202 trace objects: max=1560, avg=375, min=160, sum=75832 (in bytes)
    9 cdata objects: max=36, avg=17, min=12, sum=156 (in bytes)
    JIT state size: 7696 bytes
    global state tmpbuf size: 710772 bytes
    C type state size: 4568 bytes

    My GC walker detected for total 9683407 bytes.
    45008 microseconds elapsed in the probe handler.

The "objects" are Lua values that actually participate in garbage
collection (GC).

Primitive Lua values like numbers, booleans, nils, and light user data
do not participate in GC, so we shall never see them listed in the
output of the `lj-gc-objs` tool.

Another interesting exception is empty Lua strings, they are specially
handled by LuaJIT and they never appear in the output either.

The following types of objects participate in GC and thus considered
"GC objects" in LuaJIT 2.0:

* string: Lua strings
* upvalue: Lua Upvalues
* thread: Lua threads (i.e., Lua coroutines)
* proto: Lua function prototypes
* function: Lua functions (Lua closures) and C functions
* cdata: cdata created by the FFI API in Lua.
* table: Lua tables
* userdata: Lua user data
* trace: JIT compiled Lua code paths

Note that for the space calculated for aggregate objects like "table" and
"function", only the size of their backbones is calculated. For
example, for a Lua table, we do not follow the references to its value
objects and key objects (but we do include the size of the "key" and
"value" reference pointers themselves). Similarly, for a Lua function
object, we do not follow its upvalues either. Therefore, we can safely
add up the sizes of all the GC objects to obtain the total size
allocated by the LuaJIT GC without repeating anyone.

It's worth mentioning that the LuaJIT GC may also allocate some space
for some components in the VM itself (on demand):

* the JIT compiler state (i.e., the "JIT state size" item in the tool's output).
* the state for FFI C types (i.e., the "C type state size" item in the output).
* global state temporary buffer (i.e., the "global state tmpbuf size" item).

These allocations may not happen for trivial examples. For example, if
the JIT compiler is disabled, we won't see nonzero "JIT state size".
Similarly, if we don't use FFI in our Lua code at all, we won't see
nonzero "C type state size" either.

Like any language with a proper GC, any unused Lua objects will be
automatically freed by the LuaJIT GC. You make a Lua object become
unused by avoid referencing the object from any "GC root objects"
either directly or indirectly.

It is usually fine for Lua objects to stay a little longer then needed.Basically it's fine and also
normal to see the GC count (or the memory allocated by the GC) going
up and down during the lifetime of the process.

That is how a typical incremental GC works. The LuaJIT GC cycle
consists of various different phases. And all these phases are divided
into many small pieces that interleave with normal Lua code execution.
For example, when the GC cycle is at the "sweep-string" phase,
non-string GC objects will not freed at all until the GC cycle later
enters the "sweep" phase.

This tool is good at finding out the largest GC objects
in your already bloated Nginx worker processes. It is not really designed for debugging
individual requests due to the non-determinism of the GC on micro
levels. You should load your nginx workers by tools like ab and
weighttp or just trace workers in production, so as to make your nginx worker eat up a _lot_ of memory. The
more, the merrier. After that, run `lj-gc-objs` on your largest
`luajit` process or `nginx` worker process.

To debug individual requests, you *can* force the LuaJIT GC to free all the unsed objects at once by calling

    collectgarbage()

This forces the LuaJIT (or standard Lua interpreter) GC to run a
complete collection cycle immediately. See
http://www.lua.org/manual/5.1/manual.html#pdf-collectgarbage for more
details. But this is usually very expensive to call and it is strongly discouraged for production use.

[Back to TOC](#table-of-contents)

ngx-lj-vm-states
----------------

This tool has been renamed to [lj-vm-states](#lj-vm-states) because it is no longer specific to Nginx.

[Back to TOC](#table-of-contents)

lj-vm-states
----------------

This tool samples the LuaJIT's VM states in the specified `luajit` utility program or the specified nginx worker process (running the ngx_lua module) via kernel's timer hooks.

Other custom C processes with LuaJIT embedded can also be analyzed by this tool as long as the target C program saves the main Lua VM state (lua_State) pointer in a global C variable named `globalL`, just as in the standard `luajit` command-line utility program.

We can know how the CPU time is distributed among interpreted Lua code, (JIT) compiled Lua code, garbage collector, and etc.

The following LuaJIT VM states are analyzed:

* Interpreted

    Running interpreted Lua code
* Compiled

    Running already compiled Lua code (this including running C functions *compiled* by the `FFI` API)
* C Code (by interpreted Lua)

    Running C functions of the form `lua_CFunction` or called by the `FFI` API in the Lua interpreter.
* Garbage Collector

    Doing GC work in the garbage collector (for both compiled code and interpreted code).
* Trace exiting

    Exiting compiled Lua code and falling back to the Lua interpreter.
* JIT Compiler

    Compiling Lua code to native code in the Lua JIT compiler.

For now, this tool only supports LuaJIT v2.1.

Below are some examples:

```bash
# making the ./stap++ tool visible in PATH:
$ export PATH=$PWD:$PATH

$ lj-vm-states.sxx -x 24405 --arg time=30
Start tracing 24405 (/opt/nginx/sbin/nginx)
Please wait for 30 seconds...

Observed 457 Lua-running samples and ignored 0 unrelated samples.
C Code (by interpreted Lua): 78% (361 samples)
Interpreted: 14% (65 samples)
Garbage Collector: 4% (21 samples)
Compiled: 1% (5 samples)
JIT Compiler: 1% (5 samples)
```

In this example, we can see most of the CPU time is spent on interpreted Lua code, which means big room for future speedup via JIT compiling more hot Lua code paths.

```bash
$ lj-vm-states.sxx -x 9087
Start tracing 9087 (/opt/nginx/sbin/nginx)
Hit Ctrl-C to end.
^C
Observed 2082 Lua-running samples and ignored 0 unrelated samples.
Compiled: 97% (2024 samples)
Garbage Collector: 2% (56 samples)
Trace exiting: 0% (1 samples)
Interpreted: 0% (1 samples)
```

This example shows that almost all the CPU time is spent on compiled Lua code, which is a very good sign.

[Back to TOC](#table-of-contents)

ngx-lj-trace-exits
------------------

This tool analyzes the number of LuaJIT "trace exits" in each request handled by Nginx (with the [ngx_lua](https://github.com/chaoslawful/lua-nginx-module) module enabled).

A "trace exit" is an event that the LuaJIT VM exits a compiled trace (or a single unit of compiled Lua code path) and falls back to LuaJIT's Lua interpreter.

Basically we hope every request has some trace exits, but not too many! Too many trace exits per request usually means highly fragmented compiled Lua code paths and relatively high overhead in synchronizing the VM state when leaving the compiled code and entering interpreted mode.

Consider the following example,

    # making the ./stap++ tool visible in PATH:
    $ export PATH=$PWD:$PATH

    # assuming the nginx worker process has the pid 24391:
    $ ngx-lj-trace-exits.sxx -x 24391 --arg time=20
    Start tracing process 24391 (/opt/nginx/sbin/nginx)...
    Please wait for 20 seconds...

    5 out of 2251 requests used compiled traces generated by LuaJIT.
    Distribution of LuaJIT trace exit times per request for 5 sample(s):
    (min/avg/max: 1/1/2)
    value |-------------------------------------------------- count
        0 |                                                   0
        1 |@@@@                                               4
        2 |@                                                  1
        4 |                                                   0
        8 |                                                   0

We can see that only 5 out of 2251 requests use compiled Lua code, which is a bad sign of slowness if most of the requests go through some Lua code.

Also, in the example above, we used the `--arg time=20` option to let
the tool automatically quit after 20 seconds of sampling.

Now take a look at the following example,

    $ ngx-lj-trace-exits.sxx -x 12718 --arg time=20
    Start tracing process 12718 (/opt/nginx/sbin/nginx)...
    Hit Ctrl-C to end.
    ^C
    18022 out of 18023 requests used compiled traces generated by LuaJIT.
    Distribution of LuaJIT trace exit times per request for 18022 sample(s):
    (min/avg/max: 4/4/6)
    value |-------------------------------------------------- count
        1 |                                                       0
        2 |                                                       0
        4 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@  18022
        8 |                                                       0
       16 |                                                       0

we can see that almost all the requests used compiled traces, which is great! And we can see that almost all the requests have 4 "trace exits" in their lifetime.

[Back to TOC](#table-of-contents)

ngx-lj-lua-bt
-------------

This tool has been renamed to [lj-lua-bt](#lj-lua-bt) because it is no longer specific to Nginx.

[Back to TOC](#table-of-contents)

lj-lua-bt
-------------

This tool dumps out the current Lua backtrace in the running LuaJIT 2.1 VM in the specified "luajit" utility program or the specified nginx worker process
(with the [ngx_lua](https://github.com/chaoslawful/lua-nginx-module) module enabled).

Other custom C processes with LuaJIT embedded can also be analyzed by this tool as long as the target C program saves the main Lua VM state (lua_State) pointer in a global C variable named `globalL`, just as in the standard `luajit` command-line utility program.

This tool uses the kernel timer hook to preempt into the target nginx process. So even if the Lua code is in a tight loop or the C code called by some Lua code is spinning, we can get a proper Lua backtrace. But when the process is simply blocking on some system calls without consuming any CPU time at all, then this tool will just hang and keep waiting.

This tool is best for locating very hot or even infinite loops in some Lua code or C code called by some Lua code.

Both interpreted and compiled Lua code is supported (remember that LuaJIT ships with both a fast interpreter and an awesome JIT compiler?).

Below is an example,

    # making the ./stap++ tool visible in PATH:
    $ export PATH=$PWD:$PATH

    # assuming the nginx worker process pid is 22544:
    $ ./samples/lj-lua-bt.sxx --skip-badvars -x 22544
    Start tracing 22544 (/opt/nginx/sbin/nginx)
    @/home/agentzh/git/cf/nginx-waf/gen/lua/waf-core.lua:1283
    /waf/rulesets/lua/modsecurity_setup.lua:66
    @/home/agentzh/git/cf/nginx-waf/gen/lua/waf.lua:289
    builtin#21
    @/home/agentzh/git/cf/nginx-waf/gen/lua/waf.lua:309
    access_by_lua:3

The file paths here are for the Lua source file defining the Lua function (frames). And the number after the colon is the corresponding file line number where the corresponding source line is currently executed.

For every Lua function frame, the line number of the Lua source line currently being executed will be printed out wherever possible. Failing that, the first line of the Lua function will be printed instead.

Function frames prefixed by `C:` means C function frames and the things after `C:` are the C function names. Such C functions are only those called via the `lua_CFunction` mechanism, not LuaJIT FFI.

Function frames prefixed by `T:` are the Lua functions from which the (compiled) traces start. For traces generated by the JIT compiler, they could "encompass more than one function
and may correspond to multiple Lua stack levels". But for simplicity, we only fetch their starting Lua function as a first-order approximation for now. This approximation should be good enough for most of the common cases.

Function frames like `builtin#82` means Lua builtin functions and the number `82` here indicates the builtin function (or "fast function")'s ID. You can get the actual function name from the ID by running the [ljff.lua](https://github.com/agentzh/nginx-devel-utils/blob/master/ljff.lua) script using exactly the same LuaJIT, for example,

    $ luajit-2.1.0-alpha /path/to/nginx-devel-utils/ljff.lua 82
    FastFunc string.dump

Which means the ID 82 corresponds to the "string.dump" builtin. The IDs of builtins might be different among different versions of LuaJIT, so ensure you are using the same LuaJIT to run this Lua script.

[Back to TOC](#table-of-contents)

ngx-lj-lua-stacks
-----------------

This tool has been renamed to [lj-lua-stacks](#lj-lua-stacks) because it is no longer specific to Nginx.

[Back to TOC](#table-of-contents)

lj-lua-stacks
-------------

This tool samples Lua backtraces in the running LuaJIT 2.1 VM of the specified `luajit` process or `nginx` worker process (with the [ngx_lua](https://github.com/chaoslawful/lua-nginx-module) module). The timer hook API of the Linux kernel is used for relatively even sampling according to the CPU time usage.

It aggregates idential Lua backtraces during the sampling process so the final output data will not be very big.

Other custom C processes with LuaJIT embedded can also be analyzed by this tool as long as the target C program saves the main Lua VM state (lua_State) pointer in a global C variable named `globalL`, just as in the standard `luajit` command-line utility program.

Below is some examples:

    # making the ./stap++ tool visible in PATH:
    $ export PATH=$PWD:$PATH

    # assuming the nginx worker process pid is 6949:
    $ ./samples/lj-lua-stacks.sxx --skip-badvars -x 6949 > a.bt
    Start tracing 6949 (/opt/nginx/sbin/nginx)
    Hit Ctrl-C to end.
    ^C

By default, the tool will just keep sampling until you hit `Ctrl-C`. You can also specify the `--arg time=N` option to let the tool exit automatically after `N` seconds. For example,

    $ ./samples/lj-lua-stacks.sxx --arg time=5 \
                --skip-badvars -x 6949 > a.bt
    Start tracing 6949 (/opt/nginx/sbin/nginx)
    Please wait for 5 seconds

For the sample commands above, the tool's output data will then be in the file `a.bt`. The output format is

    <backtrace-1>
    \t<count-1>
    <backtrace-2>
    \t<count-2>
    <backtrace-3>
    \t<count-3>
    ...

Below is an example:

    C:ngx_http_lua_blah
    @/opt/gen/lua/waf-core.lua:770
    @/opt/gen/lua/waf-core.lua:1283
    /opt/gen/lua/waf-attacks.lua:38
    builtin#21
    @/opt/gen/lua/waf.lua:301
    access_by_lua:1
            97
    T:@/opt/gen/lua/waf-core.lua:770
    @/opt/gen/lua/waf-core.lua:770
    @/opt/gen/lua/waf-core.lua:1283
    /opt/gen/xss_attacks.lua:15
    @/opt/gen/lua/waf.lua:174
    builtin#21
    @/opt/gen/lua/waf.lua:301
    access_by_lua:1
            52

Each backtrace above follows the same format as in the output of the [lj-lua-bt](#lj-lua-bt) tool. The only difference is that the line numbers are always for the first source lines of the Lua functions.

This output can be consumed by the [FlameGraphs tool](https://github.com/brendangregg/FlameGraph) (created by Brenden Gregg) to generate Lua-land Flame Graphs for performance analysis. For example,

    $ stackcollapse-stap.pl a.bt > a.cbt

    $ flamegraph.pl --encoding="ISO-8859-1" \
                  --title="Lua-land on-CPU flamegraph" \
                  a.cbt > a.svg

The resulting `a.svg` file is an interactive SVG graph that can be displayed in any descent web browser, like Google Chrome:

    $ chrome a.svg

You can get much better Lua-land Flame Graphs by filtering the output of this `lj-lua-stacks` tool with the [fix-lua-bt](https://github.com/agentzh/nginx-systemtap-toolkit#fix-lua-bt) tool in my [Nginx Systemtap Toolkit](https://github.com/agentzh/nginx-systemtap-toolkit):

    $ fix-lua-bt a.bt > a2.bt

And then feed the `a2.bt` file instead to Brendan Gregg's `stackcollapse-stap.pl` tool and etc to generate the graph.

Below are some real-world sample Flame Graphs on the Lua land:

* http://agentzh.org/misc/flamegraph/lua-on-cpu-local-waf-jitted-only.svg
* http://agentzh.org/misc/flamegraph/lua-on-cpu-local-waf-interp-only.svg

By default, this tool will sample backtraces of both interpreted Lua code and compiled Lua code at the same time (remember that LuaJIT comes with both a fast interpreter and an awesome JIT compiler?).

You can choose to sample interpreted Lua code only by specifying the `--arg nojit=1` option to ignore backtrace samples for (JIT) compiled Lua code. For instance,


    $ ./samples/lj-lua-stacks.sxx --arg nojit=1 \
            --arg time=5 --skip-badvars -x $pid > a.bt

Similarly, you can choose to see samples for JIT compiled Lua code only by specifying the `--arg nointerp=1` option to ignore samples for interpreted code. For example,

    $ ./samples/lj-lua-stacks.sxx --arg nointerp=1 \
            --arg time=5 --skip-badvars -x $pid > a.bt

When you're sampling interpreted Lua backtraces, you might see some warnings like below when this tool is running:

    WARNING: user string copy fault -14 at 00000000f9b56dac
    WARNING: user string copy fault -14 at 00000000fffb4196
    WARNING: user string copy fault -14 at 0000000004014212

This warnings are normal because our timer callback might run in an arbitrary context of the LuaJIT VM (thanks to the kernel's preemption feature), which means some times the VM might be in an inconsistent state. These read/copy faults will not affect the target (nginx) processes at all because systemtap temporarily disables page faults when reading memory from the userspace. And we should not worry about the accuracy of the sampling results too much unless there are too many such warnings (like hundreds or even thousands). Thanks to the error-tolerance feature of statistics.

When sampling compiled Lua code's backtraces, we should not see these "copy fault" warnings at all because when compiled Lua code is running, the VM state is almost always consistent (because the running traces do not synchronize the VM state until they exit).

By default, this tool computes the line numbers at which the corresponding Lua functions are defined (that is, the first line of the Lua function definition). If you want accurate position for the Lua source line being executed in the resulting backtraces, you can specify the `--arg detailed=1` command-line option.

This tool is superior to LuaJIT's builtin low-overhead profiler in that

1. this tool never flushes the existing compiled traces but LuaJIT's builtin profiler always does upon profiler startups and exits,
1. this tool does not affect the LuaJIT VM's interpreter dispatcher nor JIT compiler while LuaJIT's builtin profiler requires a special running mode (and collaborations) for both the interpreter and JIT compiler.
1. this tool can construct backtraces for JIT compiled code directly while LuaJIT's builtin profiler requires falling back to the interpreter to evaluate the current backtraces.
1. this tool can be turned on and off more easily on any running (nginx) processes specified by pid. And when this tool is not running, there is strictly zero overhead in the target process.
1. Bugs in this tool has no impact on the target process, even when reading from bad memory addresses.

The overhead exposed on the target process is usually small. For example, the throughput (req/sec) limit of an nginx worker process doing simplest "hello world" requests drops by only 10% (only when this tool is running), as measured by `ab -k -c2 -n100000` when using Linux kernel 3.6.10 and systemtap 2.5. The impact on full-fledged production processes is usually smaller than even that, for instance, only 5% drop in the throughput limit is observed in a production-level Lua CDN application.

Special thanks go to Mike Pall for providing technical support in the
LuaJIT VM internals.

See also

1. The [lj-vm-states](#lj-vm-states) tool for sampling the LuaJIT VM states.
1. Brendan Gregg's article "[Flame Graphs](http://dtrace.org/blogs/brendan/2011/12/16/flame-graphs/)".

[Back to TOC](#table-of-contents)

epoll-et-lt
-----------

This tool can checks how many epoll_ctl syscalls use the epoll edge-trigger (ET) mode and how many use the epoll level-trigger (LT) model. The statistics is gathered and printed out every 1 second. This tool is not specific to Nginx.

Here is an example:

    # making the ./stap++ tool visible in PATH:
    $ export PATH=$PWD:$PATH

    $ ./samples/epoll-et-lt.sxx -x 5728
    Tracing epoll_ctl in user process 5728 (/opt/nginx/sbin/nginx)...
    Hit Ctrl-C to end.
    51 ET, 0 LT.
    384 ET, 0 LT.
    388 ET, 0 LT.
    390 ET, 0 LT.
    389 ET, 0 LT.
    394 ET, 0 LT.
    394 ET, 0 LT.
    396 ET, 0 LT.
    395 ET, 0 LT.
    384 ET, 0 LT.
    394 ET, 0 LT.
    396 ET, 0 LT.
    397 ET, 0 LT.
    ^C

We can see that Nginx is using epoll ET exclusively :)

Below is another example for KyotoTycoon servers:

    $ epoll-et-lt.sxx -x 4011
    Tracing epoll_ctl in user process 4011 (/usr/local/bin/ktserver)...
    Hit Ctrl-C to end.
    0 ET, 0 LT.
    0 ET, 3 LT.
    0 ET, 0 LT.
    0 ET, 2 LT.
    0 ET, 0 LT.
    0 ET, 0 LT.
    0 ET, 4 LT.
    0 ET, 3 LT.
    0 ET, 0 LT.
    0 ET, 5 LT.
    0 ET, 1 LT.
    0 ET, 1 LT.
    0 ET, 1 LT.
    0 ET, 0 LT.
    ^C

We can see that the `ktserver` process is using epoll LT all the time :)

[Back to TOC](#table-of-contents)

epoll-loop-blocking-distr
-------------------------

This tool can sample any specified user process for the specified time (by default, 5 seconds) and print out the distribution
of the latency between successive epoll_wait syscalls.

Essentially, it can give you a picture about the blocking latency involved in
the process's epoll-based event loop (if there is one).

Here is an example for analyzing a massively blocked Nginx worker processes in production (by really bad disk IO):

    $ ./samples/epoll-loop-blocking-distr.sxx -x 19647 --arg time=60
    Start tracing 19647...
    Please wait for 60 seconds.
    Distribution of epoll loop blocking latencies (in milliseconds)
    max/avg/min: 1097/0/0
    value |-------------------------------------------------- count
        0 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@  18471
        1 |@@@@@@@@                                            3273
        2 |@                                                    473
        4 |                                                     119
        8 |                                                      67
       16 |                                                      51
       32 |                                                      35
       64 |                                                      20
      128 |                                                      23
      256 |                                                       9
      512 |                                                       2
     1024 |                                                       2
     2048 |                                                       0
     4096 |                                                       0

Note the long tail from 4ms ~ 1.1sec.

To further analyse exactly what is blocking the epoll loop, you
can use the off-CPU and on-CPU flame graph tools:

https://github.com/agentzh/nginx-systemtap-toolkit#sample-bt

https://github.com/agentzh/nginx-systemtap-toolkit#sample-bt-off-cpu

[Back to TOC](#table-of-contents)

sample-bt-leaks
---------------

This tool can sample backtraces for memory allocations based on glibc's builtins (`malloc`, `calloc`, `realloc`) that have not been freed (via `free`) in the sampling time period.

Here is an example:

    # making the ./stap++ tool visible in PATH:
    $ export PATH=$PWD:$PATH

    # assuming the target process has the pid 16795:
    $./samples/sample-bt-leaks.sxx -x 16795 --arg time=5 \
            -D STP_NO_OVERLOAD -D MAXMAPENTRIES=10000 > a.bt
    WARNING: Start tracing 16795 (/opt/nginx/sbin/nginx)...
    WARNING: Wait for 5 sec to complete.

    $ export PATH=/path/to/FlameGraph:$PATH
    $ stackcollapse-stap.pl a.bt > a.cbt
    $ flamegraph.pl --countname=bytes \
            --title="Memory Leak Flame Graph" a.cbt > a.svg

Yu can now open the "Memory Leak Flame Graph" file, `a.svg`, in your favorite web browser.

The tools `stackcollapse-stap.pl` and `flamegraph.pl` are from the FlameGraph repository by Brendan Gregg:

https://github.com/brendangregg/FlameGraph

Below is a sample "Memory Leak Flame Graph" for a real memory leak bug
in older versions of the Nginx core:

http://agentzh.org/misc/flamegraph/nginx-leaks-2013-10-08.svg

For more details about this bug, see http://forum.nginx.org/read.php?2,241478,241478

This tool is general and not specific to Nginx, for example.

This tool requires the `uretprobes` feature in the kernel. If you are using an old kernel patched by the utrace patch, then you should be good. If you are using a mainline kernel, then you need at least 3.10.x.

This tool has relatively high overhead especially for processes without (clever) custom allocators (but still *way* faster than Valgrind memcheck). So be careful when
using this tool in production. Only use this tool in production when you really have a leak.

Please note that the `realloc` function in some builds of glibc may not have correct argument values, so you *may* see false positives on code paths doing `realloc`.

If you are seeing the systemtap error

```
ERROR: Array overflow, check MAXMAPENTRIES near identifiter ...
```

then you should increase the number in the `-D MAXMAPENTRIES=10000` command-line option.

If you are seeing the error

```
ERROR: probe overhead exceeded threshold
```

then you should specify the `-D STP_NO_OVERLOAD` command-line option.

You can find more details on Memory Leak Flame Graphs in Brendan Gregg's blog post:

http://dtrace.org/blogs/brendan/2013/08/16/memory-leak-growth-flame-graphs/

And general information about Flame Graphs here:

http://dtrace.org/blogs/brendan/2011/12/16/flame-graphs/

[Back to TOC](#table-of-contents)

ngx-lua-shdict-writes
---------------------

This tool can be used to trace write operations (`set`/`add`/`replace`/`safe_set`) to
[ngx_lua](https://github.com/chaoslawful/lua-nginx-module)'s shared dictionary zones in any running Nginx worker process
at real time.

Below is an example:

```bash
    # making the ./stap++ tool visible in PATH:
    $ export PATH=$PWD:$PATH

    # assuming one nginx worker process has the pid 28723.
    $ ./samples/ngx-lua-shdict-writes.sxx -x 28723
    WARNING: Tracing process 28723 (/opt/nginx/sbin/nginx).
    Hit Ctrl-C to end.
    [1383177226] add key=visitor::127.3.20.88 value_len=-1 dict=locks
    [1383177226] set key=visitor::127.3.20.88 value_len=18 dict=visitor_cache
    [1383177226] set key=visitor::127.3.20.88 value_len=-1 dict=locks
    [1383177226] add key=vzone::mybaz.net:127.26.29.8 value_len=-1 dict=locks
    [1383177226] set key=vzone::mybaz.net:127.26.29.8 value_len=11 dict=zone_cache
    [1383177226] set key=vzone::mybaz.net:174.96.24.8 value_len=-1 dict=locks
    [1383177226] set key=::BIC:127.0.0.2:Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/30.0.1599.101 Safari/537.36:- value_len=4 dict=visitor_cache
    [1383177226] add key=visitor::127.0.0.1 value_len=-1 dict=locks
    [1383177226] set key=visitor::127.0.0.1 value_len=18 dict=visitor_cache
    [1383177226] set key=visitor::127.0.0.1 value_len=-1 dict=locks
    [1383177226] add key=vzone::foobar.com:127.0.0.1 value_len=-1 dict=locks
    [1383177226] set key=vzone::foobar.com:127.0.0.1 value_len=11 dict=zone_cache
    ^C
```

The `--arg dict=NAME` option can be used to filter writes to a particular shared dictionary zone:

```bash
    # assuming one nginx worker process has the pid 28723.
    $ ./samples/ngx-lua-shdict-writes.sxx -x 28723 --arg dict=cpage_cache
    WARNING: Tracing process 28723 (/opt/nginx/sbin/nginx).
    Hit Ctrl-C to end.
    [1383177035] set key=cpage::/cpage/cf-error/1000s/838156:4891573 value_len=7 dict=cpage_cache
    [1383177035] set key=cpage::/cpage/block/ip-ban/171116:748534 value_len=1407861 dict=cpage_cache
    [1383177036] set key=cpage::/cpage/block/ip-ban/171116:748534 value_len=1407861 dict=cpage_cache
    [1383177040] set key=cpage::/cpage/cf-error/1000s/281904:1355323 value_len=309229 dict=cpage_cache
    [1383177042] set key=cpage::/cpage/cf-error/1000s/405903:2067154 value_len=7 dict=cpage_cache
    [1383177043] set key=cpage::/cpage/block/iuam-basic/841468:4916374 value_len=7 dict=cpage_cache
    [1383177043] set key=cpage::/cpage/block/ip-ban/171116:748534 value_len=1407861 dict=cpage_cache
    [1383177046] set key=cpage::/cpage/block/ip-ban/171116:748534 value_len=1407861 dict=cpage_cache
    [1383177047] set key=cpage::/cpage/block/basic-sec-captcha/291111:4428016 value_len=7 dict=cpage_cache
    ^C
```

[Back to TOC](#table-of-contents)

ngx-lua-shdict-info
-------------------

This tool can be used to analyzes the shared dictionary zone specified by the `--arg dict=name`
in the specified running nginx process.
This tool is very similar to [ngx-shm](https://github.com/openresty/nginx-systemtap-toolkit#ngx-shm),
just one more `rbtree black height`.

Below is an example:

```bash
    # making the ./stap++ tool visible in PATH:
    $ export PATH=$PWD:$PATH

    # assuming one nginx worker process has the pid 28723.
    $ ./samples/ngx-lua-shdict-info.sxx -x 59141 --arg dict=session01
    Start tracing 59141 (/usr/local/openresty-debug/nginx/sbin/nginx)
    shm zone "session01"
        owner: ngx_http_lua_shdict
        total size: 102400 KB
        free pages: 76792 KB (19198 pages, 1 blocks)
        rbtree black height: 11

    26 microseconds elapsed in the probe handler.
```

[Back to TOC](#table-of-contents)

ngx-single-req-latency
----------------------

Analyze the detailed latency time composition in an individual request served by an Nginx server instance. This tool can measure the time spent in the major Nginx request processing phases (like `rewrite` phase, `access` phase, and `content` phase). It will also measure Nginx upstream modules' latency on upstream connect() and etc.

By default it just tracks the first request it sees. For example,

```bash
    # making the ./stap++ tool visible in PATH:
    $ export PATH=$PWD:$PATH

    # assuming an nginx worker process's pid is 27327
    $ ./samples/ngx-single-req-latency.sxx -x 27327
    Start tracing process 27327 (/opt/nginx/sbin/nginx)...

    POST /api_json
        total: 143596us, accept() ~ header-read: 43048us, rewrite: 8us, pre-access: 7us, access: 6us, content: 100507us
        upstream: connect=29us, time-to-first-byte=99157us, read=103us

    $ ./samples/ngx-single-req-latency.sxx -x 27327
    Start tracing process 27327 (/opt/nginx/sbin/nginx)...

    GET /robots.txt
        total: 61198us, accept() ~ header-read: 33410us, rewrite: 7us, pre-access: 7us, access: 5us, content: 27750us
        upstream: connect=30us, time-to-first-byte=18955us, read=96us
```

where we use `-x <pid>` to specify an nginx worker process. We can also specify `--master <master-pid>` to monitor on all the Nginx worker processes under the master process specified.

The `--arg header=HEADER` option can be used to filter out the request by a specific request header, for instance,

```bash
    # assuming the nginx master process pid is 7088, and the request
    # being analyzed has the "Test" request header with non-empty header value:
    $ ./samples/ngx-single-req-latency.sxx --arg header=Test --master 7088
    Start tracing process 7089 7090 (/opt/nginx/sbin/nginx)...

    GET /proxy/get
        total: 2941us, accept() ~ header-read: 354us, rewrite: 100us, pre-access: 26us, access: 23us, content: 2356us
            upstream: connect=424us, time-to-first-byte=1059us, read=357us
```

[Back to TOC](#table-of-contents)

ngx-rewrite-latency-distr
-------------------------

Measure the rewrite-phase latency for each Nginx request (including subrequest) in an Nginx worker process specified and output the distribution of the latency.

By default, you hit Ctrl-C to end the sampling process:

```bash
    # making the ./stap++ tool visible in PATH:
    $ export PATH=$PWD:$PATH

    # assuming that the Nginx worker process to be analyzed is
    # of the pid 10972:
    $ ./samples/ngx-rewrite-latency-distr.sxx -x 10972
    Start tracing process 10972 (/opt/nginx/sbin/nginx)...
    Hit Ctrl-C to end.
    ^C
    Distribution of the rewrite phase latencies (in microseconds) for 478 samples:
    (min/avg/max: 19407/20465/21717)
    value |-------------------------------------------------- count
     4096 |                                                     0
     8192 |                                                     0
    16384 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@    478
    32768 |                                                     0
    65536 |                                                     0
```

You can also specify the `--arg time=SECONDS` option to let the tool quit automatically after the specified time (in seconds). For example,

```bash
    $ ./samples/ngx-rewrite-latency-distr.sxx -x 12004 --arg time=3
    Start tracing process 12004 (/opt/nginx/sbin/nginx)...
    Please wait for 3 seconds...

    Distribution of the rewrite phase latencies (in microseconds) for 46 samples:
    (min/avg/max: 0/19533/21009)
    value |-------------------------------------------------- count
        0 |@@                                                  2
        1 |                                                    0
        2 |                                                    0
          ~
     4096 |                                                    0
     8192 |                                                    0
    16384 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@       44
    32768 |                                                    0
    65536 |                                                    0
```

[Back to TOC](#table-of-contents)

ngx-lua-exec-time
-----------------

This tool measures the pure Lua code executation time (excluding any nonblocking IO time) accumulated in every request served by the specified Nginx worker at real-time and outputs the distribution of the latencies.

The time for the nginx output filters and boilerplate Lua thread initializations are excluded.

Note that both TCP sockets and stream-typed unix domain sockets are supported.

Below is an example:

```bash
    # making the ./stap++ tool visible in PATH:
    $ export PATH=$PWD:$PATH

    # assuming one nginx worker process has the pid 13501.
    $ ./samples/ngx-lua-exec-time.sxx -x 13501 --arg time=60
    Start tracing process 13501 (/opt/nginx/sbin/nginx)...
    Please wait for 60 seconds...

    Distribution of Lua code pure execution time (accumulated in each request, in microseconds) for 1605 samples:
    (min/avg/max: 92/618/2841)
    value |-------------------------------------------------- count
       16 |                                                     0
       32 |                                                     0
       64 |                                                     1
      128 |                                                    13
      256 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@    715
      512 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@  737
     1024 |@@@@@@@@                                           130
     2048 |                                                     9
     4096 |                                                     0
     8192 |                                                     0
```

The overhead exposed on the target process is usually small. For example, the throughput (req/sec) limit of an nginx worker process running a full-fledged Lua CDN application drops by only 14% (only when this tool is running), as measured by `ab -k -c2 -n100000` when using Linux kernel 3.6.10 and systemtap 2.5.

[Back to TOC](#table-of-contents)

ngx-lua-tcp-recv-time
---------------------

This tool measures the latency involved in individual [receive](https://github.com/chaoslawful/lua-nginx-module#tcpsockreceive) method calls or readers returned from the [receiveuntil](https://github.com/chaoslawful/lua-nginx-module#tcpsockreceiveuntil) method calls on [ngx_lua](https://github.com/chaoslawful/lua-nginx-module#readme) module's [ngx.socket.tcp](https://github.com/chaoslawful/lua-nginx-module#ngxsockettcp) objects in a specified Nginx worker process and outputs the distribution of the latencies.

Below is an example:

```bash
# making the ./stap++ tool visible in PATH:
$ export PATH=$PWD:$PATH

# assuming one nginx worker process has the pid 14464.
$ ./samples/ngx-lua-tcp-recv-time.sxx -x 14464 --arg time=60
Start tracing process 14464 (/opt/nginx/sbin/nginx)...
Please wait for 60 seconds...

Distribution of the ngx_lua ngx.socket.tcp receive latencies (in microseconds) for 3356 samples:
(min/avg/max: 1/74/3099)
value |-------------------------------------------------- count
    0 |                                                      0
    1 |                                                     12
    2 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@  2194
    4 |@@                                                  107
    8 |                                                      7
   16 |                                                      3
   32 |                                                     38
   64 |@@@@@@@@@@@@@@                                      631
  128 |@@@                                                 140
  256 |@@                                                  106
  512 |@                                                    82
 1024 |                                                     25
 2048 |                                                     11
 4096 |                                                      0
 8192 |                                                      0
```

See also the [ngx-lua-tcp-total-recv-time](#ngx-lua-tcp-total-recv-time) tool.

[Back to TOC](#table-of-contents)

ngx-lua-tcp-total-recv-time
---------------------------

Similar to the [ngx-lua-tcp-recv-time](#ngx-lua-tcp-recv-time) tool, but accumulate the latencies by every request served by the specified Nginx process.

This tool is useful to see how much the TCP/streaming cosocket reads contribute to the total request latency.

But note that, however, latency of cosocket reads in different ngx_lua "light threads" within the same request will be simply added up, so in case of multiple "light threads" reading at the same time will lead to larger total latency than the actual case.

Requests without TCP/stream cosocket reads will simply get skipped.

Below is an example,

```bash
# making the ./stap++ tool visible in PATH:
$ export PATH=$PWD:$PATH

# assuming one nginx worker process has the pid 14464.
$ ./samples/ngx-lua-tcp-total-recv-time.sxx  -x 14464 --arg time=60
Start tracing process 14464 (/opt/nginx/sbin/nginx)...
Please wait for 60 seconds...

Distribution of the ngx_lua ngx.socket.tcp receive latencies (accumulated in each request, in microseconds) for 649 samples:
(min/avg/max: 8/621/58875)
 value |-------------------------------------------------- count
     2 |                                                     0
     4 |                                                     0
     8 |                                                     2
    16 |                                                     0
    32 |                                                     2
    64 |@@@@@                                               40
   128 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@    329
   256 |@@@@@@@@@@@@@@                                      98
   512 |@@@@@@@@@@@@@@@                                    109
  1024 |@@@@@@@                                             51
  2048 |@                                                   13
  4096 |                                                     2
  8192 |                                                     1
 16384 |                                                     0
 32768 |                                                     2
 65536 |                                                     0
131072 |                                                     0
```

[Back to TOC](#table-of-contents)

ngx-lua-udp-recv-time
---------------------

This tool measures the latency involved in individual [receive](https://github.com/chaoslawful/lua-nginx-module#udpsockreceive) method calls on [ngx_lua](https://github.com/chaoslawful/lua-nginx-module#readme) module's [ngx.socket.udp](https://github.com/chaoslawful/lua-nginx-module#ngxsocketudp) objects in a specified Nginx worker process and outputs the distribution of the latencies.

This tool analyzes both UDP and unix domain cosockets.

Below is an example:

```bash
# making the ./stap++ tool visible in PATH:
$ export PATH=$PWD:$PATH

# assuming one nginx worker process has the pid 29906.
$ ./samples/ngx-lua-udp-recv-time.sxx -x 29906 --arg time=60
Start tracing process 29906 (/opt/nginx/sbin/nginx)...
Please wait for 60 seconds...

Distribution of the ngx_lua ngx.socket.udp receive latencies (in microseconds) for 114 samples:
(min/avg/max: 433/12468/940008)
  value |-------------------------------------------------- count
     64 |                                                    0
    128 |                                                    0
    256 |@@@@                                                9
    512 |@@@@@@@@@@@@@@@                                    30
   1024 |@@@@@@@@@@@@@@@@@@@@@@@@@@@                        54
   2048 |@@@@@@@@                                           16
   4096 |@                                                   2
   8192 |                                                    1
  16384 |                                                    0
  32768 |                                                    0
  65536 |                                                    0
 131072 |                                                    0
 262144 |                                                    1
 524288 |                                                    1
1048576 |                                                    0
2097152 |                                                    0
```

[Back to TOC](#table-of-contents)

ngx-lua-udp-total-recv-time
---------------------------

Similar to the [ngx-lua-udp-recv-time](#ngx-lua-udp-recv-time) tool, but accumulate the latencies by every request served by the specified Nginx process.

This tool is useful to see how much the UDP/unix-domain cosocket reads contribute to the total request latency.

But note that, however, latency of cosocket reads in different ngx_lua "light threads" within the same request will be simply added up, so in case of multiple "light threads" reading at the same time will lead to larger total latency than the actual case.

Requests without UDP/unix-domain cosocket reads will simply get skipped.

Below is an example,

```bash
# making the ./stap++ tool visible in PATH:
$ export PATH=$PWD:$PATH

# assuming one nginx worker process has the pid 14464.
$ ./samples/ngx-lua-udp-total-recv-time.sxx -x 14464 --arg time=60
Start tracing process 14464 (/opt/nginx/sbin/nginx)...
Please wait for 60 seconds...

Distribution of the ngx_lua ngx.socket.udp receive latencies (accumulated in each request, in microseconds) for 101 samples:
(min/avg/max: 477/60711/2991309)
  value |-------------------------------------------------- count
     64 |                                                    0
    128 |                                                    0
    256 |@                                                   2
    512 |@@@@@@@@@@@@@                                      27
   1024 |@@@@@@@@@@@@@@@@@@@@@@@@@                          51
   2048 |@@@@@@                                             12
   4096 |@                                                   3
   8192 |                                                    0
  16384 |                                                    0
  32768 |                                                    1
  65536 |                                                    0
 131072 |                                                    1
 262144 |                                                    0
 524288 |@                                                   2
1048576 |                                                    1
2097152 |                                                    1
4194304 |                                                    0
8388608 |                                                    0
```

[Back to TOC](#table-of-contents)

ngx-orig-resp-body-len
----------------------

This tool analyzes the original response body size (before going through the gzip filter module or other custom filter modules) served by Nginx and dumps out the distribution.

Below is an example:

```bash
# making the ./stap++ tool visible in PATH:
$ export PATH=$PWD:$PATH

# assuming one nginx worker process has the pid 3781.
$ ./samples/ngx-orig-resp-body-len.sxx -x 3781 --arg time=30
Start tracing process 3781 (/opt/nginx/sbin/nginx)...
Please wait for 30 seconds...

Distribution of original response body sizes (in bytes) for 1634 samples:
(min/avg/max: 0/55006/6345380)
   value |-------------------------------------------------- count
       0 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@              225
       1 |                                                     5
       2 |                                                     5
       4 |                                                     3
       8 |                                                     4
      16 |@                                                   11
      32 |@@@@@@                                              37
      64 |@@@@                                                26
     128 |@@@@@@@@@                                           56
     256 |@@@@@@@@                                            48
     512 |@@@@@@@@@@@@@                                       82
    1024 |@@@@@@@@@@@@@@@@@                                  107
    2048 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@               216
    4096 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@        258
    8192 |@@@@@@@@@@@@@@@@@@@@                               121
   16384 |@@@@@@@@@@@@@@@@@@@@@                              127
   32768 |@@@@@@@@@@@@@@@                                     94
   65536 |@@@@@@@@@@@@@@@                                     91
  131072 |@@@@@@@@@                                           56
  262144 |@@@@@                                               31
  524288 |@@@                                                 18
 1048576 |@                                                    8
 2097152 |                                                     3
 4194304 |                                                     2
 8388608 |                                                     0
16777216 |                                                     0
```

[Back to TOC](#table-of-contents)

zlib-deflate-chunk-size
-----------------------

This tool records the data chunk size fed into zlib `deflate()` and `deflateEnd()` calls in the user process specified by pid.

Below is an example:

```bash
# making the ./stap++ tool visible in PATH:
$ export PATH=$PWD:$PATH

# assuming one nginx worker process has the pid 3781.
$ ./samples/zlib-deflate-chunk-size.sxx -x 3781 --arg time=30
Start tracing process 3781 (/opt/nginx/sbin/nginx)...
Please wait for 30 seconds...

Distribution of zlib deflate chunk sizes (in bytes) for 3975 samples:
(min/avg/max: 0/2744/8192)
value |-------------------------------------------------- count
    0 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@  988
    1 |@@@                                                 61
    2 |@@                                                  40
    4 |@@                                                  41
    8 |@                                                   25
   16 |@@                                                  41
   32 |@                                                   30
   64 |@@                                                  48
  128 |@@@                                                 64
  256 |@@@                                                 76
  512 |@@@@@@@@@@                                         201
 1024 |@@@@@@@@@@@@@@@@@@@@@@@@@                          511
 2048 |@@@@@@@@@@@@@@@@@@@@@@@@                           487
 4096 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@      905
 8192 |@@@@@@@@@@@@@@@@@@@@@@                             457
16384 |                                                     0
32768 |                                                     0
```

[Back to TOC](#table-of-contents)

lj-str-tab
----------

Analayzing the structure and various statistics of the global Lua string hash table in the LuaJIT v2.1 VM.

[Back to TOC](#table-of-contents)

func-latency-distr
------------------

Calculates the latency distribution of any function-like probes which support both entry and return probes.

```
# making the ./stap++ tool visible in PATH:
$ export PATH=$PWD:$PATH

# assuming the target process has the pid 3781.

$ ./samples/func-latency-distr.sxx -x 3781 --arg func='process("$^libluajit_path").function("lj_str_new")'
Start tracing 18356 (/opt/nginx/sbin/nginx)
Hit Ctrl-C to end.
^C
Distribution of lj_str_new latencies (in nanoseconds) for 1202 samples
max/avg/min: 27463/3309/2779
value |-------------------------------------------------- count
  512 |                                                      0
 1024 |                                                      0
 2048 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@   1160
 4096 |@                                                    35
 8192 |                                                      5
16384 |                                                      2
32768 |                                                      0
65536 |                                                      0
```

```
$ func-latency-distr.sxx -x 27603 --arg func=vfs.write --arg time=10
Start tracing 27603 (/opt/nginx/sbin/nginx)
Please wait for 10 seconds...
Distribution of vfs_write latencies (in nanoseconds) for 1201 samples
max/avg/min: 545751/28313/1761
  value |-------------------------------------------------- count
    256 |                                                     0
    512 |                                                     0
   1024 |                                                     2
   2048 |                                                     0
   4096 |                                                     9
   8192 |@@@@@@@@@@@@@                                      194
  16384 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@  695
  32768 |@@@@@@@@@@@@@@@@@@                                 254
  65536 |@@                                                  41
 131072 |                                                     5
 262144 |                                                     0
 524288 |                                                     1
1048576 |                                                     0
2097152 |                                                     0
```

[Back to TOC](#table-of-contents)

ngx-count-conns
---------------

Count the number of used nginx connections and open files in the specified nginx worker process.

```
# making the ./stap++ tool visible in PATH:
$ export PATH=$PWD:$PATH

# assuming the target process has the pid 6259.
$ ./samples/ngx-count-conns -x 6259
Start tracing 6259 (/opt/nginx/sbin/nginx)...

====== CONNECTIONS ======
Max connections: 1024
Free connections: 1022
Used connections: 2

====== FILES ======
Max files: 1024
Open normal files: 2

# try another worker process
$ ngx-count-conns.sxx -x 32743
Start tracing 32743 (/opt/nginx/sbin/nginx)...

====== CONNECTIONS ======
Max connections: 32768
Free connections: 27674
Used connections: 5094

====== FILES ======
Max files: 131072
Open normal files: 2
```

[Back to TOC](#table-of-contents)

ngx-lua-count-timers
--------------------

Outputs the current timer statistics in the nginx worker process specified.

```
# making the ./stap++ tool visible in PATH:
$ export PATH=$PWD:$PATH

# assuming the target process has the pid 13982.
$ ./samples/ngx-lua-count-timers.sxx -x 13982
Start tracing 13982 (/opt/nginx/sbin/nginx)...
Running timers: 0
Max running timers: 256
Pending timers: 1
Max pending timers: 1024
```

[Back to TOC](#table-of-contents)

cpu-hogs
--------

This tool can measure how CPU time is distributed across all the running processes (and kernel threads), showing the biggest hogs.

For example, on a quite idle system, you can see a lot of "swapper" task shown on the list:

```
# making the ./stap++ tool visible in PATH:
$ export PATH=$PWD:$PATH

$ ./samples/cpu-hogs.sxx
Tracing the whole system...
Hit Ctrl-C to end.
^C
nginx-frontend 9%
nginx-backend 4%
swapper/15 3%
swapper/23 3%
swapper/19 3%
swapper/12 3%
swapper/0 3%
swapper/13 3%
swapper/21 3%
swapper/16 3%
swapper/8 3%
swapper/5 3%
pdns 3%
...
```

while on a relatively busy system, we may get something like this:

```
$ cpu-hogs.sxx --arg time=30
Tracing the whole system...
Please wait for 30 seconds...

nginx-frontend 52%
nginx-backend 15%
nginx-sslgate 6%
nginx-waf 5%
ktserver 2%
pdns 2%
swapper/0 1%
ktfs 1%
syslog-ng 1%
fancy-scheduler 1%
rh-reader 0%
```

Please note that 52% here, for example, means 52% of all of the CPU time actually *used*, instead of the total CPU capacity. Do not get confused.

[Back to TOC](#table-of-contents)

cpu-robbers
-----------

Display how frequently other processes (including user proceses and kernel threads) preempt
the target process specified by its PID (that is, stealing CPU time from the target process).
Only ouptut the biggest robbers.

```
# making the ./stap++ tool visible in PATH:
$ export PATH=$PWD:$PATH

# assuming the target process's pid is 13443
$ ./samples/cpu-robbers.sxx -x 13443
Start tracing process 13443 (/opt/nginx/sbin/nginx)...
Hit Ctrl-C to end.
^C
#1 pdns: 23% (32 samples)
#2 ksoftirqd/17: 20% (28 samples)
#3 unbound: 8% (11 samples)
#4 nginx-sslgateway: 7% (10 samples)
#5 nginx-backend: 6% (9 samples)
#6 kworker/17:1: 5% (7 samples)
#7 kworker/17:1H: 3% (5 samples)
#8 rcuos/4: 2% (4 samples)
#9 rcuos/23: 2% (3 samples)
```

This tool is very general and can be used upon any process, *not*
need to be an nginx process at all.

[Back to TOC](#table-of-contents)

ngx-pcre-dist
--------
This tool can analyse the PCRE regex executation time distribution.

It requires uretprobes support in the Linux kernel.

Also, you need to ensure that debug symbols are enabled in your Nginx build, PCRE build, and LuaJIT build. For example, if you build PCRE from source with your Nginx or OpenResty by specifying the `--with-pcre=PATH` option, then you should also specify the `--with-pcre-opt=-g` option at the same time.

You can analyze the distribution of the length of those subject string data being matched in individual runs. Note that, the time is given in microseconds (us), i.e., 1e-6 seconds.

```
# making the ./stap++ tool visible in PATH:
$ export PATH=$PWD:$PATH

# assuming the target process has the pid 6440.
$ ./samples/ngx-pcre-dist.sxx -x 6440
Start tracing 6440 (/opt/nginx/sbin/nginx)
Hit Ctrl-C to end.
^C
Logarithmic histogram for data length distribution (byte) for 195 samples:
(min/avg/max: 7/65/90)
Logarithmic histogram for data length distribution:
value |-------------------------------------------------- count
    1 |                                                     0
    2 |                                                     0
    4 |@@@@@@                                              18
    8 |@@@@@@@                                             22
   16 |@@@@@@@@                                            24
   32 |                                                     0
   64 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@        131
  128 |                                                     0
  256 |                                                     0
```

Below is an example that analyzes the PCRE regex executation time distribution for a given Nginx worker process. The `--arg exec_time=1` option is used here.

```
# assuming the target process has the pid 6440.
$ ./samples/ngx-pcre-dist.sxx -x 6440 --arg exec_time=1
Start tracing 6440 (/opt/nginx/sbin/nginx)
Hit Ctrl-C to end.
^C
Logarithmic histogram for pcre_exec running time distribution (us) for 135 sample:
(min/avg/max: 11/18/164)
value |-------------------------------------------------- count
    2 |                                                    0
    4 |                                                    0
    8 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@   97
   16 |@@@@@@@@@@@                                        23
   32 |@@@@@@                                             12
   64 |@                                                   2
  128 |                                                    1
  256 |                                                    0
  512 |                                                    0
```
Using `--arg utime=1` option can increase the accuracy of time, but it may not be supported on some platforms.
This tool supports both the ngx_lua classic API and the lua-resty-core API.

[Back to TOC](#table-of-contents)

ngx-pcre-top
--------
This tool can analyze the worst or accumulated PCRE executation time of the individual regex matches using the ngx_lua module's [ngx.re API](http://wiki.nginx.org/HttpLuaModule#ngx.re.match).

Here is an example that analyzes the longest total running time, using accumulated regex execution time:

```
# making the ./stap++ tool visible in PATH:
$ export PATH=$PWD:$PATH

# assuming the target process has the pid 6440.
$ ./samples/ngx-pcre-top.sxx --skip-badvars -x 6440
Start tracing 6440 (/opt/nginx/sbin/nginx)...
 Hit Ctrl-C to end.
^C
Top N regexes with longest total running time:
1. pattern ".": 241038us (total data size: 330110)
2. pattern "elloA": 188107us (total data size: 15365120)
3. pattern "b": 28016us (total data size: 36012)
4. pattern "ello": 26241us (total data size: 15005)
5. pattern "a": 26180us (total data size: 36012o)
```

Below is an example that analyzes the worst execution time of the individual regex matches. The `--arg worst_time=1` option is used here.

```
# assuming the target process has the pid 6440.
$ ./samples/ngx-pcre-top.sxx --skip-badvars -x 6440 --arg worst_time=1
Start tracing 6440 (/opt/nginx/sbin/nginx)
Hit Ctrl-C to end.
^C
Top N regexes with worst running time:
1. pattern "elloA": 125us (data size: 5120)
2. pattern ".": 76us (data size: 10)
3. pattern "a": 64us (data size: 12)
4. pattern "b": 29us (data size: 12)
5. pattern "ello": 26us (data size: 5)
```
Note that the time values given above are just for individual runs and are not accumulated.
Using `--arg utime=1` option can increase the accuracy of time, but it may not be supported on some platforms.
This tool supports both the ngx_lua classic API and the lua-resty-core API.

[Back to TOC](#table-of-contents)

vfs-page-cache-misses
--------
This tool can analyze the page cache misses:

1. operation miss rate: operation num that page cache missed / total operation num.
2. size miss rate: size not from page cache / total size read.

This tool can be use for processes that have multiple thread (not only for nginx process),
but it will get the wrong result when the process have some other operators that can trigger `vfs.read` but not read data from disk, like pipeline file read.

The `--arg inode=$inode` option can be used to only analyze the specified inode, `$inode` is an inode ID that can get by `ls -i /path/to/file`.

Here is an example:

```
# making the ./stap++ tool visible in PATH:
$ export PATH=$PWD:$PATH

# assuming the target process has the pid 73485.
$ ./samples/vfs-page-cache-misses.sxx -x 73485 --arg inode=7905
Tracing 73485...
Hit Ctrl-C to end.

311 vfs.read operations, 86 missed operations, operation miss rate: 27%
total read 29425 KB, 3350 pages added (page size: 4096B), size miss rate: 45%
186 vfs.read operations, 49 missed operations, operation miss rate: 26%
total read 20373 KB, 1792 pages added (page size: 4096B), size miss rate: 35%
```

[Back to TOC](#table-of-contents)

Installation
============

You need a recent enough Linux kernel (like 3.5+) *or* a older kernel with the utrace patch applied (for Linux distributions in the RedHat family, like RHEL, CentOS, and Fedora, the utrace patch should be included in their older kernels by default).

You also need to install systemtap first. It is recommended to build it from the latest source. See this document for detailed instructions: http://openresty.org/#BuildSystemtap

[Back to TOC](#table-of-contents)

Author
======

Yichun Zhang (agentzh), <agentzh@gmail.com>, CloudFlare Inc.

[Back to TOC](#table-of-contents)

Copyright and License
=====================

This module is licensed under the BSD license.

Copyright (C) 2013-2016, by Yichun "agentzh" Zhang (章亦春) <agentzh@gmail.com>, CloudFlare Inc.

All rights reserved.

Redistribution and use in source and binary forms, with or without modification, are permitted provided that the following conditions are met:

* Redistributions of source code must retain the above copyright notice, this list of conditions and the following disclaimer.

* Redistributions in binary form must reproduce the above copyright notice, this list of conditions and the following disclaimer in the documentation and/or other materials provided with the distribution.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

[Back to TOC](#table-of-contents)

See Also
========
* SystemTap Wiki Home: http://sourceware.org/systemtap/wiki
* Nginx Systemtap Toolkit: https://github.com/agentzh/nginx-systemtap-toolkit

[Back to TOC](#table-of-contents)

