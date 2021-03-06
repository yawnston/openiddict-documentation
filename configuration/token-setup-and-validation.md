# Token setup and validation

For an overview of the different token formats, see: [here](../guide/token-formats.md)

In OpenID Connect there are three types of tokens: access tokens, id tokens, and refresh tokens [See spec](https://openid.net/specs/openid-connect-core-1_0.html#Introduction). When this guide refers to _tokens_ it is referring to access tokens.

Authorization servers are responsible for token generation. Clients (server app or web app) request tokens and then use tokens to request resources. For example, a javascript web application may make API calls that require authorization, so the token is sent along in the header of every request.

Token validation needs to be configured for servers that have API endpoints that require authorization. This could be authorization servers or standalone servers, called resource servers. An example authorization server API endpoint may be '/api/User' that returns the current user. A resource server API endpoint for a note-taking app may be '/notes' that returns the user's notes.

Below shows code snippets for token generation and token validation for each token format.


# Default configuration: Opaque tokens


## Default token generation

### Authorization server
```csharp
// Startup.cs

public void ConfigureServices(IServiceCollection services)
{
    //If tokens need to be validated in a separate resource server, configure a shared ASP.NET Core DataProtection with shared key store and application name
    //See https://docs.microsoft.com/en-us/aspnet/core/security/data-protection/configuration/overview?view=aspnetcore-2.1&tabs=aspnetcore2x#setapplicationname
    services.AddDataProtection()
        .PersistKeysToFileSystem(new System.IO.DirectoryInfo(@"[UNC PATH]"))
        .SetApplicationName("[APP NAME]");

    // Register the OpenIddict services.
    // Additional configuration is only needed if using Introspection on resource servers
    services.AddOpenIddict()
        .AddCore(...)

        .AddServer(options => 
        {
            //...
            //This is required if using introspection on resource servers
            //This is not needed if resource servers will use shared ASP.NET Core DataProtection
            //options.EnableIntrospectionEndpoint("/connect/introspect");
        })
}

public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
{
    //...
 
    //must come before using MVC
    app.UseAuthentication();
 
    //...
}
```

## Default token validation

### Authorization server
```csharp
// Startup.cs

public void ConfigureServices(IServiceCollection services)
{
    services.AddOpenIddict()
        .AddCore(...)

        .AddServer(...)

        .AddValidation();
}

public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
{
    //...
    //must come before using MVC
    app.UseAuthentication();
    //...
}
```

### Resource server shared ASP.NET Core DataProtection
```csharp
// Startup.cs

public void ConfigureServices(IServiceCollection services)
{
    //If tokens need to be validated in a separate resource server, configure a shared ASP.NET Core DataProtection with shared key store and application name
    //See https://docs.microsoft.com/en-us/aspnet/core/security/data-protection/configuration/overview?view=aspnetcore-2.1&tabs=aspnetcore2x#setapplicationname
    services.AddDataProtection()
        .PersistKeysToFileSystem(new System.IO.DirectoryInfo(@"[UNC PATH]"))
        .SetApplicationName("[APP NAME]");

    services.AddOpenIddict()
        //This adds a "Bearer" authentication scheme
        .AddValidation();

    //Optionally set Bearer token authentication as default
    //services.AddAuthentication(options =>
    //{
    //    options.DefaultAuthenticateScheme = OpenIddictValidationDefaults.AuthenticationScheme;
    //    options.DefaultChallengeScheme = OpenIddictValidationDefaults.AuthenticationScheme;
    //});
}

public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
{
    //...
    //must come before using MVC
    app.UseAuthentication();
    //...
}
```
### Resource server - introspection
```csharp
// Introspection requires a request to auth server for every token so shared ASP.NET Core DataProtection is preferred.
// To use introspection, you need to create a new client application and grant it the introspection endpoint permission.
// Startup.cs

public void ConfigureServices(IServiceCollection services)
{
    services.AddAuthentication(options =>
    {
        options.DefaultScheme = OAuthIntrospectionDefaults.AuthenticationScheme;
    })
    .AddOAuthIntrospection(options =>
    {
        //example settings
        options.Authority = new Uri("http://localhost:12345/");
        options.Audiences.Add("resource-server-1");
        options.ClientId = "resource-server-1";
        options.ClientSecret = "846B62D0-DEF9-4215-A99D-86E6B8DAB342";
        options.RequireHttpsMetadata = false;

        // Note: you can override the default name and role claims:
        // options.NameClaimType = "custom_name_claim";
        // options.RoleClaimType = "custom_role_claim";
    });

    //Optionally set Bearer token authentication as default
    //services.AddAuthentication(options =>
    //{
    //    options.DefaultAuthenticateScheme = OAuthIntrospectionDefaults.AuthenticationScheme;
    //    options.DefaultChallengeScheme = OAuthIntrospectionDefaults.AuthenticationScheme;
    //});
}

public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
{
    //...
    //must come before using MVC
    app.UseAuthentication();
    //...
}
```

### Api controller
```csharp
//specify "Bearer" authentication scheme if it's not set as default
[Authorize(AuthenticationSchemes = OpenIddictValidationDefaults.AuthenticationScheme)]
//or if using introspection on resource server: 
// [Authorize(AuthenticationSchemes = OAuthIntrospectionDefaults.AuthenticationScheme)]
public class MyController : Controller
```


# Reference token format

## Reference token generation

### Authorization server

```c#
// Startup.cs

public void ConfigureServices(IServiceCollection services)
{
    // Register OpenIddict stores
    services.AddDbContext<ApplicationDbContext>(options =>
    {
        options.UseSqlServer(Configuration.GetConnectionString("DefaultConnection"));
        options.UseOpenIddict();
    });

    // Register the OpenIddict services.
    services.AddOpenIddict()
    .AddCore(options =>
    {
        options.UseEntityFrameworkCore()
            .UseDbContext<ApplicationDbContext>();
    })

    .AddServer(options => 
    {
        //...
        options.UseReferenceTokens();
    });

public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
{
    //...
    //must come before using MVC
    app.UseAuthentication();
    //...
}
```

## Reference token validation

### Authorization server
```c#
// Startup.cs

public void ConfigureServices(IServiceCollection services)
{
    services.AddOpenIddict()
        .AddCore(...) //see above

        .AddServer(...) // see above

        .AddValidation(options => 
        {
            options.UseReferenceTokens();
        });

public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
{
    //...
    //must come before using MVC
    app.UseAuthentication();
    //...
}
```

### Resource server
```c#
// Startup.cs

public void ConfigureServices(IServiceCollection services)
{
    // Register OpenIddict stores
    services.AddDbContext<AuthServerDbContext>(options =>
    {
        options.UseSqlServer(Configuration.GetConnectionString("AuthServerConnection"));
        options.UseOpenIddict();
    });


    services.AddOpenIddict()
    .AddCore(options =>
    {
        // Register the Entity Framework entities and stores.
        options.UseEntityFrameworkCore()
               .UseDbContext<AuthServerDbContext>();
    })

    //This adds a "Bearer" authentication scheme
    .AddValidation(options => 
    {
        options.UseReferenceTokens();
    });

    //Optionally set Bearer token authentication as default
    //services.AddAuthentication(options =>
    //{
    //    options.DefaultAuthenticateScheme = OpenIddictValidationDefaults.AuthenticationScheme;
    //    options.DefaultChallengeScheme = OpenIddictValidationDefaults.AuthenticationScheme;
    //});

public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
{
    //...
    //must come before using MVC
    app.UseAuthentication();
    //...
}
```

### Api controller
```c#
// Note: both OpenIddictValidationDefaults.AuthenticationScheme and JwtBearerDefaults.AuthenticationScheme are "Bearer"
//If you did not set the default authentication scheme then specify it here.
//If you get a 302 redirect to login page instead of a 401 Unauthorized then Cookie authentication is handling the request
//so scheme must be specified
[Authorize(AuthenticationSchemes = OpenIddictValidationDefaults.AuthenticationScheme)]
public class MyController : Controller
```

# JWTs

## JWT generation

### Authorization server
```c#
// Startup.cs

public void ConfigureServices(IServiceCollection services)
{
    services.AddOpenIddict()
        .AddCore(...)
        
        .AddServer(options =>
        {
            //...
            options.UseJsonWebTokens();
            //JWTs must be signed by a self-signing certificate or a symmetric key
            //Here a certificate is used. I used IIS to create a self-signed certificate
            //and saved it in /FolderName folder. See below for .csproj configuration
            options.AddSigningCertificate(
                    assembly: typeof(Startup).GetTypeInfo().Assembly,
                    resource: "AppName.FolderName.certname.pfx",
                    password: "anypassword");

        });
}

public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
{
    //...
    //must come before using MVC
    app.UseAuthentication();
    //...
}

// .csproj
// if using a certificate and its stored in your app's Resources then make sure it's published
//<ItemGroup>
//    <EmbeddedResource Include="FolderName\certname.pfx" />
//  </ItemGroup>
```

## JWT validation

### Authorization server
> [!WARNING]
> Remember, this is only needed if you have API endpoints that require token authorization. If your authorization server generates tokens that are only used by separate resource servers, then this is not needed.

```c#
// Startup.cs

public void ConfigureServices(IServiceCollection services)
{
    //this must come after registering ASP.NET Core Identity
    JwtSecurityTokenHandler.DefaultInboundClaimTypeMap.Clear();
    JwtSecurityTokenHandler.DefaultOutboundClaimTypeMap.Clear();
    services.AddAuthentication()
        .AddJwtBearer(options =>
        {
            //Authority must be a url. It does not have a default value.
            options.Authority = "this server's url, e.g. http://localhost:5051/ or https://auth.example.com/";
            options.Audience = "example: auth_server_api"; //This must be included in ticket creation
            options.RequireHttpsMetadata = false;
            options.IncludeErrorDetails = true; //
            options.TokenValidationParameters = new TokenValidationParameters()
            {
                NameClaimType = OpenIdConnectConstants.Claims.Subject,
                RoleClaimType = OpenIdConnectConstants.Claims.Role,
            };
        });
}

public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
{
    //...
    //must come before using MVC
    app.UseAuthentication();
    //...
}
```

### Resource server
```c#
// Startup.cs

public void ConfigureServices(IServiceCollection services)
{
    JwtSecurityTokenHandler.DefaultInboundClaimTypeMap.Clear();
    JwtSecurityTokenHandler.DefaultOutboundClaimTypeMap.Clear();
    //Add authentication and set default authentication scheme
    services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme) //same as "Bearer"
        .AddJwtBearer(options =>
        {
            //Authority must be a url. It does not have a default value.
            options.Authority = "auth server's url, e.g. http://localhost:5051/ or https://auth.example.com/";
            options.Audience = "example: api_server_1"; //This must be included in ticket creation
            options.RequireHttpsMetadata = false;
            options.IncludeErrorDetails = true; //
            options.TokenValidationParameters = new TokenValidationParameters()
            {
                NameClaimType = OpenIdConnectConstants.Claims.Subject,
                RoleClaimType = OpenIdConnectConstants.Claims.Role,
            };
        });
}
```

### Api controller
```c#
// Note: both OpenIddictValidationDefaults.AuthenticationScheme and JwtBearerDefaults.AuthenticationScheme are "Bearer"
//If you didn't set the default authentication scheme then specify it here.
//If you get a 302 redirect to login page instead of a 401 Unauthorized then Cookie authentication is handling the request
//so scheme must be specified
[Authorize(AuthenticationSchemes = JwtBearerDefaults.AuthenticationScheme)]
public class MyController : Controller
```
