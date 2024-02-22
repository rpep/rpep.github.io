---
title: "Go first impressions"
date: 2024-02-01T21:57:58+00:00
draft: false
featured_image: "/images/go.jpeg"
---

Go first came onto my radar about 9 years ago when I came across [MuMax](https://github.com/mumax/3) during my PhD studies. While I've touched it briefly, I wouldn't consider myself to have used it in anger until recently, when I started a job that uses it as one of it's primary languages for API development. I've had experience in writing C and C++ code, but more recently I've done a lot of Python,
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

They're much more flexible than they first appear however, as they can also be used to add methods that operate on composite types which is quite fun:

```go
type Rectangles []Rectangle

func (rs Rectangles) Print() {
    for i := range(rs) {
        fmt.Printf("%f x %f\n", rs[i].Length, rs[i].Width)
    }
}

func main() {
	rectangles := Rectangles{
		{
			Length: 1.0,
			Width:  2.0,
		},
		{
			Length: 2.0,
			Width:  4.0,
		},
	}

	rectangles.Print()
}
... 



```

It's even possible to add receivers for built in types by labelling them with your own name in this way.

# The not so good

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

```go
list := []string{"a", "b", "c"}
Print(...list)
```

But you can't spread more than one item into this:

```go
Print(...list1, ...list2)
```

## Where art thou enums...?

The closest you can get to defining an enum in Go is creating your own type:

```go
type MyFakeEnum string

const (
   MyFakeEnumA MyFakeEnum "a"
   MyFakeEnumB MyFakeEnum "b"
   MyFakeEnumC MyFakeEnum "c"
)
```

Syntactically this gets you somewhere close to an enum, but without any of the type safety a true enum would give you. For example, it's possible to do the following:

```go
func MyFunc(val MyFakeEnum) {
    ...
}

MyFunc("some value")
```

without any compiler error. Similarly, things like the built-in JSON parser will not error when marshalling into a struct, since the true type of the field is a string.

## Dependency management

I'm really not keen on the way that Go dependency management is linked to hard-coded URLs for package names. It can be frustrating when/if things inevitably move around, and I find that it also makes package names annoyingly long within the IDE.

More than once in my career, I've moved source control systems (and not just between Git servers - additionally between IBM ClearCase to Git, Mercurial to Git...). When package names have these coupled to the source control system, it causes issues, and it makes for quite noisy commits too. Going from `bitbucket.org/rpep/packagename` to `github.com/rpep/packagename`
would require changes to at least the `go.mod` and any source files referencing that package, but not only that, it breaks links that each library has to it's own dependencies if they're also hosted on a different platform. This means practically needing to add `replace` directives to your project to stop dependency resolution from failing. Vendoring your dependencies mitigates this somewhat, but it just delays the pain.
