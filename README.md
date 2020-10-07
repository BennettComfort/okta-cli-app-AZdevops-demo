# okta-cli-app-AZdevops-demo
A dotnet mvc web app with Okta sign in integration. For demo purposes only

*****

### Requirements
- [Azure Account](https://portal.azure.com/)
- [Okta Development account](https://developer.okta.com/)
- [ASP.NET CORE SDK](https://dotnet.microsoft.com/download)
- [AZ CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)

### Getting Started

Create Basic ASP.NET Core MVC App using DotNet CLI

`dotnet new mvc -n okta-cli-app`

Run Application

`dotnet run`

Open browser and navigate to `http://localhost:5000`

Add **Okta.AspNetCore** package to project

`dotnet add package Okta.AspNetCore`

Your okta-cli-app.csproj should have the following
``` csproj
<ItemGroup>
  <PackageReference Include="Okta.AspNetCore" Version="3.3.0" />
</ItemGroup>
```

My live site URL made with azure app service [https://okta-cli-app.azurewebsites.net](https://okta-cli-app.azurewebsites.net)

Modify `Startup.cs`

``` c#
using Microsoft.AspNetCore.Authentication.Cookies;
using Microsoft.AspNetCore.Authentication.OpenIdConnect;
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Okta.AspNetCore;
using System.Collections.Generic;

namespace okta_cli_app
{
    public class Startup
    {
        public Startup(IConfiguration configuration)
        {
            Configuration = configuration;
        }

        public IConfiguration Configuration { get; }

        // This method gets called by the runtime. Use this method to add services to the container.
        public void ConfigureServices(IServiceCollection services)
        {
            services.AddAuthentication(options =>
            {
                options.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
                options.DefaultChallengeScheme = OpenIdConnectDefaults.AuthenticationScheme;
            })
           .AddCookie()
           .AddOktaMvc(new OktaMvcOptions
           {
                // Replace these values with your Okta configuration
                OktaDomain = Configuration.GetValue<string>("Okta:OktaDomain"),
               ClientId = Configuration.GetValue<string>("Okta:ClientId"),
               ClientSecret = Configuration.GetValue<string>("Okta:ClientSecret"),
               Scope = new List<string> { "openid", "profile", "email" },
           });

            services.AddControllersWithViews();
        }

        // This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
        public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
        {
            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
            }
            else
            {
                app.UseExceptionHandler("/Home/Error");
                app.UseHsts();
            }
            app.UseHttpsRedirection();
            app.UseStaticFiles();

            app.UseRouting();

            app.UseAuthentication();

            app.UseAuthorization();

            app.UseEndpoints(endpoints =>
            {
                endpoints.MapControllerRoute(
                    name: "default",
                    pattern: "{controller=Home}/{action=Index}/{id?}");
            });
        }
    }
}
```

Add `AccountControllers.cs` to the project, under _controllers_ folder

``` c#
using Microsoft.AspNetCore.Authentication;
using Microsoft.AspNetCore.Authentication.Cookies;
using Microsoft.AspNetCore.Mvc;
using Okta.AspNetCore;

namespace okta_cli_app.Controllers
{
    public class AccountController : Controller
    {
        public IActionResult SignIn()
        {
            if (!HttpContext.User.Identity.IsAuthenticated)
            {
                return Challenge(OktaDefaults.MvcAuthenticationScheme);
            }

            return RedirectToAction("Index", "Home");
        }

        [HttpPost]
        public IActionResult SignOut()
        {
            return new SignOutResult(
                new[]
                {
                     OktaDefaults.MvcAuthenticationScheme,
                     CookieAuthenticationDefaults.AuthenticationScheme,
                },
                new AuthenticationProperties { RedirectUri = "/Home/" });
        }
    }
}
```

Add two links to the top bar of UI - In `_Layout.cshtml`

``` cshtml
<div class="navbar-collapse collapse d-sm-inline-flex flex-sm-row-reverse">
    @if (User.Identity.IsAuthenticated)
    {
        <ul class="nav navbar-nav navbar-right">
            <li><p class="navbar-text">Hello, @User.Identity.Name</p></li>
            <li><a class="nav-link" asp-controller="Home" asp-action="Profile" id="profile-button">Profile</a></li>
            <li>
                <form class="form-inline" asp-controller="Account" asp-action="SignOut" method="post">
                    <button type="submit" class="nav-link btn btn-link text-dark" id="logout-button">Sign Out</button>
                </form>
            </li>
        </ul>
    }
    else
    {
        <ul class="nav navbar-nav navbar-right">
            <li><a asp-controller="Account" asp-action="SignIn" id="login-button">Sign In</a></li>
        </ul>
    }
    <ul class="navbar-nav flex-grow-1">  
```

Using your Okta Developer accoutn create a new web application
- Client Credentials 
- Authorization Code

``` shell
Login redirect URIs: https://localhost:5001/authorization-code/callback
Logout redirect URIs: https://localhost:5001/signout/callback
Initiate Login URI: https://localhost:5001/authorization-code/callback
```

Add the following items from your application to the `appsettings.json` file
- OktaDomain
- ClientId
- ClientSecret

``` json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
    }
  },
  "AllowedHosts": "*",
  "Okta": {
    "OktaDomain": "https://dev-509249.okta.com",
    "ClientId": "0oaq5vf8fzPfAljzy4x6",
    "ClientSecret": "HoGyxgDTrMfNqDp4RO234CEQxDAwuXnZm2FycON-"
  }  
}
```

``` dotnet
dotnet run
```



### AZ CLI Deployment Process
 ``` shell
 az login
 ```
 
create a resource group
``` shell
az group create -l eastus -n okta-cli-rg 
```

Create an appservice plan
``` shell
az appservice plan create -n okta-cli-plan --resource-group okta-cli-rg --sku FREE
```

Create Application Service Instance
``` shell
az webapp create -g okta-cli-rg -p -okta-cli-plan -n -okta-cli-app --runtime "DOTNETCORE|3.1" --deployment-local-git
```

Create webapp deployment user
``` shell
az webapp deployment user set --user-name <username> --password <password>
```

*Make sure to change your okta application URLs to the new azurewebsites URLs*
``` shell
> http://localhost:5000 ----> https://okta-cli-app.azurewebsites.net/
```

Initialize your local okta-cli-app directory into a git repository
``` shell
git init
git add .
git commit -m "init"
```

Add your azure remote and push
``` hell
git remote add azure <deploymentLocalGitUrl-from-create-step>
```

``` shell
git push azure main
```

Browser your webapp 
``` shell
az webapp browse -n okta-cli-app
```


