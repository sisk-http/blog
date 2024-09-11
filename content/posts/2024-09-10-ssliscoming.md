+++
title = 'HTTPS is coming, natively, to Sisk'
date = 2024-09-10T20:33:48-03:00
draft = true

[params]
tags = ['HTTPS']
img = '/pimg/1726009098_2734_90024.png'
+++

That's exactly it. I am working on my own implementation of an HTTP/1.1 serializer with HTTPS support for Sisk.

Recently, I received new personnel to join the team where I work. This new team took on a project that is running with Sisk alongside me. It is a large, high-complexity project, and I was proud to present Sisk.

While showing them the project, I had to teach them how to get it working. It wasn't like a conventional project; Sisk required a much larger, tedious, and discouraging initial setup. The reason for this was SSL. Our project depended on cookie-based authentication across domains, and these were domains mapped to the computer's localhost (the so-called `hosts` file). This setup involved:

- issuing a self-signed certificate, which wasn't so simple. It required OpenSSL or another tool. Fortunately, I knew about [certgen](https://github.com/minio/certgen), which still serves me well for this purpose.
- installing IIS. I was counting on the team using Windows for this task, and they were, to my luck. IIS is the only native way to have SSL on Sisk, and it only works on Windows. If you use anything other than Windows, you won't be able to get Sisk to listen with SSL.
- configuring IIS. This was still a tedious job.
- configuring namespaces in Windows. Since Sisk works on top of `HttpListener`, Windows has to grant certain permissions for you to listen anywhere other than `localhost`. I talked about this little tutorial [here](https://docs.sisk-framework.org/docs/registering-namespace).
- editing the hosts file.
- adding ports in the Sisk configuration file.
- restarting the computer.

This entire process was only once, but it was a job that took a few hours, some unnecessary questions, and a few hiccups. It was definitely a time obstacle, and even though everything was working in the end, I wasn't happy. I needed to change that.

## The options I had

Sisk can run on SSL through a proxy. That's exactly what I built. But not as an additional application, but as just a dependency for Sisk itself, which is capable of listening and writing HTTP/1.1 messages with SSL.

All you need to do to have HTTPS with Sisk, listening on different hosts, is:

```csharp
using Sisk.Core.Http;
using Sisk.Ssl;

static async Task Main(string[] args)
{
    using var app = HttpServer.CreateBuilder()
        .UseSsl(sslListeningPort: 5555) // <-- magic happens here
        .Build();

    app.Router.MapGet("/", r =>
    {
        return new HttpResponse("Hello from HTTPS!");
    });

    await app.StartAsync();
}
```

And a message will be displayed on your screen at the first startup of the program:

![HTTPS is coming, natively, to Sisk](/scr/sslnotf.png)

Congratulations, your application with SSL is ready to be used.

## Inspiration

In addition to all the reasons I mentioned above, the inspiration is to simplify the initial setup of having an application with Sisk that listens on HTTPS with just one line of code. Soon, more details with the [SslProxy documentation](https://docs.sisk-framework.org/docs/extensions/ssl-proxy) will be available.

This project is extremely **experimental**. Building an HTTPS listener from scratch is not so simple. Sisk took a long time to become minimally stable for production. I don't think it will be different with SslProxy. Even after these tests, it is important to note that this add-on for Sisk is under development, may have bugs, and was made to facilitate development.

Do not use **SslProxy** for production. I won't stop you, but I won't guarantee that everything will work correctly. It is still better to configure Apache, Nginx, or Cloudflare for your application to be served securely over the internet. I still emphasize that Sisk was designed to work through a [proxy in production](https://docs.sisk-framework.org/docs/deploying#proxying-your-application).

## Next steps

The next steps for SslProxy are testing, testing, and testing. Almost everything is working, from websockets, SSE, chunked responses, and the primitive HTTP as well. I like to say it is "almost ready" because I know there will still be bugs.

And I will need you to report these bugs so I can work on fixing them.

You can read more about the Sisk.SslProxy in:
- [GitHub](https://github.com/sisk-http/core/tree/main/extensions/Sisk.SslProxy)
- [NuGet](https://www.nuget.org/packages/Sisk.SslProxy)
- [Documentation](https://docs.sisk-framework.org/docs/extensions/ssl-proxy)

Thank you for using Sisk.