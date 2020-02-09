# HTTP support

The hub supporting [ALPN](../alpn.md) and/or [protocol detection](../protocol-detection.md)
should also serve an HTTP endpoint for [pingers](./ping.md), but may also serve other
HTTP resources.

## Redirect

The most simple case is to redirect all requests (except ones served by [ping API](./ping.md))
to an external HTTP server, for example the hub's website.

In this case the should return `307` (temporary redirect) HTTP status code, and specify the
website address in the `location` HTTP header.

## Serving the website

The hub may also embed a full web server implementation, or act like a reverse proxy. The
hub should still serve the [ping API](./ping.md) in this case and may use other endpoints
to serve any other data.

The most common use cases might be:
- Hub status page
- Hub website
- DC web client
