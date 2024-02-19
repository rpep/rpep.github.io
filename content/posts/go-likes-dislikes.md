---
title: "Things I like and dislike about Go"
date: 2024-02-01T21:57:58+00:00
draft: false
featured_image: "/images/car.jpg"
---

Go first came onto my radar about 9 years ago when I came across [MuMax](https://github.com/mumax/3) during my PhD studies. Other than playing with modifications to this and a few Lambda functions, I've not had much
time to play with it until recently, when I started a job that uses it as one of it's primary languages for API development. I've had experience in writing C and C++ code, but more recently I've done a lot of Python,
so going back to Go has felt oddly like a throwback to those compiled languages I've used before, with a bit of modern sparkle. Now I've had a few months to reflect, here are some of my favourite parts and least favourite parts.

# The Good

## Compiler Portability

One of the things I really like with Go is that you can very straightforwardly cross compile, (so long as you don't need to utilise C libraries and so can disable the use of `cgo`). Want to develop on Mac but build for Linux? Need to compile for a different architecture? That's not an issue; just set the appropriate flag:

```bash
CGO_ENABLED=0 GOOS=linux GOARCH=arm64 go build ./...
```

To say that this is friendly compared to the C/C++ equivalent would be an understatement. There, you'd generally need to install a new compiler (e.g. gcc-mingw-w64) for each platform, install appropriate development headers for the target platform and more.

## Reflection

It's really nice in a compiled language to be able to reflect on types. Most people using Go are familiar with this from the standard library, for e.g. for directing how fields get converted from a struct into JSON or vice versa:
```go
type Rectangle struct {
   Length float64 `json:"length"`
   Width  float64 `json:"width"`
}
```
There are some great third party packages that make use of this in clever ways. The [validator](https://github.com/go-playground/validator) library lets you use additional tags for validation of struct members.


## Receiver methods

For the most part, receiver methods are used to implement object-like behaviour on structs, for e.g.:
```go
func (r Rectangle) Area() float64 {
    return r.Length * r.Width
}
```

They can also be used to add methods that operate on composite types however, which is quite fun:
```go
func (rs []Rectangle) Print() {
    for i := range(rs) {
        fmt.Printf("%s x %s\n", rs[i].Length, rs[i].Width)
    }
}
```

# The not so good

## Dependency management

I'm really not keen on the way that Go dependency management is linked to hard-coded URLs for package names. It can be frustrating when/if things inevitably move around, and I find that it also makes package names annoyingly long within the IDE.

More than once in my career, I've moved source control systems (and not just between Git servers - additionally between IBM ClearCase to Git, Mercurial to Git...). When package names have these coupled to the source control system, it causes issues, and it makes for quite noisy commits too. Going from `bitbucket.org/rpep/packagename` to `github.com/rpep/packagename`
would require changes to at least the `go.mod` and any source files referencing that package, but not only that, it breaks links that each library has to it's own dependencies if they're also hosted on a different platform. This means practically needing to add `replace` directives to your project to stop dependency resolution from failing. Vendoring your dependencies mitigates this somewhat, but it just delays the pain.

## Simple but sometimes clunky

Go is rightly championed as a language that's simple, and which avoids a lot of baggage from languages 

But sometimes that does make it feel a bit clunkier than it needs to be. When writing a function, it can take an arbitrary number of arguments, similar to C variadic argument syntax:
```go
func Print(args... string) {
    for i := range(args) {
        fmt.Printf("%s\n", args[i])
    }
}
```
This can be called by unpacking a list using the spread operatorlike so:
```
list := []string{"a", "b", "c"}
Print(...list)
```

But you can't spread more than one item into this:
```
Print(...list1, ...list2)
```



