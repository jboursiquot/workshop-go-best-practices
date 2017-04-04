# Go Best Practices Workshop | BaltimoreGolang

This material covers community-adopted best practices for Go development. Each section has bulletted summaries and code samples where appropriate. During the workshop, we'll look at a real-world codebase to see how some of these techniques are applied.

Last updated 4/4/17.

## Development Environment

### GOPATH

1. Use a per-project $GOPATH if you want tight control over the packages used in building your binaries.
2. Use the global $GOPATH (optionally) when building libraries since you typically want to avoid vendoring dependencies in library projects.
3. You can use a two-entry GOPATH (e.g. `$HOME/go/thirdparty:$HOME/go/internal`) which provides the following benefits:
-- Isolation of third-party libraries since `go get` and `go install` will work with the first entry.
-- Allows for internal/organizational packages
4. Ensure your `$GOPATH/bin` is in your `$PATH` to make it easier for binaries installed using `go install` to be found.
5. Prefer `go install` to `go build` as the former will cache build artifacts in `$GOPATH/pkg`.

> Re global vs per-project GOPATH, I always use the per-project approach and have had fewer problems that way.

### Vendoring

While we wait for the official `go dep` tool from the Go team, there are several community options you can try:

- `glide` (glide.sh) is similar to `npm` or `bundler` in usage if you're confortable with those tools already.
- `govendor`
- `gvt`


## Format & Style

1. Configure your editor or IDE to run `gofmt` or `goimports` on Save. Non-gofmt code is rare in the community.
2. Read and abide by the [Code Review Comments](https://github.com/golang/go/wiki/CodeReviewComments) the Go team advocate at a bare minimum.
3. Naming is hard sometimes. Use Andrew Gerrand's [idiomatic naming conventions](https://talks.golang.org/2014/names.slide) as a guide.
4. Package naming [conventions](https://blog.golang.org/package-names) are also available.
5. Lint your code using the superb [gometalinter](https://github.com/alecthomas/gometalinter) library.

## Configuration

1. The [12-factor app](https://12factor.net/) style of configuration is still very much relevant today and uses environment variables to provide configuration parameters to your program.
2. Allow your application to use both env vars as well as matching flags when launched.
3. For binaries, define and read your flags at the entry point of your program, func `main`.
4. For libraries/packages meant to be used by other programs, allow them to be configured through type constructors.
5. Avoid using package globals to store configuration information. That breaks separation of concern and makes testing harder.

## Project Structure

There are several ways to structure your Go programs and several phylophies that aim to justify those approaches. You may find the following useful:

* [Structuring Applications in Go](https://medium.com/@benbjohnson/structuring-applications-in-go-3b04be4ff091) by Ben Johnson
* [Standard Package Layout](https://medium.com/@benbjohnson/standard-package-layout-7cdbc8391fc1) by Ben Johnson
* [Package Oriented Design](https://www.goinggo.net/2017/02/package-oriented-design.html) by William Kennedy
* [Five suggestions for Setting up a Go project](https://dave.cheney.net/2014/12/01/five-suggestions-for-setting-up-a-go-project) by Dave Cheney
* [GithubCodeLayout](https://github.com/golang/go/wiki/GithubCodeLayout) by Eric Brown

#### Example Project Structure

```
.
├── Dockerfile 
├── Makefile
├── README.md
├── circle.yml
├── glide.lock
├── glide.yaml
├── build
├── cmd
│   ├── todoserver 
│   │   └── main.go
│   └── todocli
│       └── main.go
├── pkg
│   ├── api
│   │   ├── api.go
│   │   └── api_test.go
│   ├── storage
│   │   ├── db.go
│   │   ├── db_test.go
│   │   ├── file.go
│   │   └── file_test.go
│   └── sharing
├── public
│   ├── images
│   │   ├── image1.png
│   │   ├── image2.png
│   │   └── logo.png
│   ├── scripts
│   │   ├── jquery.min.js
│   │   ├── modernizr.min.js
│   │   └── plugins.js
│   ├── styles
│   │   ├── main.css
│   │   └── normalize.css
│   └── views
│       ├── error.html
│       ├── home.html
│       └── reset.html
└── vendor
    ├── github.com
    │   ├── davecgh
    │   ├── go-ini
    │   ├── golang
    │   ├── pkg
    │   ├── stretchr
    │   ├── valyala
    │   └── y0ssar1an
    ├── golang.org
    │   └── x
    └── google.golang.org
        └── appengine

```

Benefits of this project-agnostic layout include:

- Suitable for projects that product binaries and libraries.
- Go code is cleanly separated from non-Go assets (e.g. public)

## Logging

1. Keep it simple and log only what you can act upon. Tons of verbose logging gets in the way of troubleshooting.
2. Use structured logging (e.g. https://github.com/Sirupsen/logrus).
3. `DEBUG` and `INFO` log levels typically suffice.

## Metrics

1. Instrument signifcant portions of your program (not necessarily to instrument all the things).
2. You can capture utilization, saturation, error rates, request counts, duration and generally anything you can use a counter to track.
3. Measures can be stored and analyzed using OSS tools like [Prometheus](https://prometheus.io/) or cloud-specific offerings like AWS CloudWatch for example.

## Testing

- Use the standard library's `*testing.T` to write your tests (most of the time).
- If you/team have a clear reason to use a TDD/BDD framework (e.g. Ginkgo/Gomega, Goconvey's DSL), than do so.
- Design for testing, see [Advanced Testing with Go](https://www.youtube.com/watch?v=yszygk1cpEc) by Metchell Hashimoto
- Use small interfaces to model your dependencies.
- Focus your tests on the actual behavior being tested (i.e. don't test the `http` transport the request came on to trigger your behavior)

#### Example

```
 func foo() {
   res, err := http.Get("http://example.com")
   // ...
 }
```

In the above, `http.Get` calls on an implicit global `http.Client`, the `DefaultClient`. We can avoid the use of this global like so:

```
 func foo(client *http.Client) {
   rep, err := client.Get("http://example.com")
   // ...
 }
```

Though we made things a little better here, passing in a concrete type for that `http.Client` also forces us to use an `http.Client` during testing. We can further refactor and avoid that constraint by using an interface like so:

```
type Retriever interface {
  Do(*http.Request) (*http.Response, error)
}

func foo(r Retriever) {
  req, err := http.NewRequest(http.MethodGet, "http://example.com", nil)
  // if err != nil {...
  
  res, err := r.Do(req)
  // if err != nil {...
}
```

Using the `Retriever` interface that we know `http.Client` will satisfy (i.e. `func (c *Client) Do(req *Request) (*Response, error)`), this allows us to provide our own `Retriever` implementation during testing by simply having a `Do` method that satisfies the interface as well.

## Avoid Global State

The example above showed us how we can remove our reliance on the `http` packages `DefaultClient` and thereby make our application easier to test by swapping concreate types for interfaces. Here are other things you should be on the look out for in your code:

- package-level `init` func has package-global side effects
- log.Print uses a global log.Logger
- http.Server uses a global log.Logger

The above list is by no means exhaustive but it does show a pervasive use of global state in the standard library--which some consider to be unfortunate. That said, we can design our own applications in a way that avoids them.

## Additional Resources

- [How to write Go Code](https://golang.org/doc/code.html) - Introductory but relevant
- [Testable Examples in Go](https://blog.golang.org/examples) - Test code is code too. Treat it as such.
- [Effective Go](https://golang.org/doc/effective_go.html) - Read this several times and again as you get more confortable with Go.
- [Co Code Review Comments](https://github.com/golang/go/wiki/CodeReviewComments) - Excellent advice. Do as gophers do.
- [Go best practices, sis years in](https://peter.bourgon.org/go-best-practices-2016/) by Peter Bourgon (excellent resource that goes deeper into many of the topics duscussed above).

## Workshop Exercise 1

Write an HTTP server (REST API) to manage your TODOs.

API Expectations:

- GET /todos (list todos)
- GET /todos/:id (get single todo)
- POST /todos (add todo)
- PATCH /todos/:id (update part or all of a todo)

Assume TODO has a title and description. JSON in, JSON out.

Criteria:

- Use project-based dependency management using a vendoring tool of your choice.
- Structure your application so that it can be used as a package as well as produce a binary for your server.
- Support local storage on the file system. Extra credit: Support storage using a remote service (e.g. AWS DynamoDB, GCP Cloud Storage, etc). Think about how you could model your storage interaction so that you can swap local and remote storage.
- Support structured logging. Do not rely on the global logger.
- Use at least one interface to model your dependencies.

## Workshop Exercise 2

Refactor an existing TODO backend application to apply the best practices you've learned.

First, fork the following repo: https://github.com/mforman/todo-backend-golang, clone it locally and (attempt) to run it.

Criteria:

- There are no tests and this makes refactoring a brittle prospect. How can add tests to the existing codebase to safeguard our refactoring efforts?
- Add project-based dependency management or rely on the global $GOPATH. Your choice.
- Support structured logging. Do not rely on the global logger.
- Use at least one interface to model your dependencies.