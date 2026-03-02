# API Security Checklist

Securing an API is like running a high-end VIP lounge: you need a solid guest list, a bouncer who doesn’t take bribes, and a way to make sure no one is sneaking in through the kitchen.

---

## 1. The ID Check (Identity & Access)

Before you let anyone touch your data, you need to be 100% sure they are who they say they are.

* **Standardize Your Handshakes:** Use **OAuth2** or **OpenID Connect**. Avoid "Basic Auth" as it's trivial to decode.
* **Don’t "Roll Your Own":** Never invent your own encryption logic. Use battle-tested libraries like `bcrypt` for passwords and standard libraries for token generation.
* **MFA is Mandatory:** For sensitive actions (like changing passwords or financial transactions), require **Multi-Factor Authentication**.
* **The "3 Strikes" Rule:** Implement account lockouts or "jails."
* *Example:* If a login fails 5 times, lock the account for 15 minutes to stop brute-force bots.


* **IP VIP List:** For private or partner-only APIs, whitelist specific IP addresses to block the rest of the world entirely.

---

## 2. Token Integrity (JWT & OAuth)

If a token is a "visitor pass," you need to ensure it hasn't been forged, stolen, or altered.

* **Secret Strength:** Use a random, long string for your JWT secret. A weak secret can be cracked by scripts in seconds.
* **Dictate the Algorithm:** Don't let the client choose the encryption type in the header. Force the backend to use `HS256` or `RS256`.
* **Short Leash (TTL):** Set tokens to expire quickly.
* *Example:* A 15-minute access token is far safer than one that lasts a month.


* **White-Glove Redirects:** In OAuth, only allow redirects to URLs you have explicitly whitelisted to prevent token theft.

---

## 3. The Perimeter (Traffic & Connection)

Manage the "crowd" and keep the connection private.

* **Encrypted Tunnel:** Force **HTTPS** and use an **HSTS** header to prevent the connection from being stripped to plain text.
* **The "Machine Gun" Filter (Throttling):** Set a limit on requests.
* *Example:* Allow 100 requests per minute per IP. Anything more receives a `429 Too Many Requests`.


* **CORS Policy:** Don't use `Access-Control-Allow-Origin: *`. Explicitly list only the specific domains (like your frontend app) allowed to call your API.
* **Spike Arrest:** Use an **API Gateway** to prevent sudden traffic bursts from crashing your database.

---

## 4. Safe Intake (Input Hygiene)

Never trust what a user sends you. They might be trying to "inject" malicious code.

* **Use Proper Verbs:** Stick to REST standards. Respond with `405 Method Not Allowed` if someone tries to `DELETE` using a `GET` request.

| Action | Method | Wrong Way |
| --- | --- | --- |
| Read | `GET` | `GET /deleteUser?id=1` |
| Create | `POST` | `GET /createUser` |
| Update | `PUT/PATCH` | `POST /update` |
| Delete | `DELETE` | `GET /remove` |

* **Sanitize Everything:** Scrub inputs to prevent **SQL Injection** or **XSS**. Treat all user input as plain text, never executable code.
* **Maximum Request Size:** Limit payload sizes. Don't allow a 50MB file upload for a request that only needs a small JSON.
* **No Secrets in URLs:** Never put API keys or passwords in the address bar. Use the `Authorization` header instead.

---

## 5. Smart Processing (Logic & Data)

How you handle data internally determines if you're vulnerable to "guessing" attacks.

* **Hide the Count:** Avoid auto-incrementing IDs like `user/123`. Use **UUIDs** (e.g., `user/f47ac10b...`) so attackers can't guess the next ID.
* **Kill the "XML Bomb":** If you use XML, disable "External Entities" (XXE) to prevent tiny files from expanding into gigabytes of data in memory.
* **Background Workers:** For heavy tasks, use a **Queue**. Return a `202 Accepted` immediately so the user isn't left hanging while you process data.
* **Kill Debug Mode:** Ensure `DEBUG = False` in production to prevent leaking your folder structure or stack traces in error messages.

---

## 6. The Clean Exit (Output & Headers)

What you tell the user (and what you hide) is just as important as the data itself.

* **Generic Error Messages:** Don't tell the user *why* a login failed.
* *Good:* "Invalid credentials."
* *Bad:* "User not found" (This allows hackers to verify if an email exists in your system).


* **Mask the Data:** Never return full PII (Personally Identifiable Information).
* *Example:* Return `{"credit_card": "************1111"}` instead of the full number.


* **Shave Off Fingerprints:** Remove headers like `X-Powered-By` or `Server` that reveal your tech stack.
* **Security Headers:** Always include these:
1. `X-Content-Type-Options: nosniff` (Stop browser MIME-sniffing).
2. `X-Frame-Options: deny` (Prevent clickjacking).
3. `Content-Security-Policy: default-src 'none'` (Block unauthorized scripts).



---

## 7. The Paper Trail (Logging & Monitoring)

You can't stop what you can't see.

* **Audit Logs:** Record who accessed what and when. If a breach occurs, you need to know which records were touched.
* **Centralized Logging:** Send logs to a separate, secure server (like ELK or Splunk) so attackers can't delete the evidence.
* **Integrity Monitoring:** Set alerts for high volumes of `401 Unauthorized` or `403 Forbidden` errors; these are usually signs of a brute-force attempt.

---

## 8. The Safety Net (CI/CD)

Security is a continuous process, not a one-time event.

* **Secrets Management:** Never hardcode API keys or database passwords. Use a **Secret Manager** (like AWS Secrets Manager or HashiCorp Vault).
* **Four Eyes Principle:** No code goes to production without a **Code Review** from another person.
* **Dependency Scanning:** Use automated tools (like `Snyk`) to check for vulnerabilities in your third-party libraries before you deploy.
* **The "Undo" Button:** Always have a **Rollback Plan** to revert to the last safe version in seconds if a security hole is discovered.

---

HTTP status codes tell the client—at a glance—whether the request was a success, a confusing mess, or if the server just had a total meltdown.

---

## The Cheat Sheet: 5 Categories of "Feelings"

| Range | Category | Summary in Plain English |
| --- | --- | --- |
| **1xx** | **Informational** | "Hang on a second, I’m still thinking." |
| **2xx** | **Success** | "Got it! Everything went exactly as planned." |
| **3xx** | **Redirection** | "The thing you want is over there now." |
| **4xx** | **Client Error** | "You did something wrong (bad URL, no password, etc.)." |
| **5xx** | **Server Error** | "It’s not you, it’s me. My circuits are fried." |

---

## 2xx: The "Victory" Codes

* **200 OK:** The standard "thumbs up." The request worked, and here is your data.
* **201 Created:** Perfect for `POST` requests. "I got your data and I just finished building the new resource."
* **204 No Content:** "I did what you asked, but there’s nothing to show you" (often used for successful `DELETE` requests).

---

## 3xx: The "Detective" Codes

* **301 Moved Permanently:** "This URL is retired. Go to this new one from now on."
* **304 Not Modified:** "Nothing has changed since you last checked. Use the version you already have in your cache to save bandwidth."

---

## 4xx: The "Your Fault" Codes

These are the most important for developers to handle gracefully.

* **400 Bad Request:** "I don't understand what you just sent. The syntax is a mess."
* **401 Unauthorized:** "I don't know who you are. Please log in."
* **403 Forbidden:** "I know who you are, but you aren't allowed to touch this specific button."
* **404 Not Found:** The classic. "I looked everywhere, but that resource doesn't exist."
* **429 Too Many Requests:** "You're clicking too fast! Slow down and wait a minute." (Crucial for Rate Limiting).

---

## 5xx: The "My Fault" Codes

If you see these, the backend developer has some debugging to do.

* **500 Internal Server Error:** The generic "something exploded" code. The server encountered an error it didn't know how to handle.
* **502 Bad Gateway:** One server acting as a proxy tried to talk to another server and got an invalid response.
* **503 Service Unavailable:** "I'm currently overloaded or down for maintenance. Come back later."

---

## The Easter Egg

* **418 I'm a Teapot:** This is a real (though jokingly RFC-standardized) code from an April Fools' Day prank. It signifies that the server refuses to brew coffee because it is, in fact, a teapot.

---

Using accurate HTTP status codes is your final security layer: it prevents information leakage by masking internal logic while feeding your monitoring systems the exact telemetry needed to spot and stop an attack in real-time.
