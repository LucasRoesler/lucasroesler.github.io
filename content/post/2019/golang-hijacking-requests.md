---
title: "Trials and Tribulations when Hijacking HTTP Requests"
summary: ""
date: 2019-03-02T00:00:00+02:00
tags:
  - programming
  - golang
  - http
keywords:
  - golang
  - go
  - http
comments: false
showPagination: false
autoThumbnailImage: true
thumbnailImagePosition: left
coverSize: partial
coverImage: https://lh3.googleusercontent.com/hcegsfa76YgwX9Ba6ehkoSs91HMNwRqZUB9p5F4hpUMccXc2JzLEvLt0dUh2NTHW6A6FEbZKiQzZbFvtY88XFFQHToXuKAA53d2EtrvLjw5aX1V8mVnbdqnUrfTlGpaia0OcpvTeyVU=w2400
draft: true
---

Over the weekend I was working on an upcoming feature for OpenFaaS, streaming logs for your functions! I want the implementation to be as simple and approachable for other developers as possible. After considering various new "hip" ways to send an event stream ([server sent events][server-sent-events-mdn], [websockets][websockets-mdn], [HTTP2 push][http2-push-events-mdn]) I settled on plain HTTP using [`Keep-Alive`][keep-alive-mdn] and [`Chunked` ecncoding][chunked-encoding-mdn].

Chunked HTTP responses are a while established and supported technique. All browsers and HTTP clients need to support `Chunked` responses, otherwise they can't download large files! Unfortunately, implementing chunked responses was not as simple as I originally thought, leading me down dark and dangerous back alleys of the Go `http` and `net` packages.

<!--more-->

## Background

Both Swarm and Kubernetes expose log endpoints, so you can always access to the function logs using a command like `kubectl -n openfaas-fn logs -l "faas_function=FUNCTION-NAMED-FOO" -f` or `docker service logs FUNCTION-NAMED-FOO -f`. But when you use a FaaS system like OpenFaaS, you don't want to interact with the underly orchestration. Or, if you are the admin of the cluster, you might not want to give direct access to the cluster. Instead, we want to provide a command like `faas-cli logs FUNCTION-NAMED-FOO`.

## Basic handler

https://golang.org/pkg/net/http/#Flusher
https://golang.org/pkg/net/http/#CloseNotifier

## Timeout Woes

https://golang.org/pkg/net/http/#Server

## Hijacking

https://golang.org/pkg/net/http/#Hijacker
https://golang.org/pkg/net/#Conn
https://golang.org/pkg/net/http/httputil/#NewChunkedWriter

## Closing connections

<!-- try to write the required closing newliens for chunked encoding
see the RFC https://tools.ietf.org/html/rfc7230#section-4.1 specification of
the last chunk `last-chunk     = 1*("0") [ chunk-ext ] CRLF` -->

```go
// closeNotify will watch the connection and notify when then connection is closed
func closeNotify(ctx context.Context, c net.Conn) <-chan error {
	notify := make(chan error)

	go func() {
		buf := make([]byte, 1024)
		n, err := c.Read(buf) // blocks until non-zero read or error
		if err != nil {
			log.Printf("LogHandler: test connection: %s\n", err)
			notify <- err
			return
		}
		if n > 0 {
			log.Printf("LogHandler: unexpected data: %s\n", buf[:n])
			return
		}
	}()
	return notify
}
```

## Wrapping up

[server-sent-events-mdn]: https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events
[websockets-mdn]: https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API
[http2-push-events-mdn]: https://developer.mozilla.org/en-US/docs/Web/API/Push_API
[chunked-encoding-mdn]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Transfer-Encoding
[keep-alive-mdn]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Keep-Alive
