sabl / [patterns](../README.md#patterns) / context

# context

**context** is a pattern for injecting state and dependencies and for propagating cancellation signals. It is simple, mechanically clear, and intrinsically safe for concurrent environments. It was first demonstrated in the [golang](https://go.dev/doc/) standard library [`context` package](https://pkg.go.dev/context), published in 2014 with go 1.7.

The purpose of the context pattern is to make state injection and cancellation mechanically obvious, procedural, and local to each function. It presents an alternative to dependency injection patterns that rely on unseen mechanics to summon state and dependencies to the places where they are declaratively requested. The context pattern comes with a [cost](#consumer-requirements), while the benefits are clarity and predictability.

### Implementations

|Language|Maintainer|Source|Package|
|-|-|-|-|
|go|stdlib|github : [golang/go](https://github.com/golang/go/blob/master/src/context/context.go)|[context](https://pkg.go.dev/context)|
|JS / TS|sabl|github : [libsabl/context-js](https://github.com/libsabl/context-js)|[@sabl/context](https://www.npmjs.com/package/@sabl/context)|
<!-- |dotnet / C#|sabl|github : [libsabl/context-net](https://github.com/libsabl/context-net)|[Sabl.Context](https://www.nuget.org/packages/Sabl.Context)| -->

## Basic Pattern

### Interface
A context is a simple **interface** which provides just three features:

- `value(key)`

   A method to retrieve a value by its key. The value can be any type at all, while the key is usually a string or a [unique string-like label value](#key-type). The `value` method **must** return a null or empty value if there is no matching entry for the key requested. It should never return or throw an error.

- A means to check if the context is **already canceled**. 
- A means to respond as soon as the context is canceled.

The `value` function is essentially identical in any language, while the specific implementation of the cancellation signal varies by the language and runtime. 

Consumers of a context retrieve injected values such as ambient state or dependent services, and if applicable respond to cancellation signals.

The most common and safe pattern for getting and setting context values, including in go, is not actually contained in the context libraries themselves, but is described in detail below in [Getter / Setter Pattern](#getter--setter-pattern).

### Immutability

Mechanically, a context is a linked list of **immutable** context nodes. New key-value pairs and new cancellation checkpoints are added to a context by wrapping a parent context with functions usually called `withValue` and `withCancel`. This pattern provides intrinsic thread safety even in highly concurrent environments. Even when concurrency is not a concern, the pattern provides total clarity and predictability.

Because a context is a simple linked list, the key lookups  in the simplest implementations are `O(n)`. It is not common to have hundreds of context key-value pairs so the actual cost is usually trivial. 

That said, the context pattern works with any node that implements the minimal interface, and there is nothing preventing a determined engineer or team from creating a more efficient implementation that provides closer to `O(log n)` key lookups as long as immutability is preserved. In fact, the sabl implementation in dotnet internally uses an immutable dictionary to achieve just that.

### Consumer Requirements
 
The cost of using this pattern is that the context must be passed in as a parameter to each function that needs to use it. 

This demand is not difficult for new builds. Unmanaged coupling (aka spaghetti code) grows when authors "just" refer to an ambient symbol from within a more narrow scope -- an import, an shared variable, etc. Elaborate dependency injection frameworks instead require authors to "just" add some metadata such as a decorator or attribute, "just" add a constructor parameter, or "just" ask for a service from a service provider which is itself a shared or global instance -- spaghetti consolidated to a single noodle. The context pattern requires that authors "just" add a function parameter.

The greater challenge is adapting existing code bases. There are two methods to integrate the context pattern into existing code:

- Modify the signature of every function that must become context-aware
- Add a context as a member of an existing class / struct / module that is already present as a parameter to existing functions

`go` contains examples of both techniques. The simple relational database interfaces in the standard [`sql` package](https://pkg.go.dev/database/sql) include functions like `Query(query string, args ...any)`, a typical SQL API that accepts a string SQL statement and zero or more SQL parameter values. There is no extension point here to attach a context to, so as the context pattern was adopted throughout the go ecosystem and standard libraries, additional methods were defined with an additional context parameter, which is traditionally provided as the first parameter. The updated equivalent function here is `QueryContext(ctx context.Context, query string, args ...any)`. Note that `go` does not support function overloading, hence the pedantic function name here. 

The canonical `go` example of the other technique is the core `http` package, where the `http.Request` struct was augmented to include a context object. This allows the context to be passed down through the middleware and handler stack while still preserving the traditional two-parameter signature (request and response) of middleware and endpoint functions.

The augmentation pattern provides a useful means to add the context pattern to existing established patterns, such as the near-universal middleware and endpoint pattern for servers. How this augmentation is achieved depends on the features available in a given language and runtime. 

### Cancellation

There is one exception to the immutability rule, and a corresponding requirement on how cancellations must be used in order to avoid a memory leak.

In order to automatically cascade cancellation signals to downstream contexts, descendent cancellation checkpoints must register themselves with their closest ancestor cancellation checkpoint. This must be implemented in the internals of the `withCancel` method. 

Mechanically, and regardless of the programming language used, this means updating the internal state of the ancestor cancellation context to add a reference to the descendant context so that the ancestor knows to cancel the descendant if the ancestor is canceled. 

It is common for an ancestor context to last much longer than one or many child contexts that are created and discarded. The danger is that each descendant cancellation context registers itself with the ancestor, but then fails to unregister itself at the end of its natural life. Even when the descendant context goes out of scope and would otherwise be garbage collected in runtimes that provide garbage collection, including `go`, the ancestor would retain a reference to the descendant due to the registration. And in fact the ancestor would continue to accumulate these references with each new descendant cancellation context created.

To prevent this, any scope in which a cancelable context is created with `withCancel` **MUST** explicitly cancel the context when its work is done, and the implementation of that cancellation **MUST** remove the registration from its ancestor cancellation checkpoint if there was one. 

The method of achieving this assurance varies based on the features available in the language. In `go` this is accomplished with the `defer` keyword, while a `try ... finally` construction can be used in multiple languages and runtimes which support it.

This presents a small semantic oddity for newcomers to the pattern, in that every single cancelable context must be `cancel()`'d, even when all operations completed normally or successfully. A better verb here may be `done` or `resolve`, but for the sake of tradition and conciseness the function here is called `cancel`.

A common example is a root context for a web server which provides common services or configuration. Each request will create a child context off this common base, and the expectation is of course that the child context will be cleanly discarded when the request completes or is canceled. 

It is reasonable to use a cancellation checkpoint in the base context, such as to respond to a request to gracefully terminate the server. It is also expected that each request context provide its own cancellation checkpoint so that if the request is canceled either by the server or the client then any downstream long-running operations can also be canceled. 

If each request context is not explicitly canceled, then the server context could accumulate thousands or even millions of stale cancellation registrations. And so of course the core implementation of request handling in go creates a cancellation context and then [immediately defers a call to cancel it](https://github.com/golang/go/blob/go1.18.2/src/net/http/server.go#L1884), ensuring this classic memory leak does not occur in that package.

### Key Type

The original context pattern in `go` accepts any key type at all. String keys are naturally favored because they provide some semantic explanation of a given context value. 

Certainly all context implementations must allow the value to be of any type at all -- including null. The problem then with using plain string keys is both key collision and type safety. Two different libraries may both want to save a `"connection"` on the context, but they mean different things. Likewise there might be agreement on what `"pg-connection"` means, but a mistake in client code on the type of the value that should be supplied or expected along with that key.

The first step to avoiding this altogether is not using string keys, but instead using some mechanism that still provides semantic value but avoids accidental key collision. The technique used varies by the language or runtime features of the target environment. Here are just two examples:

- In `go`, custom types can be declared which extend primitive types. Two values are only considered equal if they are both of the same type and the same underlying value, so two values of different string-derived types will never be reported equal even if the values were initialized from the same string value.

    ```go
    type myString1 string
    type myString2 string

    const s1Hello myString1 = "hello"
    const s2Hello myString2 = "hello"

    fmt.Println(s1Hello == s2Hello) // false
    ```

- In JavaScript, the `Symbol` class provides a built-in means for exactly this sort of situation, and in fact the sabl implementation for JavaScript / TypeScript only accepts string or symbol keys.

    ```js
    const s1 = Symbol("hello")
    const s2 = Symbol("hello")
    console.log(s1 == s2) // false
    ```



### Getter / Setter Pattern

Ensuring keys do not collide still does not address type safety. Even with a `const` key value exported from some package, there is nothing preventing one from providing any value at all with that key, and nothing in the core `value(key)` API to inform tooling or runtime what the return type of the retrieved value might be. 

To address this, a pattern quickly emerged. This pattern exists only in client code, not in any implementation of the actual `context` chain itself, but it is an essential element of using the pattern well. The pattern composes two complementary techniques to guarantee type safety of context values in a way that is clear to both humans and tooling:

1. Private / non-exported key value

   Given some mechanism for [strict key comparison](#key-type), implementers first define a constant or static key value for each known context item they wish to support, but ensure this value is **not** public or exported.

2. Public / exported setter and getter

   Next, implementers define a setter function and corresponding getter function which enforce or check the type of the context value being added or retrieved, and which internally use the applicable private key value along with the basic `value` and `withValue` APIs.

We already addressed key collision [above](#key-type). By using a private key value of a non-colliding type, this pattern ensures that it is **impossible** to set a value with the private key but of the wrong value type. The exported setter method is the only possible way to set a value with the applicable key, and the setter method guarantees the type of the value, whether through static type checking or runtime validations. 

It is then safe for the exported getter to make a direct type cast of the value retrieved with the private key. Note that getter implementations still must accommodate the possibility that there is no value at all for the key.

### Example : Go

#### 1. Definition

```go
// Non-exported key type and value
type localKeyType string
const ctxKeyUser localKeyType = "user"

// Exported setter
func WithUser(ctx context.Context, u sec.User) context.Context {
	return context.WithValue(ctx, ctxKeyUser, u)
}

// Exported getter
func GetUser(ctx context.Context) (u sec.User, ok bool) {
	u, ok = ctx.Value(ctxKeyUser).(sec.User)
	return
}
```

#### 2. Usage

```go
import (
    './secctx'
)

// Add user to context in authn middleware 
app.Use(func(h http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        u, err := sec.Authenticate(r)
        if err != nil {
            http.Error(w, err.Error(), 401)
            return
        }
        ctx := secctx.WithUser(r.Context(), u)
        h.ServeHTTP(w, r.WithContext(ctx))
    })
})

// Retrieving context value
func getStatus(w http.ResponseWriter, r *http.Request) {
	u, ok := secctx.GetUser(r.Context())
    if !ok { 
        ... 
    }
    // do stuff
}
```

---
&copy; 2022 Joshua Honig. All rights reserved. See [LICENSE](../LICENSE.md).