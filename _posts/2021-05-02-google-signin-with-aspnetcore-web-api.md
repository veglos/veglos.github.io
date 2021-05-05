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

The diagram is quite straightforward. Google provides both the [Sign In button](https://developers.google.com/identity/sign-in/web/sign-in) and the [Google API Client Library for .NET](https://www.nuget.org/packages/Google.Apis.Auth/) which is recommended for token validation.

## Front-end

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8" />
    <meta name="google-signin-client_id" content="[google-signin-client_id].apps.googleusercontent.com">
    <title></title>
    <script src="https://apis.google.com/js/platform.js?onload=init" async defer></script>
</head>
<body>
    <div class="g-signin2" data-onsuccess="onSignIn"></div>
    <button onclick="signOut()">Sign Out</button>

    <script type="text/javascript">

        function onLoad() {
            console.log("onLoad()");
            gapi.load('auth2', function () {
                gapi.auth2.init();
            });
        }

        function signOut() {
            console.log("signOut()");
            var auth2 = gapi.auth2.getAuthInstance();
            auth2.signOut().then(function () {
                auth2.disconnect();

                fetch('http://localhost:5000/api/Access/signout-google', {
                    method: 'POST',
                    headers: {
                        'Accept': 'application/json',
                        'Content-Type': 'application/json'
                    },
                    body: null
                })
                    .then(response => response.json())
                    .then(data => console.log(data))
                    .catch(error => console.error('Unable to proccess the request', error));
            });

        }

        function onSignIn(googleUser) {

            const item = {
                idToken: googleUser.getAuthResponse().id_token
            };

            fetch('http://localhost:5000/api/Access/signin-google', {
                method: 'POST',
                headers: {
                    'Accept': 'application/json',
                    'Content-Type': 'application/json'
                },
                body: JSON.stringify(item)
            })
            .then(response => response.json())
            .then(data => console.log(data))
            .catch(error => console.error('Unable to proccess the request', error));
        }
    </script>
</body>

</html>
```

Not much to say. We just have to be sure about using the proper meta **google-signin-client_id**.
More information at [Integrating Google Sign-In into your web app](https://developers.google.com/identity/sign-in/web/sign-in)

## Back-end

```cs
using Google.Apis.Auth;
using GoogleSignInApi.Models;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using System;
using System.Threading.Tasks;

namespace GoogleSignInApi.Controllers
{
    [Authorize]
    [Route("api/[controller]")]
    [ApiController]
    public class AccessController : ControllerBase
    {
        [AllowAnonymous]
        [HttpPost]
        [Route("signin-google")]
        public async Task<string> SignInGoogle(GoogleSignInTokenRequest request)
        {
            try
            {
                var validationSettings = new GoogleJsonWebSignature.ValidationSettings
                {
                    Audience = new string[] { "[google-signin-client_id].apps.googleusercontent.com" }
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
            catch(Exception ex)
            {
                return $"ERROR: {ex.Message}";
            }
        }

        [AllowAnonymous]
        [HttpPost]
        [Route("signout-google")]
        public async Task<string> SignOutGoogle()
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

According to the [docs](https://developers.google.com/identity/protocols/oauth2/openid-connect#validatinganidtoken), the validation process must check the signature, some claims and the optional hd parameter (if it was specified). The Google.Apis.Auth library however, does all of that for us with the exception of the Audience and Hosted Domain claims (the latter not used in this example), which must be provided. Hence the line 24. Again, be sure to use the correct **google-signin-client_id**.