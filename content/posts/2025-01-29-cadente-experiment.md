+++
title = 'Cadente: a bold experiment for Sisk'
date = 2024-11-13T14:17:04-03:00

[params]
tags = ['HTTPS']
img = '/pimg/1726009047_1103_14877.png'
+++

You must have noticed that the SslProxy project has stopped. And yes, indeed, it will be discontinued soon.

But this does not mean that its initial purpose has ceased to exist: bringing native HTTPS to Sisk. In fact, the project has taken a new direction, which led to the construction of the Cadente project: a pure C# implementation of HTTP/1.1 with SSL support, which should soon be implemented in Sisk/Core.

The current implementation of SslProxy was quite poor. In addition to being very slow, it was not stable, not a good implementation, and not worth maintaining. An alternative was [mitmproxy](https://docs.sisk-framework.org/docs/ssl#through-mitmproxy), as it delivered well on development with HTTPS for Sisk.

But why couldn't Sisk have native SSL? In fact, it can, but you will depend on Windows for this. In fact, everything in Sisk is better on Windows: it's faster and even supports QUIC through IIS, but you depend on Windows, which increases the cost of cloud deployment.

## A bet on a future without HttpListener

The Cadente project aims to solve three problems:

- A possible deprecation of HttpListener in .NET.
- Dependence on Windows to achieve good server performance.
- Security (SSL).

And that's why it exists: to be an alternative to Microsoft's HttpListener, not an alternative to Sisk itself. The Cadente project is equivalent to ASP.NET's Kestrel, as it will be for Cadente to Sisk.

This implementation is still highly experimental and not at all suitable for production, but benchmarks have achieved good results so far. Soon, all of them will be announced. In preliminary tests, Cadente had a performance [955.40% better](https://www.techempower.com/benchmarks/#section=test&runid=4bea847f-bb2e-45f9-a723-95cba45eec14&hw=ph&test=json) than Sisk using HttpListener on Linux. The expectation is to further improve its performance, making it something better, viable, and production-ready.

### Try it out

You can test [Sisk.Cadente](https://www.nuget.org/packages/Sisk.Cadente/) today.

<script src="https://gist.github.com/CypherPotato/2c0ca0ea61ecb781aaad8ed0df1df128.js"></script>

It's worth noting that some HTTP/1.1 features are still not available, such as `Expect: 100`, but they are already on our radar. Don't hesitate to open an [issue](https://github.com/sisk-http/core) if you find something or a PR to improve what we already have.