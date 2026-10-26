---
title: Install and upgrade
description: Install JWT Bearer Authentication on a Frappe v16 bench, see what the install changes, and upgrade or remove it.
nav_title: Install
order: 1
updated: 2026-10-26
---

JWT Bearer Authentication installs like any Frappe app. It adds one settings page and changes nothing on the site until you fill that page in.

## Before you start

- A bench with Frappe v16.
- A System Manager login on the site, to fill in **JWT Auth Settings** afterwards.
- The identity provider's details: the name of the cookie or header that carries the token, the JWKS URL, the audience your application's tokens are issued for, and the provider's login and logout URLs. For Cloudflare Access these come from the Access application that protects the site. See [Configuration](configuration.md#cloudflare-access).

> [!WARNING]
> Keep a System Manager session open, or another way in, until you have checked the settings. With **Override Frappe Login** on, Frappe's `/login` page sends every guest to the provider, so a wrong **Login URL** can lock you out of the password login.

## Install

Run these from the bench directory. `<SITE_NAME>` is your site, for example `erp.example.com`.

```bash
bench get-app https://github.com/Avunu/jwt_bearer_auth --branch version-16
bench --site <SITE_NAME> install-app jwt_bearer_auth
```

`bench get-app` installs the Python packages the app needs (`PyJWT` and `requests`) and builds its desk bundle, the small script that replaces the desk's **Logout**. If you skipped the build (`--skip-assets`), run `bench build --app jwt_bearer_auth`.

Then open **JWT Auth Settings** from the desk search, fill it in and save. See [Configuration](configuration.md).

## What installing sets up

The app has no fixtures, no custom fields, no property setters, no roles and no install hooks, and its patch list is empty. Installing it adds:

- The **JWT Auth** module and its single doctype, **JWT Auth Settings**, which only a System Manager can open. Nothing is saved in it until you save the form.
- Hooks that run on every request: the token check before the request is handled and the redirects after it. Until the settings hold a cookie or header name, the token check finds no token and does nothing.
- A replacement for the website account menu: **My Account** and a **Log out** that goes through the provider.
- A desk script that replaces the desk's **Logout**.

The full list is in [What the app adds to your site](configuration.md#what-the-app-adds-to-your-site).

## Upgrade

Update the app the way you update any Frappe app: `bench update`, or pull the new code in `apps/jwt_bearer_auth` and then run these from the bench directory.

```bash
bench --site <SITE_NAME> migrate
bench build --app jwt_bearer_auth
```

The app ships no data patches, so a migration only re-syncs the **JWT Auth Settings** doctype. Your saved settings are kept.

The app was called `jwt_auth` before it was renamed to `jwt_bearer_auth`. The old addresses of its logout methods, `jwt_auth.auth.jwt_logout`, `jwt_auth.auth.on_logout` and `jwt_auth.auth.web_logout`, still work and call the new ones, so a bookmark or a cached page that uses them keeps working.

## Uninstall

```bash
bench --site <SITE_NAME> uninstall-app jwt_bearer_auth
```

Frappe removes the **JWT Auth** module and the **JWT Auth Settings** doctype with its saved values, and the app's hooks stop running: tokens no longer log anyone in, `/login` shows Frappe's own login page again, and logging out no longer sends the browser to the provider. The website account menu goes back to Frappe's own **Log out**.

The users the app created, the links it set between those users and their Contacts, and the Contacts it renamed stay on the site. They are ordinary Website Users and Contacts. People who only ever signed in through the provider have no Frappe password: set one or send them a password reset if they still need to sign in.
