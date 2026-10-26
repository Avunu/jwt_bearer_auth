---
title: Develop JWT Bearer Authentication
description: Set up the dev shell, run the tests and checks, see how CI and releases work, and follow a request through the app's code.
nav_title: Development
order: 9
updated: 2026-10-26
---

This page is for people changing the app. The managed files, the CI gates and the release flow it shares with the other Avunu apps are described once in the [app standards](https://docs.avunu.net/frappe-nix/docs/app-standards/); this page covers what is specific to this repository.

## Set up

The repository is a frappe-nix app. Its dev shell provides Python, Node, the tools CI pins, and a bench with Frappe v16 in `.frappe-nix/bench`.

```bash
nix develop
prek install
```

`direnv allow` loads the same shell when you enter the folder. `prek install` adds the git hooks that CI's lint job runs.

## Commands

Run these in the dev shell, from the repository root.

| Command                                           | What it does                                                                          |
| ------------------------------------------------- | ------------------------------------------------------------------------------------- |
| `frappe-test`                                     | Runs the app's tests on a test site, the way CI does                                  |
| `prek run --all-files`                            | Runs every lint and format hook (ruff, ssort, oxfmt, oxlint and the others)           |
| `yarn check`                                      | Checks formatting (oxfmt), lint (oxlint) and TypeScript types                         |
| `nix run --no-pure-eval .#frappe-init -- --check` | Checks that the files frappe-nix manages are unchanged; it prints `frappe-nix: clean` |

The tests are Frappe integration tests:

| File                                             | What it covers                                                                                             |
| ------------------------------------------------ | ---------------------------------------------------------------------------------------------------------- |
| `jwt_bearer_auth/test_auth.py`                   | Token validation, the guest checks, user lookup and registration, the login and logout URLs, the redirects |
| `jwt_bearer_auth/jwt_auth/hooks/test_contact.py` | Copying Contact changes to the user and renaming `[Change Me]` Contacts                                    |
| `jwt_bearer_auth/test_demo.py`                   | The demo data                                                                                              |

The token tests sign their own RS256 tokens with a key made for the test and stub the JWKS request, so they need no network and no identity provider.

## Demo data and screenshots

`jwt_bearer_auth/demo.py` fills in **JWT Auth Settings** with Cloudflare Access values for a fictitious team and registers two users through the app's own registration code: one from an existing Contact and one without. It fetches nothing and turns **Override Frappe Login** off, so the demo site's own login page stays reachable. frappe-nix's demo and screenshot tools use it; `marketplace/screenshots.ts` names the pages captured into `docs/screenshots/`. Those images are generated: do not edit them by hand.

## CI and releases

Every pull request runs the shared gates from frappe-nix: `ci / lint`, `ci / typecheck`, `ci / test`, `ci / marketplace` and `ci / pr-policy`, which checks the pull request's title and commits. A pull request that changes `docs/` also runs the `Docs` check, which builds these pages the way [docs.avunu.net](https://docs.avunu.net/jwt-bearer-auth/docs/) does.

Commits follow Conventional Commits. release-please, run by `release.yml` on `develop`, turns them into a release pull request and a changelog; a release moves the `version-16` branch, which is the branch users install. `nightly.yml` runs the integration suites and the screenshot capture every night.

## How it works

### Modules

| Path                                                  | What it holds                                                                                                      |
| ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| `jwt_bearer_auth/auth.py`                             | The `JWTAuth` class (token reading, validation, user lookup, registration, URLs), the hooks and the logout methods |
| `jwt_bearer_auth/jwt_auth/hooks/contact.py`           | The Contact `on_update` handler                                                                                    |
| `jwt_bearer_auth/jwt_auth/doctype/jwt_auth_settings/` | The **JWT Auth Settings** single doctype                                                                           |
| `jwt_bearer_auth/public/js/app.bundle.ts`             | The desk bundle that replaces the desk's logout                                                                    |
| `jwt_bearer_auth/hooks.py`                            | The hooks listed in [Configuration](configuration.md#hooks)                                                        |

### A request with a token

1. Frappe's `validate_auth` runs its own API key and OAuth checks, then every `auth_hooks` entry. The app's `validate_auth` builds a `SessionJWTAuth`, which keeps one `JWTAuth` on `frappe.local` for the rest of the request. `JWTAuth` reads **JWT Auth Settings** with `frappe.get_cached_doc`.
2. `can_auth` stops unless the session is a guest, **Enable JWT User Auth** is on, no logout is under way (`frappe.flags.jwt_logout_redirect`) and a token is found in the cookie, then the header, named in **JWT Cookie/Header**.
3. `is_valid_token` reads the audience from **JWT Private Secret** with `get_password`, fetches the JWKS with `requests` (10-second timeout), turns each key into an RSA public key with PyJWT's `RSAAlgorithm.from_jwk`, and calls `jwt.decode` with each one, `algorithms=["RS256"]` and that audience, until one succeeds. The decoded claims are kept on the object.
4. `auth` takes the `email` claim and calls `frappe.local.login_manager.login_as` for an existing User with that name, then for the user of the first Contact whose Contact Email matches, and otherwise, with registration on, calls `register_user` first.
5. `register_user` inserts the User with `ignore_permissions`, links the Contact, and sets `frappe.local.flags.commit` so that Frappe commits at the end of the request, which it would otherwise roll back for a GET. When the user must complete their profile, the target `/update-profile/<EMAIL>/edit` goes into the session data as `jwt_auth_redirect`, and the requested path into the cache as `jwt_original_location_<EMAIL>`.

### After the request

`handle_redirects`, the `after_request` hook, looks at the response in this order:

1. A guest response with `frappe.flags.jwt_logout_redirect` set becomes a 302 to that address: this is the logout redirect.
2. A `jwt_auth_redirect` in the session data becomes a 302: this is the profile redirect after registration.
3. A guest on `/login`, with **Enable JWT User Auth** and **Override Frappe Login** on, is sent to the URL built by `get_login_url` from the request's `redirect-to` argument.
4. A request for `/me` with a cached `jwt_original_location_<USER>` is sent there, and the cache entry is deleted.

### Logout

`on_logout`, the hook Frappe calls from `LoginManager.logout`, stores `get_logout_url()` in `frappe.flags.jwt_logout_redirect`. `web_logout` and `jwt_logout` call `logout` themselves and also answer with an address: a Frappe redirect response for `web_logout`, and `{"redirect_url": ...}` for `jwt_logout`, which `app.bundle.ts` follows after replacing `frappe.Application.prototype.logout`. The prototype is replaced when the bundle loads, before Frappe creates `frappe.app`, so no polling is needed. Because `logout` runs the `on_logout` hook in the same request, `jwt_logout_redirect` is set for both methods, and `handle_redirects` turns their response into a 302 to the **Logout URL**: the `web_logout` redirect is replaced by that address, and the JSON answer of `jwt_logout` never reaches `app.bundle.ts`, whose XHR follows the 302 instead.

### Contact sync

The Contact `on_update` handler renames a `[Change Me]` Contact with `frappe.enqueue("frappe.model.rename_doc.rename_doc", ...)`, using `append_number_if_name_exists` for a free name, and copies the changed name, phone, mobile and gender fields to the linked User with `ignore_permissions`.

### The rename shim

The package was renamed from `jwt_auth` to `jwt_bearer_auth`. The block between the `frappe-nix:rename-shim-begin` and `frappe-nix:rename-shim-end` markers in `hooks.py` maps the three old whitelisted method paths to the new ones through `override_whitelisted_methods`. It is written by frappe-nix's rename tool: do not edit it by hand.
