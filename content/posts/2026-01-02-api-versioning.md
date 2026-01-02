+++
title = 'API Versioning: Making the Most of RoutePrefix'
date = 2026-01-02T19:46:08-03:00
author = "Mia, CypherPotato"

[params]
tags = ['Routing']
img = '/pimg/1726025967_8537_27164.png'
+++

Have you ever shipped a breaking change to your API and watched the Slack notifications pile up from angry consumers? Yeah, me too. It's not fun. That's why API versioning is one of those things that seems optional until suddenly it's absolutely critical—and at that point, retrofitting it becomes a nightmare.

Today I want to walk through how Sisk makes API versioning surprisingly elegant using the `[RoutePrefix]` attribute combined with `RouterModule`. It's one of those patterns that, once you see it, makes you wonder why you ever did it any other way.

## The Problem with Flat Routes

When you're building a quick prototype, it's tempting to just slap routes directly onto your router: `/users`, `/products`, `/orders`. Simple enough. But what happens six months later when you need to completely redesign how authentication works, or change the response schema for a critical endpoint? You can't just break existing clients.

The classic solution is URL-based versioning: `/v1/users`, `/v2/users`. And in Sisk, this becomes almost trivially easy thanks to how `RoutePrefixAttribute` works with class inheritance.

## Building a Versioned API Structure

The `RoutePrefixAttribute` in Sisk does exactly what you'd expect—it prepends a path segment to every route defined in a class. But the magic happens when you combine this with an abstract base controller. Here's what I mean:

```csharp
public abstract class ApiController : RouterModule
{
    static T GetOrAddCurrent<T>(Func<T> getter) where T : notnull
        => HttpContext.Current.RequestBag.GetOrAdd(getter);

    public HttpContext Context => HttpContext.Current;
    public HttpRequest Request => HttpContext.Current.Request;
    public MainDbContext Database => GetOrAddCurrent(MainDbContext.Pool.CreateDbContext);
}
```

This base class extends `RouterModule`—Sisk's built-in abstraction for grouping routes, request handlers, and prefixes. Notice how it exposes `HttpContext.Current` for accessing the request context without needing to pass parameters around. That `GetOrAddCurrent` pattern is particularly clever for per-request resources like database contexts.

Now, versioning becomes a simple matter of decoration:

```csharp
[RoutePrefix("/v1/users")]
public sealed class UsersControllerV1 : ApiController
{
    [RouteGet]
    public async Task<ListUsersResponse> List()
    {
        // v1 implementation
    }
    
    [RouteGet("/<id>")]
    public async Task<UserResponse> GetById()
    {
        var id = Request.RouteParameters["id"].GetGuid();
        // ...
    }
}
```

The `[RouteGet]` attribute without a path maps to the prefix root—so `GET /v1/users` hits the `List` method. Add a path like `"/<id>"` and it becomes `GET /v1/users/{id}`. Clean and predictable.

## Evolving to V2 (Without Breaking V1)

Here's where it gets interesting. When you need a new version, you just create another controller:

```csharp
[RoutePrefix("/v2/users")]
public sealed class UsersControllerV2 : ApiController
{
    [RouteGet]
    public async Task<PaginatedUsersResponse> List()
    {
        // v2 returns paginated results
        var page = Request.Query["page"].GetInteger(1);
        var pageSize = Request.Query["pageSize"].GetInteger(20);
        // ...
    }
}
```

Both controllers coexist peacefully. V1 clients keep working while V2 clients get the shiny new paginated response. No feature flags, no conditional logic, no mess.

What I particularly appreciate about Sisk's approach is that each controller is truly independent. Want to add specific request handlers (middleware) to V2 that don't apply to V1? Just use the `HasRequestHandler` method in the constructor or apply `[RequestHandler]` attributes. The `RouterModule` base class gives you `Prefix` and `RequestHandlers` properties to configure at runtime if you prefer that over attributes.

## Wiring It All Up

Registration is straightforward. Sisk's `Router` class provides `SetObject<T>()` to scan a class for route-attributed methods:

```csharp
var router = new Router();
router.SetObject<UsersControllerV1>();
router.SetObject<UsersControllerV2>();
```

Or, if you have many controllers and want automatic discovery, `AutoScanModules<T>()` will find everything that inherits from your base controller:

```csharp
router.AutoScanModules<ApiController>();
```

Just note that `AutoScanModules` uses reflection and requires an accessible parameterless constructor on each module. It's convenient for development but worth considering the trade-offs in production builds where AOT compilation matters.

## Alternative: Programmatic Prefixes

The `[RoutePrefix]` attribute is declarative and clean, but Sisk also supports setting prefixes programmatically. Inside your `RouterModule` subclass, you can call `HasPrefix("/v1/users")` in the constructor, or set the `Prefix` property directly. This can be useful when the prefix depends on configuration or needs to be computed at runtime.

The `Router` itself also has a `Prefix` property that affects all subsequently defined routes—handy for global prefixes like `/api`.



<think>
The user wants me to rewrite just the "Trade-offs and Considerations" section from the original blog article about API versioning. They've pointed out that header-based versioning in Sisk doesn't require a RequestHandler - it can be done simply with a property in the base controller that reads the header directly.

I should update this section to reflect:
1. The simplicity of header-based versioning in Sisk (just a property)
2. Compare URL-based vs header-based approaches more fairly
3. Keep the casual, reflective tone of the original article
4. Mention the trade-offs of each approach

Let me rewrite this section with the new context.
</think>



<think>
The user wants me to add a practical example of header-based versioning to the "Trade-offs and Considerations" section. I should show how to use the `ApiVersion` property in a route to branch logic based on the version.
</think>

## Trade-offs and Considerations

Is URL-based versioning always the right choice? Honestly, it depends. Header-based versioning—using something like `Accept-Version` or `X-Api-Version`—is arguably more RESTful since the resource URL stays consistent regardless of version. And in Sisk, implementing it is surprisingly trivial:

```csharp
public abstract class ApiController : RouterModule
{
    public HttpRequest Request => HttpContext.Current.Request;
    
    public int ApiVersion =>
        Request.Headers.AsStringValueCollection()
            .GetItem("Accept-Version")
            .MaybeNullOrEmpty()?
            .GetInteger() ?? 1;
}

[RoutePrefix("/api/users")]
public sealed class UsersController : ApiController
{
    [RouteGet]
    public async Task<object> List()
    {
        return ApiVersion switch {
            1 => await UserServiceV1.ListUsers(),
            2 => await UserServiceV2.ListUsers(),
            _ => new HttpResponse(400)
        }
    }
}
```

That's it. No middleware, no request handlers, just a property on your base controller. Clients specify the version with a header:

```bash
curl http://localhost:5000/api/users                        # defaults to v1
curl -H "Accept-Version: 2" http://localhost:5000/api/users # explicit v2
```

The trade-off? Header-based versioning is less discoverable. You can't just look at a URL and know which version you're hitting. API gateways and caching layers sometimes struggle with it too—they're often configured to cache based on URL, not headers. And let's be honest: telling a frontend dev "just add this header to every request" is one more thing that can go wrong.

URL versioning (`/v1/users`, `/v2/users`) is explicit, obvious, and plays nicely with tooling. But it does mean maintaining separate controller classes—or at least separate route prefixes—which can feel heavy when V2 is 95% identical to V1.

My take? Start with URL versioning via `[RoutePrefix]` for public APIs where discoverability matters. Use header-based versioning for internal services or when you want cleaner URLs and your consumers are sophisticated enough to handle it. Sisk makes both approaches straightforward, so pick the one that fits your context.

One thing to watch regardless of approach: code duplication. When versions share most logic, extract that into services. Let your controllers be thin dispatchers that route to the right implementation. Your future self will thank you when V3 inevitably arrives.

## Wrapping Up

API versioning in Sisk isn't some elaborate framework feature—it's just a natural consequence of well-designed building blocks. `[RoutePrefix]` plus `RouterModule` plus clean inheritance gives you everything you need. Start with this pattern early, even if you think you'll never need it. Your future self will thank you when that "small" breaking change comes along and you can ship V2 with confidence.

What's your preferred versioning strategy? Have you tried other approaches with Sisk that worked well? I'd love to hear about it.