---
title: "Unix domain sockets on Windows"
date: 2019-05-27T22:57:30+01:00
draft: false

categories: ["networking"]
tags: ["bindings"]
author: "Mark Rendle"
---

I just found out from [a tweet from David Fowler](https://twitter.com/davidfowl/status/1133115112011718656)
that [Windows is getting support for Unix domain sockets](https://devblogs.microsoft.com/commandline/af_unix-comes-to-windows/),
which is pretty amazing. Unix domain sockets are an inter-process communication (IPC) feature
that use the file-system as an address space, and file-system permissions for security.
Other than that, they generally work like regular network sockets. Lots of
Linux software uses domain sockets for local connections; for example, Docker creates a `/var/run/docker.sock`
socket for communicating with the daemon on the local machine. It's more secure, incurs less system overhead
and is also faster than running on a `localhost` TCP socket.

The Windows implementation will initially support the Stream type of socket, which is like TCP,
and it appears that
[.NET Core is going to support Unix domain sockets on Windows](https://github.com/aspnet/AspNetCore/pull/10560)
pretty rapidly (it already has support for Unix domain sockets on Linux).

From the WCF point of view, this sounds like it could be a promising replacement for Named Pipe bindings,
which are used for on-host IPC on Windows. Named Pipes are not Sockets, so things like HTTP servers
(e.g. [Kestrel](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/servers/kestrel?view=aspnetcore-3.0))
don't support them. With Unix domain sockets, running a Kestrel HTTP/2 server, whether
for regular HTTP APIs or for [gRPC](/posts/wcf-vs-grpc/), is just a matter of setting options:

```csharp
.UseKestrel(options =>
{
    options.ListenUnixSocket("/var/run/myapp.sock");
})
```

At present, Unix domain sockets are only supported in the latest Insider builds of Windows 10,
so once the support lands in a .NET Core 3 preview, I'll fire up a VM and take it for a spin,
and post some performance comparisons between WCF Named Pipe bindings and .NET Core gRPC over Unix sockets.

##### Any questions?

If you're currently using WCF at your organisation and have a specific scenario you are considering
reworking for .NET Core 3.0 or .NET 5, I'd love it if you'd ping me on Twitter
([@markrendle](https://twitter.com/markrendle)) and tell me about it, or leave a comment on this
or any other post on this site.