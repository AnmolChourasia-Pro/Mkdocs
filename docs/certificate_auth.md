# Certificate-Based Authentication

To improve security, use a certificate instead of a client secret for authentication in Azure AD.

---

## Prerequisites

1. **Registered App in Azure AD** (see [Azure App Registration](azure_app_registration.md)).
2. **Certificate**: Exported `.pfx` file with a secure password.
3. **Dot Net framework**: `version 4.5` 

---

## Code Implementation

### **Set Up Configuration**
Add these settings to your `app.config` or `web.config` file:

``` xml
<appSettings>
    <add key="SpHostUrl" value="https://yoursharepointsite.sharepoint.com" />
    <add key="clientid" value="your-client-id" />
    <add key="certificate" value="path-to-your-certificate.pfx" />
    <add key="certificatePassword" value="your-certificate-password" />
    <add key="AzureTenantID" value="your-tenant-id" />
</appSettings>
```
---

The code provided is a comprehensive implementation that integrates **Azure AD authentication**, **SharePoint access using CSOM (Client-Side Object Model)**, **PKCE (Proof Key for Code Exchange)**, **token caching**, and **Hangfire for background job management** in an **ASP.NET application**.

---

### 1. **GetClientContextWithAccessToken**
This method establishes a SharePoint ClientContext, which is used to interact with SharePoint Online.

#### Key Features:

- **App-Only Context:** (Non-delegated)
     - Fetches configuration details (e.g., site URL, client ID, certificate path/password, and tenant ID).
     - Loads the X.509 certificate to authenticate with Azure AD.
     - Uses `GetAzureADAppOnlyAuthenticatedContext` to get `ClientContext` for App-Only operations.
   
- **Delegated Authentication:**
     - Reads the `AccessToken` from the session or fetches a saved token if the session token is missing.
     - If the token is invalid, it calls the `RefreshToken` method to generate a new token.
     - Uses `GetAzureADAccessTokenAuthenticatedContext` to authenticate with the token and interact with SharePoint.
``` C#
public static ClientContext GetClientContextWithAccessTokenForDelegates(bool ForDelegates = false)
{
    ClientContext clientContext = null;
    try
    {
        
        string siteUrl = ConfigurationManager.AppSettings["SpHostUrl"];
        string clientId = ConfigurationManager.AppSettings["clientid"];
        string certificatePath = ConfigurationManager.AppSettings["certificate"]; // Update with the path to your certificate
        string certificatePassword = ConfigurationManager.AppSettings["certificatePassword"];  // Update with the password for your certificate
        string tenantId = ConfigurationManager.AppSettings["AzureTenantID"];
        string resource = ConfigurationManager.AppSettings["HomeUrl"];
        if (!ForDelegates)
        {


            X509Certificate2 cert = new X509Certificate2(certificatePath, certificatePassword, X509KeyStorageFlags.MachineKeySet);

            var authenticationManager = new OfficeDevPnP.Core.AuthenticationManager();

            clientContext = authenticationManager.GetAzureADAppOnlyAuthenticatedContext(siteUrl, clientId, tenantId, cert);
            // Now you can perform CSOM operations with this context
            Web web = clientContext.Web;
            clientContext.Load(web);
            clientContext.ExecuteQuery();

            Console.WriteLine("Title: " + web.Title);
        }
        else
        {
            var authenticationManager = new OfficeDevPnP.Core.AuthenticationManager();
            try
            {
                string accessToken = Convert.ToString(HttpContext.Current.Session["AccessToken"]);
                var tokenExpiresOn = !string.IsNullOrWhiteSpace(Convert.ToString(HttpContext.Current.Session["tokenExpiresOn"])) 
                ? Convert.ToDateTime(Convert.ToString(HttpContext.Current.Session["tokenExpiresOn"])) : DateTime.MinValue;
                if (string.IsNullOrWhiteSpace(accessToken))
                {
                    var existingDetails = GetSaveAccessToken();
                    accessToken = existingDetails.AccessToken;
                    tokenExpiresOn = existingDetails.TokenExpiresOn;
                    
                }
                

                using (clientContext = authenticationManager.GetAzureADAccessTokenAuthenticatedContext(siteUrl, accessToken))
                {
                    Web web = clientContext.Web;
                    clientContext.Load(web);
                    clientContext.ExecuteQuery();
                    clientContext.Load(web.CurrentUser);
                    clientContext.ExecuteQuery();
                    Console.WriteLine("Title: " + web.Title);
                }
            }
            catch (Exception ex)
            {

                var refreshtoken = RefreshToken();
                
                if (!string.IsNullOrEmpty(refreshtoken))
                {
                    using (clientContext = authenticationManager.GetAzureADAccessTokenAuthenticatedContext(siteUrl, refreshtoken))
                    {
                        Web web = clientContext.Web;
                        clientContext.Load(web);
                        clientContext.ExecuteQuery();

                        clientContext.Load(web.CurrentUser);
                        clientContext.ExecuteQuery();

                        Console.WriteLine("Title: " + web.Title);
                    }
                }
                else
                {
                    throw new Exception("Failed to refresh token.");
                }

            }


        }
        UpdateSession(clientContext.Web.CurrentChangeToken.StringValue);
    }
    catch (Exception ex)
    {
        LogHelper.LogError("Error in GetClientContextWithAccessTokenForDelegates", ex);
        clientContext = null;
    }

    return clientContext;
}

       
```


---

### 2. **CodeCallBack**
Handles the **authorization code** flow when Azure AD redirects the user back after login. It exchanges the code for an **access token** and an **ID token** using Azure AD's PKCE flow.

#### Key Features:
- Uses the `ConfidentialClientApplicationBuilder` from MSAL to configure a client for Azure AD authentication.
- Acquires an access token using the `AuthorizationCode` grant.
- Adds PKCE support (`WithPkceCodeVerifier`) for enhanced security.
- Returns the tokens (access token, ID token) along with expiry details.

``` C#
 public static Dictionary<string, object> CodeCallBack(AuthorizationCodeReceivedNotification context)
{
    Dictionary<string, object> valuePairs = new Dictionary<string, object>();
    string siteUrl = ConfigurationManager.AppSettings["SpHostUrl"];
    string clientId = ConfigurationManager.AppSettings["clientid"];
    string certificatePath = ConfigurationManager.AppSettings["certificate"]; // Update with the path to your certificate
    string certificatePassword = ConfigurationManager.AppSettings["certificatePassword"];  // Update with the password for your certificate
    string tenantId = ConfigurationManager.AppSettings["AzureTenantID"];
    string redirectUri = ConfigurationManager.AppSettings["redirectUri"];
    string authority = String.Format(System.Globalization.CultureInfo.InvariantCulture, ConfigurationManager.AppSettings["Authority"], tenantId);
    X509Certificate2 cert = new X509Certificate2(certificatePath, certificatePassword, X509KeyStorageFlags.MachineKeySet);

    try
    {
        if (_confidentialClient == null)
        {
            _confidentialClient = ConfidentialClientApplicationBuilder.Create(clientId)
                .WithCertificate(cert)
                .WithRedirectUri(redirectUri)
                .WithAuthority(authority)
                .Build();
        }
        EnableTokenCache(_confidentialClient.UserTokenCache);

        var codeVerifier = HttpContext.Current.Request.Cookies["CodeVerifier"]?.Value;
        var scop = new[] { "https://prosaressolutions.sharepoint.com/.default", "offline_access" };

        var result = _confidentialClient.AcquireTokenByAuthorizationCode(
            scop, // Add your required scopes here
            context.Code).WithPkceCodeVerifier(codeVerifier)
            .ExecuteAsync().GetAwaiter().GetResult();


        valuePairs.Add("accessToken", result.AccessToken ?? "");

        valuePairs.Add("idToken", result.IdToken ?? "");
        valuePairs.Add("tokenExpiresOn", result.ExpiresOn);
    }
    catch (Exception ex)
    {
        LogHelper.LogError("Error -- CodeCallBack start");
        LogHelper.LogError(ex);
        LogHelper.LogError("Error -- CodeCallBack end");
    }
    return valuePairs;
}
```
---

### 3. **RefreshToken**
Acquires a new access token silently using a cached refresh token.

#### Key Features:
- Reuses the existing MSAL client to perform silent token acquisition.
- Utilizes scopes like `https://{tenant}.sharepoint.com/.default` to request access tokens specific to SharePoint.
- Updates session and user details (`UpdateHttpSession` and `UpdateUserDetailList`) with refreshed token data.
``` C#
public static string RefreshToken()
{
    string refreshToken = string.Empty;
    string siteUrl = ConfigurationManager.AppSettings["SpHostUrl"];
    string clientId = ConfigurationManager.AppSettings["clientid"];
    string certificatePath = ConfigurationManager.AppSettings["certificate"]; // Update with the path to your certificate
    string certificatePassword = ConfigurationManager.AppSettings["certificatePassword"];  // Update with the password for your certificate
    string tenantId = ConfigurationManager.AppSettings["AzureTenantID"];
    string redirectUri = ConfigurationManager.AppSettings["redirectUri"];
    string authority = String.Format(System.Globalization.CultureInfo.InvariantCulture, ConfigurationManager.AppSettings["Authority"], tenantId);
    X509Certificate2 cert = new X509Certificate2(certificatePath, certificatePassword, X509KeyStorageFlags.MachineKeySet);
    try
    {
        LogHelper.LogSuccess("RefreshToken - Started");
        if (_confidentialClient == null)
        {
            _confidentialClient = ConfidentialClientApplicationBuilder.Create(clientId)
                .WithCertificate(cert)
                .WithRedirectUri(redirectUri)
                .WithAuthority(authority)
                .Build();
        }

        EnableTokenCache(_confidentialClient.UserTokenCache);

        var accounts = _confidentialClient.GetAccountsAsync().GetAwaiter().GetResult();
        var workContext = DependencyResolver.Current.GetService<IWorkContext>();
        var curretEmail = workContext.CurrentUser.EmailId;
        var CurrAccount = accounts.Where(c => c.Username.Equals(curretEmail, StringComparison.OrdinalIgnoreCase)).FirstOrDefault();
        var scop = new[] { "https://prosaressolutions.sharepoint.com/.default", "offline_access" };
        var refreshTokenResult = _confidentialClient.AcquireTokenSilent(scop, CurrAccount)
            .ExecuteAsync()
            .GetAwaiter()
            .GetResult();
        UpdateHttpSession(refreshTokenResult);
        UpdateUserDetailList(refreshTokenResult);
        refreshToken = refreshTokenResult.AccessToken;
    }
    catch(Exception ex )
    {
        LogHelper.LogError(ex);
    }
    return refreshToken;
}
```
---

### 4. **EnableTokenCache**
- Manages token caching:
     Reads and writes the token cache to/from a local JSON file (`msal_cache.json`).
     Ensures token persistence across application restarts.

``` C# 
private static void EnableTokenCache(ITokenCache tokenCache)
{
    string cacheFilePath = Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData), "msal_cache.json");

    tokenCache.SetBeforeAccess(args =>
    {
        if (System.IO.File.Exists(cacheFilePath))
        {
            byte[] serializedCache = System.IO.File.ReadAllBytes(cacheFilePath);
            args.TokenCache.DeserializeMsalV3(serializedCache);
        }
    });


    tokenCache.SetAfterAccess(args =>
    {
        if (args.HasStateChanged)
        {
            byte[] serializedCache = args.TokenCache.SerializeMsalV3();
            System.IO.File.WriteAllBytes(cacheFilePath, serializedCache);
        }
    });
}
```
---

### 5. **OWIN Startup Configuration**
Sets up the OWIN middleware to handle authentication workflows.

#### Key Configurations:
- **Cookie Authentication**:
    - Enables cookie-based authentication with secure options (`CookieHttpOnly` and `CookieSecureOption.Always`).

- **OpenID Connect Authentication**:
    - Configures the app to use Azure AD as the identity provider.
    - Supports features like PKCE, error handling, and token exchange.

``` C#
using Microsoft.Owin;
using Owin;
using System;
using System.Threading.Tasks;
using Hangfire;
using Prosares.CLMSWeb.Helper;
using System.Web;
using Microsoft.Owin.Security.Cookies;
using Microsoft.Owin.Security;
using Microsoft.Owin.Security.OpenIdConnect;
using Microsoft.IdentityModel.Protocols.OpenIdConnect;
using Microsoft.Owin.Security.Notifications;
using Microsoft.IdentityModel.Logging;
using Prosares.CLMS.Services;
using System.Security.Cryptography;
using System.Text;
using Autofac;


[assembly: OwinStartup(typeof(Prosares.CLMSWeb.Startup))]

namespace Prosares.CLMSWeb
{
    public class Startup
    {
        // The Client ID is used by the application to uniquely identify itself to Azure AD.
        string clientId = System.Configuration.ConfigurationManager.AppSettings["clientid"];

        // RedirectUri is the URL where the user will be redirected to after they sign in.
        string redirectUri = System.Configuration.ConfigurationManager.AppSettings["redirectUri"];

        // Tenant is the tenant ID (e.g. contoso.onmicrosoft.com, or 'common' for multi-tenant)
        static string tenant = System.Configuration.ConfigurationManager.AppSettings["TenantId"];

        // Authority is the URL for authority, composed by Microsoft identity platform endpoint and the tenant name (e.g. https://login.microsoftonline.com/contoso.onmicrosoft.com/v2.0)
        string authority = String.Format(System.Globalization.CultureInfo.InvariantCulture, System.Configuration.ConfigurationManager.AppSettings["Authority"], tenant);

        public void Configuration(IAppBuilder app)
        {
            try
            {

                app.SetDefaultSignInAsAuthenticationType(CookieAuthenticationDefaults.AuthenticationType);

                app.UseCookieAuthentication(new CookieAuthenticationOptions
                {
                    CookieSameSite = Microsoft.Owin.SameSiteMode.None,
                    CookieHttpOnly = true,
                    CookieSecure = CookieSecureOption.Always, // CookieSecureOption.Always
                    AuthenticationType = "Cookies",
                    CookieManager = new Microsoft.Owin.Host.SystemWeb.SystemWebChunkingCookieManager()
                    //CookieManager = new SameSiteCookieManager(new SystemWebCookieManager())
                });

                app.UseOpenIdConnectAuthentication(
                new OpenIdConnectAuthenticationOptions
                {
                    // Sets the ClientId, authority, RedirectUri as obtained from web.config
                    ClientId = clientId,
                    Authority = authority,
                    RedirectUri = redirectUri,
                    // PostLogoutRedirectUri is the page that users will be redirected to after sign-out. In this case, it is using the home page
                    PostLogoutRedirectUri = redirectUri,
                    Scope = OpenIdConnectScope.OpenIdProfile + " offline_access",
                    // ResponseType is set to request the code id_token - which contains basic information about the signed-in user
                    ResponseType = OpenIdConnectResponseType.CodeIdToken,
                    // OpenIdConnectAuthenticationNotifications configures OWIN to send notification of failed authentications to OnAuthenticationFailed method
                    Notifications = new OpenIdConnectAuthenticationNotifications
                    {
                        AuthenticationFailed = OnAuthenticationFailed,
                        AuthorizationCodeReceived = OnAuthorizationCodeReceived,
                        RedirectToIdentityProvider = RedirectToIdentityProvider
                    }
                }
            );
            }
            catch (Exception ex)
            {
                
            }
        }

        /// <summary>
        /// Handle failed authentication requests by redirecting the user to the home page with an error in the query string
        /// </summary>
        /// <param name="context"></param>
        /// <returns></returns>
        private Task OnAuthenticationFailed(AuthenticationFailedNotification<OpenIdConnectMessage, OpenIdConnectAuthenticationOptions> context)
        {
            context.HandleResponse();
            context.Response.Redirect("/?errormessage=" + context.Exception.Message);
            return Task.FromResult(0);
        }
        private Task OnAuthorizationCodeReceived(AuthorizationCodeReceivedNotification context)
        {
            try
            {
                // Exchange the authorization code for an access token
                var dicValue = CommonServiceHelper.CodeCallBack(context);

                // Store the access token and ID token securely
                HttpContext.Current.Session["AccessToken"] = Convert.ToString(dicValue["accessToken"]);
                HttpContext.Current.Session["IdToken"] = Convert.ToString(dicValue["idToken"]);
                HttpContext.Current.Session.Add("tokenExpiresOn", dicValue["tokenExpiresOn"]);
                // Optionally store the refresh token if needed
                //context.Response.Redirect("/Home/AppInitiator");
                //HttpContext.Current.Session["RefreshToken"] = result.RefreshToken;
            }
            catch (Exception ex)
            {
                // Handle exceptions (e.g., logging)
                LogHelper.LogInformation($"OnAuthorizationCodeReceived -{ex.Message}");
            }
            return Task.FromResult(0);
        }
        private Task RedirectToIdentityProvider(RedirectToIdentityProviderNotification<OpenIdConnectMessage, OpenIdConnectAuthenticationOptions> context)
        {
            // Generate the code verifier and challenge
            var codeVerifier = GenerateCodeVerifier();
            var codeChallenge = GenerateCodeChallenge(codeVerifier);
            HttpContext.Current.Response.Cookies.Add(new HttpCookie("CodeVerifier", codeVerifier));

            // Store the code_verifier in session for later use
            //HttpContext.Current.Session["CodeVerifier"] = codeVerifier;

            // Add PKCE parameters to the authorization request
            context.ProtocolMessage.SetParameter("code_challenge", codeChallenge);
            context.ProtocolMessage.SetParameter("code_challenge_method", "S256");
            return Task.FromResult(0);
        }
        private string GenerateCodeVerifier()
        {
            using (var rng = new RNGCryptoServiceProvider())
            {
                var bytes = new byte[32];
                rng.GetBytes(bytes);
                return Base64UrlEncode(bytes);
            }
        }

        private string GenerateCodeChallenge(string codeVerifier)
        {
            using (var sha256 = SHA256.Create())
            {
                var bytes = Encoding.ASCII.GetBytes(codeVerifier);
                var hash = sha256.ComputeHash(bytes);
                return Base64UrlEncode(hash);
            }
        }

        private string Base64UrlEncode(byte[] input)
        {
            return Convert.ToBase64String(input)
                .TrimEnd('=')
                .Replace('+', '-')
                .Replace('/', '_');
        }
    }
}

```
---

### 6. **RedirectToIdentityProvider**
Implements PKCE for OpenID Connect flows:
- Generates and stores a `code_verifier` in a cookie.
- Sends a `code_challenge` (SHA-256 hash of the verifier) to Azure AD for secure token exchange.

``` c#
private Task RedirectToIdentityProvider(RedirectToIdentityProviderNotification<OpenIdConnectMessage, OpenIdConnectAuthenticationOptions> context)
{
    // Generate the code verifier and challenge
    var codeVerifier = GenerateCodeVerifier();
    var codeChallenge = GenerateCodeChallenge(codeVerifier);
    HttpContext.Current.Response.Cookies.Add(new HttpCookie("CodeVerifier", codeVerifier));

    // Store the code_verifier in session for later use
    //HttpContext.Current.Session["CodeVerifier"] = codeVerifier;

    // Add PKCE parameters to the authorization request
    context.ProtocolMessage.SetParameter("code_challenge", codeChallenge);
    context.ProtocolMessage.SetParameter("code_challenge_method", "S256");
    return Task.FromResult(0);
}
```
---

### 7. **OnAuthorizationCodeReceived**
The `OnAuthorizationCodeReceived` method is an **event handler** invoked during an OpenID Connect authentication flow when the application receives an **authorization code** from the identity provider (Azure AD, in this case). This code is exchanged for an **access token**, which is used to access protected resources (e.g., SharePoint).

- **What it Contains:** 
    - Information about the authorization code received during the OpenID Connect flow.
    - The `context.Code` property contains the authorization code.
``` C#
private Task OnAuthorizationCodeReceived(AuthorizationCodeReceivedNotification context)
{
    try
    {
        // Exchange the authorization code for an access token
        var dicValue = CommonServiceHelper.CodeCallBack(context);

        // Store the access token and ID token securely
        HttpContext.Current.Session["AccessToken"] = Convert.ToString(dicValue["accessToken"]);
        HttpContext.Current.Session["IdToken"] = Convert.ToString(dicValue["idToken"]);
        HttpContext.Current.Session.Add("tokenExpiresOn", dicValue["tokenExpiresOn"]);
    }
    catch (Exception ex)
    {
        // Handle exceptions (e.g., logging)
        LogHelper.LogInformation($"OnAuthorizationCodeReceived -{ex.Message}");
    }
    return Task.FromResult(0);
}
```
### Key Libraries Used:
1. **MSAL (Microsoft.Identity.Client)**:
     - Handles token acquisition and caching.
2. **OfficeDevPnP.Core.AuthenticationManager**:
     - Facilitates SharePoint Online authentication.
3. **OWIN Middleware**:
     - Implements authentication workflows using OpenID Connect.
4. **Autofac**:
     - Manages dependency injection.
---

### Flow Overview:
1. **Non-Delegated App Access**:
     - Authenticate using certificate and app-only permissions.
     - Interact with SharePoint without user involvement.

2. **Delegated Access**:
     - Authenticate on behalf of a user using tokens.
     - Handles token refresh when expired.
 
3. **Authentication Workflow**:
     - PKCE is used for secure authentication during OpenID Connect flows.
     - Tokens are securely cached and refreshed for seamless user experience.

This implementation ensures robust Azure AD integration for both app-only and delegated scenarios, focusing on security, scalability, and maintainability.

---

