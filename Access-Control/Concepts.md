# Access Control Concepts

This file explains the access control concepts I learned through the PortSwigger Web Security Academy labs, in my own words.

---

## 1. Unprotected Admin Functionality

**What it is:** The admin page is hidden from the menu, but it still exists and can be reached by anyone who knows the URL.

**How it is exploited:** Hidden paths can be found in `robots.txt`, page source, or by guessing common names like `/admin`. Once found, the admin functions can be used directly.

**Prevention:** Check the user's role on the server for every admin page. Hiding a URL is not security.

---

## 2. Role Controlled by a Cookie

**What it is:** The app stores the user's role in a cookie, such as `Admin=false`.

**How it is exploited:** Cookies are controlled by the browser, so the user can change `Admin=false` to `Admin=true` and get admin access.

**Prevention:** Store the role on the server side, in the session or the database. Never trust a role value sent by the client.

---

## 3. User Role Can Be Modified in User Profile (Mass Assignment)

**What it is:** When the user updates their profile, the server takes all the fields in the request and saves them to the user object.

**How it is exploited:** Extra fields like `"roleid": 2` are added to the JSON body of the email change request. The server saves them too, and the user becomes an admin.

**Prevention:** Use an allowlist so the server only accepts fields the user is allowed to change, such as `email`. Never accept fields like `role`, `isAdmin`, or `price` from the client.

---

## 4. URL-Based Access Control Can Be Circumvented (X-Original-URL)

**What it is:** A front-end guard checks the URL in the request line. The back-end application may also accept a non-standard header like `X-Original-URL` and use it as the real path.

**How it is exploited:** Send a request to an allowed URL like `/`, and add `X-Original-URL: /admin/deleteUser`. The guard allows the request, but the back-end runs the admin action.

**Prevention:** Do the access control in one place. Ignore or disable non-standard override headers like `X-Original-URL` and `X-Rewrite-URL`.

---

## 5. Method-Based Access Control

**What it is:** The access check is applied only to certain HTTP methods, such as `POST`.

**How it is exploited:** A different method, like `GET` or a fake method like `POSTX`, skips the check. The server then processes the request and performs the action.

**Prevention:** Check authorization for every HTTP method. Use default deny: if something is not explicitly allowed, block it.

---

## 6. User ID Controlled by Request Parameter (IDOR)

**What it is:** The app uses an ID in the request, like `?id=wiener`, to decide which account to show. It does not check whether the logged-in user owns that ID.

**How it is exploited:** Change the ID to another user's ID, and you see their account. With unpredictable IDs (GUIDs), the attacker first needs to find the ID somewhere else in the app, such as a blog post author link.

**Prevention:** The server must check that the logged-in user is allowed to access the requested ID. Random IDs help, but they are not a replacement for access checks.

---

## 7. User ID Controlled by Request Parameter with Data Leakage in Redirect

**What it is:** The app detects that the user is not allowed to access the page, and sends a redirect. But the redirect response still contains the sensitive data.

**How it is exploited:** The browser follows the redirect, so the user sees nothing. In Burp, the response body still shows the other user's data, such as an API key.

**Prevention:** Return nothing sensitive in a redirect response. Stop processing the request as soon as the access check fails.

---

## 8. User ID Controlled by Request Parameter with Password Disclosure

**What it is:** The account page shows the user's current password in a field, for example in an input's `value` attribute.

**How it is exploited:** Change the ID to the admin's ID, and the response shows the admin's password.

**Prevention:** Never send passwords to the client, even in hidden fields or page source. Use the access check described above.

---

## 9. Multi-Step Process with No Access Control on One Step

**What it is:** An important action happens in several steps, such as load details, submit changes, and confirm. Access control is applied to some steps but not all.

**How it is exploited:** Skip the first steps and send the request for the last step directly. The server accepts it because it assumes the earlier steps were done.

**Prevention:** Apply the same access control to every step. Also check on the server that the earlier steps were completed correctly.

---

## 10. Referer-Based Access Control

**What it is:** The app checks the `Referer` header, which the browser sends to show which page made the request, to decide if access is allowed.

**How it is exploited:** The `Referer` header is controlled by the client. An attacker can add `Referer: .../admin` to a direct request for `/admin/deleteUser`, and it is allowed.

**Prevention:** Never use the `Referer` header for access control. Check the user's role on the server for every sensitive page.

---

## 11. Location-Based Access Control

**What it is:** The app allows or blocks access based on the user's location, for example for banking or streaming services.

**How it is exploited:** Location checks are often done on the client side, so they can be bypassed with a VPN, a web proxy, or by changing the client-side geolocation.

**Prevention:** Make the location decision on the server. Do not rely only on the browser or on the IP address.

---

## 12. Unprotected Admin Functionality with Unpredictable URL

**What it is:** The admin page has a random-looking URL instead of `/admin`, so guessing it is hard. But the URL is still sent to the browser.

**How it is exploited:** The admin URL is found in the page source or in a JavaScript file. Once the URL is known, the admin functions can be used without any role check.

**Prevention:** A random URL is not security. Check the user's role on the server for every admin page and request.

---

## 13. Insecure Direct Object References (IDOR)

**What it is:** The app lets users access a file or object directly using its ID or file name, such as `/download-transcript/2.txt`. It does not check if the logged-in user owns that file.

**How it is exploited:** Change the number or name in the URL to another value, such as `/download-transcript/1.txt`. The server returns another user's file, which may contain sensitive data like a password.

**Prevention:** Before returning any object or file, check on the server that the logged-in user is allowed to access it. Do not rely on hard-to-guess or sequential names alone.

---

## Summary

Most access control bugs come from one of these mistakes:

- The server trusts data that the client controls (cookies, headers, parameters, IDs, methods).
- Access checks are missing on some pages, methods, or steps.
- The app relies on hiding things instead of checking permissions.

**Golden rule:** The server must check every request, and nothing should be allowed by default.
