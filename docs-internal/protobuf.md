# Helidon Protobuf Support Proposal

Add support for protobuf entities in the Helidon WebServer.

I.e Provide a way to send and receive protobuf messages.

## Examples

### Enable Protobuf Support

The readers and writers for the protobuf entity types will be registered by a service, similar to how other media types are supported in the webserver. All the examples below assume that the ProtobufSupport is registered as follow:

```java
Routing.builder()
    .register(ProtobufSupport.create(GreetingProtos.getDescriptor()))
    // user defined handlers...
    .build());
```

The ProtobufSupport builder accepts an optional FileDescriptor instance that will be used to lookup the parsers. If not supplied by the user, the parsers will be looked up using reflection and cached for subsequent requests.

### Process a request payload

```java
request.content().as(GreetingProtos.Greeting.class).thenAccept((greeting) -> {
    response.send("received" + greeting.getGreeting());
});
```

### Return a response

```java
response.send(GreetingProtos.Greeting.newBuilder()
    .setGreeting("Hello World!")
    .build());
```

## About protobuf

Protobuf is basically a serialization framework with a toolchain to generate code, it is agnostic of transport protocol.
One can serialize to and deserialize from bytes. See https://developers.google.com/protocol-buffers

Data structures are defined in a `.proto` file using the protocol buffers langague. See https://developers.google.com/protocol-buffers/docs/proto3

Code is generated by running the protocol buffer compiler `protoc`. This compiler not written in java but instead is compiled to native code ; executables are provided for all major platforms. A Maven plugin is available (`org.xolstice.maven.plugins:protobuf-maven-plugin`), it integrates `protoc` with Maven projects.

The generated code can be used to serialize and deserialize protobuf data types. It also provide a reflection mechanism to introspect the data types.

The protobuf compiler has a plugin interface that can be used to generate additional code. It is langage agnostic and uses the standard input and standard output. A code generation request is sent in the standard input as serialized bytes, the code generation response is sent into the standard output as serialized bytes ; a plugin uses the provided code for the request and resposne object to deserialize the request, build and serialize a response.

Any type declared in a `.proto` file will have a matching generated java class implementing `com.google.protobuf.Descriptors.Descriptors`. The `.proto` file also has a matching generated java class implementing `com.google.protobuf.Descriptors.FileDescriptor`. This class is the main can be used as the main "reflection" entrypoint.

Any protobuf object instance is a message, represented in java by the type `com.google.protobuf.Message`.

There are different level of messages:
 - `com.google.protobuf.Message` extends `com.google.protobuf.MessageLite`
 - `com.google.protobuf.MessageLite`

The main difference is that there is no "reflection" methods available from `MessageLite`.

## Another Media Type

Protobuf is transport protocol agnostic. We can support it with either HTTP1 or HTTP2.

There is no standard content-type header for protobuf, however there are some popular ones that are commonly used.

The popular headers are as follow:

- `application/x-protobuf ; messageType=x.y.Z`
- `application/x-protobuffer ; messageType=x.y.Z`

These headers can carry the message type as parameters. `application/octet-stream` is sometimes used as well with protobuf however it only describes a binary stream content and does not carry the message type information.

## Receiving Protobuf messages

Deserializing a message requires having a handle on a parser for the type that matches that message. If the message type can be determined from the request headers and if the user has provided a `FileDescriptor`, the parser can be looked-up dynamically.

If the message type is not available, the parser can be looked-up using reflection: invoke the `.parser()` static method on the requested class.

## Sending Protobuf messages

Sending protobuf message is easier given that the user provides an instance of a message and that the base type `MessageLite` provides serialization messages (`message.toByteArray()`).

## Netty Support

Since protobuf is transport agnostic there isn't much needed from Netty. However things get a little complicated because of protobuf's `Base 128 varints`, see https://developers.google.com/protocol-buffers/docs/encoding#varints

Netty provides ways to deal with that:
- https://github.com/netty/netty/blob/netty-4.1.30.Final/codec/src/main/java/io/netty/handler/codec/protobuf/ProtobufVarint32FrameDecoder.java
- https://github.com/netty/netty/blob/netty-4.1.30.Final/codec/src/main/java/io/netty/handler/codec/protobuf/ProtobufVarint32LengthFieldPrepender.java

## RPC

Protobuf defines service with RPC methods, and google provides some special `.proto` includes providing HTTP annotations.

These 2 things while in the protobuf space are strongly associated with GRPC. From the perspective of protobuf GRPC is one implementation of a toolchain for RPC services that consists of a `protoc` plugin and a stack that supports various langagues.

It would it be possible for us to define our own "Helidon" flavor of protobuf RPC services, however they wouldn't interoperate with GRPC generated code ; given the popularity of GRPC the usability of such RPC services is questionnable.

Support for the RPC and interoperability with GRPC clients should be done separately from the basic protobuf support.

## Open Issues

- Do we need to use `ProtobufVarint32FrameDecoder` and `ProtobufVarint32LengthFieldPrepender` and would that impact other non protobuf message. If yes, then how do we deal with it ? Do we "multiplex" our ForwardingHandler ?

