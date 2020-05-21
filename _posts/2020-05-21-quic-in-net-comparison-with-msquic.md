---
layout: post
title: QUIC in .NET - Comparison and Interop with msquic
categories: quic master-thesis
comment_issue_id: 3
date: 2020-05-21 17:23 +0200
---
I am slowly nearing state when my implementation is feature complete (from the perspective of my
master thesis goals), and I am slowly tying up loose ends of the implementation. There are still
around 70 TODO items scattered throughout the code, but many of them are placeholders for future
feature implementations and are not important from the thesis perspective.

Let's revisit the items mentioned at the last blog post.

## Flow Control Implementation

My implementation now advertises a fixed size window which it is willing to accept/buffer. However,
it did not have the effect on performance I expected. The performance was more or less unaffected,
so I needed to look elsewhere for performance gains.

## EventCounter API and Profiling Events

Implementing profilnig events turned out to be much more fun than I expected. I even expanded on the
idea and hacked together an analyzer for `BenchmarkDotNet` which would show me how many
packets/bytes were sent/lost when running a particular benchmark case. I also added a very verbose
event which would log incoming and outgoing UDP datagrams, for inspection. This turned out to be
very useful when debugging interop with msquic (more on that later).

## Reducing Memory Allocations

Last benchmarks showed that my implementation is quite allocation heavy and I expected to be because
of inefficient pooling of arrays for buffering data. It turned out that the arrays for buffering
were not the only problem.

One of the biggest sources of allocations was actually the set of async methods for servicing the OS
Socket. Since the only reason for using `async` was to use Async overloads on the socket, I
rewrote the implementation to use async (and await) only for receiving from a socket, and using
synchronous method for sending. It cut the amount of allocations significantly and performance
seemed to improve.

However, there are still many sources of allocations I am unable to ged rid of. Mainly allocating an
instance of `IPEndPoint` every time a datagram is sent or received. While searching for a solution,
I found that there is an [outstanding API proposal](https://github.com/dotnet/runtime/issues/30797)
for allocation-less UDP SendTo/ReceiveFrom interface. I believe that the allocations could also be
removed by using *connected UDP sockets*, but that would require large restructuralization of the
socket-related code.

## Interop with MsQuic

The items above took me only few days. I was quite curious to look at msquic source code, and
noticed that there is a sample application I could use to test interop with my implementation. So I
wrote a client counterpart to the msquic sample application and run it.

It failed horribly. MsQuic immediately close the connection on me with rather uninformative error
code (I no longer remember what it was). So I dug into msquic source code with a debugger and I was
reminded that lack of application-level protocol in during the handshake was suitable reason to
immediately close it. So I added support for ALPN, so far it was easy.

However, I still could not get the handshake to complete, my only clue was that last packet I
received from msquic could not be decrypted.

So I had an idea to reuse code I used in my tests, that could parse and modify QUIC packets, to dump
the conversation between my implementation and msquic in order to analyze it.

After a bit of tinkering and routing data from implementation to an outside logger through the
`EventCounter` API, I put together a class that would read datagram dumps from the QUIC
`EventSource` and dump it in readable form to standard output.

It turned out that I overlooked an important part of the specification, which allowed server the
connection id (originally suggested by the client) at the beginning of the connection. That was
luckily easily fixable and was also the last obstacle, I had a working demonstration of interop with
msquic, which made me quite happy after hours of stepping through foreign C code.

Just a curious note. The packet that my implementation failed to decrypt was actually a *Stateless
Reset* packet used by msquic as a last resort when it can't match incoming packets to an existing
connections (I used the old conneciton id which was not recognized).

## Fixing the MsQuic Based Implementation in .NET Runtime

After my successful interop with msquic, I wanted to see how good (or rather how bad) my
implementation would be against msquic. I tried the existing interop code, but the function
signatures have changed since then and the code immediately segfaulted somewhere in native code.

After [updating some of the key
parts](https://github.com/rzikm/dotnet-runtime/commit/8e433d57fea3dede98fe5205c83db15175c589db) of
the msquic integration in .NET I was able to run the same sample application with either managed
implementation, or msquic based one (I chose an environment variable as a switch for the
implementations).

## Performance Comparisons with MsQuic-based Implementation.

What follows are the same set of streaming comparison benchmarks I used last time, but this time
with both managed and msquic based implementation:

```ini
BenchmarkDotNet=v0.12.1, OS=manjaro 
Intel Core i5-6300HQ CPU 2.30GHz (Skylake), 1 CPU, 4 logical and 4 physical cores
.NET Core SDK=3.1.201
  [Host]     : .NET Core 3.1.3 (CoreCLR 4.700.20.11803, CoreFX 4.700.20.12001), X64 RyuJIT
  Job-QBGPIO : .NET Core 3.1.3 (CoreCLR 4.700.20.11803, CoreFX 4.700.20.12001), X64 RyuJIT

InvocationCount=1  UnrollFactor=1  
```

|       Method | DataLength |         Mean |        Error |        StdDev |       Median | Ratio | RatioSD |      Gen 0 |     Gen 1 | Gen 2 |   Allocated |
|------------- |----------- |-------------:|-------------:|--------------:|-------------:|------:|--------:|-----------:|----------:|------:|------------:|
|   **QuicStream** |      **65536** |   **1,010.2 μs** |    **443.42 μs** |   **1,221.31 μs** |     **963.9 μs** |  **5.78** |    **7.55** |          **-** |         **-** |     **-** |    **19.99 KB** |
|    SslStream |      65536 |     210.1 μs |     30.97 μs |      88.36 μs |     162.9 μs |  1.00 |    0.00 |          - |         - |     - |    33.13 KB |
| MsQuicStream |      65536 |   1,970.7 μs |    167.83 μs |     465.07 μs |   1,872.0 μs | 10.82 |    4.21 |          - |         - |     - |     3.87 KB |
|              |            |              |              |               |              |       |         |            |           |       |             |
|   **QuicStream** |    **1048576** |  **17,387.8 μs** |  **1,110.72 μs** |   **3,168.93 μs** |  **17,229.9 μs** | **16.38** |    **3.25** |          **-** |         **-** |     **-** |  **1940.82 KB** |
|    SslStream |    1048576 |   1,061.9 μs |     31.02 μs |      86.46 μs |   1,049.3 μs |  1.00 |    0.00 |          - |         - |     - |    32.52 KB |
| MsQuicStream |    1048576 |  16,072.1 μs |    440.65 μs |   1,257.21 μs |  15,858.4 μs | 15.21 |    1.84 |          - |         - |     - |    36.73 KB |
|              |            |              |              |               |              |       |         |            |           |       |             |
|   **QuicStream** |   **33554432** | **684,980.8 μs** | **44,179.60 μs** | **126,759.56 μs** | **664,396.2 μs** | **21.22** |    **4.35** | **21000.0000** | **2000.0000** |     **-** | **66055.64 KB** |
|    SslStream |   33554432 |  32,387.7 μs |    803.53 μs |   2,331.18 μs |  32,143.4 μs |  1.00 |    0.00 |          - |         - |     - |     1.19 KB |
| MsQuicStream |   33554432 | 423,751.3 μs |  8,453.93 μs |  10,063.80 μs | 427,169.0 μs | 12.21 |    0.60 |          - |         - |     - |  1145.63 KB |

There is still a room for improvement, but I am satisfied with the results for now. It seems that
the performance between managed and msquic based implementation should be comparable now. I probably
will try using my implementation inside ASP.NET QUIC samples to achieve less synthetic comparison.

## What's next?

From the thesis' perspective, the implementation can be considered complete, so I probably won't do
any drastical changes. The only think I might try to do is adding support for all properties from
`SslServerAuthenticationOptions` and `SslClientAuthenticationOptions`, which I avoided so far.

Beside that, I think I will start writing the actual thesis text. I doubt I will make it to the
deadline at the end of July. If i don't then I will have more time to revisit the implementation and
experiment more with the socket code.
