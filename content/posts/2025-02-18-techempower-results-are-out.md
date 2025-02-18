+++
title = 'TechEmpower benchmark results are out!'
date = 2025-02-18T00:00:00-03:00
author = "CypherPotato"

[params]
tags = ['HTTPS', 'Benchmark']
img = '/pimg/1726008892_5319_24258.png'
+++

My goal was never to lie about Sisk.

I understood its limitations and problems, and worked to resolve and improve it. Some commercial projects already work well with Sisk, have pleasant performance, and are easy to maintain because they are open-source.

Recently, I discovered that the HttpListener interface was very slow on Linux servers. On Windows, it is very fast, and even supports QUIC, but it depends on Windows. On Linux servers, which make up 99% of servers out there, there was an extra layer of having to use a proxy to handle SSL traffic.

The [Cadente Project](/posts/2025-01-29-cadente-experiment/) is an attempt to solve two problems at once: performance and native support for SSL. And we had good results! A preliminary test of Round 23 of the TechEmpower Benchmark showed Sisk Cadente in position 106 out of a total of 376 different frameworks, placing it in the top 48.4% of frameworks in the JSON serialization test.

The test that was supposed to be the simplest, Plaintext, did not have good results. In fact, it came in last place, but that was not the fault of Sisk. The test was poorly configured. The server returned a "Hello, world!" instead of a "Hello, World!" (note the uppercase W), which was enough to mess up the entire test. Therefore, we will not consider the plaintext test this time.

![C# results](/scr/bench.png)

## Next Steps

Our next steps are:
- To further improve the Sisk Cadente project, focusing on its performance, stability, and features.
- To embed the Sisk.Cadente abstraction in the Sisk.Core project, so that Cadente can be used as an HTTP server in Sisk instead of HttpListener.
- To ensure stability and performance for future projects.

You can help improve Sisk Cadente and Sisk.Cadente, and also help ensure that Sisk has the best performance and stability. Open an issue or send a PR to help!