---
layout: post
title:  "Facebook Sign In with ASP.NET Core Web API"
categories: howto 
tags: aspnetcore api facebook social
---

Similar to my previous post [Google Sign In with ASP.NET Core Web API](/posts/google-signin-with-aspnetcore-web-api), now with Facebook.
The complete project is hosted on [https://github.com/veglos/dotnet-facebook-signin-api](https://github.com/veglos/dotnet-facebook-signin-api).

## Diagram

![/diagram-of-facebook-sign-in](/assets/img/2021-05-05-facebook-signin-with-aspnetcore-web-api/diagram.png)
_diagram of facebook sign in_


## Pre-Requirements

We must create an App in [Facebook for Developers](https://developers.facebook.com/) and get the **App ID** and **App Secret** generated by Facebook. We will use them later for token validation.

## Front-end

First we set up the Facebook login button with the [Plugin Configurator](https://developers.facebook.com/docs/facebook-login/web/login-button). Once the button handles the Facebook SignIn process, we take care of the rest of the flow as indicated in [Manually Build a Login Flow](https://developers.facebook.com/docs/facebook-login/manually-build-a-login-flow).

Bear in mind:
* Set the correct **appId** in the first script.
* Set the **data-onlogin** in the fb-login-button.

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8" />
    <title></title>
</head>
<body>
    <div id="fb-root"></div>
    <script async defer crossorigin="anonymous" src="https://connect.facebook.net/en_US/sdk.js#xfbml=1&version=v10.0&appId=1234567890&autoLogAppEvents=1" nonce="WSrc5Zps"></script>
    <div class="fb-login-button" data-width="" data-size="large" data-button-type="login_with" data-layout="default" data-auto-logout-link="false" data-use-continue-as="false" data-onlogin="onSignIn"></div>
    <button onclick="signOut()">Sign Out</button>

    <script>
        function onSignIn(response) {

            console.log(response);

            fetch('https://localhost:5001/api/Access/signin-facebook', {
                method: 'POST',
                headers: {
                    'Accept': 'application/json',
                    'Content-Type': 'application/json'
                },
                body: JSON.stringify(response.authResponse)
            })
                .then(response => response.json())
                .then(data => console.log(data))
                .catch(error => console.error('Unable to proccess the request', error));
        }

        function signOut() {

            FB.getLoginStatus(function (response) {
                console.log(response);
              
                FB.logout(function () {

                    fetch('https://localhost:5001/api/Access/signout-facebook', {
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
            });
        }
    </script>

</body>
</html>
```

## Back-end

Once we get the token we must [verify it](https://developers.facebook.com/docs/facebook-login/manually-build-a-login-flow#checktoken)

Bear in mind:
* Set the **_settings.Value.FacebookSettings.AppID** from appsettings.json
* Set the **_settings.Value.FacebookSettings.AppSecret** from a secret manager.

```cs
using FacebookSignInApi.Models.Facebook.Requests;
using FacebookSignInApi.Services;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Options;
using System;
using System.Threading.Tasks;

namespace FacebookSignInApi.Controllers
{
    [Authorize]
    [Route("api/[controller]")]
    [ApiController]
    public class AccessController : ControllerBase
    {
        private IFacebookService _facebookService;
        private readonly IOptions<AppSettings> _settings;

        public AccessController(
            IOptions<AppSettings> settings,
            IFacebookService facebookService)
        {
            _settings = settings;
            _facebookService = facebookService;
        }

        [AllowAnonymous]
        [HttpPost]
        [Route("signin-facebook")]
        public async Task<string> SignInFacebook(FacebookSignInRequest request)
        {
            try
            {
                var inputToken = request.AccessToken;
                var accessToken = $"{_settings.Value.FacebookSettings.AppID}|{_settings.Value.FacebookSettings.AppSecret}";
                var result = await _facebookService.DebugToken(inputToken, accessToken);

                // Now we are sure the token is valid for our app, however we must verify if the user from the request is the same as the user within the token.
                if (result.data.is_valid == true && result.data.user_id == request.UserID)
                {
                    // We should now fetch the user's claims and provide her/him with an Access Token.
                    return "OK";
                }
                else
                    throw new Exception("Token is not valid");
            }
            catch (Exception ex)
            {
                return $"ERROR: {ex.Message}";
            }
        }

        [AllowAnonymous]
        [HttpPost]
        [Route("signout-facebook")]
        public async Task<string> SignOutFacebook() //should actually receive a request with the access token and the user Id.
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