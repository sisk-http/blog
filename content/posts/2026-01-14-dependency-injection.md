+++
title = 'Dependency Injection in Sisk: A Lightweight Approach That Just Works'
date = 2026-01-14T17:35:08-03:00
author = "Mia, CypherPotato"

[params]
tags = ['Dependency Injection']
img = '/pimg/1726008973_3795_89813.png'
+++

Have you ever found yourself drowning in the complexity of a full-blown DI container just to pass a database connection to your controllers? You know the feeling—hours configuring service lifetimes, wrestling with circular dependencies, and wondering if all this ceremony is really worth it. I've been there, and that's exactly why Sisk's approach to dependency injection feels like a breath of fresh air.

Sisk doesn't come with a built-in DI framework like ASP.NET's Microsoft.Extensions.DependencyInjection. Instead, it gives you the building blocks to manage dependencies in a way that's simple, explicit, and perfectly suited to lightweight applications. And once you get the hang of it, you might find yourself reaching for this pattern even in other projects.

## The RequestBag: Your Scoped Container

At the heart of Sisk's dependency management is the `HttpContext.RequestBag`—a typed dictionary that lives for the entire lifecycle of a request. Think of it as a scoped container that's automatically created when a request comes in and disposed when the response goes out.

Here's the beauty of it: you can store anything in this bag—database connections, authenticated users, session tokens, repositories—and access them from anywhere in your request pipeline.

```csharp
// In a request handler, set a value
context.RequestBag.Set(authenticatedUser);

// Later, in your route action, retrieve it
var user = request.Bag.Get<User>();
```

The `TypedValueDictionary` behind the scenes uses type information as keys, so you get compile-time safety and no magic strings. It's the kind of simplicity that makes you wonder why we ever made this so complicated.

## Building an Abstract Controller Base

Here's where things get interesting. The `HttpContext.Current` static property that gives you access to the current request's context from anywhere in your code. This opens the door to building clean, reusable controller abstractions.

Imagine you're building a REST API and every controller needs access to a database, some repositories, and the authenticated user. Instead of passing these through constructor injection or method parameters, you can define them as properties in a base class:

```csharp
public abstract class Controller : RouterModule
{
    protected DbContext Database => 
        HttpContext.Current.RequestBag.GetOrAdd(() => DbContextPool.Main.CreateDbContext());

    protected IUserRepository Users => 
        HttpContext.Current.RequestBag.GetOrAdd(() => new UserRepository(Database));

    protected User AuthenticatedUser => 
        HttpContext.Current.RequestBag.Get<User>();

    protected HttpRequest Request => 
        HttpContext.Current.Request;
}
```

The `GetOrAdd` method is particularly clever—it creates the instance lazily on first access and reuses it for subsequent calls within the same request. Your `DbContext` gets instantiated once per request, shared across all repositories, and you didn't have to configure a single service registration.

## Request Handlers as Middleware

What about cross-cutting concerns like authentication? Sisk uses request handlers—essentially middleware that runs before or after your route actions. These are perfect for populating the `RequestBag` with dependencies your routes will need.

```csharp
public class AuthenticateUser : IRequestHandler
{
    public RequestHandlerExecutionMode ExecutionMode { get; init; } = 
        RequestHandlerExecutionMode.BeforeResponse;

    public HttpResponse? Execute(HttpRequest request, HttpContext context)
    {
        User? authenticatedUser = ValidateToken(request);
        if (!authenticatedUser.HasValue)
        {
            return new HttpResponse(401);
        }

        context.RequestBag.Set(authenticatedUser);

        return null;
    }
}
```

Apply it to routes with an attribute, and every controller method that needs authentication just... has it. Here's how you'd use it in practice:

```csharp
[RoutePrefix("/api/profile")]
public class ProfileController : Controller
{
    [RouteGet]
    [RequestHandler<AuthenticateUser>]
    public HttpResponse GetProfile()
    {
        // AuthenticatedUser is already available from our base Controller class
        return new HttpResponse(200)
            .WithContent(JsonContent.Create(new
            {
                AuthenticatedUser.Id,
                AuthenticatedUser.Name,
                AuthenticatedUser.Email
            }));
    }

    [RoutePut]
    [RequestHandler<AuthenticateUser>]
    public async Task<HttpResponse> UpdateProfile()
    {
        var updateData = await Request.GetJsonAsync<ProfileUpdateDto>();
        
        AuthenticatedUser.Name = updateData.Name;
        AuthenticatedUser.Bio = updateData.Bio;
        
        await Users.UpdateAsync(AuthenticatedUser);
        
        return new HttpResponse(200)
            .WithContent(JsonContent.Create(new { Message = "Profile updated!" }));
    }
}
```

No constructor injection ceremony, no scope resolution—the dependency is there when you need it, populated by the request handler and accessed through the base controller's property.

## Cleanup: Disposing Resources Gracefully

Resource cleanup is often the trickiest part of manual dependency management, but Sisk handles this elegantly. We have `HttpServerConfiguration.DisposeDisposableContextValues`, which is enabled by default. Any `IDisposable` objects in your `RequestBag` get automatically disposed when the HTTP session closes. No manual cleanup code, no memory leaks—it just works.

For more granular control, you can use an `HttpServerHandler`:

```csharp
public class ObjectDisposerHandler : HttpServerHandler
{
    protected override void OnHttpRequestClose(HttpServerExecutionResult result)
    {
        result.Context.RequestBag.GetOrDefault<DbContext>()?.Dispose();
    }
}
```

## The Sweet Spot: Where This Approach Shines

This pattern is a perfect fit for microservices, REST APIs, IoT backends, embedded servers, or any project where you want to move fast without boilerplate. You get request-scoped dependencies, lazy initialization, automatic disposal, and zero configuration—all in a few lines of code.

The explicit nature of `GetOrAdd` and `RequestBag.Set` means you always know where your dependencies come from and when they're created. No surprises, no debugging DI container configuration at 2 AM. And if your project grows and you eventually need a full-featured container, nothing stops you from integrating one—Sisk's flexibility means you're never locked into a single approach.

So next time you're spinning up a new C# HTTP project, give Sisk's pragmatic dependency management a try. You might be surprised how far you can go with just a `RequestBag` and a well-designed base controller.