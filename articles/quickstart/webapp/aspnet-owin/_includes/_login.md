## Install and Configure Auth0 OAuth2 Middleware

The easiest way to enable authentication with Auth0 in your ASP.NET MVC application is to use the Auth0 ASP.NET OAuth2 middleware which is available in the `Auth0-ASPNET-Owin` NuGet package, so install that first:

```bash
Install-Package Auth0-ASPNET-Owin
```

There is a bug in Microsoft's OWIN implementation for System.Web, which can cause cookies to disappear on some occasions. To work around this issue, you will also need to install the `Kentor.OwinCookieSaver` NuGet package:

```bash
Install-Package Kentor.OwinCookieSaver
```

Now go to the `Configuration` method of your `Startup` class and configure the cookie middleware as well as the Auth0 middleware. Also be sure to register the [Kentor OWIN Cookie saver middleware](https://github.com/KentorIT/owin-cookie-saver) which must be added *before* any cookie handling middleware.

```cs
// Startup.cs

public void Configuration(IAppBuilder app)
{
    // Configure Auth0 parameters
    string auth0Domain = ConfigurationManager.AppSettings["auth0:Domain"];
    string auth0ClientId = ConfigurationManager.AppSettings["auth0:ClientId"];
    string auth0ClientSecret = ConfigurationManager.AppSettings["auth0:ClientSecret"];

    // Enable the Cookie saver middleware to work around a bug in the OWIN implementation
    app.UseKentorOwinCookieSaver();

    // Set Cookies as default authentication type
    app.SetDefaultSignInAsAuthenticationType(CookieAuthenticationDefaults.AuthenticationType);
    app.UseCookieAuthentication(new CookieAuthenticationOptions
    {
        AuthenticationType = CookieAuthenticationDefaults.AuthenticationType,
        LoginPath = new PathString("/Account/Login")
    });

    // Configure Auth0 authentication
    var options = new Auth0AuthenticationOptions()
    {
        Domain = auth0Domain,
        ClientId = auth0ClientId,
        ClientSecret = auth0ClientSecret,

        // If you want to request an access_token to pass to an API, then replace the audience below to 
        // pass your API Identifier instead of the /userinfo endpoint
        Provider = new Auth0AuthenticationProvider()
        {
            OnApplyRedirect = context =>
            {
                string userInfoAudience = $"https://{auth0Domain}/userinfo";
                string redirectUri = context.RedirectUri + "&audience=" + WebUtility.UrlEncode(userInfoAudience);

                context.Response.Redirect(redirectUri);
            }
        }
    };
    app.UseAuth0Authentication(options);
}
```

It is important that you register both the cookie middleware and the Auth0 middleware, as all of them are required for the authentication to work. The Auth0 middleware will handle the authentication with Auth0. Once the user has authenticated, their identity will be stored in the cookie middleware.

::: note
We are passing the Auth0 tenant's user info endpoint as the `audience` parameter to the `/authorize` endpoint. This is to ensure that all authentication reponses are [OIDC Conformant](/api-auth/intro).
:::

## Add Login and Logout Methods

Next, you will need to add `Login` and `Logout` actions to the `AccountController`.

The `Login` action will return a `ChallengeResult` which will instruct the OWIN middleware to challenge the particular piece of Authentication middleware (in the case the "Auth0" middleware) to authenticate. 

For the `Logout` action you will need to sign the user out of the cookie middleware (which will clear the local application session), as well as Auth0. For more information you can refer to the Auth0 [Logout](/logout) documentation.

```cs
// Controllers/AccountController.cs

public class AccountController : Controller
{
    public ActionResult Login(string returnUrl)
    {
        return new ChallengeResult("Auth0", returnUrl ?? Url.Action("Index", "Home"));
    }

    [Authorize]
    public void Logout()
    {
        HttpContext.GetOwinContext().Authentication.SignOut(CookieAuthenticationDefaults.AuthenticationType);
        HttpContext.GetOwinContext().Authentication.SignOut(new AuthenticationProperties
        {
            RedirectUri = Url.Action("Index", "Home")
        }, "Auth0");
    }

    [Authorize]
    public ActionResult Claims()
    {
        return View();
    }
}
```

You will also need to add the following code for the `ChallengeResult` class to your project:

```csharp
// Controllers/AccountController.cs

internal class ChallengeResult : HttpUnauthorizedResult
{
    private const string XsrfKey = "XsrfId";

    public ChallengeResult(string provider, string redirectUri)
        : this(provider, redirectUri, null)
    {
    }

    public ChallengeResult(string provider, string redirectUri, string userId)
    {
        LoginProvider = provider;
        RedirectUri = redirectUri;
        UserId = userId;
    }

    public string LoginProvider { get; set; }
    public string RedirectUri { get; set; }
    public string UserId { get; set; }

    public override void ExecuteResult(ControllerContext context)
    {
        var properties = new AuthenticationProperties { RedirectUri = RedirectUri };
        if (UserId != null)
        {
            properties.Dictionary[XsrfKey] = UserId;
        }
        context.HttpContext.GetOwinContext().Authentication.Challenge(properties, LoginProvider);
    }
}
```

## Add Login and Logout Links

Lastly, add Login and Logout links to the navigation bar. To do that, head over to `/Views/Shared/_Layout.cshtml` and add code to the navigation bar section which displays a Logout link when the user is authenticated, otherwise a Login link. These will link to the `Logout` and `Login` actions of the `AccountController` respectively:

```html
<!-- Views/Shared/_Layout.cshtml -->

<div class="navbar navbar-inverse navbar-fixed-top">
    <div class="container">
        <div class="navbar-header">
            <button type="button" class="navbar-toggle" data-toggle="collapse" data-target=".navbar-collapse">
                <span class="icon-bar"></span>
                <span class="icon-bar"></span>
                <span class="icon-bar"></span>
            </button>
            @Html.ActionLink("Application name", "Index", "Home", new { area = "" }, new { @class = "navbar-brand" })
        </div>
        <div class="navbar-collapse collapse">
            <ul class="nav navbar-nav">
                <li>@Html.ActionLink("Home", "Index", "Home")</li>
            </ul>
            <ul class="nav navbar-nav navbar-right">
                @if (User.Identity.IsAuthenticated)
                {
                    <li>@Html.ActionLink("Logout", "Logout", "Account")</li>
                }
                else
                {
                    <li>@Html.ActionLink("Login", "Login", "Account")</li>
                }
            </ul>
        </div>
    </div>
</div>
```

## Run the Application

Now when you run the application you can select the Login link to log in to the application. This will display the Login page with the Auth0 Lock component embedded in the page. The user can enter their username and password to log in, or alternatively log in with any of the social login providers you may have configured.
