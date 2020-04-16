---
layout: post
title: QUIC in .NET - The Master Thesis
categories: quic master-thesis
comment_issue_id: 1
date: 2020-04-17 13:11 +0200
---
In the past few months, I have been working on implementation of my master thesis at Charles
University. The topic of the thesis is 'QUIC protocol implementation for .NET' and it is the most
challenging journey I have emgarked on so far. Which is exactly what I was looking for because
otherwise I would feel like I got my university diploma too easily. 

It has been a about a month since I started the actual implementation phase, regularly informing my
supervisor and consultants about my progress by email. One of the consultants has voiced the idea
that if I wrote my reports in english, they could be forwarded to other people interested in my
work. I decided to expand on the idea and start writing the reports as blog posts. This post is the
first such report. Subsequent reports are likely to be shorter.

## Context of the master thesis

I would first like to expand a bit more on what exactly my master thesis is about for those hearing
about it for the first time, as it will explain some decisions I made in the source code
organization.

The latest development branch of .NET contains experimental support for QUIC protocol via interop
with msquic library (written in C), occupying the `System.Net.Quic` namespace. Because the QUIC
protocol specification is still in draft stage (and probably other reasons as well), the QUIC
support is not going to be released as part of .NET libraries anytime soon (i.e. .NET 5).

This means that the contents of the `System.Net.Quic` namespace is going to be either internal, or
not present in the release at all. However, it might be published in future releases once the
specification and implementation is final and mature.

My master thesis is (also experimental) implementation of QUIC protocol, but in managed C# only,
possibly replacing the msquic based implementation if all goes well in future. This implies that my
implementation should be done in a fork of the .NET repository itself.

However, all the QUIC classes intended to be used by user code are internal for now, so it would be
basically impossible for the reviewer of my thesis to try to use my implementation directly. Making
these classes public would help, but would require the user (including the thesis reviewer) to use
a custom build of .NET runtime, which did not feel justifiable to me. So I look for a wayt o make it
more available.

## Organization of the source code

I still wanted to keep the implementation inside the .NET runtime repository fork because that way I
can run the functional tests made for msquic based implementation against my implementation, and
also try my implementation as backend for HTTP3. 

After some thinking, I got an idea that it could be possible to take the source code files of my
implementation and build them separately as a normal .NET Core library, This library would expose
the QUIC classes and let them be used by regular .NET Core application. After some tinkering with
`<Compile Include=...>` sections inside the .csproj file I arrived at the following state (the paths
are relative to the root of the [Master thesis git
repository](https://github.com/rzikm/master-thesis)):

My fork of the dotnet runtime is referenced as a submodule at `src/dotnet-runtime`.

The actual source code of managed QUIC implementation is located in
[src/dotnet-runtime/src/libraries/Common/src/System/Net/Http/aspnetcore/Quic/Implementations/Managed](https://github.com/rzikm/dotnet-runtime/tree/master-managed-quic/src/libraries/Common/src/System/Net/Http/aspnetcore/Quic/Implementations/Managed).

The code for unit tests for managed QUIC implementation is located at:
[src/dotnet-runtime/src/libraries/System.Net.Http/tests/UnitTests](https://github.com/rzikm/dotnet-runtime/tree/master-managed-quic/src/libraries/System.Net.Http/tests/UnitTests).

The solution and projects for the standalone library build are located under `src/System.Net.Quic/`.
The `System.Net.Quic` project includes the sources mentioned above and provides public wrappers. So
it is possible to use QUIC by referencing this assembly. This is also the the solution I use for
active development, as the compile times are shorter than building the `System.Net.Http` assembly
containing the internal QUIC sources. It can be used in both Visual Studio and Rider without
complicated setup (see README at the repository root).

## The Scope of the Thesis

The official text of the thesis assignment is available on the website of CUNI's [student information system](https://is.cuni.cz/studium/eng/dipl_st/index.php?id=&tid=&do=main&doo=detail&did=224048). The
thesis does not require complete implementation of the QUIC protocol, that could be too much of a
challenge even for me. The assignment has been worded rather vaguely in order not to promise too
much and allow cuts in the scope if the implementation proved too difficult.

The thesis assignment promises a demo application demonstrating that it is possible to send data
between two endpoints through a lossy network. This implies implementing at least following parts of
the protocol:

- Serialization of primitives/frames.
- Loss detection and recovery
- Congestion control
- Handshake, integration with suitable TLS implementation (modified OpenSSL in my case)
- Flow control (advertising how much bytes the peer is willing to accept)

The features not directly related to data transport are not commited to, and will be either
implemented at the late stage of the thesis, or outside the scope of the thesis. The nonexhaustive
list of such features contains:

- Connection migration
- Peer validation
- Early (0-RTT) data

## What is Currently Implemented

Finally, I would like to summarize the result of the past month of my work. The basic skeleton of
the implementation is in place, although the code is peppered with TODO items and many things have
been rushed a bit to get as fast as possible to a first executable version. The implementation
currently can:

- negotiate transport parameters
- encrypt/decrypt outcoming/incoming packtes
- send application data (although only about 2 method overloads for doing so are implemented)
- send acknowledgements for received packets

It is already possible to send data between two local connections, provided that the sender is
throttled by application code in order not to overflow receivers OS socket buffer (loss/congestion
detection is not supported yet).

The next hot candidates for future development are (in order of priorities):

- loss detection and data retransmission
- limiting advertised flow control window (currently, endpoints advertise that they can receive all
  data the peer will send)
- negotiating application level protocol (ALPN)
- fully supporting SslClientAuthenticationOptions and SslServerAuthenticationOptions arguments
- removing unnecessary copies and internal buffering of data.
- other performance optimizations
