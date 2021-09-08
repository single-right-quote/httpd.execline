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

if you’ve ever used the publicfile `httpd`, then the setup is somewhat familiar:
`httpd.execline` expects to be run in a directory where there is a subdirectory
matching every hostname the dæmon serves requests for; it will simply mirror the
contents of every file in that subdirectory it is allowed to read.

in short: if `example.org` routed to your machine, then place a directory named
`example.org/` in `./visible-to-httpd/supported_domains/`.

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

#### `./visible-to-httpd/binaries` ###

`httpd.execline` normally chroots into the directory it runs from, making it
difficult to use dynamically linked versions of its hard dependencies. a
feasible configuration is to place statically linked dependencies into
`./binaries`:

+ [s6-portable-utils](https://skaret.org/software/s6-portable-utils/)
`s6-applyuidgid`, `s6-test`
+ [9base](https://tools.suckless.org/9base/):
`tr(1)` `read(1)`, `hoc(1)`, `sed(1)`, `grep(1)`, `urlencode(1)`,
`cleanname(1)`, `cat(1)` 
+ [toybox](http://www.landley.net/toybox/): `wc(1)`,
`date(1p)`, `printenv(1)`, `stat(1)`

we heavily rely on plan 9 regular expression semantics for `sed(1)` and
`grep(1)`; i expect translating them to coreutils or \*BSD userspace would be an
effort. so long as i am writing this code for myself, i will not perform that
effort for you.

<!-- an old version of this README explained that we use nonstandard functionality
from `s6-test`, but adjustments to the filesystem layout for configuration and
website layouts has rendered this moot. -->

### configuration ###

the directory `./visible-to-httpd/configuration` has several subdirectories for
configuring headers and error status behavior.

#### `./visible-to-httpd/configuration/Content-Type_table/` ###

a key-value store for associating extensions
with `Content-Type`s. for example, `data/Content-Type_table/html`
should probably contain the string `text/html`.

this feature can be overriden on a per-file basis in two ways, the
second overriding even the first.

1. giving a resource and extension of the form `${1}=${2}`; such files
will be served with a `Content-Type` of `{1}/${2}` (with colons in
`${1}` or `${2}` converted to periods). for example, a file named
`index.text=x:market` will always be served with a `Content-Type` of
`text/x.market`.
2. using the per-resource `overrides` folder (see below) to specify a
`Content-Type` header explicitly.

#### `./visible-to-httpd/configuration/default_headers/` ###

a collection of key-value stores to specify headers to send for all
resources associated with a particular hostname, as well as a
`-fallback` which takes effect if there is no store for that domain
(or that specific resource; see `overrides` below).

(there is presently no mechanism for combining a more specific match
for headers with a less specific one; you cannot combine unique
headers for a resource on `my-cool-web.site` with the default headers
for `my-cool-web.site, not to mention also with `-fallback`. but this
functionality is probably well worth adding.)

in subfolders matching hostnames, files named after a header should
contain the contents of that header. <!-- a personal site heavily
associated with a mastodon account would perhaps add a file
`X-Clacks-Overhead`, containing the contents `GNU Natalie Nguyen`. -->
a `Strict-Transport-Security` file is a good idea; if you find it
prudent to allow access as an onion service, an `Onion-Location` file
is a good idea. and so on.

`\r` and newlines will be stripped from filenames and file contents to
prevent trivial mischevious configurations from breaking HTTP
responses; other than this, **these HTTP header folders are not
validated syntactically or semantically**.

#### `./visible-to-httpd/configuration/error_response_pages/` ###

this directory may contain a subdirectory named after each hostname,
each containing subdirectories for a numerical HTTP status code, each
containing a required `message_body` file, an optional `Content-Type`
file (defaulting to `application/xhtml+xml; charset=utf-8`), and an
optional `headers` folder using the same scheme as the
`default_headers` header specifications.

for example: if you wanted to handle 404s at `my-cool-web.site` with
an HTML file, write the contents of said file at
`./visible-to-httpd/configuration/error_response_pages/404/my-cool-web.site/message_body`,
and place `text/html` in a file `Content-Type` in the same folder.

the error response code has a generic fallback built into the script.
you can override this using a `-fallback` domain folder, like with
domain-level `default_headers`.

#### `./visible-to-httpd/configuration/overrides/` ###

this directory allows you to override the extra headers sent along
with a resource, and attach a status code other than 200 with them. a
folder named after the specific resource (including a prepended
hostname) should may contain a `status_code` file containing a
numerical status code and optional textual message, as well as a
`headers` folder, which specifies headers using the `default_headers`
scheme.

a former official website for `httpd-execline.eerie.garden`
used to redirect to this github repository, thanks to

+ the file
`./visible-to-httpd/configuration/overrides/httpd-execline.eerie.garden/index.xhtml/status_code`
containing the text `301 moved permanently`; and
+ the file
`./visible-to-httpd/configuration/overrides/httpd-execline.eerie.garden/index.xhtml/headers/Location`
containing `https://github.com/single-right-quote/httpd.execline`
