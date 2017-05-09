---
layout: post
title: Setting Identity Server 4 Url Behind A Load Balancer
tags:
- identityserver
- c#
---

One of the problems of having an Identity Server behind a Load Balancer is to get the *Discovery Document* to show the correct urls. This post shows a solution with a custom Middleware to assign the proper url to the discovery endpoint.


## The Discovery Endpoint Url Problem

When Identity Server host is bootstrapped as below, it runs on *localhost* and all Urls in the discovery endpoint has "localhost" in them. 

`Program.cs`
```csharp
Console.Title = "IdentityServer";

var host = new WebHostBuilder()
    .UseKestrel()
    .UseUrls("http://localhost:5000")
    .UseContentRoot(Directory.GetCurrentDirectory())
    .UseIISIntegration()
    .UseStartup<Startup>()
    .Build();

host.Run();
```

Discovery Endpoint output :

`http://localhost:5000/.well-known/openid-configuration`
```json 
{
    "issuer": "http://localhost:5000",
    "jwks_uri": "http://localhost:5000/.well-known/openid-configuration/jwks",
    "authorization_endpoint": "http://localhost:5000/connect/authorize",
    "token_endpoint": "http://localhost:5000/connect/token",
    "userinfo_endpoint": "http://localhost:5000/connect/userinfo",
    "end_session_endpoint": "http://localhost:5000/connect/endsession",
    "check_session_iframe": "http://localhost:5000/connect/checksession",
    "revocation_endpoint": "http://localhost:5000/connect/revocation",
    "introspection_endpoint": "http://localhost:5000/connect/introspect",
    "frontchannel_logout_supported": true,
    "frontchannel_logout_session_supported": true,
    "REST OF IT"
}
```

If this setup is kept behind a load balancer, with a domain name of `https://identity.example.com`, the Urls will still have *localhost* in them. This could be okay if you don't use the discovery document. But in many cases you have to. (Especially that `oidc-client.js` is the ideal way to bring token support to frontend and it uses discovery endpoint to identify the urls).

We cannot configure `https://identity.example.com` in the VM, because the VM, staying behind the load balancer, does not own that domain name. Infact the load balancer does.

## Using Middleware to Solve it

We can write a simple custom middleware to feed the correct Url to the discovery endpoint. 

`Config/PublicFacingUrlMiddleware.cs`
```csharp
using IdentityServer4.Extensions;
using Microsoft.AspNetCore.Http;
using System.Threading.Tasks;

namespace MyIdentityServer.Config
{
    /// <summary>
    /// Configures the HttpContext by assigning IdentityServerOrigin.
    /// </summary>
    public class PublicFacingUrlMiddleware
    {
        private readonly RequestDelegate _next;
        private readonly string _publicFacingUri;

        public PublicFacingUrlMiddleware(RequestDelegate next, string publicFacingUri)
        {
            _publicFacingUri = publicFacingUri;
            _next = next;
        }

        public async Task Invoke(HttpContext context)
        {
            var request = context.Request;

            context.SetIdentityServerOrigin(_publicFacingUri);
            context.SetIdentityServerBasePath(request.PathBase.Value.TrimEnd('/'));

            await _next(context);
        }
    }
}
```

This middleware sets one important property in the Identity Server. The `Origin`. That's used in generating the discovery document.

The last change is to use the new middleware in the IdentityServer pipeline in `Startup.cs`.

`Startup.cs` (where identityserver is added to the pipeline)
```csharp
//app.UseIdentityServer(); This was replaced by the following 4 lines.
app.UseMiddleware<PublicFacingUrlMiddleware>(Configuration[IdentityServerPublicFacingUri]);
app.ConfigureCors();
app.ConfigureCookies();
app.UseMiddleware<IdentityServerMiddleware>();
```

Now the `PublicFacingUrlMiddleware` is in the pipeline.
You can add a configuration in `appSettings.json` to configure it to what you want. In my case it is *https://identity.example.com*.

`appsettings.json`
```json
{
  "IdentityServerPublicFacingUri":  "https://identity.example.com",
  "FileLogging": {
    "IncludeScopes": false,
    "PathFormat": "Logs/log-{Date}.txt",
    "LogLevel": {
      "Default": "Debug",
      "System": "Information",
      "Microsoft": "Information"
    }
  }
}
```

Load the Disovery Endpoint now and you will have it as follows.

`http://localhost:5000/.well-known/openid-configuration`
```json
{
    "issuer": "https://identity.example.com",
    "jwks_uri": "https://identity.example.com/.well-known/openid-configuration/jwks",
    "authorization_endpoint": "https://identity.example.com/connect/authorize",
    "token_endpoint": "https://identity.example.com/connect/token",
    "userinfo_endpoint": "https://identity.example.com/connect/userinfo",
    "end_session_endpoint": "https://identity.example.com/connect/endsession",
    "check_session_iframe": "https://identity.example.com/connect/checksession",
    "revocation_endpoint": "https://identity.example.com/connect/revocation",
    "introspection_endpoint": "https://identity.example.com/connect/introspect",
}
```

*oidc-client.js* should be now happy to take care of the frontend.

Cheers!