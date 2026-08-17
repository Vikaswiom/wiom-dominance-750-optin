# Working context — Dominance ₹750 opt-in

Everything needed to pick this up on another machine. Last updated 17 Aug 2026.

---

## 1. What this is

Project Dominance V1 (ticket #1472). CSPs are offered a new payout structure and must choose:

- **नया तरीका** — ₹750 paid at install (after a 3-month ISP recharge + proof upload), then ₹300 per
  1-month renewal ticket from month 4.
- **अभी नहीं** — current structure continues, no change.

The screen explains the structure, then captures the choice. Opt-in is written to the backend;
the decline is recorded in CleverTap only.

## 2. Files

| File | What it is |
|---|---|
| `optin.html` | **Hosted build.** Deployed to GitHub Pages. Identity comes from `?cspId=`. |
| `inapp.html` | **CleverTap inline build.** Paste into a custom-HTML in-app. Identity comes from `window.CSP_ID` / the `CSP_ID_FIXED` constant, since an inline in-app has no URL. |
| `docs/api-contract.html` | Backend API contract from the csp-rv-service team (consent + config endpoints). |
| `docs/original-untracked.html` | The original design file before any analytics or API code was added. Keep for diffing. |
| `other/renewal750-quiz-inapp.html` | Unrelated creative from the same working session — the ₹750 renewal education + 2-question quiz in-app (`Renewal750_*` events). Parked here so it isn't lost. |

`optin.html` and `inapp.html` are the same screen; they differ only in how they get `csp_id`/JWT and
in a couple of viewport rules. Any copy change must be made in **both**.

## 3. Live link

```
https://vikaswiom.github.io/wiom-dominance-750-optin/optin.html?cspId=<CSP_ID>
```

Dev team appends `cspId`. Same URL shape as the Sehat MG settle screen.

To redeploy: edit `optin.html`, commit, push to `main`. GitHub Pages (legacy build, branch `main`,
path `/`) picks it up in ~45 s. Verify with:

```
curl -s https://vikaswiom.github.io/wiom-dominance-750-optin/optin.html | grep -c Payout750_Declined
```

## 4. CleverTap events — 5 names, max 5 fires per user

| Event | When | Properties |
|---|---|---|
| `Payout750_Viewed` | page load / in-app shown | — |
| `Payout750_Progress` | ≤2× | `milestone` = `content_opened` (tapped देखें और चुनें) \| `reached_choice` (choice section 40% visible) |
| `Payout750_Confirmed` | **opt-in** confirmed — CONVERSION | `choice='new'`, `lang`, `lang_toggles`, `selection_changes`, `reached_choice`, `seconds` |
| `Payout750_Declined` | **अभी नहीं** confirmed | same properties, `choice='later'` |
| `Payout750_Closed` | terminal, once | `exit`, `choice`, `last_selected`, `reached_choice`, `lang`, `lang_toggles`, `seconds`, `api_status`, `api_error`, `api_http` |

`Confirmed` and `Declined` are mutually exclusive. `exit` takes `completed` (tapped सारे काम देखें),
`abandoned` (swiped away / backgrounded before deciding), `left_after_confirm`.

**Design rule followed here:** no event per click. Language toggles are counted into
`lang_toggles`, radio taps into `selection_changes` + `last_selected`, and API outcome into
`api_status` on the terminal event. Adding more events should be resisted — the properties already
answer the questions.

Funnel: `Viewed` → `Progress[content_opened]` → `Progress[reached_choice]` → `Confirmed`.

**Suppression:** targeting on "has not done `Payout750_Confirmed`" alone will keep re-showing the
in-app to decliners. Exclude both `Confirmed` and `Declined`.

Bridge details (learned the hard way, see the memory notes):
- Correct call is `window.CleverTap.pushEvent(name, JSON.stringify(props))`. `CleverTap.event.push()`
  is the **web** SDK and silently does nothing in a mobile in-app WebView.
- Events retry 6× with 250 ms×n back-off because the bridge is not always attached at load.
- Fire events **before** dismissing; navigating away kills the WebView and every pending retry.
- The editor's **"Include JavaScript"** checkbox must be ticked or none of this runs.

## 5. Backend call

`POST {base}/dominance/consent` — `{"csp_id": "...", "consent_choice": "OPTED_IN"}`

- Fires **only** on opt-in. "अभी नहीं" makes **no** API call (explicit product decision).
- PROD `https://csp-api.i2e1.in` — currently active. QA `https://csp-gateway-service-qa.i2e1agents.in`
  is a commented constant on the same line in the script (`var API = {...}`).
- Retries 3× with 800 ms×n back-off on network errors and 5xx. Never retries a 4xx.
- `keepalive: true` so the request survives WebView teardown.
- CSP-facing states on the done page: "आपका फ़ैसला सेव हो रहा है…" → clears on success, or
  "फ़ैसला सेव नहीं हो पाया" + a **फिर से कोशिश करें** button that re-posts.
- Outcome is reported on `Payout750_Closed` as `api_status` (`ok` / `error` / `skipped` / `idle`),
  `api_error` (`http_401`, `network`, `no_csp_id`…), `api_http`.
- `api_status = idle` means no call was needed (a decline). `skipped` means a call was needed but
  there was no `csp_id`.

Contract notes that matter: re-posting `OPTED_IN` is idempotent (`already_enrolled: true`), a
`NOT_NOW` CSP can later opt in, and an opted-in CSP **cannot** self-serve revert.

## 6. Open items for the dev team

1. **`csp_id` and JWT.** Hosted build reads `?cspId=` / `?token=` (also `window.CSP_ID` /
   `window.CSP_JWT`). Inline build has no URL — the app must inject the globals, or fill
   `CSP_ID_FIXED` / `CSP_JWT_FIXED` at the top of the script. Dev team owns this.
2. **CORS.** The hosted page calls the gateway from origin `https://vikaswiom.github.io` — must be
   allowed. The inline in-app is worse: it runs on `about:blank`, so the request carries
   `Origin: null` and will almost certainly fail preflight. **This is the main reason to prefer the
   hosted URL as the in-app.**
3. If neither is solvable, fall back to: creative records the choice in CleverTap only, and the app
   makes the authenticated POST itself after reading the event / profile flag.

## 6b. Target cohort

**332 CSP IDs** (supplied 17 Aug 2026, deduped, no malformed entries). The list is NOT in this repo —
this repo is public because GitHub Pages requires it. It lives in the private archive:

```
Vikaswiom/wiom-projects-archive -> dominance-750-optin/
  csp_ids.txt     one id per line, sorted
  csp_links.csv   csp_id,link  (per-CSP opt-in URL)
```

For the CleverTap segment, map CSP ID -> CT identity via `PROD_DB.CLEVERTAP_CSP_API.PROFILE_DATA`.

## 7. Campaign settings

Layout **Cover** (not interstitial — a full-bleed creative gets letterboxed otherwise).
**"Include JavaScript" ON and saved.** Turn CT's default close button off if the creative draws its
own exit. Test on a real device: the bridge does not exist in a desktop browser, so events and
dismissal cannot be verified there.

## 8. House rules for this kind of creative

- **Never** use `//` line comments — the CT editor can collapse the file onto one line and a `//`
  then swallows every statement after it. Block comments only.
- **No `{{ }}` personalisation tokens** inside `<script>` — they have blocked saving in this account.
  Bake values in per-segment creatives instead.
- There is no 4 KB size limit; a 17 KB creative runs fine in production.
