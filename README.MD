# Motivation
This is a test project to illustrate the issue with Play 2.6 with request headers when certain 
`Content-Encoding` is used in the request.

## Issue summary
Most of request headers are dropped when `Content-Encoding` header with values `gzip` or `deflate` is sent 
with the request.

## Details
With a default Play 2.6.6 setup, most of the request headers are not available in the actual 
`play.api.mvc.RequestHeader.headers` instance inside the controller when the compressed `Content-Encoding` 
header is sent. Here's an example:

When the controller looks like this:

```scala
@Singleton
class HomeController @Inject()(cc: ControllerComponents) extends AbstractController(cc) {
  def testGzip() = Action { implicit request: Request[AnyContent] =>
    Ok(request.headers.toString)
  }
}
```

And the following curl command is executed against a Play instance running at `http://localhost:9000`:

```bash
curl -X POST http://localhost:9000 \
-H 'Content-Encoding: gzip' \
-H 'Authorization: Bearer 123' \
-H 'X-Custom-Header: 123' -vvv
* Rebuilt URL to: http://localhost:9000/
*   Trying ::1...
* TCP_NODELAY set
* Connected to localhost (::1) port 9000 (#0)
> POST / HTTP/1.1
> Host: localhost:9000
> User-Agent: curl/7.54.0
> Accept: */*
> Authorization: Bearer 123
> Content-Encoding: gzip
> X-Custom-Header: 123
>
< HTTP/1.1 200 OK
< Referrer-Policy: origin-when-cross-origin, strict-origin-when-cross-origin
< X-Frame-Options: DENY
< X-XSS-Protection: 1; mode=block
< X-Content-Type-Options: nosniff
< Content-Security-Policy: default-src 'self'
< X-Permitted-Cross-Domain-Policies: master-only
< Date: Tue, 10 Oct 2017 12:02:09 GMT
< Content-Type: text/plain; charset=UTF-8
< Content-Length: 84
<
* Connection #0 to host localhost left intact
List((Transfer-Encoding,chunked), (Content-Type,none/none), (Content-Encoding,gzip))
```

Then you can see that headers `Authorization` and `X-Custom-Header` are missing from the 
`request` object.

To be clear, when the `Content-Encoding: gzip` is specified, no matter if the body of the request
actually include something gzipped, the situation with headers remain the same. It also doesn't 
matter if the `Content-Type` is correctly specified or not. Also to be clear,
Play can decompress the gzipped content when this Content-Encoding is specified, so that part
is fine. The problem is that headers are missing.

P.S. also worth noting that `Transfer-Encoding: chunked` is somehow inferred by Play, even though
it is not sent by the client.