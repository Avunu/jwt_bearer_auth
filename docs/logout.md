---
title: Log out
description: How logging out of the desk or the website ends both the Frappe session and the identity provider's session.
nav_title: Log out
order: 5
updated: 2026-10-26
---

Ending only the Frappe session is not enough behind an identity provider. The browser still holds the provider's token, so the next request would log the person straight back in. The app therefore ends the Frappe session and then sends the browser to the provider's **Logout URL**, which ends the provider's session too.

## Log out of the desk

1. Open the user menu in the desk sidebar.
2. Click **Logout**.

The app replaces the desk's logout with its own, which does not ask "Are you sure you want to log out?" first. It calls `jwt_bearer_auth.auth.jwt_logout` in the background, which ends the Frappe session and answers with the address to open next: the **Logout URL**, or `/login` with **Enable JWT User Auth** off. The desk then opens that address, or Frappe's login page when the answer holds none.

> [!NOTE]
> The app's logout hook also runs during that background call, so the server turns its answer into a redirect to the **Logout URL** (see [Any other logout](#any-other-logout)). The browser follows that redirect inside the background call, not in the page, so the provider's logout is requested there and the desk does not receive the address it waits for. What the page shows next depends on how the provider answers that request. Logging out from the website menu is not affected.

## Log out of the website

On the website, the account menu holds **My Account** and **Log out**. The app sets this menu for every website page.

1. Open the account menu at the top of the page.
2. Click **Log out**.

**Log out** opens `/?cmd=jwt_bearer_auth.auth.web_logout`, which ends the Frappe session and redirects the browser to the **Logout URL**.

## Any other logout

The app also hooks into Frappe's own logout. Whenever a Frappe session ends during a browser request, for example through Frappe's `/api/method/logout`, the app records the **Logout URL**, and once the request is done it turns the response into a redirect to that address. Logging a user out from the desk on someone else's behalf, or a session that simply expires, involves no request from that user's browser, so nothing is redirected.

## The address the provider returns to

When **Redirect Param** is set, the app adds it to the **Logout URL** with the full address of the logout request, for example:

```text
https://erp.example.com/cdn-cgi/access/logout?redirect_url=https://erp.example.com/?cmd=jwt_bearer_auth.auth.web_logout
```

The address is added as it is, without URL encoding, and always after a `?`, so a **Logout URL** that already has a query string gets a second `?`. If your provider uses the parameter, check that it accepts this form.

## With token login turned off

Turning **Enable JWT User Auth** off does not stop logouts from going to the **Logout URL**. The app's logout hook records the **Logout URL** whatever that setting says, and the redirect that runs after the request uses it: the website's **Log out** and the other browser logouts above still end at the provider. **Logout URL** is a required field, so there is no setting that turns this off while the app is installed and its settings are saved.
