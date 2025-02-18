+++
title = 'Sisk v1.3 is available now'
date = 2024-11-13T14:17:04-03:00
author = "CypherPotato"

[params]
tags = ['Releases']
img = '/pimg/1726009262_129_39772.png'
+++

Sisk 1.3 is being released today!

This update brings performance improvements, another way to write your favorite APIs, and something similar to dependency injection.

Sisk's commitment remains the same: to be simple to develop a quality HTTP application. Sisk has no dependencies other than the .NET ecosystem itself, and all its code is still less than 1KB in total. Sisk has a minimal footprint, focusing performance on your application's layer, not the server.

We would also like to showcase the new logo of the project. It is more modern while still referencing what the project used to be.

![Sisk logo](https://sisk-framework.org/assets/img/Presentation.png)

## Performance improvements

Sisk has a limitation: it depends on the current implementation of HttpListener to work. This implementation is great on Windows and has performance comparable to ASP.NET, but in recent tests, we found that the managed implementation of HttpListener lags significantly behind ASP.NET's Kestrel, which happens outside Windows environments. Even so, Sisk still manages to be much more performant and efficient than many other development frameworks in other languages. Our goal is not to be better than ASP.NET, but the closer we get to its performance, the more viable the choice of Sisk becomes for more projects.

In version 1.3, we achieved an average performance increase of 15% overall for request processing. This is because we improved the way the server sends the content of a response to the client. The HttpResponse object depends on HttpContent to define content to send to the client, and previously, this object had to be serialized through [HttpContent.ReadAsByteArrayAsync](https://learn.microsoft.com/en-us/dotnet/api/system.net.http.httpcontent.readasbytearrayasync?view=net-8.0). The problem is that many contents are based on ByteArrayContent, and when calling this method on this type of content, a memory buffer was created, and the response bytes were copied to this buffer. Then, this buffer was copied to the response's output stream.

Thanks to [UnsafeAccessor](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.unsafeaccessorattribute?view=net-8.0) in .NET 8, we can access the response bytes directly, without the need to create this secondary buffer. This greatly improved overall performance.

![benchmark](https://raw.githubusercontent.com/sisk-http/archive/refs/heads/master/assets/sisk-benchmark-1-3.png)

In addition, there were performance improvements outside of the request processor as well. Improvements were made in the way the server writes messages to the access log and deserializes multipart content.

## Static HttpContexts

An experimental feature is to expose the current HttpContext of a request in a local context of the execution thread. This allows you to expose members of a request outside the request method, greatly improving the readability of your code. The code snippet below shows an abstract class that functions as an API controller with an embedded database.

<script src="https://gist.github.com/CypherPotato/45a05dd23e8a99f219854bf05ad33a87.js"></script>

You can read more about this feature in the [documentation](https://docs.sisk-framework.org/docs/features/instancing).

Read all the details of this release in the [official changelog](https://github.com/sisk-http/archive/blob/master/changelogs/v1.3/v1.3.md).