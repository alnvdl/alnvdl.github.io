---
layout: post
title: "Error handling in Go"
date: "2023-09-07 19:00:00 -0300"
excerpt: A bit of history, a bit of atonement, a bit of shameless self-promotion.
---

Let's talk about error handling in Go, in 3 acts.

## A bit of history

Error handling in Go is a topic that divides opinions. Some people hate that their code ends up full of error checks. Others argue that's precisely the point: [errors are simply return values](https://go.dev/blog/errors-are-values), and these return values should of course affect the flow of your code.

I never really had a strong opinion about this. I knew I always considered exceptions in other programming languages a bit weird because they can end up being caught in distant places across the stack; but I didn't nurture strong feelings of love or hatred for exceptions either.

I started using Go in version 1.12 at work, a few weeks before version 1.13 was released. The library we used in the projects I worked in was [errgo](https://pkg.go.dev/gopkg.in/errgo.v1) (v1). Due to inertia (and fear of breaking things), we still use errgo in 2023.

In a way, errgo -- whose last release was before Go 1.13 -- was trying to anticipate a couple of things that were missing in Go at the time: error wrapping and error tracing.

Starting with Go 1.13, error wrapping [was added](https://go.dev/doc/go1.13#error_wrapping) to the `errors` package in the standard library.

As of September 2023, error tracing is the stuff of legends (and 3rd party libraries).

## A bit of atonement

Even with Go's ever evolving approach to error handling and clever 3rd party libraries, one problem remains: programmers coming into Go from other programming languages can and *will* make mistakes.

**I know it because I've seen these mistakes being made. I know it because I've made these mistakes myself.** So here's my attempt at making amends for all the mistakes I've made.

I won't go over all the possible things that can go wrong (that would be a very large list). Instead, I will suggest some good practices and contrast them with some negative examples -- or at least, my interpretation of positive and negative based on reading some official Go resources. The information condensed below was extracted from multiple official Go sources, where I think it's all a bit dispersed and hard to piece together:
- [https://go.dev/blog/error-handling-and-go](https://go.dev/blog/error-handling-and-go)
- [https://go.dev/blog/errors-are-values](https://go.dev/blog/errors-are-values)
- [https://go.dev/blog/go1.13-errors](https://go.dev/blog/go1.13-errors)
- [https://github.com/golang/go/wiki/ErrorValueFAQ](https://github.com/golang/go/wiki/ErrorValueFAQ)
- [https://pkg.go.dev/errors](https://pkg.go.dev/errors)

So here we go:

1. When handling errors in Go, we have 3 options:
- **handle it**: if you detect an error, you can take some corrective action (e.g., retry), ignore it on purpose (do nothing about it), or perhaps translate it to another error that is defined by your own package. If you ignore it on purpose, it may be a good idea to leave a comment explaining that it is not an accidental mistake.
- **wrap it**: for some errors, it might make sense to let your callers know the type or value of the error. You should wrap errors from your own packages (see #4), and you may wrap errors coming from other packages (perhaps from a 3rd party or from the standard library). For wrapping errors, you should use `fmt.Errorf` with the `%w` verb. For example, you may wrap errors from your own packages (e.g., `ErrDuplicate` defined by you), or you may choose to wrap errors from another package, such as `fs.ErrNotExist`. Be parsimonious though: every error you wrap ends up becoming part of your contract with your callers. This will be further discussed below.
- **mask it**: sometimes, an error happens because of another error, but a function can't let callers inspect that original error, because that would make these callers dependent on the source of the original error. For example, let's say you have a `store` package that happens to use the [sql](https://pkg.go.dev/database/sql) package as an implementation detail. If you let callers check against `sql.ErrNoRows`, you can't simply port your `store` package to that fancy new NoSQL database without continuing to return `sql.ErrNoRows`, or your risk breaking your callers. For masking errors, you should use `fmt.Errorf` with the `%v` verb.

2. Sometimes you will see references to **sentinel errors** when talking about errors in Go. A sentinel error is one that is defined with a fixed value by a package. For example, [`http.ErrBodyNotAllowed`](https://cs.opensource.google/go/go/+/refs/tags/go1.21.1:src/net/http/server.go;l=41) is a sentinel error. You can also make your own sentinel errors by defining them somewhere in a package:

    ```go
    var ErrValidation = errors.New("validation error")
    ```

    Based on my observations so far, sentinel errors are usually named starting with `Err`. Callers trying to detect a sentinel error should use `errors.Is(err, mypackage.ErrValidation)`.

3. Other times, you will see references to **error types**. An error type is a struct that implements the [`error` interface](https://go.dev/blog/error-handling-and-go#the-error-type). For example, [fs.PathError](https://pkg.go.dev/io/fs#PathError) is an error type. You can define your own error types like this:

    ```go
    type ValidationError struct {
        Field string
    }

    func (e *ValidationError) Error() string {
        return fmt.Sprintf("validation error: %s", e.Field)
    }
    ```

    Based on my observations so far, error types are usually named ending with `Error`. Callers trying to detect an error of a specific type could use `errors.Is(err, mypackage.ErrValidation)` if supported by that error type (see #4 below), but more commonly they will use code like this:
    ```go
    var vErr *mypackage.ValidationError
    if errors.As(err, &vErr) {
        // Handle/wrap/mask vErr.
    } else {
        // Run another check, or handle/wrap/mask err.
    }
    ```

4. Sentinels should not be returned directly, but wrapped -- even if there's nothing to add to the error message. This forces callers to always verify the error with `errors.Is`. That subtle difference means that later, you can turn a sentinel error into a custom error type without breaking backwards compatibility. Consider the following code:
    ```go
    func Validate(data string) error {
        if len(data) > 10 {
            return fmt.Errorf("%w: data: too long", ErrValidation)
        }
        return nil
    }
    ```

    By wrapping `ErrValidation`, your callers will have no option but to check the error as follows:
    ```go
    err := mypackage.Validate(data)
    // err == ErrValidation would not work, because Validate doesn't directly return ErrValidation.
    if errors.Is(err, mypackage.ErrValidation) {
        ...
    }
    ```

    Most importantly, by wrapping a sentinel, you can later decide that `Validate` will return a `ValidationError` to provide more details for callers that want it, without breaking old clients:
    ```go
    var ErrValidation = errors.New("validation error")

    type ValidationError struct {
        Field string
        Message string
    }

    func (e *ValidationError) Error() string {
        return fmt.Sprintf("validation error: %s: %s", e.Field, e.Message)
    }

    func (e *ValidationError) Is(err error) bool {
        // This makes the custom error type compatible with callers who are
        // using errors.Is.
        return err == ErrValidation
    }

    func Validate(data string) error {
        if len(data) > 10 {
            return &ValidationError{Field: "data", Message: "too long"}
        }
        return nil
    }
    ```

    The day that caller decides it needs to know the field that failed validation, it can then start using `errors.As` as explained in #3 above.

5. Avoid sentinels or error types with complex or very specific names:
   - `ErrAccountNotFound` (sentinel error): bad
   - `ErrNotFound` (sentinel error): good
   - `AccountNotFoundError` (error type): bad
   - `NotFoundError` (error type): good

   There are a couple of reasons for this:
   - As a code base grows, keeping this practice avoids an excessive number of sentinels or error types that ultimately all indicate the same thing. Consider if `ErrAccountNotFound` isn't a subcase of the more generic `ErrValidation`, for example.
   - If you do need to provide more details about `ErrNotFound`, these details could go into an error type named `NotFoundError` instead.

6. If you find yourself always wrapping the same sentinel error, it might make sense to write a non-exported function for building it:
    ```go
    func validationError(msg string) {
        return fmt.Errorf("%w: %s", ErrValidation, msg)
    }
    ```

7. Since errors are just values, you should always document the errors returned by your exported functions, be it sentinels or error types. A simple mention of "May return ErrValidation." in the Godoc comment for the function is usually enough. And when changing previously existing code, make sure to keep documentation updated: that's part of the job.

8. When in doubt between using `%w` and `%v`, go with `%v`. It's easier to expose something later with `%w` than to hide it by moving to `%v`. Errors are part of the contract of your package, and it's always a good idea to make this contract as simple as possible.

9. When writing code with multiple layers under your direct control, avoid just wrapping errors along the way. For example, if the `api` package uses the `backend` package that then uses the `store` package, the functions in the `api` package shouldn't be able to check for errors from the `store` using `errors.Is` or `errors.As`. Keeping this practice helps keep layers separate, and eases refactorings later on.

## A bit of shameless self-promotion

After many years of using Go, I can say I really like its approach to error handling. It is simple, straightforward, and it invites developers to think seriously about errors. The tooling in the language is also quite sufficient as of 2023.

Except for one bit: error tracing.

If you are developing complex applications where multiple packages fulfill a request, it helps a lot to get a list of the source code locations behind an error. Think of exceptions and their stack traces.

Unfortunately, Go doesn't say much about this problem. But with just Go 1.20+ features, it's possible to get error tracing with minimal effort. The features that enable this are:
1. The ability to implement custom `Is(error) bool`, `As(any) bool` and `Unwrap() error` functions in error types (introduced in Go 1.13).
2. The ability to retrieve custom verbs when implementing `fmt.Formatter` (introduced in Go 1.20).

And the library that implements this is [`terr`](https://pkg.go.dev/github.com/alnvdl/terr):

```go
ErrValidation := errors.New("validation error")
err1 := terr.Newf("%w: data: too long", ErrValidation)
err2 := terr.Newf("cannot fulfill request: %v", err1)
fmt.Printf("%@\n", err2)
```

You can plan with this in the [Go Playground](https://go.dev/play/p/jqIhbXRtMRP).

The code above will give you a nicely formatted kind-of-stack-trace:
```
cannot fulfill request: validation error: data: too long @ /tmp/sandbox1322570992/prog.go:13
	validation error: data: too long @ /tmp/sandbox1322570992/prog.go:12
```

`terr` only exports four functions, and most of the time you can get away with using just one: `terr.Newf`, which has the same signature and works like `fmt.Errorf`. I tried to make the [documentation](https://pkg.go.dev/github.com/alnvdl/terr) as detailed as possible, with lots of examples, including for the other three more specialized functions.

My dream is that one day terr will be deemed redundant, because Go will have added error tracing natively.

While my dream doesn't come true, you and I can rest at ease knowing that if you decide to use `terr`:
- it will introduce no extra dependencies on your projects;
- you can [get rid of it](https://pkg.go.dev/github.com/alnvdl/terr#readme-getting-rid-of-terr) just using `gofmt`;
- you can [customize the way the error tracing tree gets represented](https://pkg.go.dev/github.com/alnvdl/terr#TraceTree).
