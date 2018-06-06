---
title: Use Microsoft Graph in tab pages
description: Explains how to use the Microsoft Graph APIs in Microsoft Teams tab pages
keywords: teams graph tabs
ms.date: 03/02/2018
---

# Use Microsoft Graph in your Microsoft Teams tab pages

You might want to use [Microsoft Graph](https://developer.microsoft.com/en-us/graph/) in your [configuration](~/concepts/tabs/tabs-configuration) and [content](~/concepts/tabs/tabs-content) pages. For example, you might want to access the user's profile information, calendar, or files.

To use the Microsoft Graph APIs, you must [get an access token](https://developer.microsoft.com/en-us/graph/docs/concepts/auth_overview). When your app is running in Microsoft Teams, the only difference is that you must drive the authentication flow, as described in [Authentication](~/concepts/authentication/authentication).

Microsoft Teams also has [APIs for Microsoft Graph](https://developer.microsoft.com/en-us/graph/docs/api-reference/beta/resources/teams_api_overview). However, there are currently some limitations to be aware of; in particular, be careful if you use the Microsoft Graph APIs for [team resources](https://developer.microsoft.com/en-us/graph/docs/api-reference/beta/resources/group), rather than those for the current user, because the two have different consent models. Typically, users can directly consent to your Microsoft Teams app within a specific team. However, currently an admin must also consent to your app (as registered in Azure Active Directory) using these group APIs, at which point the app then has API access to all the groups or teams for each user. (See [Group permissions](https://developer.microsoft.com/en-us/graph/docs/concepts/permissions_reference#group-permissions) for more information.) You should therefore ensure that your Microsoft Teams app handles not having the permissions it needs, and that it respects the user's intention about the teams in which it should operate.
