# Frontend System Design: Feature Visibility & Entitlement

**Duration:** 50–60 minutes
**Format:** Whiteboard / shared doc. Pseudo-code and TypeScript interfaces are welcome. A full implementation is not expected.

---

## 1. Context

You are working on the frontend of **Acme Cloud**, a multi-tenant SaaS product. A customer (an _account_) owns many **locations**, such as clinics, branches or stores. Users pick one or more locations in a **location switcher** at the top of the app. That selection is called the **scope**, and every page in the app respects it.

Acme sells **features** such as _Insights_, _Inbox Pro_ and _Call Analytics_. A feature is purchased **per location**, not per account.

Today every product team hand-rolls its own checks for "should I show this feature?". The result is inconsistent upgrade prompts, flickering pages, stale data after purchase, and a few cases where users were shown a **Start trial** button they could not actually use.

**Your task:** design a **reusable frontend system** that any product team can adopt to decide, _for the current scope_, what the user sees for a feature.

Open [`index.html`](./index.html) in a browser. It is a throwaway mock of the behaviour we want. Spend 3–5 minutes with it before we begin, and use the **Scenario simulator** on the left to flip the world around.

> The mock is a reference for behaviour only. It is deliberately simplistic and is **not** a model of how your system should be built.

---

## 2. Concepts you need to know

| Term             | Meaning                                                                                                     |
| ---------------- | ----------------------------------------------------------------------------------------------------------- |
| **Account**      | The paying customer. It has many locations.                                                                 |
| **Location**     | The unit a feature is bought for.                                                                           |
| **Scope**        | The set of locations the user has currently selected. It can change at any time without a page reload.      |
| **Feature**      | A product capability that can be gated, for example `insights`.                                             |
| **Trial**        | A 30-day free period for one feature on one location. It can be started only once per location.             |
| **Admin**        | A user with the `MANAGE_SUBSCRIPTIONS` permission at a location. Only admins can start trials and purchase. |
| **Feature flag** | An operational switch controlled by our engineers. It is used for rollouts and kill switches.               |

---

## 3. Required behaviour

For a given feature and the user's current scope, the UI must be able to produce the following outcomes. Teams should be able to get them **without re-implementing the logic**.

| #   | Situation (across the locations in scope)                       | Expected UI                                                                                        |
| --- | --------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| 1   | Active in **all** locations                                     | Render the feature normally                                                                        |
| 2   | Active in **some** locations                                    | Render the feature **plus** an alert like "Not active in 2 of 4 locations", with an upgrade action |
| 3   | Active in **none**, and a trial or purchase is possible         | Full-page promotion with the right call to action (**Start free trial** or **Upgrade**)            |
| 4   | In **trial**                                                    | Render the feature **plus** a banner with days left. It should become more urgent near expiry      |
| 5   | Active in none and **not available** at all for these locations | A simple "not available" state                                                                     |
| 6   | The user **cannot** start a trial or purchase                   | They still see the promotion, but the action is disabled and explains why                          |
| 7   | The feature flag is **off**                                     | The feature does not exist for the user (no nav item, no teaser, no promotion)                     |
| 8   | A **dashboard teaser** is eligible                              | A dismissible card. See 3.1                                                                        |

### 3.1 Dashboard teaser rules

- Shown only when: the user is an admin, the feature is not active anywhere in scope, a trial is available, and the flag is on.
- "Maybe later" offers **Remind me in 14 days** and **Never show again**.
- The dismissal must survive a reload.
- Other teams will want to add their own teasers, and several teasers may exist on one page.

### 3.2 Things that happen while the app is open

- The user changes the scope.
- The user starts a trial or upgrades from within the app.
- Another admin upgrades a location from a different tab or device.
- A trial expires while the user is on the page.
- The feature flag is toggled by an engineer.

---

## 4. What you have to work with

You own the **frontend only**. The backend team has said they can make _reasonable_ changes if you can justify them, but assume that for the interview the following exists today.

### 4.1 Existing endpoints

<details>
<summary><b>GET /v1/features/{feature}/eligibility?locationIds=1,2,3</b></summary>

One entry per location. Each entry can fail independently.

```json
{
  "feature": "insights",
  "eligibility": [
    {
      "locationId": "1",
      "subscriptionStatus": "SUBSCRIBED",
      "trialStatus": "TRIAL_NOT_ELIGIBLE"
    },
    {
      "locationId": "2",
      "subscriptionStatus": "NOT_SUBSCRIBED",
      "trialStatus": "TRIAL_ELIGIBLE"
    },
    {
      "locationId": "3",
      "subscriptionStatus": "NOT_SUBSCRIBED",
      "trialStatus": "TRIAL_ACTIVE",
      "trialEndsAt": "2026-12-01T00:00:00Z"
    },
    { "locationId": "4", "error": { "code": "UPSTREAM_TIMEOUT" } },
    {
      "locationId": "5",
      "subscriptionStatus": "NOT_SUBSCRIBED",
      "trialStatus": "TRIAL_EXPIRED"
    },
    {
      "locationId": "6",
      "subscriptionStatus": "NOT_AVAILABLE",
      "trialStatus": "TRIAL_NOT_ELIGIBLE"
    }
  ]
}
```

Notes: this endpoint is relatively slow (300–800 ms) and is computed from billing data.

</details>

<details>
<summary><b>GET /v1/locations/feature-state?feature=insights&locationIds=1,2,3</b></summary>

A fast, cached "feature state" that is also used by the legacy app. It is updated asynchronously after billing changes.

```json
{
  "states": [
    { "locationId": "1", "state": "ACTIVE" },
    { "locationId": "2", "state": "INACTIVE" },
    { "locationId": "3", "state": "ACTIVE" }
  ]
}
```

Notes: it can **lag** the eligibility endpoint by up to a few minutes after a purchase or trial start. `state` may also be `PROMOTE`, which means "show this to the user as an upsell".

</details>

<details>
<summary><b>GET /v1/me/session</b></summary>

Loaded once at login.

```json
{
  "userId": "u_812",
  "accountId": "a_77",
  "permissionsByLocation": {
    "1": ["MANAGE_SUBSCRIPTIONS", "VIEW_REPORTS"],
    "2": ["VIEW_REPORTS"],
    "3": ["MANAGE_SUBSCRIPTIONS", "VIEW_REPORTS"]
  },
  "featureFlags": { "insights-v2": true, "inbox-pro-teaser": false }
}
```

Notes: flags are evaluated at login and do not update until the next session refresh.

</details>

<details>
<summary><b>POST /v1/features/{feature}/trial</b></summary>

```json
// request
{ "locationIds": ["2", "5"] }

// response (partial success is possible)
{
  "results": [
    { "locationId": "2", "ok": true },
    { "locationId": "5", "ok": false, "error": { "code": "NOT_ELIGIBLE" } }
  ]
}
```

</details>

<details>
<summary> <b>POST /v1/features/{feature}/subscribe</b></summary>

```json
// request
{ "locationIds": ["2", "5"] }

// response (partial success is possible)
{
  "results": [
    { "locationId": "2", "ok": true },
    { "locationId": "5", "ok": false, "error": { "code": "NOT_ELIGIBLE" } }
  ]
}
```

</details>

> **Tip:** you do not have to use these as they are. Tell us what you would keep, change or ask the backend team for, and why.

---

## 5. Constraints and non-goals

**Constraints**

- Assume a React-like component framework, but your system should not depend on any specific state library.
- Web only. No SSR.

**Out of scope**

- Billing flows, checkout UI and payment.
- Backend implementation.
- Visual design.

---

## 6. What we will discuss

We will cover roughly the following. Not everything needs to be fully solved.

1. How you model the problem, and what the key concepts are.
2. The public API of your system, as seen by a product team.
3. Where data lives, how it is fetched and how it stays correct.
4. What happens when the world changes while the app is open.
5. How other teams extend the system.
6. How you would test it and roll it out.

Think out loud. We care about your trade-offs, and about what you decide **not** to do, more than about a perfect final diagram.

---

## 7. Files in this folder

| File         | Purpose                                                                   |
| ------------ | ------------------------------------------------------------------------- |
| `README.md`  | This document                                                             |
| `index.html` | Interactive reference for the expected behaviour. Open it in any browser. |
