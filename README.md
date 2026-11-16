# CSRF-XSS-leads-to-User-data-exfil

# 🎯 CSRF Plants the Draft, Preview Runs the Code

**How a `text/plain` erotica-draft API + unsanitized Preview turned a third-party page visit into same-origin JS (username + page token)**

*by [cyrusthedev](https://github.com/cyrusthedev) · educational writeup · researcher handle: `cyrus__`*

> ⚠️ **Disclosure note.** This post is for **education**. It describes an authorized Intigriti finding on REDACTED (Aylo BBP). I used **my own researcher accounts** for every demonstration. Do not point this at real users. I am **not** publishing a copy-paste CSRF/XSS plant — defenders get the sink and the control gaps; hunters already know the patterns.

---

## TL;DR

Erotica drafts on REDACTED are created with:

```text
POST /erotica/drafts
Content-Type: text/plain
Cookie: il=<session>   // SameSite=None
body ≈ JSON  (no anti-CSRF token)
```

A logged-in user who opens an attacker page can have a draft **created in their account** without clicking “save” on REDACTED.

Draft `content` stores HTML. The editor **Preview** renders it with an unsafe sink (`dangerouslySetInnerHTML`). After a WAF-dodging event-handler lands in storage, Preview runs attacker JS in the **`www.redacted.com` origin** and can read:

- `page_params.liu_username`
- `page_params.token` (page CSRF / signed token for authenticated XHRs)

A second CSRF-able call — `POST /erotica/drafts/submit` — can push a complete draft into **`pending_review`** (moderation queue). Public story URLs stayed **404** until approval in my tests.

**Class:** CSRF (CWE-352) chained with stored XSS in Preview (CWE-79)  
**Impact:** unauthorized content + same-origin JS on Preview + username/token exfil (researcher accounts)  
**Not:** HttpOnly session-cookie theft, password ATO, or confirmed public/moderator XSS

**Severity I submitted:** Medium (CVSS 3.1 `AV:N/AC:L/PR:N/UI:R/S:U/C:L/I:L/A:N` ≈ **5.4**)  
**Honest floor if triage disagrees:** Low (if they treat Preview as “self-XSS” and ignore the CSRF plant)

---

## 🎯 Why I looked here

Story editors are HTML playgrounds. Preview is where “safe render” is often postponed.

I also knew several REDACTED state-changing endpoints already enforce a CSRF token — password change, deletes, some comments. Erotica drafts looked younger / thinner.

Two boring questions:

1. Does `POST /erotica/drafts` require the same CSRF token as password-change?  
2. What does **Preview** do with stored `content` — sanitize, or trust it as HTML?

---

## 🧩 Mapping the feature

| Surface | Role |
|---------|------|
| `POST /erotica/drafts` | Create draft (title, tags, summary, HTML `content`) |
| `POST /erotica/drafts/submit` | Move draft → `pending_review` |
| `/erotica/draft/edit/?draftId=` | Editor UI |
| **Preview** button | Client-side render of stored `content` |
| Cookie `il` | Session (`HttpOnly`, `SameSite=None` in my captures) |
| `page_params.token` | Page token used by authenticated XHRs (JS-readable) |

Intended happy path:

1. User writes a story in the editor  
2. Saves a draft  
3. Clicks Preview to proofread  
4. Submits for review when ready  

The bug is that **step 2 can be done by any site the victim visits**, and **step 3 executes whatever HTML was stored**.

---

## 🧪 The bug in one diagram

```text
[Victim — logged in on www.redacted.com]
   Cookie: il (HttpOnly, SameSite=None)
   │
   │  visits attacker page
   ▼
[Attacker origin]
   auto POST /erotica/drafts
   Content-Type: text/plain   ← simple request, cookie sent
   body: JSON with HTML content
   (no CSRF token checked)
   │
   ▼
[Victim’s draft list]
   new draft appears under their identity
   │
   │  victim opens draft → clicks Preview
   ▼
[Preview sink]
   dangerouslySetInnerHTML(content)   ❌ event handlers live
   │
   ▼
[JS in redacted.com origin]
   reads page_params.liu_username / page_params.token
   beacons to attacker collector
```

Optional integrity kick:

```text
POST /erotica/drafts/submit  (also text/plain + cookie, no CSRF token)
   → status pending_review  (public URL 404 until approval)
```

---

## 🔬 What I confirmed (researcher accounts only)

| Role | Meaning |
|------|---------|
| Account A | Victim / author (own test user) |
| Attacker page | Third-party origin hosting the CSRF form |
| Collector | Temporary HTTPS listener (not left live) |

### 1) CSRF create — no XSS required

Cross-site `POST /erotica/drafts` with `Content-Type: text/plain` and a JSON-shaped body created a new draft under the logged-in researcher account (**example plant id 4991** in my notes).

No anti-CSRF token. Session cookie rode along because of `SameSite=None`.

`text/plain` matters: it keeps the request a “simple” cross-site form post (no CORS preflight), while the server still parses JSON-looking bodies.

### 2) Preview XSS — stored HTML execution

Drafts whose `content` included an event-handler image executed in Preview (**example draft 4801** — native `alert` with `1`).

Edge WAF blocked some literal `onerror` / cookie-stealer strings (**403**). Encoding the handler name in JSON (`\u006f…`) and using string-split `Function(...)` still **stored and ran** after decode. WAF is a seatbelt, not the chassis.

### 3) Exfil from Preview

On Preview of planted drafts, the collector received (Referer `https://www.redacted.com/`):

| Beacon | Meaning |
|--------|---------|
| `/l?v=csrfimg` | CSRF plant executed / img fired |
| `/l?u=<username>` | `page_params.liu_username` |
| `/l?t=<token>` | `page_params.token` |

### 4) Submit to moderation (CSRF)

`POST /erotica/drafts/submit` with `{"draftId":…}` and `text/plain` moved a complete draft to **`pending_review`** (example story/chapter ids in research notes). Public `/erotica/{story}/{chapter}/` returned **404** until approval — so I do **not** claim drive-by XSS for anonymous readers.

---

## 🧠 Root cause (plain English)

Three controls failed together:

1. **CSRF gap** — state-changing erotica APIs trusted the session cookie alone and accepted `text/plain` JSON. Sister endpoints already required a token; these did not.  
2. **XSS gap** — Preview treated user `content` as HTML without stripping event handlers.  
3. **WAF-only thinking** — Unicode escapes and string-concat `Function` bypassed the edge filter that caught the literal keywords.

`il` being **HttpOnly** correctly blocked `document.cookie` session theft. That is why this is **not** a one-click cookie ATO. The page token in `page_params` is a different secret — readable to same-origin JS by design of the SPA.

---

## 💥 Impact

| Confirmed | Not claimed |
|-----------|-------------|
| Create drafts as the victim (integrity / reputation) | Steal HttpOnly `il` |
| Push attacker HTML into `pending_review` | Password change / full ATO from token alone (`current_password` still required) |
| Preview JS as the victim on redacted.com | Public visitor XSS while pending |
| Exfil username + page token | Confirmed moderator XSS (no third-party hits observed) |

**Realistic story:** phishing link → silent draft plant → user later hits Preview on “their” story → username + page token leave the browser. Token may unlock other same-session XHRs that trust it; password change still needs the current password. Integrity abuse of drafts/submit is already enough for a defensible **Medium**.

---

## 🛠️ Remediation (what I’d tell the team)

1. Require the **same anti-CSRF token** on `POST /erotica/drafts` and `POST /erotica/drafts/submit` as on password / delete flows.  
2. Reject **`Content-Type: text/plain`** for these JSON APIs; accept `application/json` (with token) only.  
3. Validate **`Origin` / `Referer`** as defense-in-depth.  
4. Revisit **`SameSite=None`** on `il` if cross-site embeds are not required.  
5. **Sanitize** erotica HTML on save **and** in Preview/publish — strip event handlers and `javascript:` URLs; do not feed raw user HTML into `dangerouslySetInnerHTML`.  
6. Treat WAF as secondary; Unicode-escaped handlers must not be the only gate.

---

## 🎓 Hunting lesson I’m keeping

> If sibling write endpoints already require a CSRF token, the one that doesn’t is your lead.  
> If an editor has Preview, assume stored HTML until you see a sanitizer.  
> CSRF plants remove the “self-XSS” excuse — the victim never typed the payload.

Also: prove **negative** tests (HttpOnly cookie, password change with stolen token only, public 404 while pending) so triage doesn’t invent claims you didn’t make.

---

## Timeline (personal)

- Mapped REDACTED erotica draft / Preview surfaces on an authorized Intigriti program  
- Confirmed CSRF create + Preview XSS + username/token exfil on researcher accounts  
- Confirmed submit → `pending_review` (public still 404)  
- Filed the program report (Medium / CVSS 5.4)  
- Published this **educational** version for other hunters

---

## Legal / ethics

- Authorized Intigriti / Aylo BBP testing only  
- Own researcher accounts only — no third-party customer sessions  
- Collector used for proof was temporary and taken down  
- Follow the program’s rules, rate limits, and disclosure policy  
- Rotate any page token that appeared in screenshots

---

## Thanks

To every hunter who diffs “which POSTs need a CSRF token” and then opens Preview anyway. That’s the whole finding.

— **cyrus__** / [cyrusthedev](https://github.com/cyrusthedev)

---

### Tags

`#bugbounty` `#csrf` `#storedxss` `#domxss` `#preview` `#samesite` `#intigriti` `#websecurity` `#writeup`
