+++
title = 'Speed up Sisk using streams'
date = 2024-09-14T02:56:51-03:00
draft = false

[params]
tags = ['HTTPS']
img = '/pimg/1726009208_3331_88232.png'
+++

Reading and writing content with HTTP seems simple? With C#, it's even easier. But are you doing it the right way?

If I asked you to serve an image through the `GET /cute-dog.png` route, how would you do it? Well, the simplest way would be to read the file, get its content type, and send it in the response. Something like this:

```csharp
using Sisk.Core.Http;
using Sisk.Ssl;

using var app = HttpServer.CreateBuilder()
    .UseSsl(8080)
    .Build();

app.Router.MapGet("/cute-dog.png", r =>
{
    byte[] cuteDogBytes = File.ReadAllBytes(@"cute-dog.png");

    return new HttpResponse()
        .WithContent(new ByteArrayContent(cuteDogBytes))
        .WithHeader("Content-Type", "image/png");
});

await app.StartAsync();
```

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

```csharp
app.Router.MapGet("/cute-dog.png", r =>
{
    // create a read stream from the file
    using var fileStream = File.OpenRead(@"cute-dog.png");

    // get the output stream for the response
    // no need for "using" here because the connection is properly closed
    // by the server.
    var responseStream = r.GetResponseStream();

    // send the status and headers to the client
    responseStream.SetStatus(HttpStatusCode.OK);
    responseStream.SetHeader("Content-Type", "image/png");
    responseStream.SetHeader("Content-Length", fileStream.Length);

    // copy the file read stream to the client's output
    fileStream.CopyTo(responseStream.ResponseStream);

    // close the transmission
    return responseStream.Close();
});
```

It's a slightly advanced example. I show how to send headers, status, and file content. Never send status and headers after starting to send the content! This is not allowed in HTTP, and Sisk does not support [trailers](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Trailer).

There's also an even simpler way to do this using a [StreamContent](https://learn.microsoft.com/en-us/dotnet/api/system.net.http.streamcontent?view=net-8.0) object.

```csharp
using var app = HttpServer.CreateBuilder()
    .UseSsl(8080)
    .Build();

app.Router.MapGet("/cute-dog.png", r =>
{
    var fileStream = File.OpenRead(@"cute-dog.png");

    return new HttpResponse()
        .WithHeader("Content-Type", "image/png")
        .WithContent(new StreamContent(fileStream));
});

await app.StartAsync();
```

Here, there’s no need to apply `using` to the `fileStream`, and it shouldn’t be applied. The file read stream is automatically closed when the server sends the response to the client. However, **I do not recommend** using `StreamContent` because if you encounter an error before sending the response to your client, the open streams might not close properly.

Sisk can only close streams after it reaches the server step of reading the stream content. At this stage, it's guaranteed your stream will be closed. But until it gets there, it can't guarantee that everything will go well with your stream.

To be safe, use the first example.

# And for reading?

You can also read the content sent from your client through a stream! No copying everything to memory. Additionally, the content of a request is never read automatically unless you call some read property of `HttpResponse`.

The example below deserializes a JSON message from the client content:

```csharp
using var app = HttpServer.CreateBuilder()
    .UseSsl(8080)
    .Build();

app.Router.MapPost("/users", r =>
{
    string requestBody = r.Body;
    User? userJson = JsonSerializer.Deserialize<User>(requestBody);

    if (db.Users.Add(userJson))
    {
        return new HttpResponse(200);
    }
    else
    {
        return new HttpResponse(400);
    }
});

await app.StartAsync();
```

Here, I read the content of my request into `requestBody` because I accessed `HttpRequest.Body`. This property reads the client's read stream content into a string using the encoding the client specified in its content. If no encoding is specified, Sisk will use `Encoding.Default`.

For this, I have to read all my content into memory. After that, I interpret the JSON present in this content. The point is, the native JSON library supports streams, so I can improve my code significantly with:

```csharp
var requestStream = r.GetRequestStream();
User? userJson = JsonSerializer.Deserialize<User>(requestStream);
```

And again: no `using` here. The client's read stream is automatically closed when the server finishes reading the content. Whether an error occurred during reading or not, the stream is closed. I promise.

# Conclusion

Streams are a fundamental part of Sisk. In fact, they are a fundamental part of HTTP in general, yet some servers don't implement this, or their users don't even know it exists.

Now you know. How about starting to stream your responses now?