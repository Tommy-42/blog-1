---
title: "So you want to use GoGo Protobuf"
date: 2018-02-18
subtitle: "Best practices for using GoGo Protobuf with gRPC"
tags: ["golang","protobuf", "gogoprotobuf", "grpc", "grpc-gateway"]
---

## Introduction

In the Go protobuf ecosystem there are two major implementations
to choose from. There's the official
[golang/protobuf](https://github.com/golang/protobuf), which uses
reflection to marshal and unmarshal structs, and there's
[gogo/protobuf](https://github.com/gogo/protobuf), a third party
implementation that leverages type-specific marshalling code for
extra performance, _and_ has many cool extensions you can use to
customize the generated code. `gogo/protobuf` has been recommended
as the best choice of Go serialization library in a
[large test of different implementations](https://github.com/alecthomas/go_serialization_benchmarks#recommendation).

Unfortunately, the design of `golang/protobuf` and the gRPC ecosystem
makes it hard to integrate third party implementations,
and there are certain situations where using `gogo/protobuf`
with gRPC can break unexpectedly, at _runtime_.
In this post I will try to cover best practices for
working with `gogo/protobuf` and gRPC.

## gRPC

gRPC is designed to be payload agnostic, and will work out of the
box with `gogo/protobuf`, as while it imports `golang/protobuf`,
it only uses it to
[type assert incoming interfaces](https://github.com/grpc/grpc-go/blob/dfa18343df54bda471a4b53677aa7c0d0df882d1/encoding/proto/proto.go)
into interfaces that are equally supported by all `gogo/protobuf` types.
No changes necessary here.

### Reflection

gRPC has this cool thing called
[`server reflection`](https://github.com/grpc/grpc-go/blob/master/Documentation/server-reflection-tutorial.md),
which allows a client to use a gRPC server without having to use
the servers protofile, dynamically, at runtime. Some third party
tools such as [`kazegusuri/grpcurl`](https://github.com/kazegusuri/grpcurl)
and [`fullstorydev/grpcurl`](https://github.com/fullstorydev/grpcurl)
(popular pun) have support for dynamic reflection based requests today.

Unfortunately, `gogo/protobuf` is currently not working perfectly
with server reflection, because the grpc-go implementation is
[very tightly coupled](https://github.com/grpc/grpc-go/issues/1873)
with `golang/protobuf`. This presents a couple of different scenarios
where using `gogo/protobuf` may or may not work:

1. If you use just the `protoc-gen-gofast` generator, which simply
    generates type specific marshalling and unmarshalling code,
    you'll be fine. Of course, using `protoc-gen-gofast` still
    comes with downsides, such as having to
    [regenerate the whole proto dependency tree](https://github.com/gogo/protobuf/issues/325).
2. If you use `protoc-gen-gogo*`, unfortunately, reflection will
    not work on your server. This is because
    [`gogo.pb.go`](https://github.com/gogo/protobuf/blob/master/gogoproto/gogo.pb.go)
    does not register itself with `golang/protobuf`, and reflection
    recursively resolves all imports, and will complain of
    `gogo.proto` not being found.

This is of course quite disappointing, but I've discussed
with Walter Schulze (the maintainer of `gogo/protobuf`)
how best to solve this and raised
[an issue against grpc-go](https://github.com/grpc/grpc-go/issues/1873).
If the maintainers of grpc-go do not want to make it easier
to use with `gogo/protobuf`, there are other alternatives.
I'll update this post once I know more.

## gRPC-Gateway

The gRPC-Gateway is another popular project, and at first it
might seem completely compatible `gogo/protobuf`. However,
the gRPC-Gateway [does not work with `gogo/protobuf` registered enums](https://github.com/grpc-ecosystem/grpc-gateway/issues/320).
The default JSON marshaller used by gRPC-Gateway is also unable
to marshal [non-nullable non-scalar fields](https://github.com/gogo/protobuf/issues/178).
This is just another example of a library or tool using
`golang/protobuf` directly, thus making it incompatible with
`gogo/protobuf`.

Fortunately, workarounds exist for both of these problems.
Using the [`goproto_registration` extension](https://github.com/gogo/protobuf/blob/master/extensions.md#goprotobuf-compatibility)
of `gogo/protobuf` will ensure enum resolution works.
As for the JSON marshalling problem, you have to use
the [`cockroachdb` fork of `golang/protobuf/jsonpb`](https://github.com/cockroachdb/cockroach/blob/f9f3d43ca646b6b8a84c6d09b091936ac30bc1ae/pkg/util/protoutil/jsonpb_marshal.go#L35) with the gRPC-Gateway
[`WithMarshaler` option](https://github.com/grpc-ecosystem/grpc-gateway/blob/master/runtime/marshaler_registry.go#L85).
See cockroachdb for
[an example](https://github.com/cockroachdb/cockroach/blob/f9f3d43ca646b6b8a84c6d09b091936ac30bc1ae/pkg/server/server.go#L1037).

# Conclusion

Unfortunately, while `gogo/protobuf` delivers awesome customization
options and faster marshalling, getting it working well with the
larger gRPC ecosystem is complicated. `gogo/protobuf` has it as a
stated goal to be merged back into `golang/protobuf`, and
[recent discussions](https://groups.google.com/d/msg/golang-nuts/F5xFHTfwRnY/sPv5nTVXBQAJ)
have been positive, but it's hard to say whether it'll lead to anything.
There is [an open issue](https://github.com/golang/protobuf/issues/280)
discussing the possibility of type specific marshalling and unmarshalling code.
In a perfect future, we'd have some or all of the customizability and speed of
`gogo/protobuf` with the stability and support of `golang/protobuf`.

I made
[a repo for experimenting](https://github.com/johanbrandhorst/gogoproto-experiments)
with various go proto generators
that you can check out if you want to make your own tests.

If you enjoyed this blog post, have any questions or input,
don't hesitate to contact me on
[@johanbrandhorst](https://twitter.com/JohanBrandhorst) or
under `jbrandhorst` on the Gophers Slack. I'd love to hear
your thoughts!
