# Caching

Caching can be streamlined by the addition of cache control headers in HTTP responses.

Servers SHOULD add cache control headers to their responses and MAY respect cache control
headers that they receive.

A summary of cache control headers can be found [here](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control).

Implementations SHOULD (as much as possible) be [RFC compliant](https://httpwg.org/specs/rfc7234.html). However
this specification outlines a very restricted subset of this RFC that groups can implement without needing
a library.

## Basic Implementation Using Cache Control Headers

### HTTP Server

Add `Cache-Control: max-age=<seconds>` to HTTP responses, where `<seconds>` is how long the client should
cache the resourse for.

To prevent a resource bein cached, set `<seconds>` to `0`.

*A naive implementation might use a constant value, whereas a smarter implementation
might change the value, depending on how likely the resource is to change.*

### HTTP Client

Cache resources for the number of seconds specified in `max-age`. After this time, fetch a new
copy. 
