---
title: More migration scenarios
description: This section describes additional migration scenarios and techniques for upgrading .NET Framework apps to .NET Core / .NET 5.
author: ardalis
ms.date: 02/11/2021
---

# More migration scenarios

[!INCLUDE [book-preview](../../../includes/book-preview.md)]

This section describes several different ASP.NET app scenarios and situation, and offers specific techniques for solving each of them. You can use this section to identify scenarios applicable to your app and evaluate which techniques will work for your app and its hosting environment.

## Migrate ASP.NET MVC 5 and WebApi 2 to ASP.NET Core MVC

A common scenario in ASP.NET MVC 5 and Web API 2 apps was for both products to be installed in the same application. This is a supported and relatively common approach used by many teams, but because the two products use different abstractions, there is some redundant effort needed. For example, setting up routes for ASP.NET MVC is done using methods on `RouteCollection`, such as `MapMvcAttributeRoutes()` and `MapRoute()`. But ASP.NET Web API 2 routing is managed with `HttpConfiguration` and methods like `MapHttpAttributeRoutes()` and `MapHttpRoute()`.

The `eShopLegacyMVC` app includes both ASP.NET MVC and Web API, and includes methods in its `App_Start` folder for setting up routes for both. It also supports dependency injection using Autofac, which also requires two sets of similar work to configure:

```csharp
protected IContainer RegisterContainer()
{
    var builder = new ContainerBuilder();

    var thisAssembly = Assembly.GetExecutingAssembly();
    builder.RegisterControllers(thisAssembly);      // MVC controllers
    builder.RegisterApiControllers(thisAssembly);   // Web API controllers

    var mockData = bool.Parse(ConfigurationManager.AppSettings["UseMockData"]);
    builder.RegisterModule(new ApplicationModule(mockData));

    var container = builder.Build();

    // set mvc resolver
    DependencyResolver.SetResolver(new AutofacDependencyResolver(container));

    // set webapi resolver
    var resolver = new AutofacWebApiDependencyResolver(container);
    GlobalConfiguration.Configuration.DependencyResolver = resolver;

    return container;
}
```

When upgrading these apps to use ASP.NET Core, this duplicate effort and the confusion that sometimes accompanies it is eliminated. ASP.NET Core MVC is a unified framework with one set of rules for routing, filters, and more. Dependency injection is built into ASP.NET Core itself. All of this can can be configured in `Startup.cs`, as is shown in the `eShopPorted` app in the sample.

## Migrate HttpResponseMessage to ASP.NET Core

Some ASP.NET Web API apps may have action methods that return `HttpResponseMessage`. This type does not exist in ASP.NET Core. Below is an example of its usage in a Delete action method, using the `ResponseMessage` helper method on the base `ApiController`:

```csharp
[HttpDelete]
// DELETE api/<controller>/5
public IHttpActionResult Delete(int id)
{
    var brandToDelete = service.GetCatalogBrands().FirstOrDefault(x => x.Id == id);
    if (brandToDelete == null)
    {
        return ResponseMessage(new HttpResponseMessage(HttpStatusCode.NotFound));
    }

    // demo only - don't actually delete
    return ResponseMessage(new HttpResponseMessage(HttpStatusCode.OK));
}
```

In ASP.NET Core MVC, there are helper methods available for all of the common HTTP response status codes, so the above method would be ported to the following code:

```csharp
[HttpDelete("{id}")]
public IActionResult Delete(int id)
{
    var brandToDelete = _service.GetCatalogBrands().FirstOrDefault(x => x.Id == id);
    if (brandToDelete == null)
    {
        return NotFound();
    }

    // demo only - don't actually delete
    return Ok();
}
```

If you do find that you need to return a custom status code for which no helper exists, you can always use `return StatusCode(int statusCode)` to return any numeric code you like.

## Migrate content negotiation from ASP.NET Web API to ASP.NET Core

ASP.NET Web API 2 supports [content negotiation](https://docs.microsoft.com/aspnet/web-api/overview/formats-and-model-binding/content-negotiation) natively. The sample app includes a `BrandsController` demonstrates this support, listing its results in either XML or JSON based on whether request's `Accept` header includes `application/xml` or `application/json`.

ASP.NET MVC 5 apps do not have content negotiation support built in.

Content negotiation is preferable to returning a specific encoding type, as it is more flexible and makes the API available to a larger number of clients. If you currently have action methods that return a specific format, you should consider modifying them to return a result type that supports content negotiation when you port the code to ASP.NET Core.

The following code returns data in JSON format regardless of client `Accept` header content:

```csharp
[HttpGet]
public ActionResult Index()
{
    return Json(new { Message = "Hello World!" });
}
```

[ASP.NET Core MVC supports content negotiation natively](https://docs.microsoft.com/aspnet/core/web-api/advanced/formatting), provided an appropriate [return type](https://docs.microsoft.com/aspnet/core/web-api/action-return-types) is used. Content negotiation is implemented by [ObjectResult] which is returned by the status code-specific action results returned by the controller helper methods. The previous action method, implemented in ASP.NET Core MVC and using content negotiation, would be:

```csharp
public IActionResult Index()
{
    return Ok(new { Message = "Hello World!"} );
}
```

This will default to returning the data in JSON format. XML and other formats will be used [if the app has been configured with the appropriate formatter](https://docs.microsoft.com/aspnet/core/web-api/advanced/formatting).

## Dependent libraries with .NET Framework dependencies

Many .NET Framework web apps depend on class libraries that make use of older .NET Framework classes. In some cases, these .NET Framework classes may not have an equivalent in .NET Core, or may be deprecated. This is a relatively common situation which can usually be addressed in an incremental fashion.

As an example, consider this class which exists in a .NET 4.6.1 class library as part of a legacy application:

```csharp
public class Serializing
{
    public Stream SerializeBinary(object input)
    {
        var stream = new MemoryStream();
        var binaryFormatter = new BinaryFormatter();
        binaryFormatter.Serialize(stream, input);
        stream.Seek(0, SeekOrigin.Begin);
        return stream;
    }

    public object DeserializeBinary(Stream stream)
    {
        var binaryFormatter = new BinaryFormatter();
        stream.Seek(0, SeekOrigin.Begin);
        return binaryFormatter.Deserialize(stream);
    }
}
```

This class references the BinaryFormatter class, which has been [marked obsolete for security reasons](https://docs.microsoft.com/dotnet/core/compatibility/core-libraries/5.0/binaryformatter-serialization-obsolete). In ASP.NET Core 5.0 and later, references to the `BinaryFormatter.Serialize` and `BinaryFormatter.Deserialize` methods will begin to throw a `NotSupportedException`. What steps can be taken to port an ASP.NET MVC or WebApi app to ASP.NET Core when it makes use of a shared library referencing these types and methods?

### Initial port to ASP.NET Core

The simplest initial step to support moving the app to ASP.NET Core is to migrate to an older version of ASP.NET Core such as 2.1 or 2.2 and target .NET Framework. The [referenced sample](https://github.com/dotnet-architecture/eShopModernizing) demonstrates a project, `eShopPorted`, which uses this strategy. Using this approach, the new ASP.NET Core app can continue to reference the original .NET Class library, losing no functionality.

### Next steps

After the original app has been ported to ASP.NET Core, the next step is to migrate functionality from .NET Framework libraries to .NET Standard libraries. In most cases, functionality will continue to work between .NET Framework and .NET Standard. In those situations where the .NET Framework library has not .NET Standard equivalent (or it has been marked obsolete as with `BinaryFormater`), a choice must be made. The available options typically include:

- Find and use a third party alternative package
- Author the needed functionality
- Use a different approach
- Maintain a legacy service to perform the needed task

In many cases, open source and/or commercial libraries exist that can fill any gaps between .NET Framework and .NET Standard support. This should typically be the first option considered, as the resolution may be as simple as adding a new NuGet package and perhaps authoring an [adapter](https://deviq.com/design-patterns/adapter-design-pattern) to connect its API to your app's usage of it.

In rare cases, it may make sense to author the needed functionality yourself, placing it in a new custom library your team maintains. This should usually be a last resort unless the functionality is very simple and/or well-understood by the team. Unless the functionality is central to the application's design, it's usually better to leverage a well-established, tested, and ideally supported library than to attempt to reinvent such functionality yourself.

In some cases, like with `BinaryFormatter`, the best approach may be to consider a different approach. The use of `BinaryFormatter` in web environments is strongly discouraged for security reasons, but alternatives formats like JSON and XML are quite well-supported. As the ported app is upgraded from ASP.NET Core 2.x to ASP.NET Core 3.1 or 5, it can modify its approach to use a JSON formatter from the `System.Text.Json` namespace, for example.

Finally, if there are some legacy services your app requires that you can't approach differently and which no longer exist in ASP.NET Core 3.1 and later, your last option is to expose the functionality as an internal service. Your new, ported application can reference the needed functionality by calling the .NET Framework service over HTTP and getting back the result. This approach adds some maintenance, hosting, and performance overhead, but can be effectively used to continue supporting the existing functionality while moving the bulk of the app to ASP.NET Core.

## Authentication logic using ClaimsPrincipal.Current

In ASP.NET 4.x apps, it was not unusual to see the use of `ClaimsPrincipal.Current` to get the current authenticated user's identity. In ASP.NET Core, this property is no longer set, so any code that depends on it must be [updated to use another approach](https://docs.microsoft.com/aspnet/core/migration/claimsprincipal-current). The `Thread.CurrentPrincipal` property is likewise not set in ASP.NET Core. These types were used statically in .NET Framework apps, but in ASP.NET Core dependencies like these are typically referenced through dependency injection. Thus, classes that need access to user identity will typically need to request an appropriate dependency to retrieve that information from their constructor. ASP.NET Core MVC's `ControllerBase` provides a base `User` property that can provide this functionality in controllers that inherit from it.

The most common uses of `ClaimsPrincipal.Current` were to check whether the user was logged in, and if so, to retrieve their username. In addition, it could be used to access multiple `Claims` as well as potentially multiple identities. These properties remain on the `ClaimsPrincipal` type in ASP.NET Core; the main difference is in how one gets access to the current user. This will generally vary based on where you need the information. Controllers can use the `User` property on `ControllerBase`. Utility code and static methods can request the information as a parameter rather than accessing it statically. ASP.NET Core types with access to `HttpContext` can use its `User` property, and other services that need direct access can request an `IHttpContextAccessor` to get an instance of `HttpContext` and then use its `User` property. Additional specific guidance on how to reference `HttpContext` from ASP.NET Core apps is available in the [documentation](https://docs.microsoft.com/aspnet/core/migration/claimsprincipal-current).
## References

- [ASP.NET Web API Content Negotiation](https://docs.microsoft.com/aspnet/web-api/overview/formats-and-model-binding/content-negotiation)
- [Format response data in ASP.NET Core Web API](https://docs.microsoft.com/aspnet/core/web-api/advanced/formatting)
- [BinaryFormatter serialization methods are obsolete and prohibited in ASP.NET apps](https://docs.microsoft.com/dotnet/core/compatibility/core-libraries/5.0/binaryformatter-serialization-obsolete)
- [Migrate from ClaimsPrincipal.Current](https://docs.microsoft.com/aspnet/core/migration/claimsprincipal-current)

>[!div class="step-by-step"]
>[Previous](strategies-migrating-in-production.md)
>[Next](deployment-scenarios.md)
