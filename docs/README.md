---
title: JWT Bearer Authentication
description: Sign people in to a Frappe site from the JWT that Cloudflare Access or another JWKS identity provider attaches to each request.
nav_title: Overview
order: 0
updated: 2026-10-26
---

When a Frappe site sits behind an identity provider such as Cloudflare Access, people sign in twice: once at the provider and once more with a Frappe password. JWT Bearer Authentication removes the second step. The provider attaches a signed JSON Web Token (JWT) to every request, and the app checks that token and logs in the Frappe user whose email it carries.

It is for teams that already manage their people in an identity provider and want Frappe to trust it. Cloudflare Access is the provider it is built and used with; any provider that signs RS256 tokens and publishes its keys at a JWKS URL works the same way. It runs on Frappe v16 and is free software under the [MIT license](../license.txt).

## What you get

### Sign-in from the provider's token

On each request from a visitor who is not logged in, the app reads the token from the cookie or header you name, checks its signature against the provider's keys and its audience against your application, and logs in the matching user. With **Override Frappe Login** on, guests who open Frappe's `/login` page are sent to the provider instead. See [Sign in with a token](sign-in.md).

![JWT Auth Settings filled in for Cloudflare Access: token cookie, JWKS URL, audience, and login and logout URLs](screenshots/settings-light.webp)

### Users created on first sign-in

With **Enable New User Registration** on, a valid token for an email that has no user yet creates a Website User, filled in from a Contact with that email when there is one. People without a name on file are asked to complete their profile. Later changes to the Contact are copied to the user. See [New users from a token](registration.md).

![The user list with two website users created from a token: one filled in from a Contact, one still to complete their profile](screenshots/registered-users-light.webp)

### Logout through the provider

Logging out of the desk or the website ends the Frappe session and then sends the browser to the provider's logout URL, so the provider's session ends too and the next request does not log the person straight back in. See [Log out](logout.md).

## Where to start

| You are                                | Read                                                                                                   |
| -------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| Someone who signs in to the site       | [Sign in with a token](sign-in.md), [New users from a token](registration.md) and [Log out](logout.md) |
| The administrator setting it up        | [Install and upgrade](install.md), then [Configuration](configuration.md)                              |
| The administrator when something fails | [Limitations and troubleshooting](troubleshooting.md)                                                  |
| A developer changing the app           | [Development](development.md)                                                                          |

## Requirements

- A bench with Frappe v16. The app declares `frappe >=16.0.0,<17.0.0`. It needs no other Frappe app.
- An identity provider in front of the site that issues RS256 JWTs carrying an `email` claim and publishes its public keys at a JWKS URL, such as a Cloudflare Access application.
- The Python packages `PyJWT` and `requests`, which installing the app pulls in.
- Outbound HTTPS from the Frappe server to the provider's JWKS URL.

## Source and changes

The source is at [github.com/Avunu/jwt_bearer_auth](https://github.com/Avunu/jwt_bearer_auth). Each release and its notes are on the [releases page](https://github.com/Avunu/jwt_bearer_auth/releases).
