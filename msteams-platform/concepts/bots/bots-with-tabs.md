---
title: Combine bots with tabs
description: Describes how to use tabs and bots together
keywords: teams bots tabs development
ms.date: 03/15/2018
---
# Combine bots with tabs

Bots and tabs work well together, and are often combined into a single back-end service in. This section describes best practices and common patterns for using tabs and bots together.

## Associating user identities across bot and tab

For example:
Suppose your tab application uses a proprietary ID system to secure its content. Suppose you also have a bot that can interact with the user. Typically, you’ll want to show content in the tab that is specific to the viewing user. The challenge is that the user ID in your system is likely different from the Microsoft Teams user ID. So how do you associate these two identities?
In general, the recommended approach is to sign the user in with the bot using the same identity system used to provide authentication for the tab content. You can implement this via the sign-in action, which typically logs in the user via an OAuth flow.

This flow works best if your identity provider implements the OAuth 2.0 protocol. You can then associate the Teams user ID with the user’s credentials from your own identity service. See [Authenticating in Teams](~/concepts/authentication/authentication) for detailed information.

   ![Associating identities](~/assets/images/bots/associating_contexts.png)

## Constructing deep links to tabs in messages from your bot

You may want to use tabs to show more content than can fit inside of a card, or provide a way to complete complex form-filling tasks using the tab canvas. For example, consider navigating the user to the tab when he or she clicks on the card from your bot. For this to happen, you’ll need to encode your bot’s message to include a [deep link](~/concepts/deep-links) URL, either via markup or as the target of the openUrl action.

Deep links rely on an entityId, which is an opaque value that maps to a unique entity in your system. When the tab is created, you ideally store some simple state (e.g. flag) on your backend indicating the tab has been created in the channel. When your bot constructs a message, it can target the entityId associated with that tab.

**Note:** in personal chats, because tabs are “static” and installed with the app, you can always assume their existence and thus construct deep links accordingly.

## Sending notifications for tab updates

Often you’ll want to notify the end user whenever an update or a user action occurs in a tab. An example scenario is assigning a task or ticket to a fellow team member and then notifying that team member.

There are two ways of achieving this scenario:

1. If you wish to notify an entire channel your bot can asynchronously post a message to the channel. Note that there is currently no way for a bot to proactively create the tab conversation if it wasn't created with the tab.

2. If you wish to only notify the recipient or interested parties involved with the action, your bot can send a personal chat message to the user. You should first check to see if a personal conversation between your bot and the user exists. If not, you can call `CreateConversation` to initiate the personal chat.

In both cases, use event notifications wisely and never spam the user with unnecessary updates.

## FAQ

- Do you support tabs on mobile?
  Microsoft Teams does not currently embed content on our mobile clients – we recommend that you supply a fallback URL in your tab definition such that we can load the appropriate responsive web content on mobile devices.
- How do I send ephemeral messages to a user?
  Teams does not currently support the ability for a bot to message a user in a channel such that only a specific user can view the message (everyone sees all messages in a channel). The current workaround is to send a message to the user in personal chat.