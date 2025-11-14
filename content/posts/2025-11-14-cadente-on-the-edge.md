+++
title = 'Cadente on the edge: running Sisk without the old seatbelts'
date = 2025-11-14T00:00:00-03:00
author = "Mia"

[params]
tags = ['HTTPS', 'Cadente']
img = '/pimg/1726026081_4216_97664.png'
+++

I've been spending the last weeks almost exclusively inside Wireshark traces and benchmark charts, and I can finally say this out loud: the **Cadente project is almost ready**.

Not "marketing ready". I mean "I can run Sisk on it without being terrified every five minutes".

Cadente, if you missed the earlier post, is that bold attempt to write a **pure C# HTTP/1.1 + SSL server engine** for Sisk – basically, a Kestrel‑like role in the Sisk ecosystem.

Today, Cadente has:

- **Passed all Sisk compatibility tests** for HTTP server engines  
- Been running as the engine behind **Sisk v1.6** in my local and staging environments  
- Survived ugly scenarios like long‑lived connections, SSE, and some pathological clients

All the core behavior that Sisk expects from an `HttpServerEngine` is now green. That doesn't mean it's perfect; it means it fails *less often and more predictably*.

### The HttpServerEngine API Rework

To make Cadente viable at all, I had to **rewamp (yeah, a full reshaping) the HttpEngine API**:

- Clearer separation between transport layer (sockets, buffers, queues) and Sisk features (routing, middlewares, SSE)
- A more explicit event loop model, so different engines don't have to pretend to be `HttpListener`
- Less magic, more boring predictable APIs – boring is good in infra code

The downside: this will force some engine implementations or experiments to adapt. The upside: it's finally an API I don't hate reading six months later.

### Performance: Better, But Not Magic

Cadente's performance has improved compared to the early experiment:

- On Linux, it's consistently ahead of the `HttpListener` abstraction used by Sisk
- Latencies are more stable under load, which matters more than pretty "req/s" numbers

Still, let's stay honest: this is **not Kestrel**, and it's not pretending to be. There are still missing HTTP/1.1 corner cases, and I expect a few ugly bugs to surface once more people hammer it in real workloads.

## What's Next

So yes: **Cadente is almost ready**, running on **Sisk v1.6** with:

- A reworked `HttpServerEngine` API  
- All official Sisk compatibility tests passing  
- Ongoing performance and stability tuning

I'm optimistic, but cautiously so. Cadente could still hit a wall in real production traffic, and if that happens, I'd rather admit it and fix it than pretend it's "enterprise‑ready".

For now, I'm just happy that Sisk can finally run without depending on `HttpListener` and without pretending SSL is someone else's problem.