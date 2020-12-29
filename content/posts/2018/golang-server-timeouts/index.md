---
title: "Golang long-polling: a tale of server timeouts"
summary: Extending API response in a Go server by using struct embedding
date: 2018-07-29T00:00:00+02:00
tags:
- programming
- http
- golang
- long-polling
---

I recently had a week long battle implementing HTTP long-polling. As so often
happens in long debugging session in software development, my final fix was
one line!

<!--more-->

In modern web applications realtime live updates is becoming the norm. As such, I
recently had the pleasure of implementing long-polling in one of my servers.
There are a couple of ways a web app can get updates

1.  standard polling - where the web app makes a request on a standard interval,
    say, every second.
2.  long polling - the web app makes repeated requests but each request is long
    lived and changes are immediately pushed or the request times out and the app
    starts a new request
3.  http streaming - the web app makes a long lived request but the request is
    not closed and the server sends partial responses to the app as changes
    occur, using something like [ndjson](http://ndjson.org/).
4.  web sockets - again the web app makes a single long lived request that
    allows bidirectional communication.

There are various reasons to choose any of the options 2 -- 4, just never
choose number 1! In out use case, we wanted to add read-only realtime to existing REST
APIs, so long polling seemed like a natural and simple implementation.

Unfortunately, there doesn't seem to be any standard or spec for long polling. This means "roll your own!"
We settled on the follow request sequence
{{<image src="longpolling.jpg" title="Long Polling Sequence">}}

Note that the polling timeout results in a 304 response. We also use the prefer header for sending
the polling parameters

A very simple example that replicates this long polling sequence (but with GET parameters) looks like
this

```go
package main

import (
	"context"
	"fmt"
	"io/ioutil"
	"log"
	"math/rand"
	"net/http"
	"net/http/httptest"
	"time"
)

func getLongPollDuration(r *http.Request) time.Duration {
	timeout, err := time.ParseDuration(r.URL.Query().Get("wait"))
	if err != nil {
		return 15 * time.Second
	}

	log.Printf("found custom timeout: %s", timeout)
	return timeout
}

func getResource(ctx context.Context) string {
	return "{\"id\": 1, \"updatedAt\": \"" + time.Now().Format(time.RFC3339) + "\"}"
}

func waitForResource(ctx context.Context, wait time.Duration) string {

	// randomly wait up to 15 seconds for a "resource changed event"
	r := rand.Intn(15)
	ticker := time.Tick(time.Duration(r) * time.Second)
	waiter := time.Tick(wait)

	log.Printf("will wait up to %s for the resource", wait)

	select {
	case <-ctx.Done():
		log.Printf("Received context cancel")
		return ""
	case ts := <-waiter:
		log.Printf("Received method timeout: %s", ts)
		return ""
	case ts := <-ticker:
		log.Printf("Received resource update at: %s", ts)
		return "{\"id\": 1, \"updatedAt\": \"" + ts.Format(time.RFC3339) + "\"}"
	}
}

func resourceFunc(w http.ResponseWriter, r *http.Request) {
	index := r.URL.Query().Get("index")

	if index != "" {
		timeout := getLongPollDuration(r)
		response := waitForResource(r.Context(), timeout)
		if response == "" {
			// write long poll timeout
			w.WriteHeader(http.StatusNotModified)
		}
		fmt.Fprintf(w, response)
		return

	}

	response := getResource(r.Context())
	fmt.Fprintf(w, response)
}

func main() {
	ts := httptest.NewServer(http.HandlerFunc(resourceFunc))
	defer ts.Close()

	// you should always set these timeouts, otherwise requests
	// can never timeout
	ts.Config.ReadTimeout = 10 * time.Second
	ts.Config.WriteTimeout = 10 * time.Second

	res, err := http.Get(ts.URL + "?index=2&wait=15s")
	if err != nil {
		log.Fatal(err)
	}
	resourceResp, err := ioutil.ReadAll(res.Body)
	res.Body.Close()
	if err != nil {
		log.Fatal(err)
	}

	fmt.Printf("%s\n", res.Status)
	fmt.Printf("%s", resourceResp)
}
```

You can run this at https://play.golang.org/p/kXMm-mE-tgl

My troubles, when implementing this and then exposing it with an nginx proxy
was complicated by this line

```go
	ts.Config.WriteTimeout = 10 * time.Second
```

When we fist implemented the REST api, we set the default server timeouts to
10 seconds, but, when I started implementing the long polling, I set the
minimum wait to 15 seconds. This results in sporadic 502 errors from nginx.
Nginx was actually generating an error:

```
upstream prematurely closed connection while reading response header from upstream
```

Ultimately, the error message is correct and points to the exact issue (go
closing the request due to the timeout and then my handler trying to write to
it after it is closed), but the search results online were not super helpful.
Ultimately, it took me a little over a week of on-off debugging to track it
down to my own server config, doh! That `WriteTimeout` needs to be at least as
long as your maximum allowed long polling wait value, in our case `60s`.

Hopefully, anyone else that runs into that same nginx error will now double
check their own server timeouts too. Also, checkout [this blog post by CloudFlare](https://blog.cloudflare.com/the-complete-guide-to-golang-net-http-timeouts/)
that goes into detail about the various go server timeouts
