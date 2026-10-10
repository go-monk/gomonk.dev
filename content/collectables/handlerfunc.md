+++
date = '2009-02-05'
author = 'Russ Cox'
title = 'HandlerFunc'
+++

A function type with a method that calls itself. Five lines let any plain function satisfy the `Handler` interface, with no wrapper struct in sight.

```go
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}
```

Source: [net/http in the Go standard library](https://pkg.go.dev/net/http#HandlerFunc)
