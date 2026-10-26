---
title: Configuration
description: Every field of JWT Auth Settings, example values for Cloudflare Access, and everything the app adds to a Frappe site.
nav_title: Configuration
order: 2
updated: 2026-10-26
---

The app is configured on one page, **JWT Auth Settings**. Open it from the desk search; only a System Manager can open or change it. There are no `site_config.json` keys and no other settings.

## JWT Auth Settings

| Field                            | Default | What it does                                                                                                                                                                                                                                                              |
| -------------------------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Enable JWT User Auth**         | On      | Turns token login and the `/login` override on. With it off, tokens are ignored and `/login` shows Frappe's login page. It does not stop logout from going to the **Logout URL**: see [Log out](logout.md#with-token-login-turned-off).                                   |
| **Enable New User Registration** | On      | Creates a user when a valid token carries an email that has no user yet. With it off, such a token logs nobody in. See [New users from a token](registration.md).                                                                                                         |
| **JWT Cookie/Header**            | None    | Required. The name of the cookie or HTTP header that carries the token. The app looks for a cookie of that name first and then a header.                                                                                                                                  |
| **JWKS URL**                     | None    | Required. The address of the provider's public keys in JSON Web Key Set form. The app fetches it for every token it checks.                                                                                                                                               |
| **JWT Private Secret**           | None    | Required. Despite its label, this is not a signing secret: it is the audience (`aud`) the token must be issued for. A token issued for any other audience is rejected. It is stored as a password field. If it is empty, every token is rejected.                         |
| **Override Frappe Login**        | On      | Sends guests who open Frappe's `/login` page to the **Login URL**, with the page they asked for in **Redirect Param**. It works only while **Enable JWT User Auth** is on.                                                                                                |
| **Login URL**                    | None    | Required. The provider's login page, used by **Override Frappe Login**.                                                                                                                                                                                                   |
| **Logout URL**                   | None    | Required. Where the browser goes after the Frappe session ends, so the provider's session ends too. See [Log out](logout.md).                                                                                                                                             |
| **Redirect Param**               | None    | The name of the query parameter that carries the address to come back to, added to the **Login URL** and the **Logout URL**. The form requires it while **Override Frappe Login** is on. Leave it empty, with the override off, if your provider takes no such parameter. |

The form cannot be saved until **JWT Cookie/Header**, **JWKS URL**, **JWT Private Secret**, **Login URL** and **Logout URL** are filled in.

> [!WARNING]
> **Override Frappe Login** is on by default. As soon as you save the form with **Enable JWT User Auth** on, Frappe's `/login` page sends every guest to the **Login URL**, including you once you log out. Check the **Login URL** before you save, and keep a logged-in session open while you test. [Limitations and troubleshooting](troubleshooting.md#you-are-locked-out-of-the-password-login) explains how to turn the override off from the bench.

## Cloudflare Access

With Cloudflare Access in front of the site, these values work. `<TEAM_NAME>` is your Cloudflare Zero Trust team name, `erp.example.com` stands for your site's domain, and the AUD tag is shown on the Access application's overview in the Zero Trust dashboard.

| Field                  | Value                                                                           |
| ---------------------- | ------------------------------------------------------------------------------- |
| **JWT Cookie/Header**  | `CF_Authorization` (the cookie) or `Cf-Access-Jwt-Assertion` (the header)       |
| **JWKS URL**           | `https://<TEAM_NAME>.cloudflareaccess.com/cdn-cgi/access/certs`                 |
| **JWT Private Secret** | The Access application's Application Audience (AUD) tag                         |
| **Login URL**          | `https://<TEAM_NAME>.cloudflareaccess.com/cdn-cgi/access/login/erp.example.com` |
| **Logout URL**         | `https://erp.example.com/cdn-cgi/access/logout`                                 |
| **Redirect Param**     | `redirect_url`                                                                  |

These addresses are Cloudflare's, not the app's: check them against Cloudflare's documentation for your account. The app's demo site uses the same pattern.

## Roles and permissions

The app checks no roles of its own. **JWT Auth Settings** can be read and changed by the System Manager role only.

A person logged in from a token gets exactly the permissions of their Frappe user. Users the app creates are Website Users with no roles: give them roles and desk access by hand if they need more than the website.

## What the app adds to your site

This is the complete list.

### Doctypes and fields

| What                  | Details                                                                                                        |
| --------------------- | -------------------------------------------------------------------------------------------------------------- |
| **JWT Auth Settings** | A single doctype in the **JWT Auth** module, holding the fields above. Permission: System Manager.             |
| Custom fields         | None. The app adds no custom fields or property setters to standard doctypes and ships no fixtures or patches. |

### Hooks

| Hook              | What it does                                                                                                                                                                                                                                                                |
| ----------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `auth_hooks`      | `jwt_bearer_auth.auth.validate_auth` runs on every request. For a guest with a valid token it logs in the matching user, registering one when that is on. See [Sign in with a token](sign-in.md).                                                                           |
| `after_request`   | `jwt_bearer_auth.auth.handle_redirects` runs after every request and applies the redirect to the provider after logout, the redirect to the profile page after registration, and the `/login` override.                                                                     |
| `on_logout`       | `jwt_bearer_auth.auth.on_logout` records the **Logout URL** whenever a session ends, for `after_request` to send the browser there.                                                                                                                                         |
| `doc_events`      | Contact `on_update` runs `jwt_bearer_auth.jwt_auth.hooks.contact.on_update`, which copies name, phone, mobile and gender changes to the linked user and renames placeholder Contacts. See [New users from a token](registration.md#keep-the-user-in-step-with-the-contact). |
| `website_context` | Replaces the website account menu with **My Account** (`/me`) and **Log out** (`/?cmd=jwt_bearer_auth.auth.web_logout`).                                                                                                                                                    |
| `app_include_js`  | `app.bundle.js`, loaded on every desk page, replaces the desk's **Logout** with a call to `jwt_bearer_auth.auth.jwt_logout`.                                                                                                                                                |

The app overrides no doctype classes and has no scheduled jobs.

### Whitelisted methods

| Method                            | What it does                                                                                                                                                                                                                                                                                                                                       |
| --------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `jwt_bearer_auth.auth.jwt_logout` | Ends the session and returns `{"redirect_url": ...}`: the **Logout URL**, or `/login` with **Enable JWT User Auth** off. The desk's **Logout** calls it. Because the `on_logout` hook runs in the same request, the reply reaches the browser as a redirect to the **Logout URL**, not as that JSON. See [Log out](logout.md#log-out-of-the-desk). |
| `jwt_bearer_auth.auth.web_logout` | Ends the session and redirects the browser to the **Logout URL**. The website menu's **Log out** uses it.                                                                                                                                                                                                                                          |
| `jwt_bearer_auth.auth.on_logout`  | The `on_logout` hook. It is whitelisted but does nothing useful on its own.                                                                                                                                                                                                                                                                        |

The same three methods also answer at their old addresses, `jwt_auth.auth.jwt_logout`, `jwt_auth.auth.web_logout` and `jwt_auth.auth.on_logout`, from before the app was renamed.

## Related

- [Install and upgrade](install.md)
- [Limitations and troubleshooting](troubleshooting.md)
- [How it works](development.md#how-it-works)
