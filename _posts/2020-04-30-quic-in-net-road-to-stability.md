---
layout: post
title: QUIC in .NET - Benchmarks and Road to Stability
categories: quic master-thesis
comment_issue_id: 2
date: 2020-04-30 10:30 +0200
---

It has been nearly two weeks since my last update, so I decided it's about time I summarize what was
I was up to. I had an idea that it would be nice to create performance benchmarks early in the
development, so that I am able to track performance improvements as I implement more features and
optimizations. 

I also wanted to see how my implementation compares to native options. Comparison with raw TCP stack
would be unfair, but `SslStream` which works over TCP seemed a good choice since it also needs to
perform SSL negotiation and encryption of the data.

So I put together a benchmark using the `BenchmarkDotNet` library, which would measuer how long does
it take to receive certain amount of data from a server running in the same process. It turned that
my implementation was so unstable, that the benchmarks almost never finished all the iterations.

It seems that features like loss detection and data retransmission are vital even if there is no
packet loss on local connection. Since I did not implement flow control yet, the OS socket buffer
was instantly filled to the brim and started discarding incoming data. Lesson learned.

To make the story short, I implemented packet loss detection and recovery in a day or so, and spent
rest of the last two weeks hunting bugs. Infinite cycles, deadlocks, livelocks, data races, min/max
confusion, off-by-one errors, and many other. One by one I managed to fix these and the
implementation seems quite stable now, meaning that all benchmarks run successfully to completion.

## Initial Benchmark Results

I prepared a set of 3 benchmarks, each targeting particular part of the expected usage:

- Connection establishment
- Streaming data
- Connection termination

All benchmarks are run with following configuration:

``` 
BenchmarkDotNet=v0.12.0, OS=manjaro 
Intel Core i5-6300HQ CPU 2.30GHz (Skylake), 1 CPU, 4 logical and 4 physical cores
.NET Core SDK=3.1.103
  [Host]     : .NET Core 3.1.3 (CoreCLR 4.700.20.11803, CoreFX 4.700.20.12001), X64 RyuJIT
  Job-AWMUQU : .NET Core 3.1.3 (CoreCLR 4.700.20.11803, CoreFX 4.700.20.12001), X64 RyuJIT

InvocationCount=1  UnrollFactor=1  
```

### Connection Establishment Benchmark

The first benchmark measures the connection establishment handshake. Even though QUIC allows
establishing connection by sending a single packet, I have not implemented that feature yet, so QUIC
connection requires two full roundtrips.

|         Method |     Mean |     Error |   StdDev | Ratio | RatioSD |     Gen 0 | Gen 1 | Gen 2 |  Allocated |
|--------------- |---------:|----------:|---------:|------:|--------:|----------:|------:|------:|-----------:|
|      SslStream | 6.441 ms | 0.4034 ms | 1.164 ms |  1.00 |    0.00 |         - |     - |     - |   18.66 KB |
| QuicConnection | 6.547 ms | 1.4783 ms | 4.289 ms |  1.06 |    0.72 | 1000.0000 |     - |     - | 4172.26 KB |

The results are suprisingly good, although the jitter in the `QuicConnection` is still quite large
with extreme values at 2ms and 18ms.

### Streaming Data Benchmark

Next benchmark measures how long it takes to transfer a particular chunk of data of size 64kB, 1MB,
and 32MB respectively. 

|     Method | DataLength |         Mean |         Error |        StdDev |       Median |  Ratio | RatioSD |      Gen 0 |     Gen 1 | Gen 2 |   Allocated |
|----------- |----------- |-------------:|--------------:|--------------:|-------------:|-------:|--------:|-----------:|----------:|------:|------------:|
| **QuicStream** |      **65536** |  **30,914.4 us** |  **16,044.16 us** |  **47,306.58 us** |   **1,202.8 us** | **206.28** |  **323.39** |          **-** |         **-** |     **-** |     **10920 B** |
|  SslStream |      65536 |     153.5 us |      12.55 us |      34.79 us |     136.9 us |   1.00 |    0.00 |          - |         - |     - |     33776 B |
|            |            |              |               |               |              |        |         |            |           |       |             |
| **QuicStream** |    **1048576** |  **17,649.4 us** |   **2,103.88 us** |   **6,137.11 us** |  **17,974.3 us** |  **16.55** |    **5.72** |          **-** |         **-** |     **-** |   **2674496 B** |
|  SslStream |    1048576 |   1,101.1 us |      41.80 us |     116.52 us |   1,067.3 us |   1.00 |    0.00 |          - |         - |     - |     33296 B |
|            |            |              |               |               |              |        |         |            |           |       |             |
| **QuicStream** |   **33554432** | **569,656.4 us** | **124,821.42 us** | **368,038.77 us** | **412,800.7 us** |  **17.80** |   **11.02** | **60000.0000** | **5000.0000** |     **-** | **192544992 B** |
|  SslStream |   33554432 |  32,077.4 us |     639.54 us |   1,482.22 us |  32,049.2 us |   1.00 |    0.00 |          - |         - |     - |       576 B |

Although the results are less impressive than in previous benchmark, I am actually quite satisfied. 
There is of course huge room for improvements, mainly in the amount of memory allocated. I did not
get into detailed profiling yet, but my guess is that even though I try to avoid excess allocation
by using `ArrayPool`, the sizes of the arrays I request varies too much for the pooling to work
efficiently. The buffering needs more work anyway, and if the decreased performance is caused by
congestion at the receiver side, then implementing flow control should improve it.

### Connection Termination Benchmark

I originally did not intend to measure how long it takes to terminate the connection, but I have
noticed that the cleanup stage of previous benchmark takes quite a lot of time for QUIC. This
benchmark measures solely the time needed to close a connection via call to `Dispose` method.

|         Method |       Mean |      Error |     StdDev |     Median | Ratio | RatioSD |     Gen 0 |     Gen 1 | Gen 2 |  Allocated |
|--------------- |-----------:|-----------:|-----------:|-----------:|------:|--------:|----------:|----------:|------:|-----------:|
|      SslStream |   5.529 ms |  0.1972 ms |  0.5497 ms |   5.440 ms |  1.00 |    0.00 |         - |         - |     - |   18.91 KB |
| QuicConnection | 106.344 ms | 21.5032 ms | 63.4028 ms | 126.804 ms | 19.36 |   11.99 | 1000.0000 | 1000.0000 |     - | 3841.66 KB |

I have mixed feelings about the results and can't explain them yet. It is possible that `Dispose` in
my QUIC implementation blocks for unnecesarily long time and could return sooner. Another possible
explanation is a bug in my implementation causing the connection to close via the slower
timeout-based path most of the time.

## What comes next?

There are tons of items on the current backlog, so I list only the things I plan to actually focus
on in following two weeks:

- Flow control, this will hopefully improve performance by avoiding packet discarding at receiver.
- Using `EventCounter` API to expose events such as packet loss to aid further profiling,
- Reduce memory allocation.
- Maybe? performance comparisons with msquic based implementation, since it was open-sourced now.
