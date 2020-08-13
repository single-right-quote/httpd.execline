# httpd.execline: a simple\* static webserver ###

`httpd.execline` performs the business logic of a a static HTTP mirror. it is
implemented in [execline](https://skarnet.org/software/execline/), in the same
sense that you could implement the business logic of a static HTTP server in
POSIX `sh(1)`, by wrangling Unix tools together which will actually perform the
useful tasks you want to get done. (the advantage of POSIX `sh(1)` for this job
is that it is far less verbose.)

it takes a lot of inspiration from
[publicfile](https://cr.yp.to/publicfile.html), while trying to allow some level
of customization (custom error status pages, custom HTTP headers,
file-extension/MIME-type mapping adjustments) without requiring you to edit
code; here we use using a filesystem-driven configuration where the hierarchical
file structure amounts to a simple structured key-value store.

\* “simple” here better describes functionality than implementation.

## usage ###

if you’ve ever used the publicfile `httpd`, then the setup is familiar:
`httpd.execline` expects to be run in a directory where there is a subdirectory
matching every hostname the dæmon serves requests for; it will simply mirror the
contents of every file in that subdirectory it is allowed to read.

in short: if `example.org` routed to your machine, then place a directory named
`./example.org` in the directory you’re running `./httpd.execline` from
(normally, this one).

(you should consider ensuring `httpd.execline` not have any write permissions
for the hostname-directories and their contents.)

if you’re using `daemontools`-style process supervision (runit, daemontools, s6,
or the like), *and* you already have all the dependencies (see below), including
statically linked binaries in `./binaries` (see below), then adjust
paramaterized values in `./run.template` and rename it to `./run`, and drop this
directory into wherever your process supervision suite is looking for service
directories. (if you&#8217;re not using `s6`, you should replace `s6-log` in
`./log/run.)

i haven’t used `systemd` for years, and as such, haven’t gotten around to
writing an equivalent unit file yet.

### dependencies ###

you will need a superserver to actually perform any networking; i use
[`s6-tlsserver`](https://skarnet.org/software/s6-networking/s6-tlsserver.html)
(which itself uses
[`s6-tcpserver`](https://skarnet.org/software/s6-networking/s6-tcpserver.html),
which you could use if you *don’t* need TLS), from
[`s6-networking`](https://skarnet.org/software/s6-networking/).

furthermore, we assume your kernel supports `chroot`, and that you have
userspace-level access to the feature, like GNU coreutils `chroot(1)`.

#### `./binaries` ###

`httpd.execline` normally chroots into the directory it runs from, making it
difficult to use dynamically linked versions of its hard dependencies. a
feasible configuration is to place statically linked dependencies into
`./binaries`:

+ [s6-portable-utils](https://skaret.org/software/s6-portable-utils/)
`s6-applyuidgid`, `s6-test`
+ [9base](https://tools.suckless.org/9base/):
`tr(1)` `read(1)`, `hoc(1)`, `sed(1)`, `grep(1)`, `urlencode(1)`,
`cleanname(1)`, `cat(1)` + [toybox](http://www.landley.net/toybox/): `wc(1)`,
`date(1p)`, `printenv(1)`, `stat(1)`

we heavily rely on plan 9 regular expression semantics for `sed(1)` and
`grep(1)`; i expect translating them to coreutils or \*BSD userspace would be an
effort. so long as i am writing this code for myself, i will not perform that
effort for you.

i would like to note that **s6-test receives information controlled by the
client** and is thus **difficult to replace with a different take on
`test(1p)`**; the use of `s6-test` here relies on the (non-standard! but very
useful) functionality that an argument escaped with an initial backslash is
never interpreted as an option to the program. without s6-test, handling
user-controlled input *robustly* probably requires a workaround (piping into
`grep -s`, perhaps?)

### additional, somewhat esoteric, functionality ###

#### `Content-Type`s ###

`httpd.execline` expects to see a subdirectory `./data/Content-Type_table`,
where files named after file extensions contain the MIME type such files should
be served as. for example, `data/Content-Type_table/html` should probably
contain the string `text/html`.

this feature can be overriden on a per-file basis by making its extension have
the form `${1}=${2}`; such files will be served with a `Content-Type` of
`{1}/${2}` (with colons in `${1}` or `${2}` converted to periods). (for example,
a file named `index.text=x:market` will always be served with a `Content-Type`
of `text/x.market`.)

if no `Content-Type` can be determined, `httpd.execline` falls back on
`application/octet-stream`.

#### HTTP headers ###

`httpd.execline` expects `./data/` to have another subdirectory named
`extra_headers`; and a file inside it named `default`, which may contain a
series of `\r\n`-terminated HTTP-heades inside. `default` can be overriden on a
per-file basis as follows:

say you have a client-side webapp at `${YOUR_SITE_HERE}/webapp/index.html` and
you need a Content Security Policy that differs from the one specified in
`./data/extra_headers/default`; create a file named
`./data/extra_headers/${YOUR_SITE_HERE}/webapp/index.html` containing
`\r\n`-separated headers as necessary.

the UI for this is not convenient.

#### HTTP status codes ###

a subdirectory of `./data/` named `status_override` can override HTTP status
codes on a per-file basis the same way you can override HTTP headers. i use this
for 301 redirects.

again, the interface to this feature in inconvenient.

## implementatoin details ###

### the subscripts ###

as mentioned, this script relies on several smaller subscripts, themselves often
depedent on other subscripts. we list all subscripts in the implementation
below, along with their dependencies:

+ `./get-line-from-client.execline`: read a line from the client, timing out
after 60 seconds + `./log.execline`: log, adding information useful for
debugging + `./http-error-resonse.execline`: send an http resposne indicating
error, halt(, and, optionally, log that we errored)
+ `./http-start-line-parse.execline`: parse the start line and export its
components into the environment + `./http-header-parse.execline`: parse the
headers and export them into the environment
+ `./supported-hostname-test.execline`: test if the first argument is a supported
hostname, to signal whether to short-circuiting during header parsing

`./http-error-response.execline` depends on:

+ `./log.execline`

`./http-start-line-parse.execline` depends on:

+ `./get-line-from-client.execline` + `./http-error-response.execline`: and thus
+ `./log.execline`

`./http-header-parse.execline` depends on:

+ `./get-line-from-client.execline` + `./http-error-response.execline`: and thus
+ `./log.execline`

`./supported-hostname-test.execline` depends on:

+ `./http-error-response.execline`: and thus + `./log.execline`

looking at the dependencies, we observe that `get-line-from-client.execline` and
`http-error-response.execline` are fundamental building blocks for the rest of
the script. it seems worth considering consolidating the logger into the error
response script.
