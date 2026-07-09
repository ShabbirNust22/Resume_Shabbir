# TapSign — Internal Developer & Team Report

**Audience:** Our own team (developers, product, ops, founders).
**Purpose:** Explain, in plain language, *what TapSign is*, *what we built*, *every service it offers and what each one is for*, and *how the mail system works end to end*.
**Confidentiality:** INTERNAL ONLY. This document explains our architecture. It is **not** the document we send to government, investors, or consultants — those are the separate "pitch" reports in this folder that deliberately hide *how* we built things. Do not forward this file outside the team.

> Note on scope: This describes capabilities as they exist in the codebase. It intentionally does **not** print any secret keys, credentials, connection strings, or `.env` values. If you need those, they live in the runtime `.env` / secret stores, not in a report.

---

## 1. The one-sentence version

**TapSign turns a person's phone into the thing that proves "this is really me / really us."** Anything important — a login, a payment, an email, a government portal action, an admin deletion — can be made *un-fakeable* by requiring a cryptographic approval that only the real owner's phone (backed by that phone's hardware security chip) can produce.

If you remember nothing else: **TapSign replaces "trust me, I typed the right password" with "prove it with hardware that cannot be copied or phished."**

---

## 2. The problem we exist to kill

Almost every attack that matters today comes down to **impersonation**:

1. **Phishing** — a fake email/website tricks someone into typing a password or approving something.
2. **Account takeover** — stolen passwords/OTPs let an attacker log in as you.
3. **Fake documents & fake emails** — a scammer sends an email that *looks* like it's from a bank or a ministry; the recipient can't tell it's fake.
4. **Fake receipts / fake confirmations** — a "payment confirmed" or "you're approved" message that was never really issued.
5. **Insider / careless admin actions** — someone with a login (or a left-open laptop) deletes users, moves money, or changes config.
6. **Stolen cloud keys** — a leaked AWS/GCP/Azure/GitHub key lets an attacker act as the company inside its own infrastructure.

Passwords, SMS OTPs and "click the link" verification all fail against these because **they can be copied, forwarded, intercepted, or entered on the wrong site.** TapSign's answer is a signature from a **hardware key that never leaves the owner's phone** and that is **bound to the exact action** being approved — so a phished approval is useless, and a stolen credential can't act alone.

---

## 3. The core idea (how the trust actually works — high level)

Every TapSign device (a phone) generates a private key **inside its secure hardware** — Apple's Secure Enclave, Android's Keystore / StrongBox. That private key:

- **Never leaves the chip.** Not to us, not to the cloud, not to the user.
- Is **unlocked only by the person** (Face ID / fingerprint / device PIN).
- Is **attested** — the phone proves to us, using the manufacturer's signed certificate chain, that the key really does live in genuine secure hardware on a genuine device (not an emulator, not extracted).

When something important happens, the backend sends a **challenge** ("approve login to Ministry portal at 14:03, from this browser, this action, this amount"). The phone signs *that exact challenge* with the hardware key. We verify the signature against the device's registered public key. Because:

- the signature is **bound to the specific action** (you can't reuse it for a different action),
- it is **one-time** (a nonce/anti-replay guard means it can't be replayed),
- and it can only be produced by **that** phone's hardware after the human approves,

…a phisher who tricks the user, or a thief who steals a laptop, or an attacker who steals a password, **still cannot complete the action.** They don't have the phone, they can't extract the key, and a stolen approval is bound to the wrong action so it's rejected.

That single primitive — *hardware-bound, action-bound, one-time approval* — is the engine under every product below.

---

## 4. The product surface — what we actually shipped

TapSign is not one app. It's a **platform** with a backend and many "ways to plug in." Here's the whole surface:

### 4.1 Backend (the trust brain)
- A Python (FastAPI) backend — the "tapproof-backend" — that holds the device registry, verifies every signature, runs the challenge/approval flow, stores audit trails, and exposes ~290+ API endpoints grouped by service (see §6).
- Backed by PostgreSQL (with database-level role separation and row guards so even a compromised app role can't rewrite identity bindings) and Redis for short-lived session/nonce state.

### 4.2 Native phone apps (the "signer")
- **iOS native app** (`mobile-sdk/ios-app`) — bundle id `com.tapsign.nativeapp`. Uses Secure Enclave + Face ID.
- **Android native app** (`mobile-sdk/android/tapsign-native-app`) — uses Android Keystore/StrongBox + biometric prompt.
- These are the apps that show "Approve / Deny" cards, do enrollment, handle recovery, witness approvals, and the mail "verify inbox."

### 4.3 SDKs (how customers embed TapSign)
- **Mobile SDK** (`mobile-sdk`) — iOS (Swift), Android (Kotlin), and **React Native** bridge — so a bank or government app can embed TapSign signing *inside their own app* (no separate app to install).
- **Universal Web SDK** (`platform_integrations/universal-web-sdk`) — drop-in for any website to add TapSign login/approval.
- **Guardian SDK** (`guardian_sdk/node`) — wraps sensitive server actions (delete user, move money, change config) behind a phone approval.
- **OTP SDK** (`otp_sdk/node`) — a hardened, signed OTP issuance/verification service for teams who still need OTP but want it bound and audited.

### 4.4 Browser extension (the "shield")
- A browser extension (`platform_integrations/browser_extension`) that **verifies signed emails and pages inside the browser** and renders the TapSign "Verified" badge/hover-card, and protects browser-based logins. This is what shows a green "Verified organization" badge on a signed government or bank email in Gmail.

### 4.5 Desktop vault / workstation guardian
- **Desktop Vault** (`desktop-vault`) + **Workstation Guardian** (`workstation_guardian`) — pair a computer to a phone so that unlocking secrets / doing privileged operations on a workstation requires a phone approval.

### 4.6 Marketing + operator websites
- **Public marketing site** + **Enterprise portal** + **Super-Admin console** (`SIGNTAP-WEBSITE-PROJECT`, Next.js). The enterprise portal is where an organization manages its domains, browsers, providers, billing, and device registration. The super-admin console is our internal control plane.

### 4.7 Provider proxy (the "cloud key vault")
- A set of **provider proxies** (`app/*_proxy_*`) that let an organization use AWS, GCP, Azure, GitHub, GitLab, and Bitbucket **without ever holding a long-lived cloud key in a place that can be stolen** — the real credential is minted just-in-time, server-side, and its use can be gated behind a phone approval.

---

## 5. The services, one by one — what each is and what it's for

This is the "list of services" the team asked for. Each entry: **what it is → what it's for → who uses it.** These map to the route groups in the backend (`/admin`, `/mail`, `/identity`, `/device`, `/browser`, `/guardian`, `/provider-proxy`, `/onboarding`, `/otp`, `/vault`, `/workspace`, `/card-protector`, `/passkey`, `/workstation`, `/oidc` `/saml` `/idp` `/federation`, `/billing`, `/recovery`, `/cli`, `/webhooks`, `/ai-proxy`, `/audit`, `/pricing`).

### 5.1 Identity & Device Enrollment (`/onboarding`, `/device`, `/devices`, `/identity`)
- **What:** The registration flow that binds a real person/organization to a hardware device, with attestation.
- **For:** Making sure the phone that will approve things is genuine, and tying it to an identity (a bank customer, a council taxpayer, a government staff member) via an **opaque per-institution reference** — we don't hold the institution's PII, just a reference and the device's public key.
- **Who:** Every user, once, at setup. Institutions integrate this so *their* identity check (KYC) becomes the anchor, and TapSign becomes the "keystone" that all their apps trust.

### 5.2 Passkeys (`/passkey`)
- **What:** WebAuthn/FIDO2 passkey support, complementary to hardware signing.
- **For:** Standards-based passwordless login where a full TapSign approval isn't needed.
- **Who:** Web logins, enterprise portal.

### 5.3 Authorization / Approval Sessions (`/auth`, `auth_sessions`, `/challenge`)
- **What:** The heart of the platform — issue a **request-bound challenge**, push it to the phone, collect the signed approval, and **consume it once** before the protected action runs.
- **For:** Login approval, payment approval, admin-action approval — anything that must be un-fakeable and un-replayable.
- **Who:** Every relying app (banks, gov portals, our own console).

### 5.4 Guardian (`/guardian`) — sensitive-action protection & recovery
- **What:** Wraps destructive/sensitive operations (delete user, disable account, change money route, change security config) so they require a custodian's phone approval, with optional **in-person** or **multi-witness** requirements. Also runs **witness-based recovery** (see §5.11).
- **For:** Stopping insider abuse, careless admins, and left-open laptops. "Even someone with the login can't do the dangerous thing alone."
- **Who:** Enterprise admins, super-admins, high-risk operations.

### 5.5 Mail (`/mail`) — signed & verified email
- **What:** The **anti-phishing email system** (full walkthrough in §7). It cryptographically signs outbound email from a **DNS-verified domain**, and lets recipients (in the extension, on the `/verify` page, or in the phone app's "Verify Inbox") confirm the email is genuinely from that organization and unmodified.
- **For:** Killing email impersonation — fake "your bank"/"the ministry" emails, fake receipts, fake password-reset emails.
- **Who:** Banks, government, any org that sends email people are afraid to trust. Also our own platform email (OTP, receipts, welcome).

### 5.6 Browser protection (`/browser`) & the extension
- **What:** Registers and manages the browsers an organization trusts, and drives the extension that renders verification badges and protects browser logins.
- **For:** Making phishing visible in the browser and binding browser sessions to approved devices.
- **Who:** Enterprise/government staff, end users reading signed mail.

### 5.7 Provider Proxy (`/provider-proxy`, `/connector`, `/connectors`) — cloud credential vaulting
- **What:** Server-side, just-in-time minting of cloud credentials for **AWS (STS), GCP (WIF), Azure (WIF), GitHub App, GitLab, Bitbucket**, with rate-guards, circuit breakers, tenant policy, and per-use approval gating. The long-lived secret is never handed to a browser or app.
- **For:** So a leaked key on a laptop or in a repo is worthless — there's nothing long-lived to steal, and privileged use can require a phone approval.
- **Who:** Engineering orgs, any customer running cloud infrastructure.

### 5.8 Card Protector (`/card-protector`)
- **What:** Binds a payment card / payment instrument to a TapSign device so a transaction can require hardware approval.
- **For:** Stopping card-not-present fraud and unauthorized transactions (the "protected by TapSign" claim on payments, gated on a real bind).
- **Who:** Fintech/payment integrations.

### 5.9 Vault & Workstation (`/vault`, `/workstation`, `/workstation_guardian`, desktop-vault)
- **What:** Pair a desktop/workstation to a phone; unlocking secrets or doing privileged workstation actions requires phone approval.
- **For:** Protecting developer machines, admin workstations, and local secret stores.
- **Who:** Internal ops, high-security customers.

### 5.10 Federation & SSO (`/oidc`, `/saml`, `/idp`, `/federation`)
- **What:** Standards-based SSO surfaces (OIDC, SAML, IdP, federation) so TapSign can slot into existing enterprise identity ecosystems.
- **For:** Letting big organizations adopt TapSign approval without ripping out their identity stack.
- **Who:** Enterprise/government IT.

### 5.11 Recovery (`/recovery`, `/recover`, device recovery verify)
- **What:** How a user who lost/broke their phone regains trust **without** creating a backdoor. Uses a redesigned flow: multi-factor (phone + password + PIN), **witness** approvals (people you pre-designated), owner cooling-off notifications, windowed/in-person options, and — critically — **revocation of the lost device** across all services when a new one is enrolled.
- **For:** Making "I lost my phone" survivable without making "I stole your phone" survivable.
- **Who:** Every user eventually; heavily hardened because recovery is the classic backdoor.

### 5.12 OTP SDK (`/otp`)
- **What:** A bound, signed OTP service (issuance + verification) for teams that still need OTP but want it tamper-evident and tied to a request.
- **For:** A safer OTP than SMS-only, and a migration bridge toward full hardware approval.
- **Who:** Customers mid-migration or with OTP-based flows.

### 5.13 AI Proxy (`/ai-proxy`)
- **What:** A gated proxy for AI/LLM calls.
- **For:** Letting an org use AI providers without exposing provider keys client-side, with policy/rate control.
- **Who:** Customers building AI features who want the same "no client-side secret" discipline.

### 5.14 Admin & Super-Admin control plane (`/admin`)
- **What:** The largest route group — client management, health/readiness, alerts, pricing catalog, Stripe config, recovery review queue, staff management, runtime config.
- **For:** Running the platform: onboarding clients, watching health, handling recovery reviews, editing pricing, managing internal staff.
- **Who:** Us (super-admins), gated behind QR + phone approval login.

### 5.15 Billing, Pricing, Workspace (`/billing`, `/pricing`, `/workspace`, `/usage`)
- **What:** Metering, plan catalog, quotes, invoices, Stripe checkout, per-workspace provisioning (including free-tier auto-provisioning).
- **For:** Commercial operations — turning usage into revenue.
- **Who:** Enterprise customers + our finance/ops.

### 5.16 Audit, Webhooks, Push, Security, Preflight (`/audit`, `/webhooks`, `/push`, `/security`, `preflight`)
- **What:** Tamper-evident audit log (with an append-only anchor / tamper-evidence layer), outbound webhooks, the push-notification broker (APNs / FCM / HMS), security posture endpoints, and a preflight that checks a device/environment before trusting it.
- **For:** Evidence, integration, delivery of approval prompts, and safety checks.
- **Who:** Everyone; the audit log is what lets a bank or regulator prove what happened.

### 5.17 CLI (`/cli`)
- **What:** Command-line surface for developer/operator flows.
- **For:** Scripting and automation without the web UI.
- **Who:** Developers/operators.

---

## 6. How the pieces fit together (mental model)

```
        ┌─────────────────────────────────────────────────────────────┐
        │                     RELYING PARTIES                          │
        │  Bank app · Gov portal · Council app · Our own console       │
        │  (embed the SDK, or call the API, or trust a signed email)   │
        └───────────────┬───────────────────────────┬─────────────────┘
                        │  "protect this action"     │  "verify this email/badge"
                        ▼                             ▼
        ┌─────────────────────────────────────────────────────────────┐
        │                 TAPPROOF BACKEND (trust brain)               │
        │  onboarding · identity · auth/challenge · guardian · mail    │
        │  browser · provider-proxy · vault · recovery · billing ...   │
        │  ── verifies EVERY signature, binds it to the action,        │
        │     consumes it once, writes a tamper-evident audit trail ── │
        └───────────────┬─────────────────────┬───────────────────────┘
                        │ push challenge        │ device registry / attestation
                        ▼                       ▼
        ┌───────────────────────────┐   ┌───────────────────────────┐
        │   PHONE (the SIGNER)       │   │   BROWSER EXT (the SHIELD)│
        │  Secure Enclave / Keystore │   │  renders Verified badge,  │
        │  Face ID / fingerprint     │   │  checks signed mail/pages │
        │  signs the exact challenge │   └───────────────────────────┘
        └───────────────────────────┘
```

Key discipline that shows up everywhere:
- **Secrets stay server-side.** The browser/app only ever holds *public* references (`client_key_id`, `public_device_ref`, `session_id`, receipt refs). Private keys and connector secrets never reach client code.
- **Money/critical actions need request-bound start + one-time consume**, not a generic "approved" header. This is why a replayed or phished approval fails.
- **PII stays at the institution.** We hold opaque references + public keys, not the customer's identity data.

---

## 7. How the mail system works (end to end) — the detailed walkthrough

This is the part the team specifically asked to understand. TapSign Mail is our **anti-phishing email** product. There are two custody models and one verification experience.

### 7.1 The goal
When a bank or ministry sends you an email, you should be able to see a **green "Verified organization"** badge that means: *this really came from that domain, it wasn't tampered with, and the signing was authorized.* And when it's fake, you should see a clear **red** state.

### 7.2 Domains and keys
- An organization registers a **domain** (e.g. `bank.com.gh`) and proves ownership by placing a **DNS TXT record** — this is the "DNS-verified" step. Our own `tapsign.net` is registered and DNS-verified as a hosted domain.
- Each domain has a **signing key**. There are two custody models:
  - **Hosted** (small businesses, our platform mail): the signing key is a **stable server key** we hold on the organization's behalf, and the owner's **phone blesses it** (the Secure Enclave signs the key's identity), so it's still bound to the owner's device.
  - **Self-custody** (banks, government): the organization holds **its own** signing key; **we never hold their private key.** Their phone blesses their own key.
- Keys are **rotation-proof**: when a device signing key rotates, the old key is appended to the domain's key history so previously-signed mail still verifies (this fixed the "device key may have changed" amber state on old messages).

### 7.3 Signing an outbound email
When the platform (or an org) sends a signed email:
1. The message is signed (`v=4` domain signature) by the domain's signing key.
2. A **proof block** is embedded/attached and a human-readable **verified badge** is rendered (the same badge design across extension, `/verify` page, and phone app — including an org **logo**; note: PNG/JPG for native apps, SVG only works on web/extension, so logos are re-encoded to PNG).
3. For every device that **watches** the recipient address, an inbox row + push are created so the phone's **"Verify Inbox"** shows it. (This dispatch path is `_dispatch_mail_verify_notifications`.)

### 7.4 Verifying an inbound email (the states)
A verifier (extension popup, `/verify`, or the app) resolves an email to exactly one state:

| State | Colour | Meaning |
|---|---|---|
| **Verified organization** | green | `v=4` domain-signed by a DNS-verified domain — the gold standard |
| **Sender verified** | green | signature valid, domain registered but not yet flagged "verified org" |
| **Signed, content changed** | amber | signature valid but the body was edited after signing (a reply/forward edited it) |
| **Reply / forward** | amber | quotes a signed message, but the reply itself isn't signed |
| **Couldn't verify yet — device key may have changed** | amber | proof signed by a key not in the domain's *current* set (old mail + rotated key) — solved by rotation-proof key history |
| **Not verified — sender key not registered** | red | domain has no key on record |
| **Not verified — domain not verified** | red | signed from a domain whose DNS-TXT ownership check isn't done |
| **Not verified — couldn't confirm signature** | red | signature doesn't match any of the domain's keys (throwaway/wrong key) |

The whole point: **green is un-fakeable** (you'd need the domain's key *and* — for hardware-blessed domains — the owner's phone), and every other state is a clear, explainable warning.

### 7.5 The "hardware blessing" selling point
The strongest tier: a per-domain flag `require_hardware_authorized`. When on, a `v=4` proof from that domain is treated as **NOT verified unless the signing key was blessed by the owner's phone Secure Enclave.** So even if an org is careless and its signing key/connector leaks, **nobody can sign mail in their name** — because our verifier rejects any proof whose key wasn't hardware-authorized by the owner's device. The blessing flow: backend stores a pending blessing → pushes the owner's phone → Face ID → enclave signs the signing-key identity → `/mail/domains/signing/authorize` sets `hardware_authorized=true` → badge flips.

### 7.6 Operating the mail system
- **Super-Admin** can add / DNS-verify / delete domains and set the sender name + address.
- **Message templates** (OTP/verification, welcome/org-registered, receipt) are editable without code, with the signed proof block kept non-editable so a copy change can never break the cryptography.
- **Gmail inbox logo (BIMI)** is a *separate* thing — it needs DMARC at enforcement + a `default._bimi` TXT + a paid VMC cert; it's an email-industry standard, not TapSign, and we just check/instruct.
- Every mutating mail operation (add/verify/delete domain, sender/template edits) is **phone-approval gated** so a left-open laptop can't change mail trust.

### 7.7 Why this beats normal email security
SPF/DKIM/DMARC prove "the server was allowed to send for this domain" — they do **not** prove a human/org authorized *this* message, they don't show the end-user a trustworthy badge, and a compromised sending server passes them fine. TapSign adds **content integrity + a visible, hardware-anchored, org-level verified badge that the recipient actually sees**, and (top tier) makes the signing key un-abusable without the owner's phone.

---

## 8. Push notifications (how approvals reach the phone)

- The **notification broker** routes challenges to devices via **APNs** (iOS), **FCM** (Android/Google), and **HMS** (Huawei).
- Known/fixed gotchas we should keep in mind: APNs silent (`content-available`) push vs. banner behavior, and FCM `notification_priority` handling. Canonical rules live in `tapproof-backend/docs/PUSH_NOTIFICATIONS.md`.
- Delivery telemetry (`push_delivery_*`) lets the web UI tell a user "push sent" vs. "open the app manually" when a token is missing.

---

## 9. Recovery, witnesses, and why it's hard (the honest section)

Recovery is where most security systems quietly install a backdoor. Ours is deliberately strict:

- **Bind on enroll** ("keystone"): the identity is bound to hardware at enrollment, so recovery re-binds to *new* hardware rather than trusting a password alone.
- **Multi-factor recovery:** phone + password + PIN.
- **Witnesses:** you pre-designate people; recovery finds witnesses **by identity, not by device**, notifies only the **chosen** ones, and supports **in-person or remote** approval (a per-guardian mode + "require in-person" toggle, with a Scan-QR card on the phone). Witnesses **survive** a recovery (they aren't wiped by it) and remain **anonymous** to each other.
- **Owner cooling-off:** the owner gets a cooling notification/alert even on a no-fingerprint disable path.
- **Lost device is revoked:** on completing a re-bind (or a no-rebind disable), the lost device is revoked across **all** services, with an optional cascade-move.
- **No permanent lockout offline:** design rule for TapSign Mail — a deleted/unreachable backend must never permanently lock a user out; there's a local Force-Reset (erase local passkey/password) escape so an offline user is never stranded.
- **Stuck-recovery is impossible by design:** re-initiating a recovery **supersedes** the active case, so you can't get permanently stranded in "an active case already exists."

Open items the team should know are still rough (tracked in memory): a couple of device-specific PIN dead-ends, remote-push A/B behavior on some handsets, a DB connection-slot pressure item, and a customer-app recovery-push auto-pop that currently only surfaces after a tab switch. None of these weaken the crypto; they're UX/plumbing.

---

## 10. Security posture we can state plainly (internally)

- **Hardware-bound keys**, attested to genuine secure hardware (Apple Secure Enclave, Android Keystore/StrongBox). Android root-of-trust is keyed by SPKI (Google can re-issue the RSA root), not a brittle fingerprint — which is what fixed enrollment on newer flagship devices.
- **Action-bound, one-time approvals** (anti-replay nonces; request-context hashing) — a phished or replayed approval fails.
- **Server-side-only secrets**; clients hold public refs only.
- **Database role separation + row guards** so a compromised app role can't rewrite identity bindings; consume-time signature re-checks close INSERT loopholes.
- **Tamper-evident audit log** (append-only anchor) — evidence a regulator can trust.
- **PII minimization** — institution holds identity; we hold references + public keys.
- **Phone-approval gating** on sensitive mutations across enterprise and mail.

> Reminder for the team: we had a **supply-chain incident** with a trojanized global npm CLI (documented in memory). The lesson that belongs in *this* report: our own trust model (hardware approval, server-side secrets, no long-lived client keys) is exactly the discipline that limits blast radius when a dev machine is compromised. Keep rotating secrets and keep secrets out of client code.

---

## 11. Deployment shape (so the team knows where things run)

- **TapProof backend:** runs as a local/uvicorn service in dev (e.g. `:8000`), containerized for deploy (`deploy/docker/backend.Dockerfile`, `docker-compose.backend.yml`).
- **Website (marketing/enterprise/super-admin):** Next.js, its own repo/host, talks to backend only through same-origin server proxies (no browser-held backend secret).
- **Phone apps:** iOS (`com.tapsign.nativeapp`) and Android native, built per-device; RN SDK for embedding.
- **Extension:** loaded into the browser; bundles the real logo and the verify logic.
- **Provider proxies + guardian/otp SDKs:** server-side Node/Python components customers run or call.

Exact hosts, ports, tunnels, and connection strings are in the runtime config / memory — **not** in this report by policy.

---

## 12. Glossary (so non-deep-technical teammates can follow any conversation)

- **Attestation** — the phone proving, with a manufacturer-signed certificate, that its key really lives in genuine secure hardware.
- **Secure Enclave / Keystore / StrongBox** — the tamper-resistant chip on the phone that holds the private key and never reveals it.
- **Challenge** — the exact thing the phone is asked to sign ("approve X at time T from context C").
- **Request-bound / action-bound** — the approval only works for that one action, not any other.
- **One-time consume / nonce** — the approval can be used exactly once; replays are rejected.
- **Blessing (hardware-authorized)** — the owner's phone vouching for a signing key so it can't be abused even if it leaks.
- **Custodian / witness / guardian** — people whose phone approval is required for sensitive actions or recovery.
- **Relying party** — the bank/gov/app that trusts TapSign's answer.
- **Connector** — the server-side credential an org uses to talk to us; never exposed to the browser.
- **Opaque reference** — a meaningless-to-others id that stands in for a real identity so we don't hold PII.

---

## 13. What to hand to whom

- **This file** → the internal team only.
- **`02_TAPSIGN_GOVERNMENT_INVESTOR_REPORT.md`** → Ghanaian government / investors. Describes problems solved + services offered, **hides how it's built.**
- **`03_TAPSIGN_COVER_LETTER.md`** → the Ghanaian consultant firm, as the cover for the pitch pack.
- **`04_COUNCIL_DEVELOPER_REPORT.md`** → internal team, to understand the Council platform.
- **`05_COUNCIL_PITCH_REPORT.md`** → the council/consultant audience — what it solves, no IP.

*End of internal developer report.*
