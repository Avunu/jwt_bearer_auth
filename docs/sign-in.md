---
title: Sign in with a token
description: How a request carrying the provider's JWT logs a person in to Frappe, which user it picks, and what the /login override does.
nav_title: Sign in
order: 3
updated: 2026-10-26
---

With the app set up, people do not sign in to Frappe at all. They sign in at the identity provider, the provider attaches a token to every request it lets through, and the app logs them in to Frappe from that token.

## What a person sees

1. Open any page of the site, for example `https://erp.example.com/app`.
2. The identity provider asks you to sign in, the way it is set up to (for Cloudflare Access, its login page). Frappe's own login page does not appear.
3. After you sign in, the provider sends you on to the page you asked for, and you are logged in to Frappe as the user with your email address.

If you have no Frappe user yet and registration is on, the app creates one as you arrive. See [New users from a token](registration.md).

## What the app checks

The token check runs at the start of every request, but only when all of these hold:

- **Enable JWT User Auth** is on.
- The request is not already logged in. A person with a Frappe session keeps it, whatever token arrives.
- No logout is under way in the same request.
- The request carries a token: a cookie named in **JWT Cookie/Header**, or, when there is no such cookie, a header of that name.

The app then decides whether the token is valid. It fetches the provider's keys from the **JWKS URL** and tries each one until one verifies the token. A token passes only when:

- It is signed with RS256 by one of those keys. Unsigned tokens and tokens with other algorithms fail.
- Its audience (`aud`) is the value in **JWT Private Secret**. With that field empty, every token fails.
- It has not expired, when it carries an expiry (`exp`). PyJWT, the library that decodes the token, also checks the not-before (`nbf`) and issued-at (`iat`) times when the token carries them. The issuer (`iss`) is not checked.

A token that fails is ignored: the request goes on as a guest, and no error is shown.

## Which user is logged in

A valid token must carry an `email` claim; a token without one logs nobody in. The app looks for a user in this order:

1. A user whose ID is that email. For users created from an email address, the ID is the email.
2. The first Contact found with that email, when it is linked to a user. That user is logged in, even when its own email is different.
3. When **Enable New User Registration** is on, a new Website User for that email. See [New users from a token](registration.md).

With none of these, the request goes on as a guest.

The user is logged in through Frappe's own login manager, so Frappe's checks still apply: a disabled user cannot be logged in, a user's **Restrict IP** list is enforced, and so are the user's **Login After** and **Login Before** hours.

## Send the login page to the provider

When **Override Frappe Login** is on, a guest who opens Frappe's `/login` page is sent to the **Login URL** instead. If **Redirect Param** is set, the app adds it to the URL with the page the guest asked for, so the provider can send them back there. For example, with the [Cloudflare Access values](configuration.md#cloudflare-access), a guest sent to `/login?redirect-to=/app/todo` goes to:

```text
https://<TEAM_NAME>.cloudflareaccess.com/cdn-cgi/access/login/erp.example.com?redirect_url=%2Fapp%2Ftodo
```

Without a `redirect-to` on the request, the page to come back to is `/`.

Only guests are redirected. A request that arrives at `/login` with a valid token is logged in first, and Frappe's login page then sends the logged-in user on as usual. With **Override Frappe Login** off, guests see Frappe's own login page and can sign in with a password.

## Signing in from scripts

Any client that holds a valid token for your audience can send it in the cookie or header, not only a browser that came through the provider. A script that calls the site through the provider can therefore use the same token. Frappe's own API keys and OAuth bearer tokens in the `Authorization` header keep working as before; the app's check runs after Frappe's and only for requests that are still guests.
