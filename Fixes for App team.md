# Registration API changes — action needed before release

**What changed:** the registration endpoints now require proof that an OTP was
verified. Without it they return **401** and no account is created.

**What you must do:** call `verify-otp`, keep the `verificationToken` it returns,
and send that token in the register call that follows.

**If you do nothing:** registration stops working in the app the moment this API
goes live. Please plan the change alongside the API release.

---

## 1. Why this changed

`register` and `registerusers` are open endpoints — no login, no session. They
had no way to tell a genuine post-OTP call from someone calling the URL
directly, so they assumed the OTP had happened.

We tested it. A single request, no OTP anywhere, a phone number we invented:

```
POST /api/Register/register
{ "name": "Test", "countryCode": "IND", "mobile": "9462727881" }

200 OK  →  userId 724, accessToken, refreshToken
DB      →  UserId=724 | 9462727881 | Activated=1
```

An activated account, plus working tokens, for a number we did not own. Anyone
could do this for any number — squatting on real customers' numbers, or
inflating signups.

A second problem on the SnapTrade path: when the number **already belonged to
someone**, the endpoint looked that user up and returned tokens for *their*
account. That is account takeover, not just fake signups.

---

## 2. The new flow

```
   ┌────────────────────────────────────────────────────────────┐
   │  STEP 1   POST /api/Register/send-otp                       │
   │           { "input": "9876543210" }                         │
   │           → OTP sent to the user                            │
   └───────────────────────────┬─────────────────────────────────┘
                               │
                               ▼
   ┌────────────────────────────────────────────────────────────┐
   │  STEP 2   POST /api/Register/verify-otp                     │
   │           { "input": "9876543210", "otp": "123456" }        │
   │                                                             │
   │           ★ NEW: response now contains                      │
   │             "verificationToken": "eyJhbGciOi..."            │
   │                                                             │
   │           KEEP THIS VALUE — you need it in step 3           │
   └───────────────────────────┬─────────────────────────────────┘
                               │  carry the token forward
                               ▼
   ┌────────────────────────────────────────────────────────────┐
   │  STEP 3   POST /api/Register/register                       │
   │           { ...user fields...,                              │
   │             "verificationToken": "eyJhbGciOi..." }  ★ NEW   │
   │                                                             │
   │           Server checks:                                    │
   │             1. is the token real and unexpired?             │
   │             2. does the number inside it match              │
   │                the number being registered?                 │
   │                                                             │
   │           both pass → account created + activated + tokens  │
   │           either fails → 401, nothing created               │
   └────────────────────────────────────────────────────────────┘
```

The order of calls has not changed. The only difference is that **step 3 now
proves step 2 happened**, instead of assuming it.

---

## 3. The token

- A JWT issued by `verify-otp`, valid for **10 minutes**
- It names the exact mobile or email that was verified
- Treat it as an opaque string — just store and resend it, do not parse it
- It is **not** an access token and cannot be used as one
- An access token cannot be used in its place either — the server rejects that

If the user takes longer than 10 minutes to finish the form, the token expires
and register returns 401. Send a fresh OTP and start again from step 1.

---

## 4. Endpoints affected

Three copies of each endpoint exist and **all three behave the same way**:

| Endpoint | Needs the token |
|---|---|
| `POST /api/Register/register` | **Yes** |
| `POST /api/v1/Register/app/register` | **Yes** |
| `POST /api/v1/Register/web/register` | **Yes** |
| `POST /api/SnapTrade/registerusers` | **Yes** |
| `POST /api/v1/SnapTrade/web/registerusers` | **Yes** |
| `POST /api/CAMS/RegisterUser` | **No** ★ REVERTED 2026-10-07 |

A token from any `verify-otp` works with any of the register endpoints.

### ★ CAMS RegisterUser — verificationToken REMOVED (2026-10-07)

**Previously (2026-10-05)** we added the verificationToken check to CAMS
RegisterUser as well. **This has been reverted.** CAMS consent itself is the
identity proof — the user authenticates with their PAN/mobile through the CAMS
consent popup before RegisterUser is ever called.

**Now** CAMS/RegisterUser does NOT require `verificationToken`. No OTP step
needed for the CAMS flow. The consent popup is sufficient identity verification.

**App action:** do NOT send `verificationToken` in CAMS/RegisterUser — it is
ignored. The CAMS flow stays:
```
GetConsentURL → user completes CAMS consent → FetchData → RegisterUser (no OTP needed)
```

---

## 5. Payloads and responses

### Step 1 — send OTP

**Request** `POST /api/Register/send-otp`

```json
{
  "input": "9876543210",
  "mobile": "9876543210",
  "email": ""
}
```

**Response**

```json
{
  "success": true,
  "message": "OTP sent successfully.",
  "isNew": true,
  "inputType": "mobile"
}
```

> `input` is where the OTP is sent: the **mobile for India**, the **email for
> every other country**.

---

### Step 2 — verify OTP  ★ response changed

**Request** `POST /api/Register/verify-otp`

```json
{
  "input": "9876543210",
  "otp": "123456",
  "source": "App"
}
```

**Response — success**

```json
{
  "success": true,
  "message": "OTP verified.",
  "inputType": "mobile",
  "verificationToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJwdXJwb3NlIjoib3RwX3ZlcmlmaWNhdGlvbiIsImlkZW50aWZpZXIiOiI5ODc2NTQzMjEwIn0.xxxxx"
}
```

**`verificationToken` is the new field.** Store it.

**Response — wrong OTP** (unchanged, and no token)

```json
{
  "success": false,
  "message": "Invalid OTP. 4 attempt(s) remaining"
}
```

---

### Step 3 — register  ★ request changed

**Request** `POST /api/Register/register`

```json
{
  "name": "Test User",
  "countryCode": "IND",
  "mobile": "9876543210",
  "email": null,
  "imageUrl": "",
  "version": null,
  "language": "ENG",
  "isNRI": 0,
  "isNRICountryCode": "",
  "source": "App",
  "verificationToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.xxxxx"
}
```

**`verificationToken` is the new field.** Everything else is as before.

**Response — success** (unchanged)

```json
{
  "success": true,
  "message": "User registered successfully.",
  "userId": 725,
  "isNew": true,
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIs...",
    "refreshToken": "k+WNIRlISWuSIgDRTZIT...",
    "accessTokenExpiresAt": "2026-09-29T12:21:02+05:30",
    "refreshTokenExpiresAt": "2026-10-06T12:06:02+05:30"
  }
}
```

**Response — token missing, expired or invalid** → HTTP **401**

```json
{
  "success": false,
  "message": "OTP verification required. Verify the mobile or email first."
}
```

**Response — token does not match the number being registered** → HTTP **401**

```json
{
  "success": false,
  "message": "OTP was not verified for this mobile or email."
}
```

---

### Step 3 (SnapTrade variant)

**Request** `POST /api/SnapTrade/registerusers`

```json
{
  "name": "Test User",
  "country": "IND",
  "mobile": "9876543210",
  "email": "",
  "brid": 12345,
  "product": "SCRC",
  "verificationToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.xxxxx"
}
```

**Response — success**

```json
{
  "success": true,
  "message": "User registered successfully.",
  "userId": 729,
  "isNew": true,
  "requiresOtp": false,
  "data": { "accessToken": "...", "refreshToken": "...", "accessTokenExpiresAt": "...", "refreshTokenExpiresAt": "..." }
}
```

The same two 401 responses apply. Note this now includes the case where the
number already exists — an existing account is no longer returned without proof.

> **Field naming:** `Register/register` uses `countryCode`, `SnapTrade/registerusers`
> uses `country`. That has always been so; we mention it only to save you a
> debugging session.

---

## 6. Which field is matched against the token

| Country | OTP goes to | Token is matched against |
|---|---|---|
| `IND` | mobile | `mobile` |
| everything else | email | `email` |

The comparison ignores case and surrounding spaces, but is otherwise exact. Send
the identifier in step 3 in the **same form** you sent it in step 1 — if you
verified `9876543210`, register `9876543210`, not `+919876543210`.

---

## 7. Test cases

Worth covering on your side:

| Scenario | Expected |
|---|---|
| Normal signup: send → verify → register | 200, account created |
| Register with no `verificationToken` | 401 |
| Register with a token older than 10 minutes | 401 |
| Verify number A, register number B | 401 |
| Reuse an access token as `verificationToken` | 401 |
| CAMS signup | 200, unchanged, no token needed |

All of these were verified against a running API and the live database — 39
tests across both endpoints and all route variants, all passing.

---

## 8. One more change: `/auth/activate` is gone

`POST /api/auth/activate` has been **removed** (all three route variants).

It set the activated flag for any user id with no authentication at all. It was
compiled only in Debug builds so it was never live in production, and nothing
calls it — we checked the web app, the legacy API and the mobile source. It is
gone so the risk cannot return.

**No action needed** unless you were calling it. If you were, please tell us —
activation already happens automatically inside `verify-otp`, `register`,
`registerusers` and the CAMS paths, so there should be no reason to.

---

## 9. Drop `activated` and `isPaid` if you send them

If your register payload includes either of these, please remove them:

```json
"activated": true,     // remove
"isPaid": false        // remove
```

The server has never read them on this endpoint — the stored procedure hardcodes
`Activated = 0` and takes no `@IsPaid` at all — so they were already being
ignored. They are now gone from the contract entirely.

Activation happens server-side once the OTP receipt is checked. Paid status is
set only by the billing flow. Neither is something a client can ask for, and
sending them will not cause an error — they are simply dropped.

---

## 10. Summary

1. Read `verificationToken` from the `verify-otp` response
2. Send it as `verificationToken` in `register` / `registerusers`
3. Handle **401** by sending a fresh OTP and restarting the flow
4. Leave CAMS alone
5. Send the identifier in the same format at verify and at register

Any questions on the payloads or the failure cases, come back to us before the
release — this is a breaking change and we would rather sort it out early.


---
---

# Part 2 — Pre-registration holdings now use a GeoId, not the int PRID (2026-09-30)

**What changed:** the pre-registration holdings endpoint no longer accepts the
raw integer PRID. It now takes an unguessable GUID handle called **`GeoId`**.

**Why:** anyone could iterate the sequential PRID
(`GET ...Get_Holdings_Before_Registration_Web?PRID=1,2,3…`) and read other
people's PAN-linked holdings — financial data, no login required. The int PRID
is now replaced on the wire by a GUID that can't be guessed.

**If you do nothing:** your pre-registration holdings screen stops loading, and
`pre-register-device` no longer gives you a `prid` field.

## API changes

### 1. `POST /api/SnapTrade/pre-register-device`

**Before**
```json
{ "success": true, "message": "Device pre-registered.", "prid": 10000282 }
```

**Now**
```json
{ "success": true, "message": "Device pre-registered.", "geoId": "e7fcef1d-8c22-45e8-99ba-ca7650cf7489" }
```

> The `prid` field is gone. Store `geoId` and use it for the holdings read below.

### 2. `GET /api/Holding/Get_Holdings_Before_Registration_Web`

**Before**
```
GET .../Get_Holdings_Before_Registration_Web?PRID=10000282&Country=IND
```

**Now**
```
GET .../Get_Holdings_Before_Registration_Web?GeoId=e7fcef1d-8c22-45e8-99ba-ca7650cf7489&Country=IND
```

- The query param is **`GeoId`** (the GUID from step 1), not `PRID`.
- **`GeoId` is now required.** A call with no GeoId returns **400 "GeoId is required."**
- The old `ISPRID` param is **removed** — do not send it.
- Passing the old int PRID as the value returns an empty result (0 holdings) —
  it no longer resolves to anything. That is the whole point of the fix.

Response body shape is unchanged.

## The one thing to decide with us — the register "brid"

Registration (`CAMS/RegisterUser` / `SnapTrade/registerusers`) still takes a
numeric **`brid`** (the int PRID) to remap the guest's synced holdings to the new
account. That remap on the server still runs on the int PRID.

But `pre-register-device` now returns only `geoId`, not the int. So:

- **The holdings READ is fixed and uses `geoId`.** (done, tested)
- **The register `brid`** — if your flow relied on the `prid` from
  `pre-register-device` to fill `brid`, that value is no longer returned. Please
  tell us how your app obtains `brid` today so we align the register step. There
  are two options on the table (return both geoId+prid, or move the remap to
  geoId too); we did not want to pick one without you.

Same open question exists on web and applies equally to the app.

## Test cases

| Scenario | Expected |
|---|---|
| pre-register-device | returns `geoId`, no `prid` |
| holdings read with the geoId | 200, real holdings |
| holdings read with old int PRID | 200 but 0 holdings (blocked) |
| holdings read with no GeoId | 400 "GeoId is required" |

Verified end-to-end on local: a holding seeded against a PRID is visible via its
GeoId and invisible via the int PRID.

## Summary

1. Read `geoId` from `pre-register-device` (not `prid`)
2. Send it as `GeoId` on the holdings read (not `PRID`); drop `ISPRID`
3. Tell us how you fill `brid` for register, so we finish that half together


---

## Part 2b — GetConsentURL now also returns `geoId` (2026-09-30)

`POST /api/CAMS/GetConsentURL` (and `GetConsentURL_RegUser`) response now
includes a **`geoId`** field alongside the existing `prid`:

**Now**
```json
{
  "success": true,
  "message": "Success",
  "result": { "redirectionurl": "...", "consentHandle": "..." },
  "prid": 10000284,
  "geoId": "2f45a2b4-47b4-4e9a-9560-af10fc571376"
}
```

- **`prid`** (int) — unchanged, still used for the BRID→user remap at register.
- **`geoId`** (GUID) — **new**; use it for the pre-registration holdings read
  (`Get_Holdings_Before_Registration_Web?GeoId=...`). The old int PRID no longer
  resolves on that read.

So after CAMS consent, store `geoId` from this response and send it (not the
int PRID) when you fetch the synced holdings. The `prid` is still what you pass
as `brid` into register.

Note: `GetConsentURL_RegUser` is the logged-in variant — it has no pre-reg row,
so it does not produce a geoId; logged-in users read their holdings via
`Get_Holdings_Before_Registration_App` (JWT-based) instead.


---
---

# Part 3 — AUTH-07: token storage & XSS hardening (2026-09-30)

## What's DONE (web + API, no app change needed)

1. **CSP + security headers** — added to `web.config` as
   `Content-Security-Policy-Report-Only` (blocks nothing yet; logs violations),
   plus `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`. Web-only.

2. **Server-side logout revocation** — NEW endpoint **`POST /api/auth/logout`**
   (`[Authorize]`, UserId from JWT). Revokes every active refresh token for the
   caller via the existing `Usp_Update_User_Refresh_Tokens` @UserId path (no new
   SP). Verified: active tokens 1 → 0 after logout; 401 without a token.
   - **App team: call `POST /api/auth/logout` on sign-out** (with the access
     token) so a stolen refresh token dies at logout. Fire-and-forget is fine.

## What's PENDING — needs a decision (the cookie change)

**Target:** refresh token in an `HttpOnly; Secure; SameSite=Strict` cookie,
access token in memory only. `HttpOnly` means JavaScript cannot read it, so XSS
cannot steal it — the real fix.

**Why it's not done yet:** it's a breaking, cross-cutting auth change. The API
would set the refresh token as a cookie for **web**, but **mobile apps don't use
browser cookies** — they use secure device storage (Keychain/Keystore), which is
already safer than web localStorage. So the API must support **both**:

- **Web:** refresh token issued as `Set-Cookie: HttpOnly; Secure; SameSite=Strict`;
  `/api/auth/refresh` reads it from the cookie; access token returned in the body
  and held in memory only.
- **Mobile:** refresh token stays in the response body (as today); the app keeps
  storing it in secure device storage. No cookie.

**Decision needed from the app team + backend:**
- Confirm mobile keeps the body-based refresh token (no cookie) — so this change
  is additive and does NOT break the app.
- The API distinguishes web vs mobile (e.g. an `X-Client: web|mobile` header or
  the existing `isWeb` flag) to decide cookie-vs-body.

Until that's agreed, the web keeps using localStorage — mitigated (not fixed) by
the CSP + logout revocation above.

## DONE (web-only) since
- Sanitised the API-fed `bypassSecurityTrustHtml` calls with DOMPurify: IPO
  description/business description, MF fund summary, and article body now pass
  through `sanitizeApiHtml()` (strips scripts/handlers, keeps formatting) before
  being trusted. The ~25 static/translation `trustHtml` helpers were left as-is
  (author-controlled, not an injection door). No app impact.

## Still web-only pending
- Tighten the CSP from Report-Only to enforcing: move/hash the 4 inline scripts
  in index.html, review the violation reports, then drop -Report-Only and remove
  'unsafe-inline'. No app impact once the inline scripts are handled.


---
---
---

# ═══════════════════════════════════════════════════════════
# SECTION A — CAMS-ONBOARDING
# ═══════════════════════════════════════════════════════════

Everything the CAMS pre-registration + consent + holdings flow changed, in one
place, with sample payloads and responses. All verified on local (2026-09-30).

## A.0 — Flow at a glance

```
1. pre-register-device        -> returns geoId (NOT the int prid any more)
2. GetConsentURL / _RegUser   -> returns geoId + consent redirect url
3. (user completes CAMS consent in the popup)
4. FetchData                  -> still uses the int brid (unchanged)
5. Get_Holdings_..._Web       -> now takes GeoId (NOT PRID)
```

## A.1 — The core change: int PRID -> GeoId (AUTH-06)

**Why:** the pre-registration holdings read was reachable by iterating a
sequential integer `?PRID=1..N`, exposing PAN-linked holdings of anyone mid-
onboarding. The guessable int is replaced on the wire by an unguessable GUID
handle called **`geoId`**. The int PRID never leaves the server now.

**DB:** a `GeoId uniqueidentifier` handle is stored per pre-registration row and
resolved to the internal PRID server-side. (DB-team change, already applied on
dev — must run on PROD before deploy.)

## A.2 — POST /api/SnapTrade/pre-register-device

**Request** (unchanged)
```json
{ "deviceId": "48abd569-e01a-4211-933e-36477e0b099d", "isWeb": true }
```

**Response — BEFORE**
```json
{ "success": true, "message": "Device pre-registered.", "prid": 10000282 }
```

**Response — NOW**
```json
{ "success": true, "message": "Device pre-registered.", "geoId": "e7fcef1d-8c22-45e8-99ba-ca7650cf7489" }
```

> **App action:** store `geoId`, not `prid`. The `prid` field is gone.
> Idempotent: the same `deviceId` returns the SAME `geoId` every time.

## A.3 — POST /api/CAMS/GetConsentURL  (anonymous / guest)

**Request** (unchanged)
```json
{ "mobile": "9876543210", "PAN": "ABCDE1234F", "deviceId": "...", "isWeb": "1" }
```

**Response — NOW** (`prid` is deliberately omitted; only `geoId` is returned)
```json
{
  "success": true,
  "message": "Success",
  "result": {
    "redirectionurl": "https://standardjourneyuat.camsfinserv.com/?ecreq=...",
    "consentHandle": "8404738b-dbed-4806-9ff0-e8947a51f733",
    "txnid": "acc950cb-...",
    "clienttxnid": "5da17126-...",
    "sessionId": "3932f884-..."
  },
  "geoId": "4c396fb5-bd90-4fa4-b7c4-05c6e668cfb7"
}
```

## A.4 — POST /api/CAMS/GetConsentURL_RegUser  (logged-in user, JWT)

Same as A.3 but returns BOTH `prid` and `geoId` (a logged-in user has a real
UserId; the flow still needs the int for the remap). Sends the CAMS SMS/redirect
against the user's own account.

**Response — NOW**
```json
{
  "success": true,
  "message": "Success",
  "result": { "redirectionurl": "...", "consentHandle": "..." },
  "prid": 742,
  "geoId": "9da5fded-929c-4c96-a0e7-6adfc32bf26e"
}
```

> **App action:** after consent, store the `geoId` from this response and use it
> for the holdings read (A.6). Keep the numeric value (prid) for the register
> remap if your flow uses it.

## A.5 — POST /api/CAMS/RegisterUser / InsertUserPreRegistration  [CHANGED 2026-10-05]

These call the pre-registration SP, whose parameter changed from `@PRID` (int) to
`@GeoId` (uniqueidentifier). If your app posts to `InsertUserPreRegistration`
with a `PRID` field, send `geoId` instead (or omit it on insert — the SP
generates one). Sending `PRID` now errors:
`"@PRID is not a parameter for procedure Usp_Insert_User_PreRegistration."`

### ★ CAMS/RegisterUser — verificationToken NOT required (REVERTED 2026-10-07)

CAMS consent is the identity proof. No OTP step needed for the CAMS flow.

**Request — NOW**
```json
{
  "mobile": "9876543210",
  "name": "Test User",
  "country": "IND",
  "email": "",
  "prid": 0,
  "geoId": "4c396fb5-bd90-4fa4-b7c4-05c6e668cfb7",
  "isNRI": 0,
  "isNRICountryCode": "",
  "product": "SCRC"
}
```

The CAMS flow stays:
```
1. GetConsentURL → user completes consent → FetchData
2. CAMS/RegisterUser (no OTP / verificationToken needed)
```

## A.6 — GET /api/Holding/Get_Holdings_Before_Registration_Web  [CHANGED]

**BEFORE**
```
GET .../Get_Holdings_Before_Registration_Web?PRID=10000282&Country=IND
```

**NOW**
```
GET .../Get_Holdings_Before_Registration_Web?GeoId=e7fcef1d-8c22-45e8-99ba-ca7650cf7489&Country=IND
```

- Param is **`GeoId`** (the GUID from step 1/2), not `PRID`.
- **`GeoId` is required** — a call with none returns **400 "GeoId is required."**
- The old `ISPRID` param is **removed** — do not send it.
- Passing the raw int PRID as the value resolves to **0 holdings** (blocked).

Response body shape is unchanged. Verified: a holding seeded against a PRID is
visible via its GeoId (StockCount=1) and invisible via the int PRID (0).

## A.7 — GET /api/Holding/Get_Holdings_Before_Registration_App  (NEW, JWT)

The registered-user twin of A.6. `[Authorize]`, reads holdings by the **UserId
from the JWT** (no PRID/GeoId in the URL — a caller can only read their own).
Use this for a LOGGED-IN user's holdings instead of the guest read.

```
GET .../Get_Holdings_Before_Registration_App?Country=IND
Authorization: Bearer <jwt>
```

## A.7b — GET /api/CAMS/FetchData  [CHANGED — GeoId only, Mobile REMOVED (HR-03)]

This is the call you make **after** the user finishes CAMS consent. It triggers
the holdings sync and now tells you the **real outcome**, so you know whether to
show the portfolio, wait, or send the user back to connect again.

### Request (changed — ★ Mobile param REMOVED 2026-10-05)
```
GET /api/CAMS/FetchData?GeoId=<geoId>
```
- `GeoId` — the unguessable handle you got from `GetConsentURL(_RegUser)` (AUTH-06).
  **Was `PRID` before — that is gone.** Blank/missing GeoId → `400 "GeoId is required."`
- ~~`Mobile`~~ **REMOVED** — was a PII leak (HR-03). If you still send it, it is
  silently ignored. Remove it from your code.

### Response — read THREE fields (this is the important part)

The response carries three signals that mean **different** things. The two
booleans are NOT the same flag — read them as:

| Field | Type | Question it answers |
|-------|------|---------------------|
| `success` | bool | **Did the API call work?** `true` = the sync ran and we got a valid answer (even if that answer is "nothing to show yet"). `false` = the call itself failed (server/sync error, HTTP 500). This is about the *request*, not the data. |
| `synced` | bool | **Are the user's holdings in our system now?** `true` only when holdings were actually parsed and stored. This is about the *data*. |
| `status` | string | **The exact case / what to do** — always branch on this. One of `synced` / `processing` / `no_data` / `error`. |
| `result.holdingsCount` | int | How many holdings landed (0 unless `synced:true`). |
| `result.{name,email}` | string | Prefill for the form (may be empty). ~~`mobile,panNumber,cntryCode`~~ **REMOVED (HR-03)** |

So: **`success` = "the call worked", `synced` = "holdings are in", `status` = "why / which case".**

### The two booleans combined — what each pairing means

| `success` | `synced` | What it means | `status` |
|-----------|----------|---------------|----------|
| `true` | `true` | Call worked **and** holdings are in. | `synced` |
| `true` | `false` | Call worked, but **no holdings yet** — either still arriving, or nothing came / user rejected. Look at `status` to tell which. | `processing` or `no_data` |
| `false` | `false` | **The call failed** (server/sync error). Nothing to trust in `result`. | `error` (HTTP 500) |
| `false` | `true` | **Never happens** — you cannot have holdings from a failed call. If you ever see this, treat it as an error. | — |

> Do **not** gate the UI on `success` alone — `success:true` can still mean
> "no holdings" (`synced:false`). Use `synced` for "show the portfolio vs not",
> and `status` for the precise next step below.

### What each `status` means and what the app should do

| `status` | `success` / `synced` | What happened | App action during onboarding |
|----------|----------------------|---------------|------------------------------|
| `synced` | `true` / `true` | Holdings were parsed and stored (`holdingsCount > 0`). | **Proceed** — show the imported portfolio. |
| `processing` | `true` / `false` | Call worked; CAMS data arrived but isn't parsed yet (webhook still landing). | **Wait & retry** FetchData in a few seconds (e.g. 3s, up to ~5 tries). Do NOT show "failed". |
| `no_data` | `true` / `false` | Call worked; nothing synced — consent not completed, **user rejected it**, or the webhook never arrived. | **Send the user back** to connect their account again — this is not a transient error. |
| `error` | `false` / `false` (HTTP **500**) | A real server/sync failure. `message` has the detail. | Show a generic error + retry option. |

### Samples
```
// synced
{ "success": true,  "synced": true,  "status": "synced",
  "message": "Synced 3 holding(s) for GeoId: <g>",
  "result": { "name":"...", ..., "geoId":"<g>", "holdingsCount": 3 } }

// still coming (call worked, holdings not in yet)
{ "success": true, "synced": false, "status": "processing",
  "message": "CAMS data is still being processed. Please try again shortly.",
  "result": { ..., "holdingsCount": 0 } }

// nothing / rejected (call worked, nothing to show)
{ "success": true, "synced": false, "status": "no_data",
  "message": "No CAMS holdings found. Consent may not have completed, or the data has not arrived yet.",
  "result": { ..., "holdingsCount": 0 } }

// server error
HTTP 500
{ "success": false, "status": "error", "message": "<reason>" }
```

> **Why this matters for the app specifically:** the web client detects a
> cancelled/rejected consent from the browser popup (the popup closes without
> hitting our callback) and never even calls FetchData in that case. The app has
> no popup signal — so for you, **FetchData's `status` is the only way** to tell
> "rejected / nothing came" (`no_data`) apart from "still arriving" (`processing`)
> and "done" (`synced`). Branch on `status`.

## A.8 — What is UNCHANGED in this flow
- The register remap (`brid` -> UserId) still uses the int PRID.
- Response body shapes of the holdings reads (A.6 / A.7).

## A.9 — CAMS checklist for the app team
1. Read `geoId` from `pre-register-device` and `GetConsentURL(_RegUser)`.
2. Send `GeoId` (not `PRID`) on `Get_Holdings_Before_Registration_Web`; drop `ISPRID`.
3. For a logged-in user, prefer `Get_Holdings_Before_Registration_App` (JWT).
4. Stop sending `PRID` to `InsertUserPreRegistration`.
5. Handle **400 "GeoId is required"** and empty-holdings gracefully.
6. On `FetchData` (A.7b): send `GeoId` not `PRID`, and **branch on `status`** —
   `synced` → proceed, `processing` → retry, `no_data` → reconnect, `error` → error.
   Use `synced` (not `success`) to decide whether to show the portfolio.
7. **★ REVERTED 2026-10-07:** CAMS/RegisterUser does NOT require
   `verificationToken` — CAMS consent is the identity proof. No OTP step needed.
8. **★ HR-03:** Remove `Mobile` param from `FetchData` calls. Stop reading
   `panNumber`, `mobile`, `cntryCode` from the response — they are no longer returned.
9. **★ HR-07:** Stop reading `userId` and `message` from `CheckUser` response —
   only `{ success, isNew }` is returned now. The remap no longer happens here;
   it requires authentication (HR-02 flow).


# ═══════════════════════════════════════════════════════════
# SECTION B — LOGIN / SIGNUP
# ═══════════════════════════════════════════════════════════

Everything the authentication + registration flow changed, with samples.
Verified on local (2026-09-30).

## B.0 — Summary of changes
1. **OTP now required to register** — a signed `verificationToken` from
   `verify-otp` must be sent to `register`; the identifier must match. (L2/L4)
2. **`activated` / `isPaid` dropped** — never honoured; remove from your payload. (L3)
3. **SMS-flooding rate limit** — `send-otp` is capped at 5/min per IP -> **429**.
4. **`/auth/activate` removed** — the debug endpoint is gone.
5. **Refresh token is now an HttpOnly cookie**, not in the response body. (AUTH-07)
6. **Logout revokes server-side** — new `POST /api/auth/logout`.

## B.1 — Registration: OTP proof required (L2/L4)

The registration endpoints (`Register/register`, `SnapTrade/registerusers`) now
**require** proof that the OTP was verified. Flow:

```
send-otp  ->  verify-otp (returns verificationToken)  ->  register (must send it)
```

### POST /api/Register/verify-otp — now returns a token
```json
// request
{ "input": "9876543210", "otp": "123456" }

// response — NEW field verificationToken
{
  "success": true,
  "message": "OTP verified.",
  "inputType": "mobile",
  "verificationToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJwdXJwb3NlIjoib3RwX3ZlcmlmaWNhdGlvbi..."
}
```
`verificationToken` is a 10-minute signed JWT naming the verified identifier.

### POST /api/Register/register — now requires it
```json
{
  "name": "Test User",
  "countryCode": "IND",
  "mobile": "9876543210",
  "language": "ENG",
  "isNRI": 0,
  "verificationToken": "eyJhbGci..."      // NEW, required
}
```

**Failure responses (both HTTP 401):**
```json
{ "success": false, "message": "OTP verification required. Verify the mobile or email first." }
{ "success": false, "message": "OTP was not verified for this mobile or email." }
```
The second fires when the identifier in the token != the one being registered
(stops verifying your own number and registering another number).

> **App action:** capture `verificationToken` from `verify-otp`, send it in
> `register` / `registerusers` / `CAMS/RegisterUser`. **All three** register
> endpoints now require it — CAMS is no longer exempt (see Part 1 §4 above).

## B.2 — Drop `activated` and `isPaid` (L3)

If your register payload has these, remove them — the server never read them and
they are gone from the contract:
```json
"activated": true,     // remove
"isPaid": false        // remove
```
Activation happens server-side after the OTP proof; paid status is set only by
billing.

## B.3 — SMS-flooding rate limit (AUTH-05)

`POST /api/*/send-otp` is now capped at **5 requests per minute per IP**.
Over the limit -> **HTTP 429**. Add a resend cooldown in the OTP UI so normal
users never hit it.
```
6th send within a minute -> HTTP 429 Too Many Requests
```

## B.4 — /auth/activate removed
The debug-only `POST /api/auth/activate` (set any user's activated flag with no
auth) is **deleted**. If you call it, stop — it returns 404. Real activation is
automatic in verify-otp/register.

## B.5 — Refresh token: HttpOnly cookie + body fallback (AUTH-07)  [UPDATED 2026-10-07]

The refresh token is set as an `HttpOnly; Secure; SameSite=None` cookie **AND**
returned in the response body. The cookie is the primary transport (JS cannot
read it → XSS-safe). The body copy is a fallback for cross-site callers
(e.g. localhost:4200 → uatapiv3) where the browser blocks the cookie.

### How the API behaves
- Every token-issuing endpoint (`login`, `register`, `registerusers`,
  `CAMS/RegisterUser`, `CAMS/ValidateOTP`) sets:
  ```
  Set-Cookie: islamicly_refresh_token=<token>; HttpOnly; Secure; SameSite=None; Path=/; Expires=<7d>
  ```
  **and** returns `"refreshToken": "<token>"` in the body (no longer blanked).
- **POST /api/auth/refresh** reads the refresh token from the **cookie first**;
  if the cookie is absent (cross-site block), reads it from the **request body**:
  ```json
  { "refreshToken": "<token>" }
  ```
  Must be sent with `withCredentials: true` (so the cookie is sent when available).
- **POST /api/auth/logout** (NEW) revokes all refresh tokens for the user AND
  clears the cookie. `[Authorize]`.

### Sample — login
```
POST /api/auth/login      body: { "email": "...", "password": "..." }
-> 200
   Set-Cookie: islamicly_refresh_token=<token>; HttpOnly; Secure; SameSite=None
   body: { "success": true, "data": { "accessToken": "eyJ...", "refreshToken": "<token>", ... } }
```

### Sample — refresh (cookie available, same-site)
```
POST /api/auth/refresh    body: {}   +  cookie sent automatically
-> 200  body: { "accessToken": "eyJ...", "refreshToken": "<rotated>", ... }  + rotated Set-Cookie
```

### Sample — refresh (cookie blocked, cross-site fallback)
```
POST /api/auth/refresh    body: { "refreshToken": "<token>" }
-> 200  body: { "accessToken": "eyJ...", "refreshToken": "<rotated>", ... }  + rotated Set-Cookie
```

### Mobile app
**No decision needed any more.** The refresh token is in the response body for
ALL callers (web and mobile). Mobile stores it in Keychain/Keystore as before and
sends it in the body on `/api/auth/refresh`. The cookie is a bonus for web
(same-site auto-send) — mobile doesn't need it. Both paths work.

## B.6 — Logout: POST /api/auth/logout  (NEW)
```
POST /api/auth/logout
Authorization: Bearer <access token>
-> 200 { "success": true, "message": "Logged out." }
```
Revokes every active refresh token for the user (so a stolen token dies at
logout) and clears the cookie. **App action:** call this on sign-out.

## B.7 — Login/Signup checklist for the app team
1. Send `verificationToken` (from verify-otp) in register/registerusers/**CAMS RegisterUser**.
2. Remove `activated` / `isPaid` from register payloads.
3. Handle **429** on send-otp (add a resend cooldown).
4. Stop calling `/auth/activate`.
5. Store `refreshToken` from login/register response; send it in the body on
   `/api/auth/refresh` as `{ "refreshToken": "<token>" }`. Cookie is a bonus
   for web — mobile uses the body path (B.5).
6. Call `POST /api/auth/logout` on sign-out.

---

## C. HR-02 — Logged-in CAMS re-sync fix (FIXED 2026-10-05)

**Finding:** A logged-in user doing a CAMS re-sync from the Holdings page sees
"CAMS hasn't sent your holdings yet" even after a successful consent+sync. The
holdings land in the DB under the pre-registration PRID but are never remapped
to the real UserId, so the logged-in user's holdings read returns nothing.

**Root cause:** `camsLoadHoldings` in the FE read holdings by GeoId immediately
after `FetchData` succeeded, but never called `MapPridToCurrentUser` to remap
the PRID→UserId. A wrong comment at lines 2582-2586 claimed "no
MapPridToCurrentUser call is needed here."

**Fix (FE-only, no backend or SP changes):**
- `camsLoadHoldings` now calls `camsSvc.mapBridToCurrentUser(brid, geoId)` first
  → this POSTs to `POST /api/cams/MapPridToCurrentUser` with `{ prid, geoId }`
  → the backend runs `Cams.Usp_Update_User_Holding_UID(@GeoId, @UserId)` which
  remaps all holdings from the pre-reg PRID to the JWT user's real UserId.
- After the remap (or if it fails), `camsReadHoldings` reads the holdings as
  before. On success, holdings are now under the correct UserId and display.

**App team action:** **None required.** This is a FE-only fix in the Angular web
app. The `MapPridToCurrentUser` API endpoint is unchanged — the mobile app
already has its own re-sync flow. No new endpoints, no payload changes.

---

## D. HR-03 — FetchData PII leak fix (FIXED 2026-10-05)

**Finding:** `GET /api/CAMS/FetchData` is `[AllowAnonymous]` and accepted a
`Mobile` query param. It called `Usp_Get_CAMS_User_Details_By_Mobile(@Mobile)`
which returned **full Name, Email, PAN number, and Mobile** for any phone
number — no authentication, no rate limit. Anyone could look up PII by
phone number. DPDP / SEBI exposure.

**What changed (BREAKING for app if you send Mobile):**

| Before | After |
|--------|-------|
| `GET /api/cams/FetchData?GeoId=xxx&Mobile=9876543210` | `GET /api/cams/FetchData?GeoId=xxx` |
| Response: `name, email, mobile, panNumber, cntryCode, geoId, holdingsCount` | Response: `name, email, geoId, holdingsCount` |
| Details looked up by free-text Mobile (any number) | Details looked up by GeoId→PRID (only the consent session owner) |

**Removed from response:**
- `panNumber` — was never used by any FE caller, PII exposure
- `mobile` — was never used by any FE caller, PII exposure
- `cntryCode` — was never used by any FE caller

**Removed from request:**
- `Mobile` query param — no longer needed; the new SP
  `Cams.Usp_Get_CAMS_FetchData_By_GeoId(@GeoId)` resolves the PRID from
  GeoId and returns Name + Email from demat details. No mobile-based lookup.

**App team action:**
1. **Stop sending `Mobile`** in the FetchData query string. It is now ignored
   by the backend but should be removed from app code for clarity.
2. **Stop reading `panNumber`, `mobile`, `cntryCode`** from the FetchData
   response — these fields are no longer returned.
3. If the app uses `panNumber` from this response for anything, find an
   alternative source (e.g. the user's own profile). The PAN must not come
   from an anonymous endpoint.

---

## E. HR-07 — CheckUser remap + userId leak fix (FIXED 2026-10-05)

**Finding:** `POST /api/CAMS/CheckUser` is `[AllowAnonymous]` and accepted
`{ mobile, email, geoId }`. When the mobile/email matched an existing user it:

1. **Returned the user's integer `userId`** in the response — user enumeration
2. **Called `RemapHoldingsByGeoIdAsync`** which remapped the GeoId's
   pre-registration holdings into that user's account — holdings injection

An attacker with their own GeoId (containing junk/manipulated holdings) could
POST any victim's mobile number and inject those holdings into the victim's
portfolio. They also learned whether any phone/email is registered and got the
internal DB userId.

**What changed (BREAKING if you read `userId` from CheckUser):**

| Before | After |
|--------|-------|
| Response: `{ success, message, userId, isNew }` | Response: `{ success, isNew }` |
| Remap triggered on every call for existing users | **No remap** — remap happens only after OTP-verified login (HR-02 fix) |

**Removed from response:**
- `userId` — was an integer DB ID, user enumeration risk
- `message` — removed ("User already exists." / "New user." leaked existence)

**Removed from server logic:**
- `RemapHoldingsByGeoIdAsync` call — the remap is now handled by the
  authenticated `MapBridToCurrentUser` endpoint after login (HR-02 fix),
  which is `[Authorize]` and reads UserId from the JWT

**App team action:**
1. **Stop reading `userId` from CheckUser response** — it is no longer returned.
   If you used it for anything, you must get the userId from the login/register
   response instead (which returns it after authentication).
2. **Stop reading `message` from CheckUser response** — only `success` and
   `isNew` are returned.
3. **The remap no longer happens here.** Holdings are remapped to the user only
   after they log in or register with a verified OTP. This is already handled
   by the authenticated flow. If your app relied on CheckUser to remap holdings
   before the user logged in, that path is removed — the remap now requires auth.
4. The `isNew` field works exactly as before: `true` = new user (proceed to
   register), `false` = existing user (proceed to login).

---

## A.8 — POST /api/CAMS/InsertUserPreRegistration  [CHANGED — HR-41]

### What changed (2026-10-06)

Two security fixes:

1. **Flag='U' now requires authentication.**
   Previously this endpoint was fully anonymous — anyone could call it with
   `Flag: "U"` and a known GeoId to overwrite the Mobile/PAN on someone else's
   pre-registration row. Now:
   - `Flag: "I"` (insert) — still anonymous, no change.
   - `Flag: "U"` (update) — **requires a JWT**. Anonymous callers get `401`.

2. **PRID removed from the response.**
   The response no longer includes the integer `PRID` field. Only `GeoId` is
   returned. If your app was reading `PRID` from this response, switch to
   `GeoId` (which you should already be using per AUTH-06).

### Action needed

- If you call this endpoint with `Flag: "U"`, make sure you include a valid
  `Authorization: Bearer <jwt>` header. Without it → `401`.
- Stop reading `PRID` from the response — it is no longer returned. Use `GeoId`.

---

## A.9 — CheckAlreadySynced + ValidateOTP  [BREAKING — GET→POST, HR-50]

### What changed (2026-10-06)

Two CAMS endpoints changed from **GET** to **POST** to stop sending PII (mobile,
PAN, OTP) in the URL query string — query strings end up in IIS logs, proxy
logs, browser history and Referrer headers.

### 1. POST /api/CAMS/CheckAlreadySynced

**Was:**
```
GET /api/CAMS/CheckAlreadySynced?mobile=9876543210&pan=ABCDE1234F
```

**Now:**
```
POST /api/CAMS/CheckAlreadySynced
Content-Type: application/json

{ "mobile": "9876543210", "pan": "ABCDE1234F" }
```

Response shape is unchanged: `{ success, message, result: { isHoldingsExists } }`.

### 2. POST /api/CAMS/ValidateOTP

**Was:**
```
GET /api/CAMS/ValidateOTP?OTP=123456&UserId=42
```

**Now:**
```
POST /api/CAMS/ValidateOTP
Content-Type: application/json
Authorization: Bearer <jwt>   (if available)

{ "OTP": "123456", "UserId": 42 }
```

Response shape is unchanged: `{ success, message, tokens }`.

### Action needed

1. Change `CheckAlreadySynced` from GET with query params to POST with JSON body.
2. Change `ValidateOTP` from GET with query params to POST with JSON body.
3. **GET requests to these endpoints will now return 405 Method Not Allowed.**

---

## A.10 — GetUserCamsSCSyncStats now returns CamsStatus + SnaptradeStatus  [HR-52]

### What changed (2026-10-06)

`GET /api/Holding/GetUserCamsSCSyncStats` now returns **two new columns** alongside
the existing 11 fields:

| New field | Type | Values |
|-----------|------|--------|
| `CamsStatus` | string | `SYNCED`, `CONSENT_REJECTED`, `CONSENT_OK_NO_HOLDINGS`, `CONSENT_PENDING`, `NO_CAMS_SYNC` |
| `SnaptradeStatus` | string | `SYNCED`, `CONNECTED_NO_HOLDINGS`, `NO_ST_SYNC` |

These come from the existing SP `Cams.Usp_Get_User_Cams_SC_Sync_Stats` (already
updated by the DB team). The existing fields (`IsCamsSync`, `IsSingleBroker`,
`BrokerCount`, etc.) are unchanged.

### What each status means

**CAMS:**
| Status | Meaning | Suggested UI |
|--------|---------|--------------|
| `SYNCED` | Consent active + holdings in DB | "CAMS Synced" (green) |
| `CONSENT_REJECTED` | User rejected the CAMS consent | "CAMS Consent Rejected" (red) |
| `CONSENT_OK_NO_HOLDINGS` | Consent active but zero holdings arrived | "CAMS Consent OK · No Holdings" (amber) |
| `CONSENT_PENDING` | Consent row exists but no response yet | "CAMS Consent Pending" (amber) |
| `NO_CAMS_SYNC` | No CAMS consent row for this user | (don't show anything) |

**SnapTrade:**
| Status | Meaning | Suggested UI |
|--------|---------|--------------|
| `SYNCED` | Broker connected + holdings in DB | "SnapTrade Synced" (green) |
| `CONNECTED_NO_HOLDINGS` | Broker connected but no holdings | "SnapTrade Connected · No Holdings" (amber) |
| `NO_ST_SYNC` | No SnapTrade mapping for this user | (don't show anything) |

### Action needed

1. Read `CamsStatus` and `SnaptradeStatus` from the `GetUserCamsSCSyncStats` response.
2. Use them to show sync status in the portfolio/holdings screen — these give
   precise state vs the binary `IsCamsSync` flag which can't distinguish
   "never synced" from "consent pending" from "consent OK but no holdings".
3. Existing fields are unchanged — no breaking change, purely additive.
