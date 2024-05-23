---
layout: post
title: Swagger is being removed from .NET 9
image: /assets/img/posts/api_explorer.webp
date: 2024-05-23 08:00 +0100
categories: [".NET", "ASP.NET Core"]
tags: ["openapi", "swagger", "dotnet"]
pin: true
---

You've probably heard or seen on the internet that Swagger (a.k.a Swachbuckle) will be deprecated with the release of .NET 9. So I'll try to go into more details about that and discuss the options to providing OpenAPI support for ASP.NET Core Web APIs starting from .NET 9.

## Is it official ?
Yes it's an official announcement made by the ASP.NET Core team through a [Github issue](https://github.com/dotnet/aspnetcore/issues/54599), which spread some concern amongst the .NET community.

The inclusion of the `Swachbuckle.AspNetCore` package as a dependency dates back to .NET 5, which provided built-in support for OpenAPI along with a web-based UI to test and interact with an API.

As described by the author of the announcement, the shipping of the `Swachbuckle.AspNetCore` package as a dependency of ASP.NET Core web API templates will be suspended beginning with .NET 9.

## Why ?
The reason why Microsoft decided to ditch the package is the lack of interactivity from the maintainer, although the package repo showed some activity recently, but there still numerous issues that need to be addressed and resolved, besides there was no official release for .NET 8.

The ASP.NET Core team's plan is to get rid of the dependency on `Swachbuckle.AspNetCore` from the web API templates and extend the in-house library `Microsoft.AspNetCore.OpenApi` to provide OpenAPI support.

That does not mean the `Swachbuckle` packages are going anywhere, they can still be used like any other packages such as [NSwag](https://github.com/RicoSuter/NSwag) which allows —in addition to OpenAPI support— client and server generation from an OpenAPI document.

## Goodbye Swagger UI
The fact that OpenAPI support becomes a first-class citizen in ASP.NET Core, doesn't mean we'll be having all the Swagger tools that were provided by 3<sup>rd</sup> party libraries and allowed interacting with Open API documents, like the Swagger UI, client and server generation tools, etc...

Unfortunately the Swagger aspect is being removed with the release of .NET 9, and developers will have to rely on other libraries or custom development to bring back a UI to test and interact with an ASP.NET Core web API.

## Current state-of-the-art
When you generate a new project using the ASP.NET Core minimal web API template as an option, your `Program.cs` file will look like this :

```csharp
var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
// Learn more about configuring Swagger/OpenAPI at https://aka.ms/aspnetcore/swashbuckle
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();

var summaries = new[]
{
    "Freezing", "Bracing", "Chilly", "Cool", "Mild", "Warm", "Balmy", "Hot", "Sweltering", "Scorching"
};

app.MapGet("/weatherforecast", () =>
{
    var forecast =  Enumerable.Range(1, 5).Select(index =>
        new WeatherForecast
        (
            DateOnly.FromDateTime(DateTime.Now.AddDays(index)),
            Random.Shared.Next(-20, 55),
            summaries[Random.Shared.Next(summaries.Length)]
        ))
        .ToArray();
    return forecast;
})
.WithName("GetWeatherForecast")
.WithOpenApi();

app.Run();

record WeatherForecast(DateOnly Date, int TemperatureC, string? Summary)
{
    public int TemperatureF => 32 + (int)(TemperatureC / 0.5556);
}
```
{: file="Program.cs"}
{: .nolineno }

In the preceding code :
- `builder.Services.AddSwaggerGen();` adds OpenAPI/Swagger services to the DI container of ASP.NET Core.
- `app.UseSwagger();` configures the HTTP request pipeline to serve the OpenAPI document.
- `app.UseSwaggerUI();` configures the HTTP request pipeline to serve the Swagger UI endpoints.

The `csproj` file holds a reference to the `Swachbuckle.AspNetCore` NuGet package :
```xml
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.OpenApi" Version="8.0.5" />
    <PackageReference Include="Swashbuckle.AspNetCore" Version="6.4.0" />
  </ItemGroup>

</Project>
```
{: file=".csproj"}
{: .nolineno }

## What are the options with .NET 9 ?

As mentionned earlier, with the release of .NET 9 the `Swachbuckle.AspNetCore` package will be removed as a dependency from web API templates.

If you don't stick to LTS versions of .NET (which is something I discourage), there a few changes that needs to be made to switch to using the Microsoft's OpenAPI library.

### OpenAPI document generation

First thing to do is to get rid of the package, now `csproj` files will look like the following :

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.OpenApi" Version="9.0.*-*" />
  </ItemGroup>

</Project>
```
{: file=".csproj"}
{: .nolineno }

The extension methods provided by `Swachbuckle.AspNetCore` to register OpenAPI/Swagger-related services will be replaced by the following line :

```csharp
builder.Services.AddOpenApi();
```
{: file="Program.cs"}
{: .nolineno }

And to enable the endpoint serving the generated OpenAPI document in JSON format, the following line will be replacing the one provided by the `Swachbuckle` package :
```csharp
app.MapOpenApi();
```
{: file="Program.cs"}
{: .nolineno }

The updated `Program.cs` file now looks like the following :
```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddOpenApi();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
}

app.UseHttpsRedirection();

// ...

app.Run();
```
{: file="Program.cs"}
{: .nolineno }


With that the OpenAPI document is ready to be served at the `/openapi/v1.json` endpoint.

### Options for OpenAPI document customization

Numerous options are provided for customizing the generated OpenAPI document :
- The document can be customized by changing it's name, OpenAPI version, the route the OpenAPI document is served at.
- Caching is an option to avoid document generation on each HTTP request.
- The access to an OpenAPI endpoint can be limited to only authorized users.

### Customizing with Transformers

The OpenAPI document can also be customized with transformers, which is useful for scenarios like adding top-level information to an OpenAPI document, adding parameters to all operations, modifying descriptions of parameters and so on.

This sample from Microsoft's [documentation](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis/aspnetcore-openapi?view=aspnetcore-9.0&tabs=netcore-cli) demonstrates the use of a transformer to add JWT bearer-related schemes to the OpenAPI document's top level :

```csharp
using Microsoft.AspNetCore.Authentication;
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.OpenApi;
using Microsoft.Extensions.DependencyInjection;

var builder = WebApplication.CreateBuilder();

builder.Services.AddAuthentication().AddJwtBearer();

builder.Services.AddOpenApi(options =>
{
    options.UseTransformer<BearerSecuritySchemeTransformer>();
});

var app = builder.Build();

app.MapOpenApi();

app.MapGet("/", () => "Hello world!");

app.Run();

internal sealed class BearerSecuritySchemeTransformer(IAuthenticationSchemeProvider authenticationSchemeProvider) : IOpenApiDocumentTransformer
{
    public async Task TransformAsync(OpenApiDocument document, OpenApiDocumentTransformerContext context, CancellationToken cancellationToken)
    {
        var authenticationSchemes = await authenticationSchemeProvider.GetAllSchemesAsync();
        if (authenticationSchemes.Any(authScheme => authScheme.Name == "Bearer"))
        {
            var requirements = new Dictionary<string, OpenApiSecurityScheme>
            {
                ["Bearer"] = new OpenApiSecurityScheme
                {
                    Type = SecuritySchemeType.Http,
                    Scheme = "bearer", // "bearer" refers to the header name here
                    In = ParameterLocation.Header,
                    BearerFormat = "Json Web Token"
                }
            };
            document.Components ??= new OpenApiComponents();
            document.Components.SecuritySchemes = requirements;
        }
    }
}
```
{: file="Program.cs"}
{: .nolineno}

And that's about it when it comes to what Microsoft will be providing with it's in-house OpenAPI support. No UI, no tools to generate client code, So what are the options if we want ad-hoc testing to be built into our APIs ?

## OpenAPI tooling options

### Keep using the Swagger UI

You can still use the Swagger UI package, which you'll have to install manually and configure to serve the Swagger UI endpoint :

```bash
dotnet add package Swashbuckle.AspNetCore.SwaggerUI
```
{: file=".NET CLI"}
{: .nolineno}

Configure the Swagger UI middleware with the OpenAPI document route :
```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddOpenApi();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();

    // Configuring the Swagger UI endpoint
    app.UseSwaggerUI(options =>
    {
        options.SwaggerEndpoint("/openapi/v1.json", "v1");
    });
}

app.UseHttpsRedirection();

// ...

app.Run();
```
{: file="Program.cs"}
{: .nolineno }

### Use Scalar interactive API documentation

[Scalar](https://scalar.com/) is an open-source highly customizable platform that offers a set of tools for interacting with APIs, generating and customizing API references based on an OAS file.

![Scalar](/assets/img/posts/scalar_dark.png){: .light }
![Scalar](/assets/img/posts/scalar_light.png){: .dark }
_Scalar API references page - https://docs.scalar.com/_

Scalar can be integrated as a NuGet package into ASP.NET Core web APIs :
```bash
dotnet add package Scalar.AspNetCore
```
{: file=".NET CLI"}

To configure the middleware that will be serving the Scalar API reference UI add the following lines to your `Program.cs` :
```csharp
// ...

if (app.Envitonment.IsDevelopment())
{
    app.MapScalarApiReference();
}

// ...
```
{: file="Program.cs"}

By default Scalar uses the OpenAPI document route `/openapi/v1.json` which can be customized using the middleware options.

The Scalar API reference UI is servet at `/scalar/v1` by default which can also be customized.

![Scalar](/assets/img/posts/scalar_net_dark.png){: .light }
![Scalar](/assets/img/posts/scalar_net_light.png){: .dark }

## Wrap Up

In conclusion, ASP.NET Core team's decision will provide a more streamlined and integrated approach for handling OpenAPI support within the ASP.NET Core ecosystem.

Developers can still leverage the Swagger UI package a.k.a Swachbuckle or switch to modern alternatives like the Scalar API documentation platform that offers a more robust and customizable solution for generating and interacting with API references.

Exploring with new customization and tooling options will ensure that API documentation and testing remain efficient and effective, it will also help the community to continue building and maintaining high-quality web APIs in the evolving .NET landscape.