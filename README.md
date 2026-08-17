# Wiom — Project Dominance ₹750 Opt-in (CSP)

Hindi/English opt-in screen for the new payout structure: **₹750 at install + ₹300 per renewal ticket**.

Full working context (decisions, open items, campaign settings): **[CONTEXT.md](CONTEXT.md)**

## Files

| File | What |
|---|---|
| `optin.html` | Hosted build (this is what the link serves) |
| `inapp.html` | CleverTap inline custom-HTML build |
| `docs/api-contract.html` | Backend consent API contract |
| `docs/original-untracked.html` | Design file before analytics/API were added |
| `other/renewal750-quiz-inapp.html` | Separate creative: ₹750 renewal education + quiz in-app |

**Live URL**

```
https://vikaswiom.github.io/wiom-dominance-750-optin/optin.html?cspId=<CSP_ID>
```

## URL / runtime contract

| Input | How | Notes |
|---|---|---|
| CSP id | `?cspId=` (also accepts `?csp_id=`) or `window.CSP_ID` | required for the consent POST |
| JWT | `?token=` (also `?jwt=`) or `window.CSP_JWT` | sent as `Authorization: Bearer <jwt>` |

Without a CSP id the page still works end-to-end; the POST is skipped and reported as
`api_status = skipped`.

## Backend

`POST https://csp-api.i2e1.in/dominance/consent` with `{"csp_id": "...", "consent_choice": "OPTED_IN"}`
— fired **only** when the CSP confirms the new structure. "अभी नहीं" is a UI-only decline and
makes no API call. Retries 3x on network/5xx, never on 4xx. QA host:
`https://csp-gateway-service-qa.i2e1agents.in` (one constant at the top of the script).

The gateway must allow the `https://vikaswiom.github.io` origin for the fetch to succeed.

## CleverTap events (5 names, max 5 fires per user)

| Event | When | Key properties |
|---|---|---|
| `Payout750_Viewed` | page load | — |
| `Payout750_Progress` | <=2x | `milestone`: `content_opened`, `reached_choice` |
| `Payout750_Confirmed` | opt-in confirmed (conversion) | `choice='new'`, `lang`, `lang_toggles`, `selection_changes`, `seconds` |
| `Payout750_Declined` | "अभी नहीं" confirmed | `choice='later'`, same properties |
| `Payout750_Closed` | terminal | `exit`, `choice`, `last_selected`, `api_status`, `api_error`, `api_http` |

`Confirmed` and `Declined` are mutually exclusive - exactly one can fire.
Language toggles and radio selections are counted into properties rather than fired as events.
