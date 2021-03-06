# Interfaces

An interface in go is a contract specifying which method signatures a type must have. Crucially, it is specified by the user of the interface, not the code which satisfies it.

## Keep interfaces simple

> The bigger the interface, the weaker the abstraction
> – Rob Pike

Interfaces are at their most powerful when they express a simple contract that any type can easily conform to. If they start to demand a laundry list of functions \(_more than a few_ is a good rule of thumb\), they have very little advantage over a concrete type as an argument, because the caller is not going to be able to create an alternative type without substantially recreating the original.

Here are some examples of useful interfaces from the standard library, you'll notice that all of these extremely popular interfaces have one thing in common - they all require very few functions.

[error](https://golang.org/ref/spec#Errors)

Error represents an error condition, and only returns a string with a description of the error.

```go
type error interface {
    Error() string
}
```

[fmt.Stringer](https://golang.org/pkg/fmt/#Stringer)

Used when formatting values for fmt.Printf and friends.

```go
type Stringer interface {
        String() string
}
```

[io.Reader ](https://golang.org/pkg/io/#Reader)

Read reads len\(p\) bytes from the data stream.

```go
type Reader interface {
        Read(p []byte) (n int, err error)
}
```

[io.Writer](https://golang.org/pkg/io/#Writer)

Write writes len\(p\) bytes from p to a data .stream.

```go
type Writer interface {
        Write(p []byte) (n int, err error)
}
```

[http.ResponseWriter](https://golang.org/pkg/net/http/#ResponseWriter)

HTTP handlers are passed a ResponseWriter to write the HTTP response to a request. 

```go
type ResponseWriter interface {
        // Header returns the header map used to set headers.
        Header() Header

        // Write writes the data to the connection as part of an HTTP reply.
        Write([]byte) (int, error)

        // WriteHeader writes an HTTP response header with just a status code - used for errors.
        WriteHeader(int)
}
```

## Avoid the empty Interface

The empty interface is written like this, but unlike most interfaces it requires no methods \(hence the empty brackets\):

```go
interface{}
```

It is the equivalent of:

```go
type MyInterface interface {
}
```

Don't overuse empty interface - _it means nothing_. If you find yourself using empty interface and then switching on type, consider instead defining separate functions which operate on concrete types. Don't try to use empty interface as a poor man's generics - it is possible, but it subverts the type system and makes it very easy to cause panics.

## Accept interfaces, return types

Interfaces are a way of avoiding tight coupling between different packages, so they are most useful when defined at their point of use, and only used there. If you export an interface as a return type, you are forcing others to use this interface only forever, or to attempt to cast your interface back into a concrete type.

Do not design interfaces for mocking, design them for real world use, and don't add methods to them before you have a concrete use for the methods. The exception to this is of course the extremely common error interface.

## Avoid mixing interface and concrete types

When defining interfaces in a package, don't also provide the implementation in the same package, because the receiver should define interfaces, not the implementer – you may sometimes need to break this guideline.

* Define interfaces in the package that uses them
* Define concrete types in the package that uses the package that accepts an interface

## Don't use pointers to interface

You probably meant to use a pointer to your real type, or just a plain old Interface. You don't need a pointer to interfaces.

## Don't compare interface to nil

An interface will only be nil when both their type and value fields are nil, so comparing interface to nil can have unexpected results. Don't do that. You can read more about this problem in the, but in summary if any concrete value has ever been stored in the interface, the interface will not be nil.

```go
// E is a type conforming to the error interface  
type E struct{}  
func (e *E) Error() string { return fmt.Sprintf("%p", e) }

// typederror returns an interface  
func typederror() error {  
    var e *E = nil  
    return e // e is not nil as expected  
}
```

You can read more about the internals of interfaces in [Go Data Structures: Interfaces](https://research.swtch.com/interfaces) on Russ Cox's blog.

