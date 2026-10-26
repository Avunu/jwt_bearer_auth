---
title: Limitations and troubleshooting
description: What JWT Bearer Authentication does not do, and how to find out why a token does not log anyone in.
nav_title: Troubleshooting
order: 8
updated: 2026-10-26
---

## Limitations

- **RS256 and the `email` claim only.** Tokens must be signed with RS256, and only the `email` claim is used to find the user. The issuer (`iss`) is not checked. Expiry is checked when the token carries an `exp` claim, as PyJWT does by default.
- **No key cache.** The app fetches the JWKS again for every token it checks, with a 10-second timeout, so each sign-in from a token makes one request to the provider. While the JWKS URL cannot be reached, nobody can sign in with a token.
- **One identity provider per site.** The settings hold a single cookie or header name, JWKS URL and audience.
- **No check that a request came through the provider.** Any client that holds a valid token for your audience can send it in the cookie or header. Keep the site reachable only through the provider.
- **Guests only.** The token is read only while a request is not logged in. A person who already has a Frappe session keeps it, even when the provider now sends a token for someone else, until they log out.
- **New users are Website Users.** Users created from a token have no roles. Give them roles and desk access by hand if they need it.
- **The first matching Contact.** When several Contacts share an email, the app uses the first one the database returns, both to find a linked user and to fill in a new one.
- **Back to the path only.** After a new user completes their profile, the app returns them to the path of the page they first asked for, without its query string.
- **The logout return address.** With **Redirect Param** set, the address added to the **Logout URL** is the logout request's own address, not encoded, and always after a `?`. See [Log out](logout.md#the-address-the-provider-returns-to).
- **The desk's logout is redirected in the background.** The desk's **Logout** calls `jwt_bearer_auth.auth.jwt_logout`, and the app's own logout hook turns that call's answer into a redirect to the **Logout URL**, which the browser follows inside the background call. See [Log out of the desk](logout.md#log-out-of-the-desk).
- **Logout always goes to the provider.** Turning **Enable JWT User Auth** off does not stop browser logouts from being redirected to the **Logout URL**. See [Log out](logout.md#with-token-login-turned-off).

## Troubleshooting

The app shows no error messages of its own. A token that fails any check is ignored, and the request goes on as a guest. It can write two warnings to the `jwt_auth` log, `logs/jwt_auth.log` in the bench directory:

| Warning                                                            | Cause and fix                                                                                                                        |
| ------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------ |
| `JWT Auth Settings has no JWT Private Secret; rejecting the token` | **JWT Private Secret** is empty, so every token is rejected. Enter your application's audience and save.                             |
| `could not load the JWKS from <URL>: <ERROR>`                      | The server could not fetch or read the **JWKS URL**: wrong address, no outbound access, a timeout, or a reply that is not a key set. |

> [!NOTE]
> Frappe's loggers keep warnings only when the `DEV_SERVER` environment variable is set, which `bench start` does. Under a production process manager they keep errors only, so these lines do not appear there. Check the settings by hand instead, as below.

### Nobody is logged in from the token

The person reaches the site but stays a guest, or sees Frappe's login page. Check, in this order:

1. **Enable JWT User Auth** is on.
2. The token arrives under the name in **JWT Cookie/Header**. For Cloudflare Access, the cookie is `CF_Authorization` and the header is `Cf-Access-Jwt-Assertion`. Names are matched exactly for cookies.
3. **JWT Private Secret** holds the audience the token was issued for. For Cloudflare Access, that is the Access application's AUD tag, not a key or a client secret.
4. The **JWKS URL** answers from the Frappe server. Fetch it from there:

   ```bash
   curl -sS https://<TEAM_NAME>.cloudflareaccess.com/cdn-cgi/access/certs
   ```

   It must return JSON with a `keys` list.

5. The token carries an `email` claim, and either a user with that email exists, a Contact with that email is linked to a user, or **Enable New User Registration** is on.
6. The token has not expired. A provider whose clock is far from the server's can produce tokens that are not yet valid or already expired.

### You are locked out of the password login

With **Override Frappe Login** on, `/login` sends every guest to the **Login URL**. If that address is wrong, or the provider does not let you in, turn the override off from the bench directory:

```bash
bench --site <SITE_NAME> execute frappe.db.set_single_value --args "['JWT Auth Settings', 'enable_login', 0]"
```

`/login` then shows Frappe's login page again. Fix the settings, then turn **Override Frappe Login** back on.

### The browser loops between the site and the provider

The provider lets the person in, the token does not log them in, and `/login` sends them back to the provider. The token is not being accepted: work through [Nobody is logged in from the token](#nobody-is-logged-in-from-the-token). While you do, turn **Override Frappe Login** off so that `/login` stops redirecting.

### "User ... is disabled. Please contact your System Manager."

The token is valid and names a user who is disabled in Frappe. Frappe refuses to start the session and shows this message. Tick **Enabled** on the user in **User** if they should have access.

### "Login not allowed at this time"

The token is valid, but the user has **Login After** or **Login Before** hours set and the request falls outside them. Frappe refuses to start the session. Clear or change those hours in **User** if the person should have access now.

### Logging out logs straight back in

The Frappe session ends, but the provider's does not, so its token logs the person in again on the next request. Check that **Logout URL** is the provider's logout address. For Cloudflare Access, it is `https://erp.example.com/cdn-cgi/access/logout` on your site's own domain. See [Log out](logout.md).

### A new user was not created

- **Enable New User Registration** is off. Turn it on, or create the user by hand.
- A Contact with that email is already linked to a user. The app logs in that user instead of creating one. See [Which user is logged in](sign-in.md#which-user-is-logged-in).

### A Contact is still called `[Change Me]`

The rename runs as a background job. Check that the bench's workers are running, and see the job in **RQ Job** if it failed. The Contact is renamed only once its first, middle and last names (or, with no name, its company name) make a full name other than `[Change Me]`.
