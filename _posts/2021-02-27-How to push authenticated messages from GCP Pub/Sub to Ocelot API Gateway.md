
In this new entrance, I will show you how you can use GCP's Authentication scheme to secure the messages incoming from GCP's Pub/Sub to Ocelot API Gateway.

I'll assume you want to do something like the image shows, external services could be even a GKE cluster. The pattern itself has nothing wrong, but by default, there is no authentication scheme being used apart from HTTPS.




Therefore we will configure the Ocelot push subscription to make use of the Oauth2 authentication, in the Ocelot side we will dispatch the token validation to a Google library that will make all the work for us.

### What tools will be used?

Google.Apis.Auth

[https://github.com/googleapis/google-api-dotnet-client]()

Ocelot

[https://github.com/ThreeMammals/Ocelot]()

### What will be implemented?

Basically, we will create a custom authentication handler that will delegate to Google.Apis.Auth the token validation task. This is the recommended way of  implement token validation according to

[https://cloud.google.com/pubsub/docs/push?hl=en]()

Lest start!
### 1.-create a Pub/Sub push subscription

![pubsub_conf.png]({{site.baseurl}}/images\how_to_push_auth\pubsub_conf.png)

### Notes

- Only Service Accounts can be used
- Audience is optional
- This option is only available for Push subscriptions
- In the next URL, further details can be found


[https://cloud.google.com/pubsub/docs/push]()

### 2.-create the custom token handler

The custom token handler is composed of three parts

### 2.1.-Implement AuthenticationHandler<AuthenticationSchemeOptions> class

AuthenticationSchemeOptions holds convenience properties used to declare the configuration needed, even though is a good idea to make a subclass from it, I will skip this step because this is not needed for our purpose.

The method HandleAuthenticateAsync will be called in execution time on every request,  AuthenticationHandler will give us access to the context, logging as well other actions.

 ```
public class CustomAuthHandler_ : AuthenticationHandler<AuthenticationSchemeOptions>
{
    public CustomAuthHandler_(IOptionsMonitor options, ILoggerFactory logger, 
                                    UrlEncoder encoder, ISystemClock clock) : base(options, logger, encoder, clock)
    {
    }

    protected override Task HandleAuthenticateAsync()
    {
        throw new NotImplementedException();
    }
}
 ```
 
  
A possible implementation could look like this:

  ```
protected override async Task HandleAuthenticateAsync()
{
    AuthenticationTicket ticket = null;

    try
    {
        if (!AuthenticationHeaderValue.TryParse(Request.Headers[AUTHORIZATION_HEADER], out AuthenticationHeaderValue headerValue))
            return AuthenticateResult.Fail("Authorization header is not present or valid!");
                    
        ///Token validation ahead
        var validPayload = await GoogleJsonWebSignature.ValidateAsync(headerValue.Parameter);
        if (validPayload is null)
            return AuthenticateResult.Fail("Authorization header is not present or valid!");

         ////Transform the payload into a list of ClaimIdentity
         var identities = ToClaimsIdentity(validPayload);
         ticket = new AuthenticationTicket(new ClaimsPrincipal(identities), validPayload.Type);

      }
      catch (InvalidJwtException ex)
      {
          return AuthenticateResult.Fail(ex.Message);
      }

    return AuthenticateResult.Success(ticket);
}
  

/// <summary>
        /// Transform payload into a list of ClaimsIdentity 
        /// </summary>
        /// <param name="payload"></param>
        /// <returns></returns>
        private List<ClaimsIdentity> ToClaimsIdentity(GoogleJsonWebSignature.Payload payload)
        {
            var claims = new List<Claim>()
            {
                new Claim("scope", payload.Scope ?? string.Empty),
                new Claim("prn", payload.Prn ?? string.Empty),
                new Claim("hd", payload.HostedDomain ?? string.Empty),
                new Claim("email", payload.Email ?? string.Empty),
                new Claim("email_verified", payload.EmailVerified.ToString()),
                new Claim("name", payload.Name ?? string.Empty),
                new Claim("given_name", payload.GivenName ?? string.Empty),
                new Claim("family_name", payload.FamilyName ?? string.Empty),
                new Claim("iss", payload.Issuer ?? string.Empty),
                new Claim("jti", payload.JwtId ?? string.Empty),
                new Claim("typ", payload.Type ?? string.Empty),
                new Claim("exp", payload.ExpirationTimeSeconds.HasValue ? payload.ExpirationTimeSeconds.Value.ToString() : string.Empty),
                new Claim("nbf", payload.NotBeforeTimeSeconds.HasValue ? payload.NotBeforeTimeSeconds.Value.ToString() : string.Empty),
                new Claim("target_audience", payload.TargetAudience ?? string.Empty),
                new Claim("iat", payload.IssuedAtTimeSeconds.HasValue ? payload.IssuedAtTimeSeconds.Value.ToString() : string.Empty)
            };
            foreach (var audience in payload.AudienceAsList)
            {
                claims.Add(new Claim("aud", audience ?? string.Empty));
            }

            return new List<ClaimsIdentity>(1) { new ClaimsIdentity(claims, "custom") };
        }
  
  ```
  
GoogleJsonWebSignature.ValidateAsync is fast and efficient, due to the fact it caches Google's certificates, anyway if you need to, you can indicate it to always refresh the certificate. Finally, if everything goes ok with the validation then I transform the payload into a List<ClaimIdentity>, but this last step is not necessary.  

### 3.-Create an extension method to simplify the custom handler utilization
  
  ```
public static class CustomAuthBuilderExtension
{
    public static AuthenticationBuilder AddCustomAuth(this AuthenticationBuilder builder)
    {
        // Add custom authentication scheme with  custom handler
        return builder.AddScheme("custom", o => { });
    }
}
  ```
  
### 4.-Finally put all the pieces together
  
  ```
public void ConfigureServices(IServiceCollection services)
{
    services.AddOcelot();
    services.AddAuthentication(options =>
    {
        ///Force the same scheme for all requests
        options.DefaultScheme = "custom";
     })
     .AddCustomAuth();    
}
  ```
  
It makes sense to force only one type of authentication scheme if you use Ocelot as a BFF, but in more complex scenarios you may want to mix authenticated and not authenticated routes.
  
## Final thoughts 

Is important to understand how to integrate Ocelot with different authentication solutions, therefore in future releases, I will dive into other scenarios as Firebase authentication and user session management with Redis.
