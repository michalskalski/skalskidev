+++
title = "About absence of a sum type in Go"
date = 2026-02-11
+++

Recently I was building an infrastructure-as-code library for Google Cloud. One of the things I needed was a function to create [Eventarc](https://docs.cloud.google.com/eventarc/docs) triggers. The function needed to accept a destination - where should events be delivered?

Looking at the [Eventarc REST API](https://docs.cloud.google.com/eventarc/docs/reference/rest/v1/projects.locations.triggers#Destination), the destination is defined as a union field. It can be one of several things: a Cloud Run service, an HTTP endpoint, a Cloud Function, a GKE service, or a Workflow. Exactly one must be set.

```jsonc
{
  // Union field descriptor can be only one of the following:
  "cloudRun": { },
  "cloudFunction": "...",
  "gke": { },
  "workflow": "...",
  "httpEndpoint": { }
}
```

This is an example of [sum type](https://en.wikipedia.org/wiki/Tagged_union) (also called a tagged union or discriminated union). A value that can be exactly one of N known alternatives. Go doesn't have sum types, and how you work around that absence turns out to be an interesting exercise.

My function signature looked something like this:

```go
func NewTrigger(name string, dest Destination, eventType EventType) error
```

The question was how to define `Destination` so that callers are guided toward correct usage and invalid states are caught early.

The simplest thing that could work was a struct. Unexported fields, a constructor function:

```go
type Destination struct {
    cloudRun *cloudRunConfig
}

type cloudRunConfig struct {
    service string
    region  string
}

func CloudRunDestination(service, region string) Destination {
    return Destination{cloudRun: &cloudRunConfig{service: service, region: region}}
}
```

It looks reasonable. The fields are unexported, so callers can't mess with the internals. They use the constructor, they get a valid destination. But there's a problem that's easy to miss.

In Go, every type has a zero value. For a struct, that means all fields set to their zero values. Which means this compiles perfectly:

```go
dest := Destination{}
err := NewTrigger("my-trigger", dest, someEvent)
```

The caller passed an empty `Destination`. No Cloud Run config, no HTTP endpoint, nothing. The code compiles. The linter is happy. You won't find out until runtime apply the configuration when it fails with some cryptic error about a missing destination.

It gets worse as you add more destination types. With two fields someone could theoretically set both, or neither. With five you need validation that checks "exactly one of these N fields is non-nil". It's doable, but you're reimplementing by hand what a type system could give you.

I went looking for better patterns and found discussions around "sealed interfaces" - a pattern used in Go's [standard library](https://cs.opensource.google/go/go/+/refs/tags/go1.26.0:src/testing/testing.go;l=888-918). There's a [Reddit thread](https://www.reddit.com/r/golang/comments/1mlhqt4/breaking_the_misconception_of_the_sealed_interface/) debating the merits of this approach. The idea is to define an unexported interface that only types within your package can implement. Since the interface or its method is unexported, nobody outside the package can satisfy it.

Here's what I ended up with:

```go
type Destination struct {
    dest destinationType
}

type destinationType interface {
    isDestination() // unexported method seals the interface
}

type cloudRunConfig struct {
    service string
    region  string
}

func (*cloudRunConfig) isDestination() {}

func CloudRunDestination(service, region string) Destination {
    return Destination{dest: &cloudRunConfig{service: service, region: region}}
}
```

_[full example on go playground](https://go.dev/play/p/6XIDSemn1kn)_

I think it is better approach than the previous one. The `destinationType` interface can only be implemented inside the package, so the set of valid destination types is closed. You can't accidentally set multiple variants because there's only one field. When I add an HTTP endpoint variant later, it's just another type that implements `isDestination()`, and `Destination` still holds exactly one of them.

But the zero value problem didn't go away:

```go
dest := Destination{} // dest.dest is nil — compiles, still invalid
```

I still need a runtime check inside `NewTrigger`:

```go
if dest.dest == nil {
    return errors.New("destination not set; use a constructor like CloudRunDestination()")
}
```

It works. The error message is clear. The invalid state is caught before it causes damage downstream. But it's caught at runtime, not at compile time. The compiler doesn't help you here.

The Go community has been discussing sum types for years. There's [proposal #19412](https://github.com/golang/go/issues/19412) from 2017 and a more specific [proposal #57644](https://github.com/golang/go/issues/57644) that suggests extending interfaces to support type unions:

```go
type Destination interface {
    CloudRunConfig | HttpEndpointConfig | GKEConfig
}
```

This would make the sealed interface pattern a first-class language feature instead of an idiom you have to know about. One of the [comments on that proposal](https://github.com/golang/go/issues/57644#issuecomment-1373008575) describes almost exactly the pattern I ended up using - an unexported method to seal the interface - and points out how the proposal would reduce the boilerplate and make the intent explicit.

But there's a catch that the proposal authors themselves acknowledge. In Go, every interface has a zero value, and that zero value is `nil`. Even with union interfaces in the language, you'd still be able to write:

```go
var dest Destination // nil — compiles fine
```

The proposal text says it directly:

> this is a form of sum type in which there is always another possible option, namely nil. Sum types in most languages do not work this way, and this may be a reason to not add this functionality to Go.

The zero value principle is deeply embedded in Go's design. It's a deliberate choice that makes many things simpler. You can declare a `sync.Mutex` or a `bytes.Buffer` and use it immediately, no constructor needed. But when you're modeling something where "nothing" is not a valid state, that same principle works against you.

This got me curious about how Rust handles the same problem. In Rust, the destination type would be:

```rust
enum Destination {
    CloudRun { service: String, region: String, path: Option<String> },
    HttpEndpoint { uri: String },
    Gke { cluster: String, location: String, namespace: String, service: String },
    Workflow(String),
}
```

_[example on rust playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&gist=80dc35f74027277a61d2bc3339dc67fd)_

That's it. No wrapper struct. No sealed interface. No unexported methods. No constructors needed. The enum definition says everything: a `Destination` is one of these four things, and nothing else.

There's no zero value. You can't construct a `Destination` without choosing a variant. The type system forces the decision, and it forces it at compile time.

When you consume the destination, exhaustive pattern matching makes sure you handle every case:

```rust
fn configure_trigger(dest: Destination) {
    match dest {
        Destination::CloudRun { service, region, path } => { /* ... */ }
        Destination::HttpEndpoint { uri } => { /* ... */ }
        Destination::Gke { .. } => { /* ... */ }
        Destination::Workflow(name) => { /* ... */ }
    }
}
```

If I add a fifth variant six months later, every `match` expression that doesn't handle it becomes a compile error. The compiler tells me exactly where I need to update. In Go, adding a new type that implements `destinationType` compiles everywhere silently. I have to search through type switches and hope I didn't miss any. For an internal library where my team owns all the code - that exhaustiveness is what I want. A missed destination type means a misconfigured trigger that fails at deploy time, so I'd rather hear about it from the compiler. For a public library, that same property becomes disruptive - adding a new variant breaks downstream consumer and forces a major release. Rust acknowledges this with the [`#[non_exhaustive]`](https://doc.rust-lang.org/reference/attributes/type_system.html#r-attributes.type-system.non_exhaustive) attribute, which lets you keep exhaustiveness within your own crate while allowing external callers to gracefully ignore new variants.

Go's simplicity is real and valuable. The zero value concept eliminates an entire class of "forgot to initialize" bugs that other languages struggle with. The sealed interface pattern I used works in practice - the runtime error is caught early, the message is helpful.

However there's something satisfying about a language that lets you say "a destination is one of these things" and then enforces that statement everywhere, at compile time, with no extra work. The Rust enum doesn't need discipline or idiom knowledge. It doesn't need runtime validation because the invalid state simply doesn't exist.
