---
layout: post
title:  "Google Sign In with ASP.NET Core Web API"
categories: jekyll 
tags: aspnetcore api google social
---

I couldn't find a simple, short and straight to the point article about this topic, so I made my own.
The complete project is hosted on [https://github.com/veglos/dotnet-google-signin-api](https://github.com/veglos/dotnet-google-signin-api).

## Diagram

![/diagram-of-google-sign-in](/assets/img/2021-05-02-google-signin-with-aspnetcore-web-api/diagram.png)
_diagram of google sign in_

Google provides both the [Sign In button](https://developers.google.com/identity/sign-in/web/sign-in) and the [Google API Client Library for .NET](https://www.nuget.org/packages/Google.Apis.Auth/) which is recommended for token validation.

## Pre-Requirements

We must create a Google Cloud Platform Project and create an OAuth 2.0 Client ID credential [https://console.cloud.google.com/apis/credentials](https://console.cloud.google.com/apis/credentials). Once the credential has been created, Google will provide us with a **Client ID** and a **Client secret**. We will use them later. 

Finally, we must declare the Authorized JavaScript origins URIs. For development purposes I have set it up as **https://localhost:5001**.

## Front-end

Not much to say. We just have to be sure to set the proper meta **google-signin-client_id**, that is, the **Client ID** we got earlier.
The **onSignIn()** function sends the token_id to the back-end server to be validated. The token contains the Google's userId.
The **signOut()** will logout from Google, but could also revoke a refresh token.
More information at [Integrating Google Sign-In into your web app](https://developers.google.com/identity/sign-in/web/sign-in)

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8" />
    <meta name="google-signin-client_id" content="1234567890-1q2w3e4r5t6y7u8i.apps.googleusercontent.com">
    <title></title>
    <script src="https://apis.google.com/js/platform.js?onload=init" async defer></script>
</head>
<body>
    <div class="g-signin2" data-longtitle="true" data-onsuccess="onSignIn"></div>
    <button onclick="signOut()">Sign Out</button>

    <script type="text/javascript">

        function onLoad() {
            console.log("onLoad()");
            gapi.load('auth2', function () {
                gapi.auth2.init();
            });
        }

        function onSignIn(googleUser) {

            const item = {
                idToken: googleUser.getAuthResponse().id_token
            };

            fetch('https://localhost:5001/api/Access/signin-google', {
                method: 'POST',
                headers: {
                    'Accept': 'application/json',
                    'Content-Type': 'application/json'
                },
                body: JSON.stringify(item)
            })
            .then(response => response.json())
            .then(data => console.log(data))
            .catch(error => console.error('Unable to process the request', error));
        }

        function signOut() {
            console.log("signOut()");
            var auth2 = gapi.auth2.getAuthInstance();
            auth2.signOut().then(function () {
                auth2.disconnect();

                // optionally should send the access token to revoke the refresh token too
                fetch('https://localhost:5001/api/Access/signout-google', {
                    method: 'POST',
                    headers: {
                        'Accept': 'application/json',
                        'Content-Type': 'application/json'
                    },
                    body: null
                })
                    .then(response => response.json())
                    .then(data => console.log(data))
                    .catch(error => console.error('Unable to process the request', error));
            });

        }
    </script>
</body>

</html>
```


## Back-end

The **SignInGoogle()** method validates the token and then should fetch the user's claims and return an Access Token and a Refresh Token.

According to the [docs](https://developers.google.com/identity/protocols/oauth2/openid-connect#validatinganidtoken), the validation process must check the signature, some claims and the optional hd parameter (if it was specified). The Google.Apis.Auth library however, does all of that for us with the exception of the Audience and Hosted Domain claims (the latter not used in this example), which must be provided. Hence the line 24. Again, be sure to use the correct **google-signin-client_id**.

```cs
using Google.Apis.Auth;
using GoogleSignInApi.Models;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Options;
using System;
using System.Threading.Tasks;

namespace GoogleSignInApi.Controllers
{
    [Authorize]
    [Route("api/[controller]")]
    [ApiController]
    public class AccessController : ControllerBase
    {
        private readonly IOptions<AppSettings> _settings;

        public AccessController(IOptions<AppSettings> settings)
        {
            _settings = settings;
        }

        [AllowAnonymous]
        [HttpPost]
        [Route("signin-google")]
        public async Task<string> SignInGoogle(GoogleSignInTokenRequest request)
        {
            try
            {
                var validationSettings = new GoogleJsonWebSignature.ValidationSettings
                {
                    Audience = new string[] { _settings.Value.GoogleSettings.ClientID }
                };

                var token = request.IdToken;
                var payload = await GoogleJsonWebSignature.ValidateAsync(token, validationSettings);

                // Now we are sure the user has authenticated via Google and we can proceed to do anything
                // like fetch the user's claims and provide her/him with an Access Token.
                return "OK";
            }
            catch (InvalidJwtException ex)
            {
                return $"ERROR: InvalidJwt - {ex.Message}";
            }
            catch (Exception ex)
            {
                return $"ERROR: {ex.Message}";
            }
        }

        [AllowAnonymous]
        [HttpPost]
        [Route("signout-google")]
        public async Task<string> SignOutGoogle() //should actually receive a request with the access token and the user Id.
        {
            try
            {
                // Proceed to take any actions if needed such as disabling or deleting the user's refresh token.
                return "OK";
            }
            catch (Exception ex)
            {
                return $"ERROR: {ex.Message}";
            }
        }
    }
}

```
