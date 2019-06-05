---
title: "WCF vs gRPC - Round 2"
date: 2019-06-05T18:48:36+01:00
draft: false

categories: ["comparisons"]
tags: ["wcf", "grpc"]
author: "Mark Rendle"
---

After my previous post [comparing WCF to gRPC](https://unwcf.com/posts/wcf-vs-grpc), a couple of people
on Twitter and in the comments asked which WCF binding I had used for the performance comparison. The answer
to that was "whatever the default binding is", which is basic HTTP binding. As Clemens Vaster pointed out, that
is not an "apples-to-apples" comparison, and a fairer WCF vs gRPC test would use NetTCP binding. So I re-ran
my performance tests using NetTCP binding in a simple console host for the WCF service.

##### WCF using NetTCP

Here's what the host code looks like for this run:

```csharp
namespace DummyServiceHost
{
    class Program
    {
        static void Main(string[] args)
        {
            var dummyService = new DummyService(new DummyRepo());
            var host = new ServiceHost(dummyService);

            var binding = new NetTcpBinding();
            host.AddServiceEndpoint(
                typeof(IDummyService), binding,
                "net.tcp://localhost:5000/dummy"
            );

            host.Open();

            Console.ReadLine();
        }
    }
}
```

And the client code:

```csharp
[GlobalSetup]
public void Setup()
{
    _client = new Dummy.DummyServiceClient(
        new NetTcpBinding(),
        new EndpointAddress("net.tcp://localhost:5000/dummy")
    );
}
```

I knew this would be faster than using HTTP binding, but I was surprised by just *how much faster* it is. Here are the
[BenchmarkDotNet](https://benchmarkdotnet.org/) results:

| Method |     Mean |    Error |   StdDev |  Gen 0 | Gen 1 | Gen 2 | Allocated |
|------- |---------:|---------:|---------:|-------:|------:|------:|----------:|
|    Wcf | 211.5 us | 1.625 us | 1.440 us | 0.7324 |     - |     - |   3.52 KB |

OK, wow. That's pretty fast. In fact, that's nearly five times faster than the gRPC implementation. Which I was not
expecting, because I know gRPC is fast. So I went back to the gRPC version of the code to see what, if anything, I
could do about it.

##### gRPC Streams vs Repeats

The gRPC implementation of the service in the first test used a [stream](https://grpc.io/docs/guides/concepts/#server-streaming-rpc)
method, which fires back every result as it becomes available. This is a great feature when you're working with a
real-world scenario that is grabbing data a row at a time from a database, for example, both because you save on the memory overhead
of building a huge in-memory list or array and because the client can start doing its thing with the data before
the database query has finished on the server.

But in this case, it is again not an apples-to-apples comparison to what the WCF application is doing, which is creating
a single response with a list of results in it. So I changed the gRPC application to work the same way, returning a
complete list in a single response.

Here's the `dummy.proto` file for the new approach:

```protobuf
syntax = "proto3";

package Dummy;

service ProtoDummyService {
  rpc GetDataStream(GetDataStreamRequest) returns (GetDataStreamResponse) {}
}

message GetDataStreamRequest {
  int32 min = 1;
  int32 max = 2;
}

message GetDataStreamResponse {
	repeated CompositeType value = 1;
}

message CompositeType {
  bool BoolValue = 1;
  string StringValue = 2;
  int32 IntValue = 3;
}
```

Instead of returning a `stream CompositeType`, we create an explicit response message using the `repeated`
modifier on the `CompositeType value` field. The ASP.NET Core 3.0 implementation method for this now looks like this:

```csharp
public partial class DummyService : ProtoDummyService.ProtoDummyServiceBase
{
    public override Task<GetDataStreamResponse> GetDataStream(
        GetDataStreamRequest request, ServerCallContext context)
    {
        var response = new GetDataStreamResponse();
        response.Value.AddRange(GetDataStreamImpl(request.Min, request.Max));
        return Task.FromResult(response);
    }
}
```

And the consuming client code looks like this (much more like the WCF client code now):

```csharp
[Benchmark]
public async Task<int> Grpc()
{
    var response = await _client.GetDataStreamAsync(
        new GetDataStreamRequest {Min = 0, Max = 42}
    );

    var sum = 0;
    foreach (var item in response.Value)
    {
        sum += item.IntValue;
    }
    return sum;
}
```

Here are the BenchmarkDotNet results for *this* run:

| Method |     Mean |    Error |   StdDev |  Gen 0 | Gen 1 | Gen 2 | Allocated |
|------- |---------:|---------:|---------:|-------:|------:|------:|----------:|
|   Grpc | 190.1 us | 1.984 us | 1.856 us | 1.4648 |     - |     - |   1.14 KB |

##### Analysis

Here are those results side-by-side for easy comparison:

| Method |     Mean |    Error |   StdDev |  Gen 0 | Gen 1 | Gen 2 | Allocated |
|------- |---------:|---------:|---------:|-------:|------:|------:|----------:|
|    WCF | 211.5 us | 1.625 us | 1.440 us | 0.7324 |     - |     - |   3.52 KB |
|   gRPC | 190.1 us | 1.984 us | 1.856 us | 1.4648 |     - |     - |   1.14 KB |

WCF has made a much better showing this time around using the more representative network binding. gRPC is still
a tiny bit faster, but we're only talking 20μs (that's microseconds, or millionths of a second) difference
now. The memory allocations are much closer as well; WCF is only allocating about three times as much
memory as gRPC instead of nearly 20 times as much from the previous test. Interestingly, gRPC actually
causes twice as many heap allocations as WCF, but again, the actual numbers are tiny. Both are better than their
counterparts from the previous test.

I'd like to thank the people who pointed out the flaw in my previous benchmarking experiment, which led
to this re-run and improvement of both the WCF and the gRPC solutions. I'm still hard at work on the automated
conversion tool, which will be made public in the near future, and it will offer a choice when converting
WCF `OperationContract` methods that return lists, so you can decide whether a streaming response or a
repeated message type is more appropriate on a case-by-case basis.

The tool is also going to assist with migrating other types of .NET Framework code to .NET Core 3.0. I can't wait
to actually share all the details with the world. Subscribe to this site's [RSS feed](https://unwcf.com/index.xml)
or follow [me on Twitter](https://twitter.com/markrendle) to hear about it when it happens.
