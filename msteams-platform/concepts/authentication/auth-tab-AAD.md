---
title: Authentication for tabs using Azure Active Directory
description: Describes authentication in Teams and how to use it in tabs
keywords: teams authentication tabs AAD
ms.date: 03/01/2018
---
# Authenticate a user in a Microsoft Teams tab

There are many services that you may wish to consume inside your Teams app, and most of those services require authentication and authorization to get access to the service. Services include Facebook, Twitter, and of course Teams. Users of Teams have user profile information stored in Azure Active Directory (AAD) using Microsoft Graph and this article will focus on authentication using AAD to get access to this information.

OAuth is an open standard for authentication used by AAD and many other service providers. Understanding OAuth is a prerequisite for working with authentication in Teams and AAD. The examples below use the OAuth2 Implicit Grant flow with the goal of eventually reading the user's profile information from AAD and Graph.

The code in this article comes from the Teams sample app [Microsoft Teams tab authentication sample (Node)](https://github.com/OfficeDev/microsoft-teams-sample-complete-node). It contains a static tab that requests an access token for Microsoft Graph and shows the current user's basic profile information from Azure AD.

For a general overview of authentication flow for tabs see the topic [Authentication flow in tabs](~/concepts/authentication/auth-flow-tab).

Authentication flow in tabs differs slightly from authentication flow in bots.

## Configuring identity providers

See the topic [Configure identity providers](~/concepts/authentication/auth-configure) for detailed steps on configuring OAuth 2.0 callback redirect URL(s) when using Azure Active Directory as an identity provider.

## Initiate authentication flow

Authentication flow should be triggered by a user action. You should not open the authentication pop-up automatically because this is likely to trigger the browser's pop-up blocker as well as confuse the user.

Add a button to your configuration or content page to enable the user to sign in when needed. This can be done in the tab [configuration](~/concepts/tabs/tabs-configuration) page or any [content](~/concepts/tabs/tabs-content) page.

AAD, like most identity providers, does not allow its content to be placed in an iframe. This means that you will need to add a pop-up page to host the identity provider. In the following example this page is `/tab-auth/simple-start`. Use the `microsoftTeams.authenticate()` function of the Microsoft Teams client SDK to launch this page when your button is selected.

```javascript
microsoftTeams.authentication.authenticate({
    url: window.location.origin + "/tab-auth/simple-start",
    width: 600,
    height: 535,
    successCallback: function (result) {
        getUserProfile(result.accessToken);
    },
    failureCallback: function (reason) {
        handleAuthError(reason);
    }
});
```

### Notes

* The URL you pass to `microsoftTeams.authentication.authenticate()` is the start page of the authentication flow. In this example that is `/tab-auth/simple-start`. This should match what you registered in the [AAD Application Registration Portal](https://apps.dev.microsoft.com).

* Authentication flow must start on a page that's on your domain. This domain should also be listed in the [`validDomains`](~/resources/schema/manifest-schema#validdomains) section of the manifest. Failure to do so will result in an empty pop-up.

* Failing to use `microsoftTeams.authentication.authenticate()` will cause a problem with the popup not closing at the end of the sign in process.

## Navigate to the authorization page from your popup page

When your popup page (`/tab-auth/simple-start`) is displayed the following code is run. The main goal of this page is to redirect to your identity provider so the user can sign in. This redirection could be done on the server side using HTTP 302, but in this case it is done on the client side using with a call to `window.location.assign()`. This also allows `microsoftTeams.getContext()` to be used to retrieve hinting information which can be passed to AAD.

```javascript
microsoftTeams.getContext(function (context) {
    // Generate random state string and store it, so we can verify it in the callback
    let state = _guid(); // _guid() is a helper function in the sample
    localStorage.setItem("simple.state", state);
    localStorage.removeItem("simple.error");
    // Go to the Azure AD authorization endpoint
    let queryParams = {
        client_id: "YOUR_APP_ID_HERE",
        response_type: "id_token token",
        response_mode: "fragment",
        resource: "https://graph.microsoft.com/User.Read openid",
        redirect_uri: window.location.origin + "/tab-auth/simple-end",
        nonce: _guid(),
        state: state,
        // The context object is populated by Teams; the loginHint attribute
         // is used as hinting information
        login_hint: context.loginHint,
    };

    let authorizeEndpoint = "https://login.microsoftonline.com/" + context.tid + "/oauth2/authorize?" + toQueryString(queryParams);
    window.location.assign(authorizeEndpoint);
});
```

After the user completes authorization, the user is redirected to the callback page you specified for your app at `/tab-auth/simple-end`.

### Notes

* See [get user context information](~/concepts/tabs/tabs-context) for help building authentication requests and URLs. For example, you can use the user's login name as the `login_hint` value for Azure AD sign-in, which means the user might need to type less. Remember that you should not use this context directly as proof of identity since an attacker could load your page in a malicious browser and provide it with any information they want.
* Although the tab context provides useful information regarding the user, don't use this information to authenticate the user whether you get it as URL parameters to your tab content URL or by calling the `microsoftTeams.getContext()` function in the Microsoft Teams client SDK. A malicious actor could invoke your tab content URL with its own parameters, and a web page impersonating Microsoft Teams could load your tab content URL in an iframe and return its own data to the `getContext()` function. You should treat the identity-related information in the tab context simply as hints and validate them before use.
* The `state` parameter is used to confirm that the service calling the callback URI is the service you called. If the `state` parameter in the callback does not match the parameter you sent during the call the return call is not verified and should be terminated.
* The `microsoftTeams.navigateCrossDomain()` function is not available in the context of the identity provider's popup. As a result, it is not necessary to include the identity provider's domain in the `validDomains` list in the app's manifest.json file.

## The callback page

In the last section you called the AAD authorization service and passed in user and app information so that AAD could present the user with its own monolithic authorization experience. Your app has no control over what happens in this experience. All it knows is what is returned when AAD calls the  callback page that you provided (`/tab-auth/simple-end`).

In this page you need to determine success or failure based on the information returned by AAD and call `microsoftTeams.authentication.notifySuccess()` or `microsoftTeams.authentication.notifyFailure()`. If the login was successful you will have access to service resources.

````javascript
// Split the key-value pairs passed from Azure AD
// getHashParameters is a helper function that parses the arguments sent
// to the callback URL by Azure AD after the authorization call
let hashParams = getHashParameters(); 
if (hashParams["error"]) {
    // Authentication/authorization failed
    microsoftTeams.authentication.notifyFailure(hashParams["error"]);
} else if (hashParams["access_token"]) {
    // Get the stored state parameter and compare with incoming state
    // This validates that the data is coming from Azure AD
    let expectedState = localStorage.getItem("simple.state");
    if (expectedState !== hashParams["state"]) {
        // State does not match, report error
        microsoftTeams.authentication.notifyFailure("StateDoesNotMatch");
    } else {
        // Success: return token information to the tab
        microsoftTeams.authentication.notifySuccess({
            idToken: hashParams["id_token"],
            accessToken: hashParams["access_token"],
            tokenType: hashParams["token_type"],
            expiresIn: hashParams["expires_in"]
        })
    }
} else {
    // Unexpected condition: hash does not contain error or access_token parameter
    microsoftTeams.authentication.notifyFailure("UnexpectedFailure");
}
````

This code parses the key-value pairs received from Azure AD in `window.location.hash` using the `getHashParameters()` helper function. If it finds an `access_token`, and the `state` value is the same as the one provided at the start of the authentication flow, it returns the access token to the tab by calling `notifySuccess()`; otherwise it reports an error with `notifyFailure()`.

### Notes

`NotifyFailure()` has the following predefined failure reasons:

* `CancelledByUser` the user closed the popup window before completing the authentication flow.
* `FailedToOpenWindow` the popup window could not be opened. When running Microsoft Teams in a browser, this typically means that the window was blocked by a popup blocker.

If successful, you can refresh or reload the page and show content relevant to the now-authenticated user. If authentication fails, display an error message.

Your app can set its own session cookie so that the user need not sign in again when they return to your tab on the current device.

For more information on Single Sign-On (SSO) see the article [Silent authentication](~/concepts/authentication/auth-silent-AAD).

For more information on using AAD authentication for bots, see [Authentication for bots (AAD)](~/concepts/authentication/auth-bot-AAD)

## Samples

For sample code showing the tab authentication process using AAD see:

* [Microsoft Teams tab authentication sample (Node)](https://github.com/OfficeDev/microsoft-teams-sample-complete-node)
* [Microsoft Teams tab authentication sample (C#)](https://github.com/OfficeDev/microsoft-teams-sample-complete-csharp)
