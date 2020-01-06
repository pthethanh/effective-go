This page collects common comments made during reviews of Go code and some best practices collected from different sources.

You can view this as a supplement to [Effective Go](https://golang.org/doc/effective_go.html).

## Gofmt

Run [gofmt](https://golang.org/cmd/gofmt/) on your code to automatically fix the majority of mechanical style issues. Almost all Go code in the wild uses `gofmt`. The rest of this document addresses non-mechanical style points.

An alternative is to use [goimports](https://godoc.org/golang.org/x/tools/cmd/goimports), a superset of `gofmt` which additionally adds (and removes) import lines as necessary.

As a developer, we often forget to run `go fmt` hence the best way to avoid pushing your code with a bad format is to configure your IDE to run `go fmt` whenever the code is saved.

## Go vet

Run `go vet` on your code to check if any suspicious piece of code before every single commit. Vet will examine your code and reports suspicious constructs, such as Printf calls whose arguments do not align with the format string. Vet uses heuristics that do not guarantee all reports are genuine problems, but it can find errors not caught by the compilers. This is very useful as we - developers some time make silly mistakes when we are in rush :)

Run`go vet` for all of sub-packages of your code by following command:

`go vet ./...`

## Comment Sentences

See <https://golang.org/doc/effective_go.html#commentary>. Comments documenting declarations should be full sentences, even if that seems a little redundant. This approach makes them format well when extracted into godoc documentation. Comments should begin with the name of the thing being described and end in a period:

```go
// Request represents a request to run a command.
type Request struct { ...

// Encode writes the JSON encoding of req to w.
func Encode(w io.Writer, req *Request) { ...
```

## Contexts

Values of the context.Context type carry security credentials, tracing information, deadlines, and cancellation signals across API and process boundaries. Go programs pass Contexts explicitly along the entire function call chain from incoming RPCs and HTTP requests to outgoing requests.

Most functions that use a Context should accept it as their first parameter:

```go
func F(ctx context.Context, /* other arguments */) {}
```

A function that is never request-specific may use context.Background(), but err on the side of passing a Context even if you think you don't need to. The default case is to pass a Context; only use context.Background() directly if you have a good reason why the alternative is a mistake.

Don't add a Context member to a struct type; instead add a ctx parameter to each method on that type that needs to pass it along. The one exception is for methods whose signature must match an interface in the standard library or in a third party library.

Don't create custom Context types or use interfaces other than Context in function signatures.

If you have application data to pass around, put it in a parameter, in the receiver, in globals, or, if it truly belongs there, in a Context value.

Contexts are immutable, so it's fine to pass the same ctx to multiple calls that share the same deadline, cancellation signal, credentials, parent trace, etc.

Read more about context at:

1. [Context isn’t for cancellation](https://dave.cheney.net/2017/08/20/context-isnt-for-cancellation)
2. [Context is for cancelation](https://dave.cheney.net/2017/01/26/context-is-for-cancelation)

## Copying

To avoid unexpected aliasing, be careful when copying a struct from another package. For example, the bytes.Buffer type contains a `[]byte` slice and, as an optimization for small strings, a small byte array to which the slice may refer. If you copy a `Buffer`, the slice in the copy may alias the array in the original, causing subsequent method calls to have surprising effects.

In general, do not copy a value of type `T` if its methods are associated with the pointer type, `*T`.

## Declaring Empty Slices

When declaring an empty slice, prefer

```go
var t []string
```

over

```go
t := []string{}
```

The former declares a nil slice value, while the latter is non-nil but zero-length. They are functionally equivalent—their `len` and `cap` are both zero—but the nil slice is the preferred style.

Note that there are limited circumstances where a non-nil but zero-length slice is preferred, such as when encoding JSON objects (a `nil` slice encodes to `null`, while `[]string{}` encodes to the JSON array `[]`).

When designing interfaces, avoid making a distinction between a nil slice and a non-nil, zero-length slice, as this can lead to subtle programming errors.

For more discussion about nil in Go see Francesc Campoy's talk [Understanding Nil](https://www.youtube.com/watch?v=ynoY2xz-F8s).

## Crypto Rand

Do not use package `math/rand` to generate keys, even throwaway ones. Unseeded, the generator is completely predictable. Seeded with `time.Nanoseconds()`, there are just a few bits of entropy. Instead, use `crypto/rand`'s Reader, and if you need text, print to hexadecimal or base64:

```go
import (
    "crypto/rand"
    // "encoding/base64"
    // "encoding/hex"
    "fmt"
)

func Key() string {
    buf := make([]byte, 16)
    _, err := rand.Read(buf)
    if err != nil {
        panic(err)  // out of randomness, should never happen
    }
    return fmt.Sprintf("%x", buf)
    // or hex.EncodeToString(buf)
    // or base64.StdEncoding.EncodeToString(buf)
}
```

## Doc Comments

All top-level, exported names should have doc comments, as should non-trivial unexported type or function declarations. See <https://golang.org/doc/effective_go.html#commentary> for more information about commentary conventions.

## Don't Panic

See <https://golang.org/doc/effective_go.html#errors>. Don't use panic for normal error handling. Use error and multiple return values.

## Error Strings

Error strings should not be capitalized (unless beginning with proper nouns or acronyms) or end with punctuation, since they are usually printed following other context. That is, use `fmt.Errorf("something bad")` not `fmt.Errorf("Something bad")`, so that `log.Printf("Reading %s: %v", filename, err)` formats without a spurious capital letter mid-message. This does not apply to logging, which is implicitly line-oriented and not combined inside other messages.

## Examples

When adding a new package, include examples of intended usage: a runnable Example, or a simple test demonstrating a complete call sequence.

Read more about [testable Example() functions](https://blog.golang.org/examples).

## Goroutine Lifetimes

When you spawn goroutines, make it clear when - or whether - they exit.

Goroutines can leak by blocking on channel sends or receives: the garbage collector will not terminate a goroutine even if the channels it is blocked on are unreachable.

Even when goroutines do not leak, leaving them in-flight when they are no longer needed can cause other subtle and hard-to-diagnose problems. Sends on closed channels panic. Modifying still-in-use inputs "after the result isn't needed" can still lead to data races. And leaving goroutines in-flight for arbitrarily long can lead to unpredictable memory usage.

Try to keep concurrent code simple enough that goroutine lifetimes are obvious. If that just isn't feasible, document when and why the goroutines exit.

## Handle Errors

Do not discard errors using `_` variables. If a function returns an error, check it to make sure the function succeeded. Handle the error, return it, or, in truly exceptional situations, panic.

When checking an error, don't check for a specific text in error, check its type instead. Don't write:
```go
if strings.Contains(err.Error(), "timeout"){
    // do something
}
```

Instead, write:
```go
if nerr, ok := err.(net.Error); ok && nerr.Timeout() {
    // do something
}
if err != nil {
    log.Fatal(err)
}
```

If you are using Go 1.13 afterward, better look at https://blog.golang.org/go1.13-errors. It offers a very nice way to wrap, check error.

Read more about error at:
1. [Effective Go - Errors](https://golang.org/doc/effective_go.html#errors)
1. [Error handling and go](https://blog.golang.org/error-handling-and-go)
2. [Errors are values](https://blog.golang.org/errors-are-values)

## Imports

Avoid renaming imports except to avoid a name collision; good package names should not require renaming. In the event of collision, prefer to rename the most local or project-specific import.

Imports are organized in groups, with blank lines between them. The standard library packages are always in the first group.

```go
package main

import (
	"fmt"
	"hash/adler32"
	"os"

	"appengine/foo"
	"appengine/user"

        "github.com/foo/bar"
	"rsc.io/goversion/version"
)
```

[goimports](https://godoc.org/golang.org/x/tools/cmd/goimports) will do this for you.

## In-Band Errors

In C and similar languages, it's common for functions to return values like -1 or null to signal errors or missing results:

```go
// Lookup returns the value for key or "" if there is no mapping for key.
func Lookup(key string) string

// Failing to check a for an in-band error value can lead to bugs:
Parse(Lookup(key))  // returns "parse failure for value" instead of "no value for key"
```

Go's support for multiple return values provides a better solution. Instead of requiring clients to check for an in-band error value, a function should return an additional value to indicate whether its other return values are valid. This return value may be an error, or a boolean when no explanation is needed. It should be the final return value.

```go
// Lookup returns the value for key or ok=false if there is no mapping for key.
func Lookup(key string) (value string, ok bool)
```

This prevents the caller from using the result incorrectly:

```go
Parse(Lookup(key))  // compile-time error
```

And encourages more robust and readable code:

```go
value, ok := Lookup(key)
if !ok  {
    return fmt.Errorf("no value for %q", key)
}
return Parse(value)
```

This rule applies to exported functions but is also useful for unexported functions.

Return values like nil, "", 0, and -1 are fine when they are valid results for a function, that is, when the caller need not handle them differently from other values.

Some standard library functions, like those in package "strings", return in-band error values. This greatly simplifies string-manipulation code at the cost of requiring more diligence from the programmer. In general, Go code should return additional values for errors.

## Indent Error Flow

Try to keep the normal code path at a minimal indentation, and indent the error handling, dealing with it first. This improves the readability of the code by permitting visually scanning the normal path quickly. For instance, don't write:

```go
if err != nil {
	// error handling
} else {
	// normal code
}
```

Instead, write:

```go
if err != nil {
	// error handling
	return // or continue, etc.
}
// normal code
```

If the `if` statement has an initialization statement, such as:

```go
if x, err := f(); err != nil {
	// error handling
	return
} else {
	// use x
}
```

then this may require moving the short variable declaration to its own line:

```go
x, err := f()
if err != nil {
	// error handling
	return
}
// use x
```

## Initialisms

Words in names that are initialisms or acronyms (e.g. "URL" or "NATO") have a consistent case. For example, "URL" should appear as "URL" or "url" (as in "urlPony", or "URLPony"), never as "Url". As an example: ServeHTTP not ServeHttp. For identifiers with multiple initialized "words", use for example "xmlHTTPRequest" or "XMLHTTPRequest".

This rule also applies to "ID" when it is short for "Identity Document" (which is pretty much all cases when it's not the "id" as in "ego", "superego"), so write "appID" instead of "appId".

Code generated by the protocol buffer compiler is exempt from this rule. Human-written code is held to a higher standard than machine-written code.

## Interfaces

Go interfaces generally belong in the package that uses values of the interface type, not the package that implements those values. The implementing package should return concrete (usually pointer or struct) types: that way, new methods can be added to implementations without requiring extensive refactoring.

Do not define interfaces on the implementor side of an API "for mocking"; instead, design the API so that it can be tested using the public API of the real implementation.

Do not define interfaces before they are used: without a realistic example of usage, it is too difficult to see whether an interface is even necessary, let alone what methods it ought to contain.

```go
package consumer  // consumer.go

type Thinger interface { Thing() bool }

func Foo(t Thinger) string { … }
package consumer // consumer_test.go

type fakeThinger struct{ … }
func (t fakeThinger) Thing() bool { … }
…
if Foo(fakeThinger{…}) == "x" { … }
// DO NOT DO IT!!!
package producer

type Thinger interface { Thing() bool }

type defaultThinger struct{ … }
func (t defaultThinger) Thing() bool { … }

func NewThinger() Thinger { return defaultThinger{ … } }
```

Instead return a concrete type and let the consumer mock the producer implementation.

```go
package producer

type Thinger struct{ … }
func (t Thinger) Thing() bool { … }

func NewThinger() Thinger { return Thinger{ … } }
```

## Line Length

There is no rigid line length limit in Go code, but avoid uncomfortably long lines. Similarly, don't add line breaks to keep lines short when they are more readable long--for example, if they are repetitive.

Most of the time when people wrap lines "unnaturally" (in the middle of function calls or function declarations, more or less, say, though some exceptions are around), the wrapping would be unnecessary if they had a reasonable number of parameters and reasonably short variable names. Long lines seem to go with long names, and getting rid of the long names helps a lot.

In other words, break lines because of the semantics of what you're writing (as a general rule) and not because of the length of the line. If you find that this produces lines that are too long, then change the names or the semantics and you'll probably get a good result.

This is, actually, exactly the same advice about how long a function should be. There's no rule "never have a function more than N lines long", but there is definitely such a thing as too long of a function, and of too stuttery tiny functions, and the solution is to change where the function boundaries are, not to start counting lines.

## Mixed Caps

See <https://golang.org/doc/effective_go.html#mixed-caps>. This applies even when it breaks conventions in other languages. For example an unexported constant is `maxLength` not `MaxLength` or `MAX_LENGTH`.

Also see [Initialisms](https://github.com/golang/go/wiki/CodeReviewComments#initialisms).

## Named Result Parameters

Consider what it will look like in godoc. Named result parameters like:

```go
func (n *Node) Parent1() (node *Node)
func (n *Node) Parent2() (node *Node, err error)
```

will stutter in godoc; better to use:

```go
func (n *Node) Parent1() *Node
func (n *Node) Parent2() (*Node, error)
```

On the other hand, if a function returns two or three parameters of the same type, or if the meaning of a result isn't clear from context, adding names may be useful in some contexts. Don't name result parameters just to avoid declaring a var inside the function; that trades off a minor implementation brevity at the cost of unnecessary API verbosity.

```go
func (f *Foo) Location() (float64, float64, error)
```

is less clear than:

```go
// Location returns f's latitude and longitude.
// Negative values mean south and west, respectively.
func (f *Foo) Location() (lat, long float64, err error)
```

Naked returns are okay if the function is a handful of lines. Once it's a medium sized function, be explicit with your return values. Corollary: it's not worth it to name result parameters just because it enables you to use named returns. Clarity of docs is always more important than saving a line or two in your function.

Finally, in some cases you need to name a result parameter in order to change it in a deferred closure. That is always OK.

## Naked Returns

See [Named Result Parameters](https://github.com/golang/go/wiki/CodeReviewComments#named-result-parameters).

## Package Comments

Package comments, like all comments to be presented by godoc, must appear adjacent to the package clause, with no blank line.

```go
// Package math provides basic constants and mathematical functions.
package math
/*
Package template implements data-driven templates for generating textual
output such as HTML.
....
*/
package template
```

For "package main" comments, other styles of comment are fine after the binary name (and it may be capitalized if it comes first), For example, for a `package main` in the directory `seedgen` you could write:

```go
// Binary seedgen ...
package main
```

or

```go
// Command seedgen ...
package main
```

or

```go
// Program seedgen ...
package main
```

or

```go
// The seedgen command ...
package main
```

or

```go
// The seedgen program ...
package main
```

or

```go
// Seedgen ..
package main
```

These are examples, and sensible variants of these are acceptable.

Note that starting the sentence with a lower-case word is not among the acceptable options for package comments, as these are publicly-visible and should be written in proper English, including capitalizing the first word of the sentence. When the binary name is the first word, capitalizing it is required even though it does not strictly match the spelling of the command-line invocation.

See <https://golang.org/doc/effective_go.html#commentary> for more information about commentary conventions.

## Package Names

All references to names in your package will be done using the package name, so you can omit that name from the identifiers. For example, if you are in package chubby, you don't need type ChubbyFile, which clients will write as `chubby.ChubbyFile`. Instead, name the type `File`, which clients will write as `chubby.File`. Avoid meaningless package names like util, common, misc, api, types, and interfaces. See <http://golang.org/doc/effective_go.html#package-names> and<http://blog.golang.org/package-names> for more.

## Pass Values

Don't pass pointers as function arguments just to save a few bytes. If a function refers to its argument `x` only as `*x` throughout, then the argument shouldn't be a pointer. Common instances of this include passing a pointer to a string (`*string`) or a pointer to an interface value (`*io.Reader`). In both cases the value itself is a fixed size and can be passed directly. This advice does not apply to large structs, or even small structs that might grow.

## Receiver Names

The name of a method's receiver should be a reflection of its identity; often a one or two letter abbreviation of its type suffices (such as "c" or "cl" for "Client"). Don't use generic names such as "me", "this" or "self", identifiers typical of object-oriented languages that gives the method a special meaning. In Go, the receiver of a method is just another parameter and therefore, should be named accordingly. The name need not be as descriptive as that of a method argument, as its role is obvious and serves no documentary purpose. It can be very short as it will appear on almost every line of every method of the type; familiarity admits brevity. Be consistent, too: if you call the receiver "c" in one method, don't call it "cl" in another.

## Receiver Type

Choosing whether to use a value or pointer receiver on methods can be difficult, especially to new Go programmers. If in doubt, use a pointer, but there are times when a value receiver makes sense, usually for reasons of efficiency, such as for small unchanging structs or values of basic type. Some useful guidelines:

- If the receiver is a map, func or chan, don't use a pointer to them. If the receiver is a slice and the method doesn't reslice or reallocate the slice, don't use a pointer to it.
- If the method needs to mutate the receiver, the receiver must be a pointer.
- If the receiver is a struct that contains a sync.Mutex or similar synchronizing field, the receiver must be a pointer to avoid copying.
- If the receiver is a large struct or array, a pointer receiver is more efficient. How large is large? Assume it's equivalent to passing all its elements as arguments to the method. If that feels too large, it's also too large for the receiver.
- Can function or methods, either concurrently or when called from this method, be mutating the receiver? A value type creates a copy of the receiver when the method is invoked, so outside updates will not be applied to this receiver. If changes must be visible in the original receiver, the receiver must be a pointer.
- If the receiver is a struct, array or slice and any of its elements is a pointer to something that might be mutating, prefer a pointer receiver, as it will make the intention more clear to the reader.
- If the receiver is a small array or struct that is naturally a value type (for instance, something like the time.Time type), with no mutable fields and no pointers, or is just a simple basic type such as int or string, a value receiver makes sense. A value receiver can reduce the amount of garbage that can be generated; if a value is passed to a value method, an on-stack copy can be used instead of allocating on the heap. (The compiler tries to be smart about avoiding this allocation, but it can't always succeed.) Don't choose a value receiver type for this reason without profiling first.
- Finally, when in doubt, use a pointer receiver.

## Synchronous Functions

Prefer synchronous functions - functions which return their results directly or finish any callbacks or channel ops before returning - over asynchronous ones.

Synchronous functions keep goroutines localized within a call, making it easier to reason about their lifetimes and avoid leaks and data races. They're also easier to test: the caller can pass an input and check the output without the need for polling or synchronization.

If callers need more concurrency, they can add it easily by calling the function from a separate goroutine. But it is quite difficult - sometimes impossible - to remove unnecessary concurrency at the caller side.

## Useful Test Failures

Tests should fail with helpful messages saying what was wrong, with what inputs, what was actually got, and what was expected. It may be tempting to write a bunch of assertFoo helpers, but be sure your helpers produce useful error messages. Assume that the person debugging your failing test is not you, and is not your team. A typical Go test fails like:

```go
if got != tt.want {
	t.Errorf("Foo(%q) = %d; want %d", tt.in, got, tt.want) // or Fatalf, if test can't test anything more past this point
}
```

Note that the order here is actual != expected, and the message uses that order too. Some test frameworks encourage writing these backwards: 0 != x, "expected 0, got x", and so on. Go does not.

If that seems like a lot of typing, you may want to write a [table-driven test](https://github.com/golang/go/wiki/TableDrivenTests).

Another common technique to disambiguate failing tests when using a test helper with different input is to wrap each caller with a different TestFoo function, so the test fails with that name:

```go
func TestSingleValue(t *testing.T) { testHelper(t, []int{80}) }
func TestNoValues(t *testing.T)    { testHelper(t, []int{}) }
```

In any case, the onus is on you to fail with a helpful message to whoever's debugging your code in the future.

## Variable Names

Variable names in Go should be short rather than long. This is especially true for local variables with limited scope. Prefer `c` to `lineCount`. Prefer `i` to `sliceIndex`.

The basic rule: the further from its declaration that a name is used, the more descriptive the name must be. For a method receiver, one or two letters is sufficient. Common variables such as loop indices and readers can be a single letter (`i`, `r`). More unusual things and global variables need more descriptive names.

## Use Makefile for common commands

Use a Makefile to collect common commands that every developer in your projects need to execute every day. You can include fundamental commands like `go fmt`, `go vet`, `go test` or building docker images there. 

Here is an example:

```shell
SHELL:=/bin/bash
PROJECT_NAME=myapp
GO_BUILD_ENV=CGO_ENABLED=0 GOOS=linux GOARCH=amd64
GO_FILES=$(shell go list ./... | grep -v /vendor/)

BUILD_VERSION=$(shell cat VERSION)
BUILD_TAG=$(BUILD_VERSION)
DOCKER_IMAGE=$(PROJECT_NAME):$(BUILD_TAG)

.SILENT:

all: fmt vet install test

build:
	$(GO_BUILD_ENV) go build -v -o $(PROJECT_NAME).bin .

install:
	$(GO_BUILD_ENV) go install

vet:
	go vet $(GO_FILES)

fmt:
	go fmt $(GO_FILES)

test:
	go test $(GO_FILES) -cover

integration_test:
	go test -tags=integration $(GO_FILES)

docker: build
	docker build -t $(DOCKER_IMAGE) .;\
        rm -f $(PROJECT_NAME).bin 2> /dev/null; \
```

Once the Makefile is created, everyone just need to type simple command like: `make all`, `make build` or `make docker`.

## Don't use default http.Client & http.Server in production

Don't use default http.Client in production code. The `http.DefaultClient` is very convenient as you can very easily do HTTP request by `http.Get(url)` or `http.DefaultClient.Get(url)` without the need to initialize the client yourself but it should not be used for production. 

The reason is the default client is created with Timeout is 0, which means `no timeout`. This means when the target server has outage or something like that, the connection from the client will be hang forever and will cause the user request (if one is associated) to be hang as well.

The best way to avoid the above problem is to define a custom http.Client with timeout and reuse it in your application:

```go
var c = &http.Client{
  Timeout: time.Second * 10,
}
res, err := c.Get(url)
```

Or you can control more over the transport by creating a custom one:

```go
var ts = &http.Transport{
  Dial: (&net.Dialer{
    Timeout: 5 * time.Second,
  }).Dial,
  TLSHandshakeTimeout: 5 * time.Second,
}
var c = &http.Client{
  Timeout: time.Second * 10,
  Transport: ts,
}
res, err := c.Get(url)
```

The same is applied for `http.Server`, create a custom server for using in production instead of using the default one:

```go
server := http.Server{
	Addr:         *addr,
	ReadTimeout:  30 * time.Second,
	WriteTimeout: 30 * time.Second,
}
if err := server.ListenAndServe(); err != nil {
	log.Panicf("main: listening on %s failed: %v", server.Addr, err)
}
```

Read more about this at:

1. [Don't use default http client](https://medium.com/@nate510/don-t-use-go-s-default-http-client-4804cb19f779)
2. [The complete guide to golang http timeout](https://blog.cloudflare.com/the-complete-guide-to-golang-net-http-timeouts/)

## Add context to errors

Wrapping errors with a custom message provides context as it gets propagated up the stack. Make sure the root error is still accessible somehow for type checking.

Don't write:

```go
file, err := os.Open("foo.txt")
if err != nil {
	return err
}
```

Instead, write:

```go
import "github.com/pkg/errors"

// ...

file, err := os.Open("foo.txt")
if err != nil {
	return errors.Wrap(err, "failed to open foo.txt")
}
```

If you are using Go 1.13 afterward, better look at https://blog.golang.org/go1.13-errors. It offers a very nice way to wrap, check error.
## Avoid global variables

Global variables make testing and readability hard. It also makes everyone in the same package can access it even they don't need it and should not able to access to it. The best practice is to set it as a dependency of a struct and use dependency injection to inject it whenever you need it (often in main.go)

So, don't write:

```go
var db *sql.DB

func main() {
	db = // ...
	http.HandleFunc("/drop", DropHandler)
	// ...
}

func DropHandler(w http.ResponseWriter, r *http.Request) {
	db.Exec("DROP DATABASE prod")
}
```

Instead, use dependency injection to inject it into the struct:

```go
func main() {
	db := // ...
	handlers := Handlers{DB: db}
	http.HandleFunc("/drop", handlers.DropHandler)
	// ...
}

type Handlers struct {
	DB *sql.DB
}

func (h *Handlers) DropHandler(w http.ResponseWriter, r *http.Request) {
	h.DB.Exec("DROP DATABASE prod")
}
```

## Project Structure/Layout

It's always a good idea to structure your code to make it readable and extensible.
If you're writing a library/framework, source code of the Go SDK is a good place to look at. But if you are writing an application which composed of different modules, it's recommended to look at [go-project-layout]( https://github.com/golang-standards/project-layout). A sample implementation of the layout can be found at the [robusta-project](https://github.com/pthethanh/robusta).

For beginner: A simple approach to check if you have a good code, good design is to write unit test for your code. Most beginner write their code and cannot even write unit test for their code because of complexity of the code/design, especially when external dependencies such as database, mail, message queue, ....  are needed.



## References

1. [Effective Go](https://golang.org/doc/effective_go.html)
2. [Go Code Review Comments](https://github.com/golang/go/wiki/CodeReviewComments)
3. [Best practices for a new go developer](https://blog.rubylearning.com/best-practices-for-a-new-go-developer-8660384302fc)
4. [Error handling in Go](https://blog.golang.org/error-handling-and-go)
5. [Errors are values](https://blog.golang.org/errors-are-values)
6. [Slice tricks](https://github.com/golang/go/wiki/SliceTricks)
7. [Table driven testing](https://github.com/golang/go/wiki/TableDrivenTests)
8. [HTTP services best practices - Mat Ryer](https://medium.com/statuscode/how-i-write-go-http-services-after-seven-years-37c208122831)
9. [Go best practices - Peter Bourgon](https://peter.bourgon.org/go-best-practices-2016/)
10. [Twelve Go Best Practices -Francesc Campoy Flores](https://talks.golang.org/2013/bestpractices.slide#1)
11. [Don’t use Go’s default HTTP client in production](https://medium.com/@nate510/don-t-use-go-s-default-http-client-4804cb19f779)
12. [Go project layout](https://medium.com/golang-learn/go-project-layout-e5213cdcfaa2)
13. [Building APIs - Mat Ryer](https://go-talks.appspot.com/github.com/matryer/golanguk/building-apis.slide)
14. [Context isn't for cancellation - Dave Cheney](https://dave.cheney.net/2017/08/20/context-isnt-for-cancellation)
15. [The package level logger anti pattern - Dave Cheney](https://dave.cheney.net/2017/01/23/the-package-level-logger-anti-pattern)
