---
title: "WCF vs gRPC"
date: 2019-05-23T18:55:34+01:00
draft: false

categories: ["comparisons"]
tags: ["wcf", "grpc"]
author: "Mark Rendle"
---

One of the alternatives [recommended by Microsoft](https://devblogs.microsoft.com/dotnet/net-core-is-the-future-of-net/) 
for organizations looking for a migration path away from WCF
on .NET Framework is [gRPC](https://grpc.io): a low-overhead, high-performance, cross-platform RPC framework.
The upcoming .NET Core 3.0 has first-class support for gRPC; out of the box, you can create a new project with `dotnet new grpc`.

This creates an ASP.NET Core 3.0 project with the usual `Program.cs` (remember, ASP.NET Core apps are console apps)
and `Startup.cs`, plus an example gRPC service called Greet.

There are two new things in this project: a `greet.proto` file in the `Protos` folder, and a `GreeterService.cs` file
in the `Services` folder. Let's take a look at them.


##### `Protos/greet.proto`

{{< gist markrendle 58b30c96cdf2ee457cc8c40169f3b74b "greet.proto" >}}

This is the Protobuf [Interface Definition Language](https://developers.google.com/protocol-buffers/docs/proto3) (IDL),
a custom language for specifying data structures and service contracts. The first line tells whatever tooling you're
using that you're using v3 of the language; the default is v2. The second line provides a name for the generated package;
in .NET Core this will be used as the namespace for the generated code.

Then we have the actual definitions: a `service` and two `message`s.

The `service` is like the `[ServiceContract]` interface in a WCF project; it's a declaration of the methods (or endpoints) that the
service will have. Each method is declared using the `rpc` token, then the name of the method, the parameter type, and the return type.
gRPC methods can only have a single parameter, which must be a known `message` type. You can't just use `int` or `string` here.

The `message` is like the `[DataContract]` types in WCF; it declares a data transport object (DTO). Each field has a type, a name,
and an index, which starts at `1`. As you add new fields, you increment the index.

The `proto` file is used to generate all the gRPC plumbing code, including data objects and their Protobuf serialization
implementations, and base classes for your services. It may seem strange to declare these things in a separate text file if
you're used to working in WCF using C# or VB, but there are code generators for almost every language and platform currently
in use. Once you've got your `proto` file, you can generate clients for Java, Python, JavaScript (Node and browser), C/C++, Ruby,
Go, Rust and more.

And trust me: you don't want to see the code that it generates. Let's just say it prioritizes performance over beauty.

##### `Services/GreeterService.cs`

{{< gist markrendle 58b30c96cdf2ee457cc8c40169f3b74b "GreeterService.cs" >}}

This is where you write your implementation code. The `Greeter.GreeterBase` base class is generated by Protobuf, which is
integrated with MSBuild so you don't need to install anything beyond the `Grpc.AspNetCore.Server` and `Google.Protobuf`
NuGet packages. That base class defines virtual methods for all the declared `rpc` methods in the `proto` file, with request
and response types that have also been automatically generated. (If you're following along and wondering where the code for
all these classes is, it's in the `obj` folder in your project.)

The `ServerCallContext` is the gRPC equivalent of `HttpContext` in an ASP.NET Core MVC application. There you can access
authentication information, headers, and other metadata about the call. As with a WCF contract, you simply handle the request
and return a response object.

##### `Startup.cs`

{{< gist markrendle 58b30c96cdf2ee457cc8c40169f3b74b "Startup.cs" >}}

The ASP.NET Core `Startup` class is responsible for the application setup, similar to `Global.asax` in an ASP.NET application.
Here we register the gRPC services with the Dependency Injection provider (line 5), and add specific services with the
`UseEndpoints` call (lines 12-15). We can now run this application from the command line on Windows or Linux, or in IIS
on Windows, and connect to it from any client application that has used the same `.proto` file to generate client access
code.

##### Client code

You can drop a `.proto` file in a .NET Core 3.0 client application with the `Grpc.Core` and `Grpc.Tools` NuGet packages
and it will generate a client class for accessing the server, much like generating client classes using "Add Service Reference"
in a .NET application. Here's an example of a simple console application using the client class generated from the `Greeter`
proto file:

{{< gist markrendle 58b30c96cdf2ee457cc8c40169f3b74b "Client.cs" >}}

### Performance comparison

**UPDATE:** *These tests were run using the basic HTTP binding for WCF, which is not the most appropriate
binding to compare to gRPC. [This follow-up post](/posts/wcf-vs-grpc-round-2) shows a more representative
comparison, using NetTCP binding for WCF (and also a modified version of the gRPC solution).*

So how does gRPC perform compared to WCF? Here are the results from BenchmarkDotNet comparing a sample WCF service to an
equivalent .NET Core 3.0 gRPC service, which was generated using an automated conversion tool currently in development.

**The WCF ServiceContract implementation:**

{{< gist markrendle 58b30c96cdf2ee457cc8c40169f3b74b "DummyServiceWcf.cs" >}}

**Auto-converted code for gRPC:**

{{< gist markrendle 58b30c96cdf2ee457cc8c40169f3b74b "DummyServiceGrpc.impl.cs" >}}

**Auto-generated gRPC wrapper code:**

{{< gist markrendle 58b30c96cdf2ee457cc8c40169f3b74b "DummyServiceGrpc.cs" >}}

Note that in this case, because the original `GetDataStream` method returned an `IEnumerable`, the conversion tool
created a method with a `stream` response, which lets gRPC sends responses asynchronously. The wrapper code takes care
of running this from the original `IEnumerable`-returning method from the WCF code.

Benchmarking these two implementations with the auto-generated client code for each server gives the following results:

| Method |     Mean |     Error |    StdDev |  Gen 0 | Gen 1 | Gen 2 | Allocated |
|------- |---------:|----------:|----------:|-------:|------:|------:|----------:|
|   gRPC | 0.971 ms |  18.39 us |  26.37 us | 3.9063 |     - |     - |   1.48 KB |
|    WCF | 1.406 ms | 0.0278 ms | 0.0390 ms | 3.9063 |     - |     - |  19.38 KB |

So it looks like gRPC is about 33% faster than WCF, and allocates nearly 95% less memory, which is a pretty good win
for an auto-converted project.

*Again, please refer to [this post](/posts/wcf-vs-grpc-round-2) for a more accurate comparison using WCF NetTCP binding.*

Here's a video showing the conversion tool (working title "Uplift" may change before release) in use in VS2019, and the
benchmarks running:

{{< vimeo 338031391 >}}

To receive updates on the tools shown here, and other new features under development,
please subscribe to this site's [RSS feed](https://unwcf.com/index.xml).
