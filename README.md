# HttpRouter [![Build Status](https://travis-ci.org/spuranam/httprouter.png?branch=master)](https://travis-ci.org/spuranam/httprouter) [![Coverage](http://gocover.io/_badge/github.com/spuranam/httprouter?0)](http://gocover.io/github.com/spuranam/httprouter) [![GoDoc](http://godoc.org/github.com/spuranam/httprouter?status.png)](http://godoc.org/github.com/spuranam/httprouter)

HttpRouter is a lightweight high performance HTTP request router
(also called *multiplexer* or just *mux* for short) for [Go](http://golang.org/).

In contrast to the [default mux](http://golang.org/pkg/net/http/#ServeMux) of Go's net/http package, this router supports
variables in the routing pattern and matches against the request method.
It also scales better.

The router is optimized for high performance and a small memory footprint.
It scales well even with very long paths and a large number of routes.
A compressing dynamic trie (radix tree) structure is used for efficient matching.

## Features
**Only explicit matches:** With other routers, like [http.ServeMux](http://golang.org/pkg/net/http/#ServeMux),
a requested URL path could match multiple patterns. Therefore they have some
awkward pattern priority rules, like *longest match* or *first registered,
first matched*. By design of this router, a request can only match exactly one
or no route. As a result, there are also no unintended matches, which makes it
great for SEO and improves the user experience.

**Stop caring about trailing slashes:** Choose the URL style you like, the
router automatically redirects the client if a trailing slash is missing or if
there is one extra. Of course it only does so, if the new path has a handler.
If you don't like it, you can [turn off this behavior](http://godoc.org/github.com/spuranam/httprouter#Router.RedirectTrailingSlash).

**Path auto-correction:** Besides detecting the missing or additional trailing
slash at no extra cost, the router can also fix wrong cases and remove
superfluous path elements (like `../` or `//`).
Is [CAPTAIN CAPS LOCK](http://www.urbandictionary.com/define.php?term=Captain+Caps+Lock) one of your users?
HttpRouter can help him by making a case-insensitive look-up and redirecting him
to the correct URL.

**Parameters in your routing pattern:** Stop parsing the requested URL path,
just give the path segment a name and the router delivers the dynamic value to
you. Because of the design of the router, path parameters are very cheap.

**Zero Garbage:** The matching and dispatching process generates zero bytes of
garbage. In fact, the only heap allocations that are made, is by building the
slice of the key-value pairs for path parameters. If the request path contains
no parameters, not a single heap allocation is necessary.

**Best Performance:** [Benchmarks speak for themselves](https://github.com/spuranam/go-http-routing-benchmark).
See below for technical details of the implementation.

**No more server crashes:** You can set a [Panic handler](http://godoc.org/github.com/spuranam/httprouter#Router.PanicHandler) to deal with panics
occurring during handling a HTTP request. The router then recovers and lets the
PanicHandler log what happened and deliver a nice error page.

Of course you can also set **custom [NotFound](http://godoc.org/github.com/spuranam/httprouter#Router.NotFound) and  [MethodNotAllowed](http://godoc.org/github.com/spuranam/httprouter#Router.MethodNotAllowed) handlers** and [**serve static files**](http://godoc.org/github.com/spuranam/httprouter#Router.ServeFiles).

## Usage
This is just a quick introduction, view the [GoDoc](http://godoc.org/github.com/spuranam/httprouter) for details.

Let's start with a trivial example:
```go
package main

import (
    "fmt"
    "github.com/spuranam/httprouter"
    "net/http"
    "log"
)

func Index(w http.ResponseWriter, r *http.Request) {
    fmt.Fprint(w, "Welcome!\n")
}

func Hello(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "hello, %s!\n", r.URL.Query().Get(":name"))
}

func main() {
    router := httprouter.New()
    router.GET("/", Index)
    router.GET("/hello/:name", Hello)

    log.Fatal(http.ListenAndServe(":8080", router))
}
```

### Named parameters
As you can see, `:name` is a *named parameter*.
The values are accessible via `http.Request.URL`.
You can get the value of a parameter by using the r.URL.Query().Get(":name") method

Named parameters only match a single path segment:
```
Pattern: /user/:user

 /user/gordon              match
 /user/you                 match
 /user/gordon/profile      no match
 /user/                    no match
```

**Note:** Since this router has only explicit matches, you can not register static routes and parameters for the same path segment. For example you can not register the patterns `/user/new` and `/user/:user` for the same request method at the same time. The routing of different request methods is independent from each other.

### Catch-All parameters
The second type are *catch-all* parameters and have the form `*name`.
Like the name suggests, they match everything.
Therefore they must always be at the **end** of the pattern:
```
Pattern: /src/*filepath

 /src/                     match
 /src/somefile.go          match
 /src/subdir/somefile.go   match
```

## How does it work?
The router relies on a tree structure which makes heavy use of *common prefixes*,
it is basically a *compact* [*prefix tree*](http://en.wikipedia.org/wiki/Trie)
(or just [*Radix tree*](http://en.wikipedia.org/wiki/Radix_tree)).
Nodes with a common prefix also share a common parent. Here is a short example
what the routing tree for the `GET` request method could look like:

```
Priority   Path             Handle
9          \                *<1>
3          ├s               nil
2          |├earch\         *<2>
1          |└upport\        *<3>
2          ├blog\           *<4>
1          |    └:post      nil
1          |         └\     *<5>
2          ├about-us\       *<6>
1          |        └team\  *<7>
1          └contact\        *<8>
```
Every `*<num>` represents the memory address of a handler function (a pointer).
If you follow a path trough the tree from the root to the leaf, you get the
complete route path, e.g `\blog\:post\`, where `:post` is just a placeholder
([*parameter*](#named-parameters)) for an actual post name. Unlike hash-maps, a
tree structure also allows us to use dynamic parts like the `:post` parameter,
since we actually match against the routing patterns instead of just comparing
hashes. [As benchmarks show](https://github.com/spuranam/go-http-routing-benchmark),
this works very well and efficient.

Since URL paths have a hierarchical structure and make use only of a limited set
of characters (byte values), it is very likely that there are a lot of common
prefixes. This allows us to easily reduce the routing into ever smaller problems.
Moreover the router manages a separate tree for every request method.
For one thing it is more space efficient than holding a method->handle map in
every single node, for another thing is also allows us to greatly reduce the
routing problem before even starting the look-up in the prefix-tree.

For even better scalability, the child nodes on each tree level are ordered by
priority, where the priority is just the number of handles registered in sub
nodes (children, grandchildren, and so on..).
This helps in two ways:

1. Nodes which are part of the most routing paths are evaluated first. This
helps to make as much routes as possible to be reachable as fast as possible.
2. It is some sort of cost compensation. The longest reachable path (highest
cost) can always be evaluated first. The following scheme visualizes the tree
structure. Nodes are evaluated from top to bottom and from left to right.

```
├------------
├---------
├-----
├----
├--
├--
└-
```

## Where can I find Middleware *X*?
This package just provides a very efficient request router with a few extra
features. The router is just a [http.Handler](http://golang.org/pkg/net/http/#Handler),
you can chain any http.Handler compatible middleware before the router,
for example the [Gorilla handlers](http://www.gorillatoolkit.org/pkg/handlers).
Or you could [just write your own](http://justinas.org/writing-http-middleware-in-go/),
it's very easy!

### Multi-domain / Sub-domains
Here is a quick example: Does your server serve multiple domains / hosts?
You want to use sub-domains?
Define a router per host!
```go
// We need an object that implements the http.Handler interface.
// Therefore we need a type for which we implement the ServeHTTP method.
// We just use a map here, in which we map host names (with port) to http.Handlers
type HostSwitch map[string]http.Handler

// Implement the ServerHTTP method on our new type
func (hs HostSwitch) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	// Check if a http.Handler is registered for the given host.
	// If yes, use it to handle the request.
	if handler := hs[r.Host]; handler != nil {
		handler.ServeHTTP(w, r)
	} else {
		// Handle host names for wich no handler is registered
		http.Error(w, "Forbidden", 403) // Or Redirect?
	}
}

func main() {
	// Initialize a router as usual
	router := httprouter.New()
	router.GET("/", Index)
	router.GET("/hello/:name", Hello)

	// Make a new HostSwitch and insert the router (our http handler)
	// for example.com and port 12345
	hs := make(HostSwitch)
	hs["example.com:12345"] = router

	// Use the HostSwitch to listen and serve on port 12345
	log.Fatal(http.ListenAndServe(":12345", hs))
}
```

### Basic Authentication
Another quick example: Basic Authentification (RFC 2617) for handles:

```go
package main

import (
	"bytes"
	"encoding/base64"
	"fmt"
	"log"
	"net/http"
	"strings"

	"github.com/spuranam/httprouter"
)

func BasicAuth(h httprouter.Handle, user, pass []byte) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		const basicAuthPrefix string = "Basic "

		// Get the Basic Authentication credentials
		auth := r.Header.Get("Authorization")
		if strings.HasPrefix(auth, basicAuthPrefix) {
			// Check credentials
			payload, err := base64.StdEncoding.DecodeString(auth[len(basicAuthPrefix):])
			if err == nil {
				pair := bytes.SplitN(payload, []byte(":"), 2)
				if len(pair) == 2 &&
					bytes.Equal(pair[0], user) &&
					bytes.Equal(pair[1], pass) {

					// Delegate request to the given handle
					h(w, r)
					return
				}
			}
		}

		// Request Basic Authentication otherwise
		w.Header().Set("WWW-Authenticate", "Basic realm=Restricted")
		http.Error(w, http.StatusText(http.StatusUnauthorized), http.StatusUnauthorized)
	}
}

func Index(w http.ResponseWriter, r *http.Request) {
	fmt.Fprint(w, "Not protected!\n")
}

func Protected(w http.ResponseWriter, r *http.Request) {
	fmt.Fprint(w, "Protected!\n")
}

func main() {
	user := []byte("gordon")
	pass := []byte("secret!")

	router := httprouter.New()
	router.GET("/", Index)
	router.GET("/protected/", BasicAuth(Protected, user, pass))

	log.Fatal(http.ListenAndServe(":8080", router))
}
```

## Chaining with the NotFound handler

**NOTE: It might be required to set [Router.HandleMethodNotAllowed](http://godoc.org/github.com/spuranam/httprouter#Router.HandleMethodNotAllowed) to `false` to avoid problems.**

You can use another [http.Handler](http://golang.org/pkg/net/http/#Handler), for example another router, to handle requests which could not be matched by this router by using the [Router.NotFound](http://godoc.org/github.com/spuranam/httprouter#Router.NotFound) handler. This allows chaining.

### Static files
The `NotFound` handler can for example be used to serve static files from the root path `/` (like an index.html file along with other assets):
```go
// Serve static files from the ./public directory
router.NotFound = http.FileServer(http.Dir("public")).ServeHTTP
```

But this approach sidesteps the strict core rules of this router to avoid routing problems. A cleaner approach is to use a distinct sub-path for serving files, like `/static/*filepath` or `/files/*filepath`.
