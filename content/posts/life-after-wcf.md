---
title: "Life after WCF"
date: 2019-05-16T16:52:52+01:00
draft: false
noSummary: false

categories: []
tags: []
author: "Mark Rendle"
---

At the //BUILD 2019 conference, Microsoft announced that after .NET Core 3.0
(which is due to be released in Q3 of 2019) the [next major release would
be called .NET 5](https://devblogs.microsoft.com/dotnet/introducing-net-5/).
This is intended to prevent the extra confusion that might
result from there being a .NET 4.x and a .NET Core 4.x existing in the same
universe. But .NET 5 will be the evolution of .NET Core, not .NET Framework.

It seems pretty clear that the old .NET Framework is going
into legacy support mode, and while Microsoft are committed to continuing to support
it long into the future, with security updates and so forth, it is unlikely that
there will be much innovation or development on the platform. With both WPF and Windows Forms
being ported to .NET Core 3.0, and made open source like the rest of Core, the future of
those technologies seems assured and the migration path should be straightforward.

<!--more-->

##### What about WCF?

Windows Communication Foundation is not being ported to .NET 5, and the server-side
part of WCF is not&mdash;at least, not yet&mdash;an 
open source technology, so currently there is no option for a community effort to fork the project
and attempt to port it to .NET Core themselves.

Even if there were that option, would it be the right way to go? WCF was originally released
as part of .NET Framework 3.0 at the end of 2006. The world of distributed systems was very
different then: JSON was barely a thing, and Remote Procedure Calls (RPC) using XML-serialized SOAP messages was the 
prevailing standard for Service Oriented Architectures. The term "microservices" would not
be coined for another five years. And technology moves forward ever-faster, so in the years
since WCF was conceived, people have come up with better solutions to the problems it sought
to solve.

### Alternatives to WCF

Here are some possible alternatives for creating distributed systems, service-oriented architectures,
or microservices, that are popular and well-supported today.

##### Basic HTTP APIs

At the simplest end, we now have basic HTTP APIs: you make a request to a URI, and it responds
with data, hopefully in the format you requested (JSON, XML, etc.). This includes APIs that
strictly conform with the ReST architectural style, but also simple "CRUD-over-HTTP" APIs that just
use GET, PUT, POST and DELETE requests to retrieve, store and manage data. These APIs can apply
security using any of the available HTTP authentication options, and can be made secure simply
by applying SSL/TLS to the connection. For basic SOAP-over-HTTP or SOAP-over-TCP request/response
WCF applications, an HTTP API is a good potential alternative.

HTTP APIs created with .NET Core 2.x can be documented using Swagger, which includes the ability
to read the API metadata from a known endpoint and [generate client library code](https://swagger.io/tools/swagger-codegen/).
Visual Studio 2019 and 2017 both include support for adding "REST API" clients to a project
from a Swagger URL.

##### gRPC

Initially developed at Google, gRPC uses HTTP/2 for transport and Protocol Buffers (Protobuf)
as the wire format. It is a promising drop-in replacement for some of WCF's more complicated
abilities like full-duplex messaging over TCP. gRPC has a very similar approach to WCF in that
the creator of the service declares contracts, in this case using a dedicated Interface Definition
Language (IDL), from which the base code for both server and client can be generated. The code is
very efficient and takes care of serializing/deserializing data-transfer objects and processing
requests and responses. Because the Protobuf serialization format is very compact, gRPC generates
much less network traffic than a SOAP service, or a JSON or XML HTTP API. Protobuf is also faster
and more memory efficient than text-based serialization, resulting in meaningful performance and
scalability gains.

gRPC also supports request/response streaming, where a client can asynchronously send a stream of
requests and receive a stream of responses, which is a possible alternative to full-duplex WCF
services.

In the current ASP.NET Core 3.0 preview there is already support for this code-generation step,
fully integrated with the MSBuild process, and in Visual Studio 2019 the experience is almost seamless,
with tooling support like syntax highlighting and IntelliSense for `.proto` files.

There are gRPC code generators available for most platforms or languages, including C/C++, Go, Java,
Node, PHP, Python, Ruby and in-browser JavaScript, so interop between systems is easy to implement. Protobuf
is also resilient to changes in object structures, helping to avoid breaks when a service adds new fields
to an object, for example.

##### WebSockets / SignalR

WebSockets are an open standard for maintaining persistent connections between client and server, and
sending arbitrary messages in both directions. That makes them another potential alternative for WCF's
full-duplex messaging. Although WebSockets were originally designed for browsers, they can also be used
from C#, Java and other platforms.

Because WebSockets are a low-level networking solution, they are completely agnostic about the format of
messages sent and received, so you can use JSON, Protobuf, MessagePack or any format you like, as long
as the code at either end of the socket can parse it.

SignalR is an ASP.NET project that provides a very simple wrapper over WebSockets, making it easy to create
servers as part of an ASP.NET Core project, and to access those servers from JavaScript, C# and Java applications.
It has out-of-the-box support for serialization using JSON and MessagePack (a binary protocol similar to Protobuf),
and messages sent over WebSockets have minimal network overhead. Microsoft Azure has a SignalR as a Service
offering which makes it easy to deploy, maintain and scale SignalR applications in the cloud.

### Are these long term solutions?

There are two parts to this question. First, will these protocols themselves be around for long enough to
justify investing in them? And second, will Microsoft continue to provide first-class support for them?

As to the first part, HTTP and WebSockets are both standards of the Open Web, and will likely be around for
a long time. gRPC and Protobuf are Google technologies and are not governed by an independent standards
body, but they are well-documented and completely open source (Apache 2.0 for gRPC and BSD for Protobuf).
Adoption of gRPC is sufficiently widespread and includes large enough companies that even if Google themselves
stopped developing and using it, the community would continue to maintain it. SignalR is a Microsoft technology,
like WCF, and so there is always the possibility that they will stop developing it at some future point. But unlike
WCF, SignalR is fully open source (Apache 2.0) and could be forked and maintained by the community if necessary.

As to the second part, Microsoft seem committed to supporting popular, open technologies, and this may well
mean that in the future, if a "better" alternative to something becomes available and sees widespread adoption,
they might focus more development energy on that alternative. One may hope that this will be an additive
approach, rather than just replacement. But in the worst case, we are talking about open source solutions
which are developed in the open, rather than the proprietary, closed-source solutions of the old Microsoft.
If, at some point in the future, support for a particular part of the .NET or ASP.NET framework is deprecated,
a community-maintained fork of the relevant project could continue to support it.

In the end, no technology, language, framework or solution comes with a lifetime guarantee. Things get deprecated,
new things arrive to replace them, and our job as developers is to try to keep up. The best we can do is look
carefully at the available options, pick the one that is right for our project right now, and try to write our own
business and application logic in a way that makes it as easy as possible to switch an underlying dependency out
for a new one in the future. Sadly, "as easy as possible" may not always be particularly easy.