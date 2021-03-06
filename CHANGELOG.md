# Change Log

A human readable change log between our released versions can be found in here.

## v0.1.15 - 2016-05-27

### Bug fixes

* Fix panicing on serving cached zero sized files
* Fix from 'github.com/MStoykov/mp4' for very rare index out of bounds

## v0.1.14 - 2016-01-14

### Bug fixes

* Properly close log files. Previous versions were leaking open files in certain situations. It was happening after configuration reload in cases when the log file has been renamed, moved or removed.

## v0.1.13 - 2015-12-28

### Bug fixes

* Fix parsing pid files with whitespace around the actual pid.

* Handler Proxy was balancing on the path + query which will not work well with consistent hashing. It`s now only on path.

### Development

* Now the git vars in the MakeFile can be overwritten with env variables for build where git is not present.

## v0.1.12 - 2015-12-14

### Configuration

* `io_transfer_size` configuration directive is split into two new directives: `max_io_transfer_size` and `min_io_transfer_size`. The old one is removed. Generally, nedomi will try to read/write with `max_io_transfer_size` bytes per operation. But in cases when a connection is limit rated (a.k.a. throttled) the `min_io_transfer_size` bytes will be the lower bound of bytes written per operation before sleeping. Lower values of `min_io_transfer_size` will result in finer limit rating for the cost of more CPU and IO load. The default values are: `max_io_transfer_size` - 1m, `min_io_transfer_size` - 128k.

## v0.1.11 - 2015-12-10

### New Stuff

* Optimizations which reduce memory allocations. This results in much less garbage collection.

* An common API for limit rating a connection. It can be used by any module/handler. See the next point for the first one. This API exposes an efficient method for maintaining a constant speed for the downstream connection. The main improvement is that this method does not result in excesive `os.Stat` calls as was the previous.

* The throttling handle is rewritten and uses the new API.

## v0.1.10 - 2015-12-04

### New Stuff

* Access logs will now identify the virtual host for which the request was made by the `Host` header sent by the client. Previously it was always identified by its primary name. That way information for the request was lost. For unknown virtual hosts or virtual hosts which do not match any location, the `[unknown-location]` string is appended to the virtual host identification.

### Bug fixes

* Upstream basic authentication challenges are no longer passed down to the client.

## v0.1.9 - 2015-12-01

### New Stuff

* Upstream addresses now support HTTP Basic Authenticate. This is achieved by adding the user and password to the upstream URL. Example: "http://user@password:domain.com" 

### Bug fixes

* Served bytes were not correctly logged in the access log for requests which were satisfied using `sendfile` and friends.

### Development

* All logs for particular request now have an unique Request ID printed alongside. Previously the `http.Request` memory address was used. This was causing different requests to use the same ID since `http.Request` objects are reused.

### Removed

* The `via` handler is removed. The same thing can be achieved with the `headers` handler and its `add_headers` directive.

## v0.1.8 - 2015-11-18

### New Stuff

* The `io_transfer_size` directive is introduced. It can control the size of all IO operations done by nedomi. Basically, this is the size used in `sendfile`, `write` and `read` network calls. Previously this value was hard coded to 128k and this is now the default.

* As a performance improvements, some checks are removed when serving files from cache. Most notably, now nedomi does not check if the size of the stored object match the configured one. Tampering with the cache files can now result in weird behaviour.

### Bug fixes

* Retrying an upstream was causing a goroutine leaks because the initial upstream request was not cleaned up correctly.

* Reloading the cache from storage was not working when `skip_cache_key_in_path` is used.

* Making a cache zone smaller via reload of the configuration was not removing all discarded file objects from the cache algorithm memory properly.

* Upstream requests will use proper `Host` header without any configuration. There was an unintended behaviour where this header was set to the IP address of the upstream.

* Configuration reloading had no effect on `write_timeout` and `read_timeout` directives. They are now properly set.

* There was a bug where in some situations connections would time out server side earlier than configured.

### Development

* Some error messages have their priority lowered. Logging only errors will result in much small log files now.

## v0.1.7 - 2015-11-11

### Bug fixes

* A major bug in retrying with different upstream (introduced in v0.1.6) is fixed. It was causing a crash. The bug manifested itself when the configured upstream for retrying was not found.

* Cache zones in the status page are now sorted. Previously every refresh was causing a reordering of the zones.

## v0.1.6 - 2015-11-10

### New Stuff

* Further extend the efficient buffer copying mechanism from v0.1.5. It now works in tandem with the flv handler and the rewrite headers handler.

* Upstream retrying in the proxy handler is now possible. A handler can now define a new upstream to be retried via the `try_other_upstream_on_code` setting. Using it certain HTTP error codes from the first upstream can be configured to cause an additional request (retry) from a second upstream.

## v0.1.5 - 2015-11-10

### New Stuff

* Nedomi can now utilise more efficient OS syscalls for copying data in some situations. For example, with cached files in Linux the `sendfile` syscall will be used to bridge the opened file and network socket directly in kernel space. For other systems, their respectful method for efficient copying will be used. This behaviour is made possible by the golang's standard library.

## v0.1.4 - 2015-11-09

### New Stuff

* Updated the [mp4 lib](https://github.com/MStoykov/mp4) used for pseudo streaming to its version 0.1.3. The previous version had a bug which was leading to permanently blocked go routines.

## v0.1.3 - 2015-11-05

### Bug fixes

* Read and write time outs (as set by `read_timeout` and `write_timeout`) were actually deadlines. Requests were expected to finish *all* of their reading and writing in the specified number of seconds. This is now changed to actual time outs on the connection. A time out will happen only if the connection is actually stalled. Reading or writing a single byte every `read/write_timeout` period means the connection will not be marked as stalled.

* Pulls a bug fix from the upstream mp4 library. In some situations the atom `udta' was not parsed correctly. The effect of this prematurely terminated request. 

## v0.1.2 - 2015-11-03

### New Stuff

* The `skip_cache_key_in_path` option is added for a cache zone. By default it is `false` and the old behaviour for cache paths is used: inside the cache directory there will be one directory for every cache key used in this cache zone. When set to `true` no "cache key directory" will be used and all cached files will be rooted in the cache zone directory.

* The `ketama` upstream balancing algorithm is renamed to `legacyketama`. We are planning a better implementation of the algorithm but this one is still in use in some installations and must be kept for backward interoperability. **Important!** This change requires a change in the configuration file between v0.1.(0|1) and this one: `ketama` must become `legacyketama`.

* New option `upstream_hash_prefix` in the proxy handler. It is a string which will be used in front of the URIs as a key for the consistent hashing upstream balancing algorithms.

### Removed

* The `jump` upstream balancing algorithm is removed. It turns out it is not suitable for our purposes at all.

### Bug fixes

* Numerous bug fixes in the balancing algorithms in corner cases. They all use a thread-safe random number generator now.

### Development

* The severity of many error messages is lowered to info or debug.

## v0.1.1 - 2015-10-29

### New Stuff

 * More useful error messages on wrong syntax in the config file. An error will now be pointed out and some context will be printed.
 * All access log lines now contain identification for which vhost they refer to.
 * The purge module uses HTTP return code to signal that a purge was now unsuccessful for some reason.

### Bug fixes

 * The combination between "Date" and "Cache-Control: max-age" response headers is more RFC compliant. The Date will be stored as-is from the origin server and max-age will be calculated as time after this Date.
 * A bug in resizing down a cache zone with reload (`kill -HUP`) made it possible for this zone to be larger than expected. In some situations it resulted in crashes.

### Development

 * Starting with this one all release branches will have their dependencies vendored. For this end the `GO15VENDOREXPERIMENT` is used and it can be seen in the Makefile.
 * Support for `go1.4` is dropped.

## v0.1.0 - 2015-10-26

 * Initial Release
