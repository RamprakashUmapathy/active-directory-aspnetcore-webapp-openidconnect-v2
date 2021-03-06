# Signing out

This page explains how sign-out works

## What sign out involves

Signing out from a Web app is about more than removing the information about the signed-in account from the Web App's state.
The Web app must also redirect the user to the Microsoft identity platform v2.0 `logout` endpoint to sign out. When your web app redirects the user to the `logout` endpoint, this endpoint clears the user's session from the browser. If your app did not go to the `logout`, the user would reauthenticate to your app without entering their credentials again, because they would have a valid single sign-in session with the Microsoft Identity platform v2.0 endpoint.

To learn more, see the [Send a sign-out request](https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-protocols-oidc#send-a-sign-out-request) paragraph in the [Microsoft Identity platform v2.0 and the OpenID Connect protocol](https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-protocols-oidc) conceptual documentation

## Application registration

During the application registration, you will have registered a post logout URI. In our tutorial, you registered `https://localhost:44321/signout-oidc` in the **Logout URL** field of the **Advanced Settings** section in the **Authentication** page. For details see for instance, [
Register the webApp app](https://github.com/Azure-Samples/active-directory-aspnetcore-webapp-openidconnect-v2/tree/master/1-WebApp-OIDC/1-1-MyOrg#register-the-webapp-app-webapp)

## About code

### Signout button

The sign out button is exposed in `Views\Shared\_LoginPartial.cshtml` and only displayed when there's an authenticated account (that is when the user has previously signed in)

```html
@using Microsoft.Identity.Web
@if (User.Identity.IsAuthenticated)
{
    <ul class="nav navbar-nav navbar-right">
        <li class="navbar-text">Hello @User.GetDisplayName()!</li>
        <li><a asp-area="AzureAD" asp-controller="Account" asp-action="SignOut">Sign out</a></li>
    </ul>
}
else
{
    <ul class="nav navbar-nav navbar-right">
        <li><a asp-area="AzureAD" asp-controller="Account" asp-action="SignIn">Sign in</a></li>
    </ul>
}
```

### `Signout()` action of the `AccountController`

Pressing the **Sign out** button on the web app, triggers the `SignOut` action on the `Account` controller. In previous versions of the ASP.NET core templates, this controller
was embedded with the Web App, but this is no longer the case as it's now part of the ASP.NET Core framework itself. The code for the `AccountController` is available from the ASP.NET core repository at
from <https://github.com/aspnet/AspNetCore/blob/master/src/Azure/AzureAD/Authentication.AzureAD.UI/src/Areas/AzureAD/Controllers/AccountController.cs>, and what it does is:

- set an openid redirect URI to `/Account/SignedOut` so that the controller is called back when Azure AD has performed the sign out
- call `Signout()`, which lets the OpenId connect middleware contact the Microsoft identity platform `logout` endpoint which:

  - clears the session cookie from the browser,
  - and finally calls back the **logout URL**, which, by default, displays the signed out view page [SignedOut.html](https://github.com/aspnet/AspNetCore/blob/master/src/Azure/AzureAD/Authentication.AzureAD.UI/src/Areas/AzureAD/Pages/Account/SignedOut.cshtml) also provided part of ASP.NET Core.

### Intercepting the call to the logout endpoint

The ASP.NET Core OpenIdConnect middleware enables your app to intercept the call to the Microsoft identity platform logout endpoint by providing an OpenIdConnect event named `OnRedirectToIdentityProviderForSignOut`. The web app uses it to attempt to avoid the select account dialog to be presented to the user when signing out. This interception is done in the `AddAzureAdV2Authentication` of the `Microsoft.Identity.Web` reusable library. See [StartupHelpers.cs L58-L66](https://github.com/Azure-Samples/active-directory-aspnetcore-webapp-openidconnect-v2/blob/b87a1d859ff9f9a4a98eb7b701e6a1128d802ec5/Microsoft.Identity.Web/StartupHelpers.cs#L58-L66)

```CSharp
public static IServiceCollection AddAzureAdV2Authentication(this IServiceCollection services,
                                                            IConfiguration configuration)
{
    ...
    services.Configure<OpenIdConnectOptions>(AzureADDefaults.OpenIdScheme, options =>
    {
        ...
        options.Authority = options.Authority + "/v2.0/";
        ...
        // Attempt to avoid displaying the select account dialog when signing out
        options.Events.OnRedirectToIdentityProviderForSignOut = async context =>
        {
            var user = context.HttpContext.User;
            context.ProtocolMessage.LoginHint = user.GetLoginHint();
            context.ProtocolMessage.DomainHint = user.GetDomainHint();
            await Task.FromResult(0);
        };
    }
}
```

### Intercepting the callback after logout - Single Sign Out

Your application can also intercept the after logout event, for instance to clear the entry of the token cache associated with the account that signed out. We'll see in the second part of this tutorial (about the Web app calling a Web API), that the web app will store access tokens for the user in a cache. Intercepting the after logout callback enables your web application to remove the user from the token cache. This is illustrated in the `AddMsal()` method of [StartupHelper.cs L137-143](https://github.com/Azure-Samples/active-directory-aspnetcore-webapp-openidconnect-v2/blob/b87a1d859ff9f9a4a98eb7b701e6a1128d802ec5/Microsoft.Identity.Web/StartupHelpers.cs#L137-L143)

The Logout Url that you have registered for your application enables you to implement single sign out. Indeed, the Microsoft identity platform logout endpoint will call the Logout URL registered with your application. This call happens whether or not the sign-out was initiated from your web app, or from another web app or the browser. For more information, see [Single sign-out](https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-protocols-oidc#single-sign-out) in the conceptual documentation

```CSharp
public static IServiceCollection AddMsal(this IServiceCollection services, IEnumerable<string> initialScopes)
{
    services.AddTokenAcquisition();

    services.Configure<OpenIdConnectOptions>(AzureADDefaults.OpenIdScheme, options =>
    {
     ...
        // Handling the sign-out: removing the account from MSAL.NET cache
        options.Events.OnRedirectToIdentityProviderForSignOut = async context =>
        {
            // Remove the account from MSAL.NET token cache
            var _tokenAcquisition = context.HttpContext.RequestServices.GetRequiredService<ITokenAcquisition>();
            await _tokenAcquisition.RemoveAccount(context);
        };
    });
    return services;
}
```