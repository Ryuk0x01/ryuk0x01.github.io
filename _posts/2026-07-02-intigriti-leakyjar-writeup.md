---
title: "LeakyJar Write-up | Top 3 Intigriti Challenge Write-up 🏆"
date: 2026-07-02
categories: [Write-ups, Intigriti]
tags: [intigriti, web-security, ctf, write-up, csrf]

image:
  path: /assets/img/posts/leakyjar/winner.png
  alt: "LeakyJar Challenge - Top 3 Intigriti Write-up"
---


Intigriti June Bonus Challenge 2026 — Leaky Jar Write-up
========================================================


Introduction
------------

The objective of this challenge was to retrieve the administrator’s secret recipe containing the flag.

At first glance, the application looked like a simple recipe management platform where users could create private recipes, share them with other users, and report recipes to an administrator through a review bot.

Rather than searching immediately for payloads, I started by understanding the application’s functionality and identifying every feature that could modify user data or interact with the administrator.

This write-up explains the complete reconnaissance process, the reasoning that led to identifying the vulnerable endpoint, and how the CSRF vulnerability was leveraged to obtain the administrator’s private vault.

Reconnaissance
--------------

I started by exploring the application’s functionality to understand the available features before attempting any exploitation.

The homepage revealed the main areas of the application:

*   **Recipes** — Publicly available recipes.
*   **Bakers** — List of registered users.
*   **My Recipe Box** — A private vault where users can store personal recipes.
*   **Report a Recipe** — A page allowing users to submit URLs for review by the administrator bot.

At this stage, nothing appeared obviously vulnerable. Therefore, I decided to inspect each feature individually, focusing on functionality capable of modifying user data or interacting with privileged users.

![Figure 1— Homepage showing the main application features.](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*SncFnMXoOWJCuny2meycNw.png)


Exploring the Application
-------------------------

I continued exploring each section of the application to identify features that could potentially expose sensitive functionality.

The **Recipes** page only displayed public content and did not appear to expose any attack surface.

The **Bakers** page listed registered users, which later became useful because the `/share` functionality requires a valid username.

The most interesting page, however, was **My Recipe Box**. This page allows authenticated users to:

*   Store private recipes.
*   Share their entire recipe vault with another user.

Since both actions modify application state, this page became the primary focus for further analysis.

![Figure 2— The Recipe Box page exposing functionality for storing and sharing private recipes.](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*WwgCXOvnWjm2pO237ForXg.png)

Inspecting the Share Functionality
----------------------------------

Since the **Share my Recipe Box** feature performs a state-changing action, I decided to inspect how it was implemented.

The page contains a simple HTML form that submits a `POST` request to the `/share` endpoint. The only user-controlled parameter is the target username.

While reviewing the form, I noticed that it did not contain any hidden fields commonly used for request validation, such as a CSRF token. At this point, I did not immediately conclude that the application was vulnerable, but the absence of an anti-CSRF mechanism made this endpoint particularly interesting for further investigation.

![Figure 3 — HTML form used to share the recipe box with another user.](https://miro.medium.com/v2/resize:fit:1380/format:webp/1*ZqQ85HtMqO53zdNHxrpHUg.png)

Identifying the Attack Surface
------------------------------

At this stage, I shifted my focus from exploring the application’s features to identifying endpoints capable of performing sensitive actions.

The `/share` endpoint immediately stood out for several reasons:

*   It uses a `POST` request, indicating that it modifies application state.
*   The action grants another user access to the owner’s private recipe vault.
*   The HTML form only requires a single parameter (`username`).
*   No anti-CSRF token or other request validation mechanism was visible in the page source.

Although these observations alone were not sufficient to confirm a vulnerability, they suggested that the endpoint was a strong candidate for Cross-Site Request Forgery (CSRF) testing.

Another important clue came from the challenge itself: users could submit external URLs through the **Report a Recipe** feature, and those URLs would later be visited by an authenticated administrator bot.

Combining these observations led to the following hypothesis:

> _If the administrator bot could be tricked into visiting an attacker-controlled webpage, and if the_ `_/share_` _endpoint accepted cross-site POST requests without CSRF protection, the administrator could unknowingly share their private recipe vault with an attacker-controlled account._

This hypothesis became the basis for the next stage of the investigation.

![Figure 4— The /share request only requires the target username and contains no anti-CSRF token or other request validation parameter.](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*wLTPWQHM3vSx_Ueg37IJ2A.png)

Developing the Exploit
----------------------

After confirming that the `/share` endpoint accepted a simple authenticated `POST` request without any visible CSRF protection, I attempted to reproduce the same request from an external webpage.

The goal was straightforward: create a page that automatically submits a form to the `/share` endpoint while an authenticated administrator is viewing it.

If successful, the administrator’s browser would include its authenticated session cookies, causing the application to process the request as if it had been intentionally initiated by the administrator.

![Figure 5 — Auto-submitting HTML page used to trigger a cross-site POST request to the /share endpoint.](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*AnVa7q1u0gi5A9tnX1edbg.png)

Delivering the Payload
----------------------

Once the CSRF page was ready, the next step was to deliver it to the administrator.

The challenge provides a **Report a Recipe** feature, allowing participants to submit a URL that is later visited by an authenticated administrator bot.

I hosted the HTML page on a publicly accessible server and submitted its URL through the report form. Since the page automatically submitted the form on load, no additional interaction from the administrator was required.

![Figure 6 — Submitting the attacker-controlled page to the administrator bot.](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*Ze2BcqycMjJH5dhHTjHVCw.png)

Exploitation
------------

When the administrator bot visited the submitted URL, the browser automatically rendered the attacker’s HTML page.

The embedded JavaScript immediately submitted the hidden form to the `/share` endpoint.

Because the administrator was already authenticated, the browser automatically included the administrator’s session cookies with the request.

As a result, the application processed the request as a legitimate action performed by the administrator and shared the administrator’s private recipe vault with my account.

![Figure 7 — The administrator’s recipe vault is now shared with the attacker’s account.](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*IEMovOvc0USlh1z86R6XVQ.png)

Retrieving the Flag
-------------------

After the administrator’s recipe box was successfully shared with my account, I navigated back to the **My Recipe Box** page.

The administrator’s private recipes were now accessible under the shared recipes section. Among them was the **Master Baker’s Secret Recipe**, which contained the challenge flag.

```
INTIGRITI{019ef404-1e44-7748-bdcf-ca7b12dbfee0}
```
![Figure 8 — The administrator’s shared recipe box revealing the challenge flag.](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*Qjc7fcOgOLjst47jmPK74g.png)

Root Cause
----------

The vulnerability exists because the `/share` endpoint performs a sensitive state-changing action without implementing any protection against Cross-Site Request Forgery (CSRF).

The endpoint accepts authenticated `POST` requests based solely on the user's session cookies and does not verify whether the request originated from the legitimate application.

As a result, an attacker can craft an external webpage that silently submits a request on behalf of an authenticated user, causing unintended actions to be executed.

Mitigation
----------

The vulnerability can be mitigated by implementing standard CSRF protections, including:

*   Synchronizer CSRF tokens.
*   Origin and Referer header validation.
*   Appropriate SameSite cookie attributes.
*   Additional confirmation for sensitive actions such as sharing an entire recipe vault.

Conclusion
----------

This challenge highlights how a seemingly simple feature can become security-critical when combined with an automated administrator bot.

Although the `/share` endpoint appeared straightforward, the absence of CSRF protection allowed an attacker to abuse the administrator's authenticated session and gain unauthorized access to sensitive data.

By combining careful reconnaissance with an understanding of browser behavior and CSRF mechanics, it was possible to identify the intended attack path and successfully retrieve the flag.