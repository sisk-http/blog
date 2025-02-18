+++
title = 'Speed up Sisk using streams'
date = 2024-09-14T02:56:51-03:00
draft = false
author = "CypherPotato"

[params]
tags = ['HTTPS']
img = '/pimg/1726009208_3331_88232.png'
+++

Reading and writing content with HTTP seems simple? With C#, it's even easier. But are you doing it the right way?

If I asked you to serve an image through the `GET /cute-dog.png` route, how would you do it? Well, the simplest way would be to read the file, get its content type, and send it in the response. Something like this:

<script src="https://gist.github.com/CypherPotato/0286e6af1bc6012070d4d3ef009f3a45.js"></script>

Do you see any problem in the code above? If I open my browser and access `https://localhost:8080/cute-dog.png`, this would be the result:

![sher name is Vaquinha](/scr/cutedog.png)

Just kidding! You definitely don’t have that photo on your computer, so some error will happen on your machine, or you might see another dog (I don't guarantee it's cute).

But let's go back to the code. The result is what I expect, but every time I access `GET /cute-dog.png`, the server will need to read the file, store the content in memory, and then send it to the client. In doing so, I must trust that the GC will collect that memory after sending the file.

But what if I access this file 100 times? The server will need to read the file 100 times, store the content in memory 100 times, and then send it to the client 100 times. If the file I'm sending is an image, it might not bother the server much, but what if I'm serving a `big-file.zip` with 150 MB of content?

That's what I want to address.

# HTTP is about streaming

Exactly that. It's not the video streaming you're thinking of, but content streaming. HTTP works over TCP, and the content is sent in a continuous byte stream between the client and the server. This streaming is one-way, but there is a time to read and a time to write for each party.

When I request a file from a server via `GET`, *commonly* the client expects the server to respond with the file's content. Instead of reading the file's content into memory, we can simply copy the file's read stream to the client's write stream.

How? I have an example.

<script src="https://gist.github.com/CypherPotato/85d63402fb1c98ba062014d2161eaf7e.js"></script>

It's a slightly advanced example. I show how to send headers, status, and file content. Never send status and headers after starting to send the content! This is not allowed in HTTP, and Sisk does not support [trailers](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Trailer).

There's also an even simpler way to do this using a [StreamContent](https://learn.microsoft.com/en-us/dotnet/api/system.net.http.streamcontent?view=net-8.0) object.

<script src="https://gist.github.com/CypherPotato/6310a21e551a240532c39aab2cc7abcc.js"></script>

Here, there’s no need to apply `using` to the `fileStream`, and it shouldn’t be applied. The file read stream is automatically closed when the server sends the response to the client. However, **I do not recommend** using `StreamContent` because if you encounter an error before sending the response to your client, the open streams might not close properly.

Sisk can only close streams after it reaches the server step of reading the stream content. At this stage, it's guaranteed your stream will be closed. But until it gets there, it can't guarantee that everything will go well with your stream.

To be safe, use the first example.

# And for reading?

You can also read the content sent from your client through a stream! No copying everything to memory. Additionally, the content of a request is never read automatically unless you call some read property of `HttpResponse`.

The example below deserializes a JSON message from the client content:

<script src="https://gist.github.com/CypherPotato/6baf6d46b86f73b9ca76c6d289f1ffbb.js"></script>

Here, I read the content of my request into `requestBody` because I accessed `HttpRequest.Body`. This property reads the client's read stream content into a string using the encoding the client specified in its content. If no encoding is specified, Sisk will use `Encoding.Default`.

For this, I have to read all my content into memory. After that, I interpret the JSON present in this content. The point is, the native JSON library supports streams, so I can improve my code significantly with:

<script src="https://gist.github.com/CypherPotato/bdd5b7b10647bffb14b9480c0875fd49.js"></script>

And again: no `using` here. The client's read stream is automatically closed when the server finishes reading the content. Whether an error occurred during reading or not, the stream is closed. I promise.

# Conclusion

Streams are a fundamental part of Sisk. In fact, they are a fundamental part of HTTP in general, yet some servers don't implement this, or their users don't even know it exists.

Now you know. How about starting to stream your responses now?