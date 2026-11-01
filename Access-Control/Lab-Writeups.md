# Access Control Lab Writeups

This file has my short writeups for the 13 PortSwigger Web Security Academy Access Control labs. Each entry has the goal, what I did, and the key takeaway.

All screenshots are in one PDF: [screenshots/Access-Control-screenshots.pdf](screenshots/Access-Control-screenshots.pdf)

---

## Lab 1: Unprotected Admin Functionality

**Goal:** Find the hidden admin panel and delete the user `carlos`.

**What I did:**
- Found the admin panel at `/admin`, which was not linked in the normal UI.
- Opened the admin panel and deleted `carlos`.

**Takeaway:** Hiding a URL is not access control. The server must check the user's role on every admin page.

---

## Lab 2: Unprotected Admin Functionality with Unpredictable URL

**Goal:** Find the admin panel, which has a random-looking URL, and delete `carlos`.

**What I did:**
- Found the random admin URL in the page source.
- Opened the admin panel and deleted `carlos`.

**Takeaway:** A random URL is not security. If the URL is sent to the browser, anyone can find it.

---

## Lab 3: User Role Controlled by Request Parameter

**Goal:** Become an admin and delete `carlos`.

**What I did:**
- Logged in as `wiener`. The app stored the role in a cookie called `Admin`, set to `false`.
- Changed the cookie to `Admin=true` in Burp Repeater, and the server returned the admin page.
- Deleted `carlos` from the admin panel.

**Takeaway:** The role must be stored on the server. A cookie value can be changed by the user.

![Lab 3 - Admin cookie](screenshots/Access-Control-screenshots.pdf)

---

## Lab 4: User Role Can Be Modified in User Profile

**Goal:** Change my own role to admin through the profile update.

**What I did:**
- Captured the email change request `POST /my-account/change-email` in Burp. The JSON body had only the `email` field.
- Added `"roleid": 2` to the JSON body and sent it.
- The response showed `"roleid": 2`, so the role was changed. Then I opened the admin panel and deleted `carlos`.

**Takeaway:** This is mass assignment. The server must accept only the fields the user is allowed to change.

---

## Lab 5: User ID Controlled by Request Parameter

**Goal:** Read the account of `carlos` and get his API key.

**What I did:**
- Logged in as `wiener` and saw the URL `/my-account?id=wiener`.
- Changed `id` to `carlos` in Repeater and got his account page, including his API key.

**Takeaway:** This is an IDOR (Insecure Direct Object Reference). The server must check that the logged-in user is allowed to access the requested ID.

---

## Lab 6: User ID Controlled by Request Parameter with Unpredictable User IDs

**Goal:** Read the account of `carlos`, but his ID is a random GUID.

**What I did:**
- Could not guess the GUID.
- Found `carlos`'s GUID in the author link of a blog post in the page HTML.
- Used the GUID in `/my-account?id=` and got his account, including his API key.

**Takeaway:** Random IDs make guessing harder, but they are not a replacement for an access check. If an ID is shown elsewhere in the app, an attacker can collect it.

---

## Lab 7: User ID Controlled by Request Parameter with Data Leakage in Redirect

**Goal:** Get the API key of `carlos` from a redirect response.

**What I did:**
- Sent `GET /my-account?id=carlos` in Burp Repeater.
- The response was a redirect to the login page, but the response body still contained `carlos`'s account data.
- Copied the API key from the response body and submitted it.

**Takeaway:** A redirect does not stop the data from being sent. The server must stop processing as soon as the access check fails, and must not return sensitive data.

---

## Lab 8: User ID Controlled by Request Parameter with Password Disclosure

**Goal:** Get the administrator's password.

**What I did:**
- Viewed my own account page and found that the password was present in the page HTML.
- Changed `id` to `administrator` and got the admin's password from the response.
- Logged in as `administrator`, opened the admin panel, and deleted `carlos`.

**Takeaway:** Passwords must never be sent to the client, even in hidden fields or page source.

---

## Lab 9: Insecure Direct Object References (IDOR)

**Goal:** Get the password of `carlos` by accessing another user's data.

**What I did:**
- Found the live chat feature, and the chat transcript was downloadable from `/download-transcript/2.txt`.
- Changed the file number in the request to `/download-transcript/1.txt`, and the server returned another user's chat transcript.
- The transcript showed the password of `carlos`, which I used to log in as `carlos`.

**Takeaway:** Before returning any file or object, the server must check that the logged-in user is allowed to access it. Sequential file names make guessing easy.

---

## Lab 10: Multi-Step Process with No Access Control on One Step

**Goal:** Promote `wiener` to admin, as a normal user.

**What I did:**
- Studied the admin flow: load user details, submit changes, and confirm.
- Found that the confirm step did not check access, while the earlier steps did.
- Sent the confirm request directly to `/admin-roles` with `action=upgrade&confirmed=true&username=wiener`, and the server returned 302. `wiener` became admin.

**Takeaway:** Every step of a sensitive process needs the same access control, not just some of them.

---

## Lab 11: Method-Based Access Control Can Be Circumvented

**Goal:** Promote `wiener` to admin, even though the POST method is blocked for non-admin users.

**What I did:**
- Logged in as admin, promoted `carlos`, and sent the request to Burp Repeater.
- Logged in as `wiener` in a private window and used `wiener`'s session cookie. The response was "Unauthorized" for POST.
- Changed the method to `POSTX`, which is a fake method. The response changed to "Missing parameter", which showed the access check was skipped for this method.
- Changed the method to `GET`, moved the parameters to the URL, and set `username=wiener`. The response was 302, and `wiener` became admin.

**Takeaway:** The access check must apply to every HTTP method. Use default deny, so anything not explicitly allowed is blocked.

---

## Lab 12: URL-Based Access Control Can Be Circumvented

**Goal:** Delete `carlos` through the admin path, even though the front-end blocks direct access to `/admin`.

**What I did:**
- A direct request to `/admin` was blocked by the front-end.
- Sent a request to `/`, which the front-end allowed, and added the header `X-Original-URL: /admin/delete`.
- The request with `?username=carlos` in the request line returned 302, and `carlos` was deleted.

**Takeaway:** The access check and the application must use the same URL. Non-standard headers like `X-Original-URL` must be ignored on the server.

---

## Lab 13: Referer-Based Access Control

**Goal:** Upgrade `wiener` to admin, using the admin sub-page, without admin access.

**What I did:**
- The main `/admin` page had a strong check, but the sub-page `/admin-roles` only checked the `Referer` header.
- Sent `GET /admin-roles?username=wiener&action=upgrade` with the header `Referer: https://<lab-host>/admin`. The response was 302, and `wiener` became admin.

**Takeaway:** The `Referer` header is controlled by the client, so it must never be used for access control.

---

## Summary

All 13 labs were solved. The main patterns were:

- The server trusted data from the client: cookies, JSON fields, headers, IDs, methods, and the Referer header.
- Access checks were missing on some pages, methods, or steps.
- The app relied on hiding URLs instead of checking permissions.
