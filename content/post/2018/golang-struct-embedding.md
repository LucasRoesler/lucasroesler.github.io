---
title: Struct embedding for custom dev responses
summary: Extending API response in a Go server by using struct embedding
date: 2018-01-27T00:00:00-06:00
categories:
- programming
- golang
tags:
- golang
keywords:
- golang
comments: false
showPagination: false
autoThumbnailImage: true
---

I had to learn about Go struct embedding the other day and I wanted to document my
fix. In my use case I wanted to add some extra information to an API response to include
some meta data from kubernetes cluster and/or some extra error information.

<!--more-->

When I first implemented this particular API I was just learning Go and for simplicity
_to get things done_ I just used two structs.  For example:

```go
    dev := true
    type APIResponse struct {
        ID string `json:"id"`
        Name string `json:"name"`
    }

    type DevAPIResponse struct {
        ID string `json:"id"`
        Name string `json:"name"`
        DevData string `json:"devData"`
    }

    var payloadBytes []byte
    payload := APIResponse{ID: "1", Name: "sample"}
    if dev {
        devPayload := DevAPIResponse {
            ID: payload.ID,
            Name: payload.Name,
            DevData: "some special internal value, e.g. a k8s object id",
        }

        payloadBytes, _ = json.Marshal(devPayload)
    } else {
        payloadBytes, _ = json.Marshal(payload)
    }
    fmt.Printf("%s\n", payloadBytes)
```
You can run this at https://play.golang.com/p/WAT2OKkxVq-

But, I was recently refactoring the original code and I could not stand the idea of is basically
a duplicated struct and the need to keep them in sync.  I struggled to find a good example of
extending a struct like this.  Plenty of toy examples that involve Dogs or Balls, but no actual
practical  examples. So, I spent time to learn and play with Go struct embedding and ended up
with something that looks like this

```go
    dev := true
    type APIResponse struct {
        ID string `json:"id"`
        Name string `json:"name"`
    }

    var payloadBytes []byte
    payload := APIResponse{ID: "1", Name: "sample"}
    if dev {
        devPayload := struct {
            APIResponse  // embedded
            DevData  string `json:"devData"`
        }{
            DevData: "some special internal value, e.g. a k8s object id",
        }
        devPayload.APIResponse = payload

        payloadBytes, _ = json.Marshal(devPayload)
    } else {
        payloadBytes, _ = json.Marshal(payload)
    }

    fmt.Println(payloadBytes)
```
https://play.golang.org/p/V-qqsGazbAW

If this were an actual request handler, this would actually look like
```go
func APiHandler(w http.ResponseWriter, r *http.Request) {
    var payloadBytes []byte
    var err error
    payload := APIResponse{ID: "1", Name: "sample"}
    // dev is some global flag
    if dev {
        devPayload := struct {
            APIResponse  // embedded
            DevData  string `json:"devData"`
        }{
            DevData: "some special internal value, e.g. a k8s object id",
        }
        devPayload.APIResponse = payload

        payloadBytes, err = json.Marshal(devPayload)
    } else {
        payloadBytes, err = json.Marshal(payload)
    }

    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    w.Write(payloadBytes)
}
```

Hopefully someone else finds this useful.